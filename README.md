## Phase 1: Provider Completion & Regression Verification

### 1. Inspection Findings (from the uploaded source)
Reviewing the Phase 1 execution paths across `IptvProvider`, `IptvProviderFactory`, `M3uIptvProvider`, `XtreamIptvProvider`, `XtreamClientFactory`, `XtreamAccountManager`, `XtreamRepository`, `StreamResolver`, `M3uParser`, `OkHttpDataSourceHelper`, `UnifiedCatalogRepository`, `UrlViewModel`, `StreamFailoverEngine`, and `PlaybackRecoveryManager` surfaced the following concrete defects:

1. **Xtream HTTP/HTTPS Redirect Handling Broken (`network/XtreamClientFactory.kt`)**:
   - `XtreamClientFactory.createService` sets `.followRedirects(false)` and `.followSslRedirects(false)` without attaching `NetworkSecurityValidator.RedirectGuardInterceptor`. Any Xtream panel that issues 301/302/307/308 redirects (e.g., HTTP→HTTPS or load-balanced edge nodes) fails authentication and catalog sync.
   - `normalizeUrl` also fails if a user pastes a URL ending in `/player_api.php` or `/get.php?...`, producing `.../player_api.php/player_api.php`.
2. **Missing Scheme in Stored Xtream Server URLs (`data/repository/XtreamRepository.kt`, `security/XtreamAccountManager.kt`)**:
   - `XtreamClientFactory` prepends `http://` when omitted, so authentication succeeds, but `XtreamRepository.importFullAccount` saved `serverUrl` without normalizing the scheme. As a result, `XtreamIptvProvider` and `StreamResolver` constructed schemeless Live/VOD/Series/EPG URLs (`host:port/live/...`) which `NetworkSecurityValidator` rejected at playback time.
3. **Provider Custom Headers Overridden by Default Client Headers (`network/OkHttpDataSourceHelper.kt`)**:
   - `DynamicStreamInterceptor` checked `if (originalRequest.header("User-Agent") == null)` before applying `matchedContext.headers.userAgent`. If Media3/OkHttp pre-populated a default header, channel-specific `User-Agent` / `Referer` / `Cookie` / `Authorization` headers were ignored.
4. **Protocol Sniffing Fails on Unbounded Live Streams (`provider/StreamResolver.kt`)**:
   - `sniffRemoteProtocol` called `BoundedResponseReader.readBytes(it, 65_536L)`, which requires EOF before 64 KB. Live IPTV servers that ignore `Range: bytes=0-4095` stream past 64 KB, causing `MaxBytesExceededException` (caught and turned into `null`). It also lacked a scheme guard against non-HTTP(S) URLs and masked `XtreamAccountManager.CredentialsResolution.NeedsReauthentication` behind a generic message.
5. **M3U Open Failures Masked Underlying Cause (`playlist/M3uParser.kt`)**:
   - `openReader` returned `null` on HTTP error codes (e.g. 401/403/404) and caught network/SSRF exceptions, hiding the actual failure reason behind `"Could not open playlist source"`.
6. **Stuck `SYNCING` State on Sync Cancellation (`data/repository/UnifiedCatalogRepository.kt`, `viewmodel/UrlViewModel.kt`)**:
   - Updating `PlaylistSource.syncState` inside a `CancellationException` path without `NonCancellable` immediately threw `CancellationException` on the DB suspend call, leaving the source stuck in `SyncState.SYNCING`.
7. **Stale Stream Context & Index Desync on Failover (`player/PlaybackRecoveryManager.kt`, `player/StreamFailoverEngine.kt`)**:
   - `PlaybackRecoveryManager.executeCandidateFailover` only unregistered `currentSession.resolvedUri`, leaving rewritten/redirected URLs from the failed endpoint in `OkHttpDataSourceHelper.streamContextRegistry`, and omitted `resolvedUrlOverride`.
   - `StreamFailoverEngine.registerCandidates` set `activeIndices[channelKey] = -1` when `selectOptimalCandidate` returned `null`, causing `getActiveCandidate(channelKey)` to return `null` even when `PlaybackSessionManager` fell back to `safeCandidates.first()`.

---

### 2. Required Fixes

#### `network/XtreamClientFactory.kt`
```kotlin
package vid.mat.app.network

import okhttp3.OkHttpClient
import retrofit2.Retrofit
import retrofit2.converter.gson.GsonConverterFactory
import vid.mat.app.security.NetworkSecurityContext
import vid.mat.app.security.NetworkSecurityValidator
import vid.mat.app.security.XtreamAccountManager
import java.io.IOException
import java.util.concurrent.TimeUnit

object XtreamClientFactory {

    @Throws(IOException::class)
    fun createService(baseUrl: String, okHttpClient: OkHttpClient): XtreamService {
        val formattedUrl = normalizeUrl(baseUrl)

        val validation = NetworkSecurityValidator.validateUri(formattedUrl, NetworkSecurityContext.XTREAM_API)
        if (validation !is NetworkSecurityValidator.UrlValidationResult.Valid) {
            val reason = (validation as? NetworkSecurityValidator.UrlValidationResult.Invalid)?.reason
                ?: "Xtream server URL failed security validation"
            throw IOException("SSRF / Network Policy Violation: $reason")
        }

        val client = okHttpClient.newBuilder()
            .dns(NetworkSecurityValidator.SafeDns(NetworkSecurityContext.XTREAM_API))
            .addInterceptor(NetworkSecurityValidator.RedirectGuardInterceptor { NetworkSecurityContext.XTREAM_API })
            .connectTimeout(20, TimeUnit.SECONDS)
            .readTimeout(30, TimeUnit.SECONDS)
            .followRedirects(false)
            .followSslRedirects(false)
            .build()

        return Retrofit.Builder()
            .baseUrl(validation.sanitizedUrl)
            .client(client)
            .addConverterFactory(GsonConverterFactory.create())
            .build()
            .create(XtreamService::class.java)
    }

    private fun normalizeUrl(url: String): String {
        val normalized = XtreamAccountManager.normalizeServerUrl(url)
        return if (normalized.endsWith("/")) normalized else "$normalized/"
    }
}
```

#### `security/XtreamAccountManager.kt`
Add `normalizeServerUrl` and apply it on both save and credential resolution (preserving existing stored rows):
```kotlin
    companion object {
        private const val TAG = "XtreamAccountManager"

        fun normalizeServerUrl(rawUrl: String): String {
            var clean = rawUrl.trim().substringBefore('#').substringBefore('?')
            if (!clean.startsWith("http://", ignoreCase = true) &&
                !clean.startsWith("https://", ignoreCase = true)
            ) {
                clean = "http://$clean"
            }
            clean = clean.trimEnd('/')
            if (clean.endsWith("/player_api.php", ignoreCase = true)) {
                clean = clean.substring(0, clean.length - "/player_api.php".length)
            } else if (clean.endsWith("/get.php", ignoreCase = true)) {
                clean = clean.substring(0, clean.length - "/get.php".length)
            }
            return clean.trimEnd('/')
        }
    }
```
And inside `saveOrUpdateAccount`:
```kotlin
        val cleanServer = normalizeServerUrl(serverUrl)
```
And inside `resolveCredentials`:
```kotlin
            is CryptoUtils.DecryptionResult.Success -> {
                CredentialsResolution.Success(normalizeServerUrl(account.serverUrl), account.username, decryption.plaintext)
            }
```

#### `data/repository/XtreamRepository.kt`
Update `cleanServer` in `importFullAccount` to use the canonical normalizer:
```kotlin
    val cleanServer = XtreamAccountManager.normalizeServerUrl(serverUrl)
```

#### `network/OkHttpDataSourceHelper.kt`
In `DynamicStreamInterceptor.intercept`, ensure explicit channel/provider headers take precedence over any default request headers:
```kotlin
                val explicitUa = NetworkSecurityValidator.sanitizeHeaderValue(matchedContext.headers.userAgent?.takeIf { it.isNotBlank() })
                if (explicitUa != null) {
                    requestBuilder.header("User-Agent", explicitUa)
                } else if (originalRequest.header("User-Agent") == null) {
                    requestBuilder.header("User-Agent", DEFAULT_USER_AGENT)
                }

                NetworkSecurityValidator.sanitizeHeaderValue(matchedContext.headers.referer)?.takeIf { it.isNotBlank() }?.let {
                    requestBuilder.header("Referer", it)
                }
                NetworkSecurityValidator.sanitizeHeaderValue(matchedContext.headers.cookie)?.takeIf { it.isNotBlank() }?.let {
                    requestBuilder.header("Cookie", it)
                }
                NetworkSecurityValidator.sanitizeHeaderValue(matchedContext.headers.authorization)?.takeIf { it.isNotBlank() }?.let {
                    requestBuilder.header("Authorization", it)
                }
```

#### `provider/StreamResolver.kt`
1. In live channel resolution, track `lastResolutionError`:
```kotlin
            val providerName = source?.name ?: "Default"
            var lastResolutionError: String? = null
            val candidates = buildList {
                for (ep in endpoints) {
                    ...
                    val resolutionResult = if (needsXtreamResolution && source != null) {
                        val normalizedChannel = mapToNormalizedChannel(channel)
                        val normalizedEndpoint = mapToNormalizedEndpoint(ep)
                        providerFactory.createProvider(source)
                            .resolveStream(normalizedChannel, normalizedEndpoint)
                    } else null
                    val resolved = resolutionResult?.getOrNull()
                    if (resolutionResult?.isFailure == true) {
                        lastResolutionError = resolutionResult.exceptionOrNull()?.message
                    }

                    val fallbackCreds = if (resolved == null && ep.url.contains("/X-AUTH-INJECT/") && source?.xtreamAccountId != null) {
                        when (val credsRes = xtreamAccountManager.resolveCredentials(source.xtreamAccountId)) {
                            is XtreamAccountManager.CredentialsResolution.Success -> credsRes
                            XtreamAccountManager.CredentialsResolution.NeedsReauthentication -> {
                                lastResolutionError = "Xtream credentials require re-authentication"
                                null
                            }
                            is XtreamAccountManager.CredentialsResolution.Error -> {
                                lastResolutionError = credsRes.message
                                null
                            }
                        }
                    } else null
```
And when `candidates.isEmpty()`:
```kotlin
            if (candidates.isEmpty()) {
                return Result.failure(
                    IllegalStateException(
                        lastResolutionError ?: "No playable or authorized endpoints available for ${channel.title}"
                    )
                )
            }
```
2. Replace `sniffRemoteProtocol` so it reads at most 4096 bytes directly (safe on infinite live streams that ignore `Range`) and skips non-HTTP(S) schemes:
```kotlin
    private suspend fun sniffRemoteProtocol(url: String, headers: OkHttpStreamingClient.StreamHeaders): String? =
        withContext(Dispatchers.IO) {
            runCatching {
                val resolvedUrl = if (url.contains("/X-AUTH-INJECT/") &&
                    !headers.xtreamUser.isNullOrBlank() && !headers.xtreamPass.isNullOrBlank()
                ) {
                    url.replace(
                        "/X-AUTH-INJECT/",
                        "/${android.net.Uri.encode(headers.xtreamUser)}/${android.net.Uri.encode(headers.xtreamPass)}/"
                    )
                } else url
                if (!resolvedUrl.startsWith("http://", ignoreCase = true) &&
                    !resolvedUrl.startsWith("https://", ignoreCase = true)
                ) {
                    return@withContext null
                }
                val context = NetworkSecurityValidator.resolveStreamContext(resolvedUrl)
                val requestBuilder = Request.Builder().url(resolvedUrl).header("Range", "bytes=0-4095")
                headers.userAgent?.takeIf { it.isNotBlank() }?.let { requestBuilder.header("User-Agent", it) }
                headers.referer?.takeIf { it.isNotBlank() }?.let { requestBuilder.header("Referer", it) }
                headers.cookie?.takeIf { it.isNotBlank() }?.let { requestBuilder.header("Cookie", it) }
                headers.authorization?.takeIf { it.isNotBlank() }?.let { requestBuilder.header("Authorization", it) }
                NetworkSecurityValidator.executeSafeHttpCall(okHttpClient, requestBuilder.build(), context).use { response ->
                    if (!response.isSuccessful) return@withContext null
                    val stream = response.body?.byteStream() ?: return@withContext null
                    val buffer = ByteArray(4096)
                    var total = 0
                    while (total < buffer.size) {
                        val read = stream.read(buffer, total, buffer.size - total)
                        if (read <= 0) break
                        total += read
                    }
                    if (total == 0) return@withContext null
                    val sample = if (total == buffer.size) buffer else buffer.copyOf(total)
                    when (ContentTypeSniffer.detectFromUrl(resolvedUrl, sample)) {
                        ContentTypeSniffer.DetectedContentType.M3U8 -> "m3u8"
                        ContentTypeSniffer.DetectedContentType.MPD -> "mpd"
                        ContentTypeSniffer.DetectedContentType.MPEG_TS -> "ts"
                        ContentTypeSniffer.DetectedContentType.MP4 -> "mp4"
                        else -> null
                    }
                }
            }.getOrNull()
        }
```

#### `playlist/M3uParser.kt`
Propagate HTTP status codes and `IOException` reasons from `openReader` instead of returning `null`:
```kotlin
    private fun openReader(
        context: Context,
        source: Any,
        httpClient: OkHttpClient
    ): Pair<BufferedReader, okhttp3.Response?> {
        return when (source) {
            is Uri -> {
                val stream = context.contentResolver.openInputStream(source)
                    ?: throw java.io.IOException("Unable to open local playlist URI")
                Pair(createCharsetSafeReader(stream), null)
            }
            is String -> {
                if (source.startsWith("http://", ignoreCase = true) || source.startsWith("https://", ignoreCase = true)) {
                    val response = RemoteResourceClient.fetch(
                        url = source,
                        context = NetworkSecurityContext.REMOTE_PLAYLIST,
                        httpClient = httpClient
                    ) { builder ->
                        builder.header("User-Agent", DEFAULT_USER_AGENT)
                        builder.header("Accept", "*/*")
                    }
                    if (!response.isSuccessful) {
                        val code = response.code
                        val msg = response.message
                        response.close()
                        throw java.io.IOException("Playlist HTTP error $code: $msg")
                    }
                    val body = response.body ?: run {
                        response.close()
                        throw java.io.IOException("Playlist response body was empty")
                    }
                    Pair(createCharsetSafeReader(body.byteStream()), response)
                } else {
                    val file = java.io.File(source)
                    if (!file.exists() || !file.canRead()) {
                        throw java.io.IOException("Local playlist file does not exist or is unreadable")
                    }
                    Pair(createCharsetSafeReader(file.inputStream()), null)
                }
            }
            else -> throw IllegalArgumentException("Unsupported playlist source type: ${source::class.simpleName}")
        }
    }
```
And in `parseExtended`:
```kotlin
        val (reader, response) = openReader(context, source, httpClient)
        httpResponse = response
```

#### `data/repository/UnifiedCatalogRepository.kt` & `viewmodel/UrlViewModel.kt`
Ensure cancellation updates `PlaylistSource.syncState` under `NonCancellable`:
In `UnifiedCatalogRepository.kt` (import `kotlinx.coroutines.NonCancellable`):
```kotlin
    } catch (e: kotlinx.coroutines.CancellationException) {
        withContext(NonCancellable) {
            runCatching { sourceDao.updateSyncState(sourceId, SyncState.FAILED, "Sync cancelled") }
        }
        throw e
    }
```
In `UrlViewModel.kt` (import `kotlinx.coroutines.NonCancellable`):
```kotlin
        } catch (e: Exception) {
            withContext(NonCancellable) {
                runCatching { sourceDao.updateSyncState(sourceId, SyncState.FAILED, e.message ?: "Cancelled") }
            }
            if (e is CancellationException) {
```

#### `player/PlaybackRecoveryManager.kt` & `player/StreamFailoverEngine.kt`
In `PlaybackRecoveryManager.executeCandidateFailover`:
```kotlin
                    val streamKey = candidate.uri.toString()
                    OkHttpDataSourceHelper.clearStreamContexts()
                    OkHttpDataSourceHelper.registerStreamContext(
                        streamKey,
                        OkHttpDataSourceHelper.StreamContext(
                            headers = candidate.headers,
                            resolvedUrlOverride = streamKey
                        )
                    )
```
In `StreamFailoverEngine.registerCandidates`:
```kotlin
    val optimal = smartSelector.selectOptimalCandidate(candidates)
    val idx = if (optimal != null) candidates.indexOf(optimal).coerceAtLeast(0) else 0
    activeIndices[channelKey] = idx
```

---

### 3. Phase 1 Report

- **Completed functionality (verified in code)**:
  - M3U remote/local/single-stream authentication (`M3uIptvProvider.authenticate`) and Xtream `player_api.php` authentication + account-status validation (`XtreamIptvProvider.authenticateWithInfo`).
  - Xtream redirect handling and base-URL normalization (`XtreamClientFactory`, `XtreamAccountManager`, `XtreamRepository`).
  - Xtream VOD (`get_vod_info`) and Series (`get_series_info`) detail lookups (`XtreamIptvProvider`), with graceful degradation in `VodRepository` / `SeriesRepository` for M3U providers.
  - Custom HTTP header propagation (`User-Agent`, `Referer`, `Cookie`, `Authorization`) and `/X-AUTH-INJECT/` credential injection across `StreamResolver` and `OkHttpDataSourceHelper`.
  - Protocol sniffing on live/unbounded streams without `MaxBytesExceededException` false failures.
  - Provider source switching (`UnifiedCatalogViewModel.selectSource`), cancellation state cleanup (`UnifiedCatalogRepository`, `UrlViewModel`), and clean stream context replacement during failover (`PlaybackRecoveryManager`, `StreamFailoverEngine`).

- **Remaining defects**:
  - `PlaylistRepository.syncPlaylist` (used if legacy callers sync M3U directly via `PlaylistRepository` rather than `UnifiedCatalogRepository`/`UrlViewModel`) still invokes `M3uParser.parse` instead of `M3uParser.parseExtended`, omitting `channelNumber`, `tvgName`, and catch-up attributes; Phase 2 will consolidate/align playlist synchronization.
  - `M3uIptvProvider.authenticate` only validates URI/file accessibility; remote M3U HTTP errors are surfaced during `fetchCatalog` (now with full HTTP status/exception messages from `M3uParser.openReader`).

- **Modified classes**:
  - `network/XtreamClientFactory.kt`
  - `security/XtreamAccountManager.kt`
  - `data/repository/XtreamRepository.kt`
  - `network/OkHttpDataSourceHelper.kt`
  - `provider/StreamResolver.kt`
  - `playlist/M3uParser.kt`
  - `data/repository/UnifiedCatalogRepository.kt`
  - `viewmodel/UrlViewModel.kt`
  - `player/PlaybackRecoveryManager.kt`
  - `player/StreamFailoverEngine.kt`

- **Integration points**:
  - `XtreamIptvProvider` ↔ `XtreamClientFactory` ↔ `NetworkSecurityValidator.RedirectGuardInterceptor`
  - `XtreamRepository` / `XtreamIptvProvider` ↔ `XtreamAccountManager` ↔ `CryptoUtils`
  - `StreamResolver` ↔ `IptvProviderFactory` ↔ `PlaybackSessionManager` ↔ `OkHttpDataSourceHelper` ↔ `PlaybackRecoveryManager` ↔ `StreamFailoverEngine`
  - `UnifiedCatalogRepository` / `UrlViewModel` ↔ `M3uParser` ↔ `PlaylistSourceDao`

- **Verification results**:
  - Static execution-path verification completed against all Phase 1 classes in the provided snapshot; all authentication, URL normalization, credential resolution, header injection, protocol sniffing, and failover context paths trace cleanly without type or contract mismatches.

- **Limitations preventing full verification**:
  - No Gradle compiler/emulator or live M3U/Xtream provider endpoints are accessible in this environment, so runtime compilation, HTTP redirect integration, and live playback against real IPTV servers remain unexecuted here.

Ready to proceed to **Phase 2: Playlist and catalog integrity** when you say **start phase 2**.

