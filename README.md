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




## Phase 2: Playlist and Catalog Integrity

### 1. Inspection Findings (from the uploaded source)
Reviewing the Phase 2 synchronization, channel identity, duplicate cleanup, override preservation, and partial-import protection paths across `PlaylistRepository`, `UnifiedCatalogRepository`, `PlaylistSyncEngine`, `ResilientSyncPolicy`, `PlaylistFingerprint`, `DuplicateCleanupRepository`, `DuplicateDetectionEngine`, `ChannelDao`, `ChannelOrganizationRepository`, and `AppLifecycleSupervisor` surfaced the following concrete defects:

1. **Legacy `PlaylistRepository.syncPlaylist` Uses Incompatible Identity & Bypasses Partial/Shrinkage Guards (`data/repository/PlaylistRepository.kt`)**:
   - `syncPlaylist` still called `M3uParser.parse(...)` instead of `M3uParser.parseExtended(...)`, omitting `channelNumber`, `tvgName`, `catchupDays`, `catchupSource`, `CategoryNormalizer`, and `ChannelNameNormalizer`.
   - Computing `PlaylistFingerprint.computeIdentity` with raw `event.channel.title` instead of `ChannelNameNormalizer.normalize(vu.title)` caused `channelIdentity` mismatches against `UrlViewModel` and `M3uIptvProvider`, breaking user overrides (`ChannelUserOverrideEntity`) and favorites (`live_identity:...`) if a playlist was synced via `PlaylistRepository`.
   - `syncPlaylist` ignored `ParseEvent.Done(total, truncated)` (allowing truncated playlists to destructively delete remaining channels), ignored `ResilientSyncPolicy.EvaluationResult.RequiresUserApproval` (allowing 0-channel or >50%-shrinkage feeds to wipe the catalog), and did not use `NonCancellable` on `CancellationException` when resetting `SyncState.SYNCING`.
2. **VOD / Series Catalogs Unprotected Against Empty/Partial Provider Responses (`data/repository/UnifiedCatalogRepository.kt`)**:
   - `UnifiedCatalogRepository.syncProviderCatalog` evaluated `ResilientSyncPolicy.evaluateBatchSafety` only on Live channels. If an Xtream server returned an empty or truncated `get_vod_streams` or `get_series` response (HTTP 200 `[]`) while Live channels succeeded, `reconcileMoviesNonDestructively` / `reconcileSeriesNonDestructively` would delete existing VOD movies or series and their watch progress.
   - Additionally, `incomingMovies` and `incomingSeries` were not deduplicated by `streamId`/`seriesId` before reconciliation.
3. **Channel Overrides Ignored by `countAll()`, `getFavorites()`, and `getRecentlyPlayed()` (`data/dao/ChannelDao.kt`)**:
   - `ChannelDao.countAll()` counted hidden channels (`cuo.isHidden = 1`), causing UI channel counts (`UrlListFragment`, category drawer) to disagree with the visible list.
   - `ChannelDao.getFavorites()` and `ChannelDao.getRecentlyPlayed()` did not join `channel_user_overrides`, allowing hidden channels to appear in Favorites/Recents and ignoring custom titles, logos, groups, and channel numbers.
4. **Duplicate Merge Did Not Migrate `UnifiedFavoriteEntity` Ledger (`data/repository/DuplicateCleanupRepository.kt`)**:
   - `DuplicateAction.MERGE_ENDPOINTS` only toggled `channelDao.setFavorite(primaryChannelId, true)` without migrating the favorite in `UnifiedFavoriteDao`. Because `ProfileFavoriteSynchronizer` treats `channels.isFavorite` as a projection of `unified_favorites`, the merged favorite would be wiped on the next profile switch or favorite toggle, and a dangling `live_identity:<deleted>` row remained.

---

### 2. Required Fixes

#### `data/repository/PlaylistRepository.kt`
Update imports to include `kotlinx.coroutines.NonCancellable`, `kotlinx.coroutines.withContext`, `vid.mat.app.provider.normalization.CategoryNormalizer`, and `vid.mat.app.provider.normalization.ChannelNameNormalizer`, and replace `syncPlaylist`:
```kotlin
    suspend fun syncPlaylist(
        context: Context,
        sourceId: Long,
        forceSync: Boolean = false,
        onProgress: ((processed: Int, total: Int) -> Unit)? = null
    ): Result<Int> {
        val source = sourceDao.getById(sourceId)
            ?: return Result.failure(IllegalArgumentException("Playlist source with ID $sourceId not found"))

        if (source.sourceUrl.isBlank()) {
            return Result.failure(IllegalStateException("No source URL configured for source $sourceId"))
        }

        sourceDao.updateSyncState(sourceId, SyncState.SYNCING, null)

        return try {
            AppLogger.i(TAG, "Starting sync for playlist source ${source.name} (ID: $sourceId)")
            val syncItems = mutableListOf<PlaylistSyncEngine.SyncItem>()
            var wasTruncated = false

            M3uParser.parseExtended(context, source.sourceUrl, httpClient).collect { event ->
                when (event) {
                    is M3uParser.ExtendedParseEvent.Channel -> {
                        val entry = event.entry
                        val vu = entry.channel
                        val canonicalTitle = ChannelNameNormalizer.normalize(vu.title).ifBlank { vu.title }
                        val unifiedGroup = CategoryNormalizer.normalize(vu.groupTitle)
                        val identity = PlaylistFingerprint.computeIdentity(
                            sourceId = sourceId,
                            externalId = vu.tvgId,
                            tvgId = vu.tvgId,
                            title = canonicalTitle
                        )

                        val channel = Channel(
                            sourceId = sourceId,
                            channelIdentity = identity,
                            externalId = vu.tvgId,
                            channelNumber = entry.channelNumber ?: (syncItems.size + 1),
                            title = vu.title,
                            logoUrl = vu.logoUrl,
                            groupTitle = unifiedGroup,
                            tvgId = vu.tvgId,
                            tvgName = entry.tvgName ?: vu.tvgId,
                            catchupDays = entry.catchupDays,
                            catchupSource = entry.catchupSource,
                            userOrder = syncItems.size
                        )

                        val path = vu.url.substringBefore('?').substringBefore('#').lowercase()
                        val inferredFormat = when {
                            path.endsWith(".m3u8") -> "m3u8"
                            path.endsWith(".mpd") -> "mpd"
                            path.endsWith(".mp4") || path.endsWith(".m4v") || path.endsWith(".mov") -> "mp4"
                            path.endsWith(".mkv") -> "mkv"
                            path.endsWith(".ts") || path.endsWith(".m2ts") -> "ts"
                            else -> "auto"
                        }

                        val endpoint = StreamEndpoint(
                            channelId = 0L,
                            url = vu.url,
                            priority = 0,
                            containerFormat = inferredFormat,
                            resolutionLabel = vu.resolution ?: "Auto",
                            userAgent = vu.userAgent,
                            referer = vu.referer,
                            cookie = vu.cookie,
                            authorization = vu.authorization
                        )

                        syncItems.add(PlaylistSyncEngine.SyncItem(channel, listOf(endpoint)))
                    }
                    is M3uParser.ExtendedParseEvent.Done -> wasTruncated = event.truncated
                    is M3uParser.ExtendedParseEvent.Error -> {
                        throw IllegalStateException(event.message, event.cause)
                    }
                    else -> Unit
                }
            }

            if (wasTruncated) {
                val reason = "Playlist exceeded maximum supported capacity and was truncated; aborting sync to preserve existing catalog."
                sourceDao.updateSyncState(sourceId, SyncState.FAILED, reason)
                return Result.failure(IllegalStateException(reason))
            }

            val existingCount = channelDao.countForSource(sourceId)
            val safetyCheck = ResilientSyncPolicy.evaluateBatchSafety(
                incomingCount = syncItems.size,
                existingCount = existingCount,
                sourceName = source.name,
                forceSync = forceSync
            )

            when (safetyCheck) {
                is ResilientSyncPolicy.EvaluationResult.Rejected -> {
                    sourceDao.updateSyncState(sourceId, SyncState.FAILED, safetyCheck.reason)
                    return Result.failure(IllegalStateException(safetyCheck.reason))
                }
                is ResilientSyncPolicy.EvaluationResult.RequiresUserApproval -> {
                    sourceDao.updateSyncState(sourceId, SyncState.FAILED, "Pending User Approval")
                    return Result.failure(SyncRequiresApprovalException(safetyCheck.reason, sourceId))
                }
                ResilientSyncPolicy.EvaluationResult.Accepted -> Unit
            }

            val syncResult = syncEngine.syncSource(
                sourceId = sourceId,
                incomingItems = syncItems,
                onProgress = onProgress
            )

            AppLogger.i(TAG, "Playlist sync successful for source $sourceId. Processed: ${syncResult.totalProcessed}")
            Result.success(syncResult.totalProcessed)
        } catch (e: kotlinx.coroutines.CancellationException) {
            withContext(NonCancellable) {
                runCatching { sourceDao.updateSyncState(sourceId, SyncState.FAILED, "Sync cancelled") }
            }
            throw e
        } catch (e: Exception) {
            AppLogger.e(TAG, "Failed syncing playlist $sourceId — preserving last known good catalog", e)
            sourceDao.updateSyncState(sourceId, SyncState.FAILED, e.message)
            Result.failure(e)
        }
    }
```

#### `data/repository/UnifiedCatalogRepository.kt`
In `syncProviderCatalog`, evaluate `ResilientSyncPolicy` for fetched VOD/Series catalogs as well, and deduplicate by `streamId`/`seriesId` before reconciling:
```kotlin
        if (catalog.isMoviesFetched) {
            val existingMovieCount = vodMovieDao.getAllForSource(sourceId).size
            when (val vodSafety = ResilientSyncPolicy.evaluateBatchSafety(
                incomingCount = catalog.movies.size,
                existingCount = existingMovieCount,
                sourceName = "${source.name} (VOD)",
                forceSync = forceSync
            )) {
                is ResilientSyncPolicy.EvaluationResult.Rejected -> {
                    sourceDao.updateSyncState(sourceId, SyncState.FAILED, vodSafety.reason)
                    return@withContext Result.failure(IllegalStateException(vodSafety.reason))
                }
                is ResilientSyncPolicy.EvaluationResult.RequiresUserApproval -> {
                    sourceDao.updateSyncState(sourceId, SyncState.FAILED, "Pending User Approval")
                    return@withContext Result.failure(SyncRequiresApprovalException(vodSafety.reason, sourceId))
                }
                ResilientSyncPolicy.EvaluationResult.Accepted -> Unit
            }
        }

        if (catalog.isSeriesFetched) {
            val existingSeriesCount = seriesDao.getAllForSource(sourceId).size
            when (val seriesSafety = ResilientSyncPolicy.evaluateBatchSafety(
                incomingCount = catalog.series.size,
                existingCount = existingSeriesCount,
                sourceName = "${source.name} (Series)",
                forceSync = forceSync
            )) {
                is ResilientSyncPolicy.EvaluationResult.Rejected -> {
                    sourceDao.updateSyncState(sourceId, SyncState.FAILED, seriesSafety.reason)
                    return@withContext Result.failure(IllegalStateException(seriesSafety.reason))
                }
                is ResilientSyncPolicy.EvaluationResult.RequiresUserApproval -> {
                    sourceDao.updateSyncState(sourceId, SyncState.FAILED, "Pending User Approval")
                    return@withContext Result.failure(SyncRequiresApprovalException(seriesSafety.reason, sourceId))
                }
                ResilientSyncPolicy.EvaluationResult.Accepted -> Unit
            }
        }
```
And in `reconcileMoviesNonDestructively` / `reconcileSeriesNonDestructively`, deduplicate incoming items:
```kotlin
    private suspend fun reconcileMoviesNonDestructively(sourceId: Long, incomingMovies: List<NormalizedVodMovie>) {
        val uniqueIncoming = incomingMovies.distinctBy { it.streamId }
        val existingByStreamId = vodMovieDao.getAllForSource(sourceId).associateBy { it.streamId }
        val incomingStreamIds = uniqueIncoming.map { it.streamId }.toSet()
        // ... iterate uniqueIncoming ...
```
```kotlin
    private suspend fun reconcileSeriesNonDestructively(sourceId: Long, incomingSeries: List<NormalizedSeries>) {
        val uniqueIncoming = incomingSeries.distinctBy { it.seriesId }
        val existingBySeriesId = seriesDao.getAllForSource(sourceId).associateBy { it.seriesId }
        val incomingSeriesIds = uniqueIncoming.map { it.seriesId }.toSet()
        // ... iterate uniqueIncoming ...
```

#### `data/dao/ChannelDao.kt`
Update `getFavorites()`, `getRecentlyPlayed()`, and `countAll()` to respect `channel_user_overrides`:
```kotlin
    @Query("""
        SELECT c.channelId, c.sourceId, c.channelIdentity, c.externalId,
               COALESCE(cuo.customChannelNumber, c.channelNumber) AS channelNumber,
               COALESCE(NULLIF(cuo.customTitle, ''), c.title) AS title,
               COALESCE(NULLIF(cuo.customLogoUrl, ''), c.logoUrl) AS logoUrl,
               COALESCE(NULLIF(cuo.customGroupTitle, ''), c.groupTitle) AS groupTitle,
               c.tvgId, c.tvgName, c.isFavorite,
               COALESCE(cuo.userOrderIndex, c.userOrder) AS userOrder,
               c.catchupDays, c.catchupSource, c.lastPlayedAtMs, c.createdAtMs, c.updatedAtMs
        FROM channels c
        LEFT JOIN channel_user_overrides cuo ON c.channelIdentity = cuo.channelIdentity
        WHERE COALESCE(cuo.isHidden, 0) = 0 AND c.isFavorite = 1
        ORDER BY COALESCE(cuo.userOrderIndex, c.userOrder) ASC, title ASC
        LIMIT 50
    """)
    fun getFavorites(): Flow<List<Channel>>

    @Query("""
        SELECT c.channelId, c.sourceId, c.channelIdentity, c.externalId,
               COALESCE(cuo.customChannelNumber, c.channelNumber) AS channelNumber,
               COALESCE(NULLIF(cuo.customTitle, ''), c.title) AS title,
               COALESCE(NULLIF(cuo.customLogoUrl, ''), c.logoUrl) AS logoUrl,
               COALESCE(NULLIF(cuo.customGroupTitle, ''), c.groupTitle) AS groupTitle,
               c.tvgId, c.tvgName, c.isFavorite,
               COALESCE(cuo.userOrderIndex, c.userOrder) AS userOrder,
               c.catchupDays, c.catchupSource, c.lastPlayedAtMs, c.createdAtMs, c.updatedAtMs
        FROM channels c
        LEFT JOIN channel_user_overrides cuo ON c.channelIdentity = cuo.channelIdentity
        WHERE COALESCE(cuo.isHidden, 0) = 0 AND c.lastPlayedAtMs IS NOT NULL
        ORDER BY c.lastPlayedAtMs DESC
        LIMIT 20
    """)
    fun getRecentlyPlayed(): Flow<List<Channel>>

    @Query("""
        SELECT COUNT(*) FROM channels c
        LEFT JOIN channel_user_overrides cuo ON c.channelIdentity = cuo.channelIdentity
        WHERE COALESCE(cuo.isHidden, 0) = 0
    """)
    fun countAll(): Flow<Int>
```
*(Keep `countForSource(sourceId: Long)` unchanged so raw source counts for `ResilientSyncPolicy` remain accurate.)*

#### `data/repository/DuplicateCleanupRepository.kt`
Inject `UnifiedFavoriteDao` and `ActiveProfileProvider`, and migrate/clean up `unified_favorites` entries when merging or deleting duplicate channels:
```kotlin
@Singleton
class DuplicateCleanupRepository @Inject constructor(
    @ApplicationContext private val context: Context,
    private val database: VideoUrlDatabase,
    private val channelDao: ChannelDao,
    private val endpointDao: StreamEndpointDao,
    private val overrideDao: ChannelUserOverrideDao,
    private val favoriteDao: UnifiedFavoriteDao,
    private val activeProfileProvider: ActiveProfileProvider,
    private val engine: DuplicateDetectionEngine
) {
```
Inside `resolveGroup`:
```kotlin
    suspend fun resolveGroup(group: DuplicateChannelGroup, action: DuplicateAction) = withContext(Dispatchers.IO) {
        val activeProfileId = activeProfileProvider.awaitInitializedProfile().id
        database.withTransaction {
            when (action) {
                DuplicateAction.MERGE_ENDPOINTS -> {
                    AppLogger.i(TAG, "Merging endpoints for '${group.bestCandidate.channel.title}'")
                    val primaryChannel = group.bestCandidate.channel
                    val primaryChannelId = primaryChannel.channelId
                    val existingUrls = group.bestCandidate.endpoints.map { it.url }.toHashSet()
                    val lowestPrimaryPriority = group.bestCandidate.endpoints.minOfOrNull { it.priority } ?: 0

                    val allEndpointsToAttach = group.duplicates
                        .flatMap { it.endpoints }
                        .distinctBy { it.url }
                        .filterNot { it.url in existingUrls }
                        .mapIndexed { idx, ep ->
                            ep.copy(
                                endpointId = 0L,
                                channelId = primaryChannelId,
                                priority = lowestPrimaryPriority - 1 - idx
                            )
                        }

                    if (allEndpointsToAttach.isNotEmpty()) {
                        endpointDao.insertBatch(allEndpointsToAttach)
                    }

                    val primaryFavTarget = FavoriteTargetIds.live(primaryChannel.channelIdentity)
                    var primaryIsFav = primaryChannel.isFavorite ||
                        favoriteDao.isFavorite(primaryFavTarget, activeProfileId)

                    group.duplicates.forEach { dup ->
                        val dupFavTarget = FavoriteTargetIds.live(dup.channel.channelIdentity)
                        if (dup.channel.isFavorite || favoriteDao.isFavorite(dupFavTarget, activeProfileId)) {
                            if (!primaryIsFav) {
                                favoriteDao.upsert(
                                    UnifiedFavoriteEntity(
                                        targetId = primaryFavTarget,
                                        profileId = activeProfileId,
                                        type = FavoriteType.LIVE_CHANNEL,
                                        title = primaryChannel.title,
                                        subtitle = primaryChannel.tvgName,
                                        logoUrl = primaryChannel.logoUrl,
                                        streamUrl = group.bestCandidate.endpoints.firstOrNull()?.url.orEmpty(),
                                        groupTitle = primaryChannel.groupTitle
                                    )
                                )
                                channelDao.setFavorite(primaryChannelId, true)
                                primaryIsFav = true
                            }
                            favoriteDao.deleteByTarget(dupFavTarget, activeProfileId)
                        }
                        overrideDao.deleteByIdentity(dup.channel.channelIdentity)
                        channelDao.delete(dup.channel)
                    }
                }

                DuplicateAction.HIDE_DUPLICATES -> {
                    AppLogger.i(TAG, "Hiding duplicate channels for '${group.bestCandidate.channel.title}'")
                    val now = System.currentTimeMillis()
                    group.duplicates.forEach { dup ->
                        val existing = overrideDao.getByIdentity(dup.channel.channelIdentity)
                        val updated = (existing ?: ChannelUserOverrideEntity(
                            channelIdentity = dup.channel.channelIdentity
                        )).copy(
                            isHidden = true,
                            updatedAtMs = now
                        )
                        overrideDao.upsert(updated)
                    }
                }

                DuplicateAction.DELETE_DUPLICATES -> {
                    AppLogger.i(TAG, "Permanently deleting ${group.duplicates.size} duplicate channels for group ${group.groupId}")
                    group.duplicates.forEach { dup ->
                        favoriteDao.deleteByTarget(FavoriteTargetIds.live(dup.channel.channelIdentity), activeProfileId)
                        overrideDao.deleteByIdentity(dup.channel.channelIdentity)
                        channelDao.delete(dup.channel)
                    }
                }

                DuplicateAction.IGNORE_GROUP -> {
                    AppLogger.d(TAG, "Ignoring duplicate group: ${group.normalizedTitle}")
                    persistIgnoredGroup(group.normalizedTitle)
                }
            }
        }
    }
```

---

### 3. Phase 2 Report

- **Completed functionality (verified in code)**:
  - Process-death sync state recovery (`AppLifecycleSupervisor.reconcileOrphanedSyncStates` → `PlaylistSourceDao.resetOrphanedSyncStates`) and coroutine cancellation cleanup (`UnifiedCatalogRepository`, `UrlViewModel`, `PlaylistRepository`).
  - Consistent `PlaylistFingerprint` identity computation across `M3uIptvProvider`, `UrlViewModel`, `PlaylistRepository`, and `XtreamIptvProvider`, plus non-destructive `PlaylistSyncEngine` endpoint/channel merging.
  - Preservation of `ChannelUserOverrideEntity` across playlist syncs and enforcement of hidden/renamed/reordered overrides across `ChannelDao.getPagedChannels`, `UnifiedCatalogDao.getPagedChannels`, `ChannelDao.countAll`, `ChannelDao.getFavorites`, and `ChannelDao.getRecentlyPlayed`.
  - Partial/truncated import and catastrophic shrinkage protection (`ResilientSyncPolicy` + `isComplete` / `wasTruncated`) across Live channels, VOD movies, and Series in `UnifiedCatalogRepository`, `UrlViewModel`, and `PlaylistRepository`.
  - Duplicate clustering, endpoint merging, hiding, and `UnifiedFavoriteEntity` ledger migration in `DuplicateDetectionEngine` and `DuplicateCleanupRepository`.

- **Remaining defects**:
  - None identified in Phase 2 code paths; note that `DuplicateCleanupRepository` cleans/migrates `UnifiedFavoriteEntity` for the active profile only (`UnifiedFavoriteDao` currently exposes `deleteByTarget(targetId, profileId)` rather than cross-profile deletion by `targetId`).

- **Modified classes**:
  - `data/repository/PlaylistRepository.kt`
  - `data/repository/UnifiedCatalogRepository.kt`
  - `data/dao/ChannelDao.kt`
  - `data/repository/DuplicateCleanupRepository.kt`

- **Integration points**:
  - `PlaylistRepository` / `UrlViewModel` / `UnifiedCatalogRepository` ↔ `M3uParser.parseExtended` ↔ `PlaylistFingerprint` ↔ `ResilientSyncPolicy` ↔ `PlaylistSyncEngine`
  - `ChannelDao` / `UnifiedCatalogDao` ↔ `ChannelUserOverrideDao` ↔ `ChannelOrganizationRepository`
  - `DuplicateCleanupRepository` ↔ `DuplicateDetectionEngine` ↔ `UnifiedFavoriteDao` / `ActiveProfileProvider`
  - `AppLifecycleSupervisor` ↔ `PlaylistSourceDao.resetOrphanedSyncStates`

- **Verification results**:
  - Static code-path verification completed for all synchronization, identity, override projection, partial-import abort, and duplicate-resolution flows. Room query projections in `ChannelDao` match all `Channel` columns and `ChannelUserOverrideDao` schema.

- **Limitations preventing full verification**:
  - No Gradle build/Room annotation processor or device/instrumentation test runner is available in this environment to compile and execute Room migration/DAO queries or live playlist sync tests.

Ready to proceed to **Phase 3: Live player** when you say **start phase 3**.








