Those compiler errors come from 13 files where helper constants/imports, a class closing brace, or parameter order were missing or mismatched. Fixing those 13 files will clear all of the listed errors (including the cascading errors in `PlaylistRepository`, `PlaylistIntelligenceViewModel`, `FastChannelZapper`, `PlaybackEngine`, and `EpgProgramDetailsBottomSheet`).

Here is **Compile-Fix Batch 1 (5 files)**:
1. `catchup/UnifiedCatchupEngine.kt` — Adds `xtreamDateFormat` in `companion object`.
2. `di/AppModule.kt` — Fixes `provideStreamResolver(...)` argument order (`okHttpClient, unifiedCatchupEngine`).
3. `error/recovery/SmartRecoveryManager.kt` — Adds missing `import vid.mat.app.player.StreamFailureReason`.
4. `network/OkHttpDataSourceHelper.kt` — Closes `DynamicStreamInterceptor` class before `buildFactory(...)`.
5. `player/PlaybackRecoveryManager.kt` — Fixes `onActiveStreamChanged` default lambda (`= { _, _ -> }`).

```python
#!/usr/bin/env python3
from pathlib import Path

BASE_DIR = Path("/home/username/AndroidStudioProjects/Ytvmate/app/src/main/java/vid/mat/app")

FILES = {
    "catchup/UnifiedCatchupEngine.kt": """package vid.mat.app.catchup

import android.net.Uri
import vid.mat.app.data.dao.ChannelDao
import vid.mat.app.data.dao.PlaylistSourceDao
import vid.mat.app.data.dao.StreamEndpointDao
import vid.mat.app.data.model.CatchupMetadata
import vid.mat.app.data.model.CatchupType
import vid.mat.app.data.model.StreamEndpoint
import vid.mat.app.epg.model.EpgProgram
import vid.mat.app.provider.StreamDescriptor
import vid.mat.app.security.XtreamAccountManager
import java.text.SimpleDateFormat
import java.util.Date
import java.util.Locale
import java.util.TimeZone
import javax.inject.Inject
import javax.inject.Singleton

@Singleton
class UnifiedCatchupEngine @Inject constructor(
    private val channelDao: ChannelDao,
    private val endpointDao: StreamEndpointDao,
    private val sourceDao: PlaylistSourceDao,
    private val xtreamAccountManager: XtreamAccountManager
) {
    companion object {
        private const val TAG = "UnifiedCatchupEngine"
        private const val MS_PER_DAY = 86_400_000L
        private val xtreamDateFormat = SimpleDateFormat("yyyy-MM-dd:HH-mm", Locale.US).apply {
            timeZone = TimeZone.getTimeZone("UTC")
        }
    }

    suspend fun resolveCatchupDescriptor(
        channelId: Long,
        startMs: Long,
        stopMs: Long
    ): Result<StreamDescriptor> {
        if (channelId <= 0L || startMs <= 0L || stopMs <= startMs) {
            return Result.failure(IllegalStateException("Catch-up is not available for this time range"))
        }

        val channel = channelDao.getById(channelId)
            ?: return Result.failure(IllegalStateException("Channel not found"))

        val nowMs = System.currentTimeMillis()
        val ageMs = nowMs - stopMs
        val retentionMs = channel.catchupDays.toLong() * MS_PER_DAY
        if (channel.catchupDays <= 0 || ageMs >= retentionMs) {
            return Result.failure(IllegalStateException("Catch-up is not available or has expired for this program"))
        }

        val durationSec = (stopMs - startMs) / 1000L
        if (durationSec <= 0L) {
            return Result.failure(IllegalStateException("Invalid program duration for catch-up"))
        }

        val endpoints = endpointDao.getEndpointsForChannel(channelId)
            .sortedWith(
                compareByDescending<StreamEndpoint> { it.isWorking }
                    .thenByDescending { it.priority }
                    .thenBy { it.failureCount }
            )
        val primaryEndpoint = endpoints.firstOrNull()
            ?: return Result.failure(IllegalStateException("No endpoints available for channel '${channel.title}'"))

        val source = sourceDao.getById(channel.sourceId)
        val accountId = source?.xtreamAccountId
        if (accountId != null) {
            return when (val creds = xtreamAccountManager.resolveCredentials(accountId)) {
                is XtreamAccountManager.CredentialsResolution.Success -> {
                    val formattedDate = synchronized(xtreamDateFormat) { xtreamDateFormat.format(Date(startMs)) }
                    val durationMin = (durationSec / 60L).coerceAtLeast(1L)
                    val cleanServer = creds.serverUrl.trimEnd('/')
                    val streamId = channel.externalId?.takeIf { it.isNotBlank() }
                        ?: primaryEndpoint.url.substringAfterLast('/').substringBefore('.')
                            .takeIf { it.isNotBlank() }
                        ?: return Result.failure(IllegalStateException("Missing stream ID for Xtream catch-up"))

                    val uri = "$cleanServer/timeshift/X-AUTH-INJECT/$durationMin/$formattedDate/$streamId.m3u8"
                    Result.success(
                        StreamDescriptor(
                            streamUri = uri,
                            containerFormat = "m3u8",
                            headers = buildMap {
                                put("User-Agent", primaryEndpoint.userAgent ?: "IPTVSmartersPlayer")
                                primaryEndpoint.referer?.value?.let { put("Referer", it) }
                                primaryEndpoint.cookie?.value?.let { put("Cookie", it) }
                                primaryEndpoint.authorization?.value?.let { put("Authorization", it) }
                            },
                            userAgent = primaryEndpoint.userAgent ?: "IPTVSmartersPlayer",
                            isLive = false,
                            xtreamUser = creds.username,
                            xtreamPass = creds.pass
                        )
                    )
                }
                XtreamAccountManager.CredentialsResolution.NeedsReauthentication ->
                    Result.failure(IllegalStateException("Xtream credentials require re-authentication"))
                is XtreamAccountManager.CredentialsResolution.Error ->
                    Result.failure(IllegalStateException(creds.message))
            }
        }

        val inferredType = inferCatchupType(primaryEndpoint.url, channel.catchupSource)
        val metadata = CatchupMetadata(
            channelId = channelId,
            catchupType = inferredType,
            days = channel.catchupDays,
            sourceTemplate = channel.catchupSource
        )
        val resolvedUrl = CatchupUrlBuilder.buildCatchupUrl(
            streamUrl = primaryEndpoint.url,
            metadata = metadata,
            startEpochMs = startMs,
            durationSeconds = durationSec
        )

        return Result.success(
            StreamDescriptor(
                streamUri = resolvedUrl,
                containerFormat = if (resolvedUrl.contains(".ts", ignoreCase = true)) "ts" else "m3u8",
                headers = buildMap {
                    primaryEndpoint.userAgent?.let { put("User-Agent", it) }
                    primaryEndpoint.referer?.value?.let { put("Referer", it) }
                    primaryEndpoint.cookie?.value?.let { put("Cookie", it) }
                    primaryEndpoint.authorization?.value?.let { put("Authorization", it) }
                },
                userAgent = primaryEndpoint.userAgent,
                isLive = false
            )
        )
    }

    suspend fun resolveCatchupStream(channelId: Long, program: EpgProgram): String? {
        val descriptor = resolveCatchupDescriptor(channelId, program.startMs, program.stopMs).getOrNull()
            ?: return null
        return if (!descriptor.xtreamUser.isNullOrBlank() && !descriptor.xtreamPass.isNullOrBlank()) {
            descriptor.streamUri.replace(
                "/X-AUTH-INJECT/",
                "/${Uri.encode(descriptor.xtreamUser)}/${Uri.encode(descriptor.xtreamPass)}/"
            )
        } else {
            descriptor.streamUri
        }
    }

    private fun inferCatchupType(streamUrl: String, sourceTemplate: String?): CatchupType {
        val tmpl = sourceTemplate?.lowercase().orEmpty()
        return when {
            tmpl.contains("timeshift_abs") || tmpl.contains("video-") || streamUrl.contains("flussonic", ignoreCase = true) ->
                CatchupType.FLUSSONIC
            tmpl.contains("timeshift=") -> CatchupType.SHIFT
            tmpl.isNotBlank() -> CatchupType.APPEND
            else -> CatchupType.DEFAULT
        }
    }
}
""",

    "di/AppModule.kt": """package vid.mat.app.di

import android.content.Context
import dagger.Module
import dagger.Provides
import dagger.hilt.InstallIn
import dagger.hilt.android.qualifiers.ApplicationContext
import dagger.hilt.components.SingletonComponent
import okhttp3.OkHttpClient
import vid.mat.app.ads.promo.PromoAccountingDao
import vid.mat.app.catchup.UnifiedCatchupEngine
import vid.mat.app.data.dao.*
import vid.mat.app.data.database.VideoUrlDatabase
import vid.mat.app.data.repository.*
import vid.mat.app.dvr.dao.DvrDao
import vid.mat.app.epg.EpgAlignmentEngine
import vid.mat.app.epg.EpgDao
import vid.mat.app.epg.EpgRepository
import vid.mat.app.favorites.ProfileFavoriteSynchronizer
import vid.mat.app.favorites.UnifiedFavoriteRepository
import vid.mat.app.favorites.dao.UnifiedFavoriteDao
import vid.mat.app.history.WatchHistoryDao
import vid.mat.app.history.WatchHistoryRepository
import vid.mat.app.network.OkHttpStreamingClient
import vid.mat.app.player.NewPlayer
import vid.mat.app.player.NewPlayerImpl
import vid.mat.app.player.StreamFailoverEngine
import vid.mat.app.player.abr.IntelligentAbrController
import vid.mat.app.player.capability.DeviceCodecCapabilityDetector
import vid.mat.app.player.intelligence.BandwidthEstimator
import vid.mat.app.player.intelligence.PlaybackIntelligence
import vid.mat.app.player.intelligence.StreamHealthStore
import vid.mat.app.player.performance.StreamPerformanceMonitor
import vid.mat.app.player.quality.SmartStreamSelector
import vid.mat.app.player.sync.AudioDelayAudioProcessor
import vid.mat.app.player.sync.PlayerSynchronizationController
import vid.mat.app.player.sync.PlayerSynchronizationControllerImpl
import vid.mat.app.profile.ActiveProfileProvider
import vid.mat.app.profile.dao.ProfileDao
import vid.mat.app.provider.IptvProviderFactory
import vid.mat.app.provider.StreamResolver
import vid.mat.app.security.ParentalControlManager
import vid.mat.app.security.ParentalGatekeeper
import vid.mat.app.security.XtreamAccountManager
import javax.inject.Singleton

@Module
@InstallIn(SingletonComponent::class)
object AppModule {
    @Provides
    @Singleton
    fun provideDatabase(@ApplicationContext context: Context): VideoUrlDatabase =
        VideoUrlDatabase.getInstance(context)

    @Provides fun provideVideoUrlDao(db: VideoUrlDatabase): VideoUrlDao = db.videoUrlDao()
    @Provides fun providePlaylistSourceDao(db: VideoUrlDatabase): PlaylistSourceDao = db.playlistSourceDao()
    @Provides fun provideChannelDao(db: VideoUrlDatabase): ChannelDao = db.channelDao()
    @Provides fun provideStreamEndpointDao(db: VideoUrlDatabase): StreamEndpointDao = db.streamEndpointDao()
    @Provides fun provideEpgDao(db: VideoUrlDatabase): EpgDao = db.epgDao()
    @Provides fun provideWatchHistoryDao(db: VideoUrlDatabase): WatchHistoryDao = db.watchHistoryDao()
    @Provides fun provideXtreamAccountDao(db: VideoUrlDatabase): XtreamAccountDao = db.xtreamAccountDao()
    @Provides fun provideDvrDao(db: VideoUrlDatabase): DvrDao = db.dvrDao()
    @Provides fun provideProfileDao(db: VideoUrlDatabase): ProfileDao = db.profileDao()
    @Provides fun provideChannelUserOverrideDao(db: VideoUrlDatabase): ChannelUserOverrideDao = db.channelUserOverrideDao()
    @Provides fun provideCustomGroupDao(db: VideoUrlDatabase): CustomGroupDao = db.customGroupDao()
    @Provides fun provideWatchProgressDao(db: VideoUrlDatabase): WatchProgressDao = db.watchProgressDao()
    @Provides fun provideUnifiedFavoriteDao(db: VideoUrlDatabase): UnifiedFavoriteDao = db.unifiedFavoriteDao()
    @Provides fun provideVodMovieDao(db: VideoUrlDatabase): VodMovieDao = db.vodMovieDao()
    @Provides fun provideSeriesDao(db: VideoUrlDatabase): SeriesDao = db.seriesDao()
    @Provides fun provideSearchDao(db: VideoUrlDatabase): SearchDao = db.searchDao()
    @Provides fun provideUnifiedCatalogDao(db: VideoUrlDatabase): UnifiedCatalogDao = db.unifiedCatalogDao()
    @Provides fun providePromoAccountingDao(db: VideoUrlDatabase): PromoAccountingDao = db.promoAccountingDao()

    @Provides
    @Singleton
    fun provideOkHttpClient(): OkHttpClient = OkHttpStreamingClient.epgClient

    @Provides
    @Singleton
    fun provideVideoUrlRepository(dao: VideoUrlDao, favoriteRepository: UnifiedFavoriteRepository): VideoUrlRepository =
        VideoUrlRepository(dao, favoriteRepository)

    @Provides
    @Singleton
    fun provideChannelRepository(channelDao: ChannelDao, endpointDao: StreamEndpointDao, favoriteRepository: UnifiedFavoriteRepository): ChannelRepository =
        ChannelRepository(channelDao, endpointDao, favoriteRepository)

    @Provides
    @Singleton
    fun providePlaylistSourceRepository(sourceDao: PlaylistSourceDao): PlaylistSourceRepository =
        PlaylistSourceRepository(sourceDao)

    @Provides
    @Singleton
    fun provideEpgRepository(dao: EpgDao, okHttpClient: OkHttpClient): EpgRepository =
        EpgRepository(dao, okHttpClient)

    @Provides
    @Singleton
    fun provideWatchHistoryRepository(
        dao: WatchHistoryDao,
        activeProfileProvider: ActiveProfileProvider,
        @ApplicationContext context: Context
    ): WatchHistoryRepository =
        WatchHistoryRepository(dao, activeProfileProvider, context)

    @Provides
    @Singleton
    fun provideXtreamAccountManager(dao: XtreamAccountDao): XtreamAccountManager =
        XtreamAccountManager(dao)

    @Provides
    @Singleton
    fun provideParentalControlManager(@ApplicationContext context: Context): ParentalControlManager =
        ParentalControlManager(context)

    @Provides
    @Singleton
    fun provideParentalGatekeeper(
        parentalControlManager: ParentalControlManager,
        activeProfileProvider: ActiveProfileProvider
    ): ParentalGatekeeper = ParentalGatekeeper(parentalControlManager, activeProfileProvider)

    @Provides
    @Singleton
    fun provideUnifiedFavoriteRepository(
        favoriteDao: UnifiedFavoriteDao,
        activeProfileProvider: ActiveProfileProvider,
        parentalGatekeeper: ParentalGatekeeper,
        synchronizer: ProfileFavoriteSynchronizer
    ): UnifiedFavoriteRepository =
        UnifiedFavoriteRepository(favoriteDao, activeProfileProvider, parentalGatekeeper, synchronizer)

    @Provides
    @Singleton
    fun provideAudioDelayAudioProcessor(): AudioDelayAudioProcessor = AudioDelayAudioProcessor()

    @Provides
    @Singleton
    fun providePlayerSynchronizationController(
        @ApplicationContext context: Context,
        audioDelayProcessor: AudioDelayAudioProcessor
    ): PlayerSynchronizationController =
        PlayerSynchronizationControllerImpl(context, audioDelayProcessor)

    @Provides
    @Singleton
    fun provideDeviceCodecCapabilityDetector(): DeviceCodecCapabilityDetector =
        DeviceCodecCapabilityDetector()

    @Provides
    @Singleton
    fun provideStreamHealthStore(): StreamHealthStore =
        StreamHealthStore()

    @Provides
    @Singleton
    fun provideStreamPerformanceMonitor(): StreamPerformanceMonitor =
        StreamPerformanceMonitor()

    @Provides
    @Singleton
    fun provideBandwidthEstimator(): BandwidthEstimator =
        BandwidthEstimator()

    @Provides
    @Singleton
    fun provideIntelligentAbrController(
        bandwidthEstimator: BandwidthEstimator,
        codecDetector: DeviceCodecCapabilityDetector
    ): IntelligentAbrController =
        IntelligentAbrController(bandwidthEstimator, codecDetector)

    @Provides
    @Singleton
    fun provideSmartStreamSelector(
        codecDetector: DeviceCodecCapabilityDetector,
        healthStore: StreamHealthStore,
        bandwidthEstimator: BandwidthEstimator
    ): SmartStreamSelector = SmartStreamSelector(codecDetector, healthStore, bandwidthEstimator)

    @Provides
    @Singleton
    fun providePlaybackIntelligence(
        @ApplicationContext context: Context,
        failoverEngine: StreamFailoverEngine,
        performanceMonitor: StreamPerformanceMonitor,
        codecDetector: DeviceCodecCapabilityDetector,
        healthStore: StreamHealthStore,
        bandwidthEstimator: BandwidthEstimator,
        abrController: IntelligentAbrController
    ): PlaybackIntelligence =
        PlaybackIntelligence(
            context,
            failoverEngine,
            performanceMonitor,
            codecDetector,
            healthStore,
            bandwidthEstimator,
            abrController
        )

    @Provides
    @Singleton
    fun provideIptvProviderFactory(
        @ApplicationContext context: Context,
        httpClient: OkHttpClient,
        xtreamAccountManager: XtreamAccountManager
    ): IptvProviderFactory = IptvProviderFactory(context, httpClient, xtreamAccountManager)

    @Provides
    @Singleton
    fun provideUnifiedCatchupEngine(
        channelDao: ChannelDao,
        endpointDao: StreamEndpointDao,
        sourceDao: PlaylistSourceDao,
        xtreamAccountManager: XtreamAccountManager
    ): UnifiedCatchupEngine = UnifiedCatchupEngine(channelDao, endpointDao, sourceDao, xtreamAccountManager)

    @Provides
    @Singleton
    fun provideEpgAlignmentEngine(
        epgDao: EpgDao,
        sourceDao: PlaylistSourceDao
    ): EpgAlignmentEngine = EpgAlignmentEngine(epgDao, sourceDao)

    @Provides
    @Singleton
    fun provideStreamResolver(
        @ApplicationContext context: Context,
        channelDao: ChannelDao,
        endpointDao: StreamEndpointDao,
        sourceDao: PlaylistSourceDao,
        videoUrlDao: VideoUrlDao,
        vodMovieDao: VodMovieDao,
        seriesDao: SeriesDao,
        providerFactory: IptvProviderFactory,
        xtreamAccountManager: XtreamAccountManager,
        parentalGatekeeper: ParentalGatekeeper,
        okHttpClient: OkHttpClient,
        unifiedCatchupEngine: UnifiedCatchupEngine
    ): StreamResolver = StreamResolver(
        context, channelDao, endpointDao, sourceDao, videoUrlDao,
        vodMovieDao, seriesDao, providerFactory, xtreamAccountManager,
        parentalGatekeeper, okHttpClient, unifiedCatchupEngine
    )

    @Provides
    @Singleton
    fun provideMediaRepository(
        channelDao: ChannelDao,
        endpointDao: StreamEndpointDao,
        videoUrlDao: VideoUrlDao
    ): MediaRepository = DefaultMediaRepository(channelDao, endpointDao, videoUrlDao)

    @Provides
    @Singleton
    fun provideNewPlayer(impl: NewPlayerImpl): NewPlayer = impl
}
""",

    "error/recovery/SmartRecoveryManager.kt": """package vid.mat.app.error.recovery

import androidx.media3.common.PlaybackException
import kotlinx.coroutines.CoroutineScope
import kotlinx.coroutines.Dispatchers
import kotlinx.coroutines.Job
import kotlinx.coroutines.SupervisorJob
import kotlinx.coroutines.delay
import kotlinx.coroutines.flow.MutableStateFlow
import kotlinx.coroutines.flow.StateFlow
import kotlinx.coroutines.flow.asStateFlow
import kotlinx.coroutines.launch
import vid.mat.app.error.ErrorClassifier
import vid.mat.app.error.NetworkHealthMonitor
import vid.mat.app.player.NewPlayer
import vid.mat.app.player.StreamCandidate
import vid.mat.app.player.StreamFailoverEngine
import vid.mat.app.player.StreamFailureReason
import javax.inject.Inject
import javax.inject.Singleton

@Singleton
class SmartRecoveryManager @Inject constructor(
    private val failoverEngine: StreamFailoverEngine
) {
    private val scope = CoroutineScope(SupervisorJob() + Dispatchers.Main)
    private var recoveryJob: Job? = null

    private val _recoveryState = MutableStateFlow(PlaybackTroubleshooterState())
    val recoveryState: StateFlow<PlaybackTroubleshooterState> = _recoveryState.asStateFlow()

    fun dismiss() {
        recoveryJob?.cancel()
        recoveryJob = null
        _recoveryState.value = PlaybackTroubleshooterState()
    }

    fun handlePlaybackFault(
        error: PlaybackException,
        channelKey: String,
        currentCandidateId: String,
        networkMonitor: NetworkHealthMonitor,
        player: NewPlayer,
        onAlternateElected: (StreamCandidate) -> Unit,
        onTerminalFailure: (PlaybackTroubleshooterState) -> Unit = {}
    ) {
        recoveryJob?.cancel()
        recoveryJob = scope.launch {
            val classification = ErrorClassifier.classify(error)
            val initialSteps = listOf(
                TroubleshooterStep(RecoveryStepType.CHECK_NETWORK, "Verify Network Connectivity", StepExecutionStatus.IN_PROGRESS),
                TroubleshooterStep(RecoveryStepType.TRY_ALTERNATE_ENDPOINT, "Test Alternate Stream Endpoints", StepExecutionStatus.PENDING),
                TroubleshooterStep(RecoveryStepType.TEST_DECODER_FALLBACK, "Check Hardware Codec Compatibility", StepExecutionStatus.PENDING),
                TroubleshooterStep(RecoveryStepType.RETRY_STREAM_CONNECTION, "Re-establish Stream Handshake", StepExecutionStatus.PENDING)
            )

            _recoveryState.value = PlaybackTroubleshooterState(
                isTroubleshooting = true,
                steps = initialSteps,
                rootCauseSummary = classification.userMessage,
                recommendedAction = "Diagnosing stream failure and evaluating recovery routes...",
                channelKey = channelKey
            )

            delay(350L)

            val isNetworkOk = networkMonitor.isConnected
            updateStep(
                type = RecoveryStepType.CHECK_NETWORK,
                status = if (isNetworkOk) StepExecutionStatus.SUCCESS else StepExecutionStatus.FAILED,
                detail = if (isNetworkOk) "Internet connection online (${networkMonitor.networkType.value.name})" else "No active internet connection"
            )

            if (!isNetworkOk) {
                updateStep(RecoveryStepType.TRY_ALTERNATE_ENDPOINT, StepExecutionStatus.SKIPPED, "Network offline")
                updateStep(RecoveryStepType.TEST_DECODER_FALLBACK, StepExecutionStatus.SKIPPED, "Network offline")
                updateStep(RecoveryStepType.RETRY_STREAM_CONNECTION, StepExecutionStatus.FAILED, "Waiting for network")
                val terminalState = _recoveryState.value.copy(
                    isTroubleshooting = false,
                    isTerminalFailure = true,
                    canForceRetry = false,
                    recommendedAction = "Please check your Wi-Fi or mobile data connection and try again."
                )
                _recoveryState.value = terminalState
                onTerminalFailure(terminalState)
                return@launch
            }

            delay(250L)
            updateStep(RecoveryStepType.TRY_ALTERNATE_ENDPOINT, StepExecutionStatus.IN_PROGRESS)
            val failureReason = failoverEngine.classifyError(error)
            val alternate = failoverEngine.selectNextCandidate(channelKey, currentCandidateId, failureReason)

            if (alternate != null) {
                updateStep(
                    RecoveryStepType.TRY_ALTERNATE_ENDPOINT,
                    StepExecutionStatus.SUCCESS,
                    "Switched to backup source (${alternate.resolutionLabel})"
                )
                updateStep(RecoveryStepType.TEST_DECODER_FALLBACK, StepExecutionStatus.SKIPPED, "Backup source elected")
                updateStep(RecoveryStepType.RETRY_STREAM_CONNECTION, StepExecutionStatus.SUCCESS, "Handshaking backup endpoint")

                _recoveryState.value = _recoveryState.value.copy(
                    isTroubleshooting = false,
                    recommendedAction = "Connecting to alternate stream endpoint..."
                )
                onAlternateElected(alternate)
                delay(900L)
                if (!_recoveryState.value.isTerminalFailure) {
                    _recoveryState.value = PlaybackTroubleshooterState()
                }
            } else {
                updateStep(
                    RecoveryStepType.TRY_ALTERNATE_ENDPOINT,
                    StepExecutionStatus.FAILED,
                    "No working alternate endpoints available"
                )

                val isDecoderFault = failureReason == StreamFailureReason.DECODER_FAILURE ||
                    failureReason == StreamFailureReason.UNSUPPORTED_CODEC
                updateStep(
                    type = RecoveryStepType.TEST_DECODER_FALLBACK,
                    status = if (isDecoderFault) StepExecutionStatus.FAILED else StepExecutionStatus.SUCCESS,
                    detail = if (isDecoderFault) "Device decoder incompatible with stream format" else "Hardware decoder OK"
                )

                updateStep(
                    type = RecoveryStepType.RETRY_STREAM_CONNECTION,
                    status = StepExecutionStatus.FAILED,
                    detail = "Origin stream unreachable after retries"
                )

                val terminalState = _recoveryState.value.copy(
                    isTroubleshooting = false,
                    isTerminalFailure = true,
                    canForceRetry = true,
                    recommendedAction = "All stream sources exhausted. Tap Force Retry to clear cooldowns or Retry Now."
                )
                _recoveryState.value = terminalState
                onTerminalFailure(terminalState)
            }
        }
    }

    private fun updateStep(
        type: RecoveryStepType,
        status: StepExecutionStatus,
        detail: String? = null
    ) {
        val updated = _recoveryState.value.steps.map { step ->
            if (step.type == type) step.copy(status = status, detailMessage = detail ?: step.detailMessage)
            else step
        }
        _recoveryState.value = _recoveryState.value.copy(steps = updated)
    }
}
""",

    "network/OkHttpDataSourceHelper.kt": """package vid.mat.app.network

import android.content.Context
import android.net.Uri
import androidx.annotation.OptIn
import androidx.media3.common.util.UnstableApi
import androidx.media3.datasource.DataSource
import androidx.media3.datasource.DefaultDataSource
import androidx.media3.datasource.okhttp.OkHttpDataSource
import okhttp3.HttpUrl.Companion.toHttpUrlOrNull
import okhttp3.Interceptor
import okhttp3.OkHttpClient
import okhttp3.Response
import vid.mat.app.security.NetworkSecurityContext
import vid.mat.app.security.NetworkSecurityValidator
import java.util.concurrent.ConcurrentHashMap

@OptIn(UnstableApi::class)
object OkHttpDataSourceHelper {

    private const val DEFAULT_USER_AGENT =
        "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/120.0.0.0 Safari/537.36"

    data class StreamContext(
        val headers: OkHttpStreamingClient.StreamHeaders = OkHttpStreamingClient.StreamHeaders(),
        val resolvedUrlOverride: String? = null,
        val host: String? = null,
        val securityContext: NetworkSecurityContext = NetworkSecurityContext.IPTV_STREAM,
        val creationTimeMs: Long = System.currentTimeMillis()
    )

    private val streamContextRegistry = ConcurrentHashMap<String, StreamContext>()

    fun clearStreamContexts() {
        streamContextRegistry.clear()
    }

    fun registerStreamContext(mediaId: String, context: StreamContext) {
        streamContextRegistry[mediaId] = context
    }

    fun unregisterStreamContext(mediaId: String) {
        streamContextRegistry.remove(mediaId)
    }

    private fun directoryPrefix(url: String): String {
        val noQuery = url.substringBefore('?').substringBefore('#')
        val lastSlash = noQuery.lastIndexOf('/')
        return if (lastSlash > "https://".length) noQuery.substring(0, lastSlash + 1) else noQuery
    }

    private fun originKey(url: String): String? {
        val parsed = url.toHttpUrlOrNull() ?: return null
        return "${parsed.scheme.lowercase()}://${parsed.host.lowercase()}:${parsed.port}"
    }

    private fun findDeterministicContext(requestUrl: String): StreamContext? {
        streamContextRegistry[requestUrl]?.let { return it }

        val cleanRequestUrl = requestUrl.substringBefore('?').substringBefore('#')
        val prefixMatches = streamContextRegistry.entries
            .filter { (key, ctx) ->
                val keyPrefix = directoryPrefix(key)
                val overridePrefix = ctx.resolvedUrlOverride?.let(::directoryPrefix)
                (keyPrefix.isNotBlank() && (requestUrl.startsWith(keyPrefix) || cleanRequestUrl.startsWith(keyPrefix))) ||
                    (!overridePrefix.isNullOrBlank() && (requestUrl.startsWith(overridePrefix) || cleanRequestUrl.startsWith(overridePrefix)))
            }
            .sortedByDescending { it.key.length }

        if (prefixMatches.isNotEmpty()) {
            return prefixMatches.maxByOrNull { it.value.creationTimeMs }?.value
        }

        val reqOrigin = originKey(requestUrl)
        if (reqOrigin != null) {
            val originMatches = streamContextRegistry.entries.filter { (key, ctx) ->
                originKey(key) == reqOrigin || (ctx.resolvedUrlOverride != null && originKey(ctx.resolvedUrlOverride) == reqOrigin)
            }
            if (originMatches.isNotEmpty()) {
                return originMatches.maxByOrNull { it.value.creationTimeMs }?.value
            }
        }

        return null
    }

    private fun resolveSecurityContextForUrl(url: String): NetworkSecurityContext {
        findDeterministicContext(url)?.let { return it.securityContext }
        return NetworkSecurityValidator.resolveStreamContext(url)
    }

    class DynamicStreamInterceptor : Interceptor {
        override fun intercept(chain: Interceptor.Chain): Response {
            val originalRequest = chain.request()
            var requestUrl = originalRequest.url.toString()
            val requestBuilder = originalRequest.newBuilder()

            val matchedContext = findDeterministicContext(requestUrl)

            if (matchedContext != null) {
                if (matchedContext.headers.xtreamUser != null && matchedContext.headers.xtreamPass != null) {
                    if (requestUrl.contains("/X-AUTH-INJECT/")) {
                        val encodedUser = Uri.encode(matchedContext.headers.xtreamUser)
                        val encodedPass = Uri.encode(matchedContext.headers.xtreamPass)
                        val rewrittenUrl = requestUrl.replace("/X-AUTH-INJECT/", "/$encodedUser/$encodedPass/")
                        requestUrl = rewrittenUrl
                        requestBuilder.url(rewrittenUrl)
                        streamContextRegistry[rewrittenUrl] = matchedContext.copy(resolvedUrlOverride = rewrittenUrl)
                    }
                } else if (matchedContext.resolvedUrlOverride != null && requestUrl.startsWith("xtream://")) {
                    requestBuilder.url(matchedContext.resolvedUrlOverride)
                }

                val ua = NetworkSecurityValidator.sanitizeHeaderValue(matchedContext.headers.userAgent?.takeIf { it.isNotBlank() })
                    ?: DEFAULT_USER_AGENT
                if (originalRequest.header("User-Agent") == null) requestBuilder.header("User-Agent", ua)

                NetworkSecurityValidator.sanitizeHeaderValue(matchedContext.headers.referer)?.takeIf { it.isNotBlank() }?.let {
                    if (originalRequest.header("Referer") == null) requestBuilder.header("Referer", it)
                }
                NetworkSecurityValidator.sanitizeHeaderValue(matchedContext.headers.cookie)?.takeIf { it.isNotBlank() }?.let {
                    if (originalRequest.header("Cookie") == null) requestBuilder.header("Cookie", it)
                }
                NetworkSecurityValidator.sanitizeHeaderValue(matchedContext.headers.authorization)?.takeIf { it.isNotBlank() }?.let {
                    if (originalRequest.header("Authorization") == null) requestBuilder.header("Authorization", it)
                }
            } else {
                if (originalRequest.header("User-Agent") == null) {
                    requestBuilder.header("User-Agent", DEFAULT_USER_AGENT)
                }
            }

            val response = chain.proceed(requestBuilder.build())
            val finalUrl = response.request.url.toString()
            if (matchedContext != null && finalUrl != requestUrl && !streamContextRegistry.containsKey(finalUrl)) {
                val sameOrigin = originKey(finalUrl) == originKey(requestUrl)
                val preservedHeaders = if (sameOrigin) {
                    matchedContext.headers
                } else {
                    matchedContext.headers.copy(cookie = null, authorization = null, xtreamUser = null, xtreamPass = null)
                }
                streamContextRegistry[finalUrl] = matchedContext.copy(headers = preservedHeaders, resolvedUrlOverride = finalUrl)
            }
            return response
        }
    }

    fun buildFactory(
        context: Context,
        diagnosticsListener: okhttp3.EventListener.Factory? = null
    ): DataSource.Factory {
        val baseOkHttp: OkHttpClient = OkHttpStreamingClient
            .buildDynamicClient(DynamicStreamInterceptor(), diagnosticsListener)
            .newBuilder()
            .dns(NetworkSecurityValidator.ContextDispatchingDns(NetworkSecurityContext.IPTV_STREAM))
            .addInterceptor(
                NetworkSecurityValidator.RedirectGuardInterceptor { url ->
                    resolveSecurityContextForUrl(url.toString())
                }
            )
            .followRedirects(false)
            .followSslRedirects(false)
            .build()
        val okHttpFactory = OkHttpDataSource.Factory(baseOkHttp)
        return DefaultDataSource.Factory(context, okHttpFactory)
    }
}
""",

    "player/PlaybackRecoveryManager.kt": """package vid.mat.app.player

import androidx.media3.common.C
import androidx.media3.common.MediaItem
import androidx.media3.common.MimeTypes
import androidx.media3.common.PlaybackException
import androidx.media3.exoplayer.ExoPlayer
import androidx.media3.exoplayer.mediacodec.MediaCodecRenderer
import androidx.media3.exoplayer.mediacodec.MediaCodecUtil
import kotlinx.coroutines.CoroutineScope
import kotlinx.coroutines.Dispatchers
import kotlinx.coroutines.Job
import kotlinx.coroutines.SupervisorJob
import kotlinx.coroutines.cancel
import kotlinx.coroutines.delay
import kotlinx.coroutines.flow.MutableSharedFlow
import kotlinx.coroutines.flow.MutableStateFlow
import kotlinx.coroutines.flow.StateFlow
import kotlinx.coroutines.flow.asStateFlow
import kotlinx.coroutines.flow.first
import kotlinx.coroutines.launch
import kotlinx.coroutines.withContext
import kotlinx.coroutines.withTimeoutOrNull
import vid.mat.app.data.NoResponse
import vid.mat.app.data.ReplaceStream
import vid.mat.app.data.SkipItem
import vid.mat.app.data.StreamExceptionResponse
import vid.mat.app.data.repository.MediaRepository
import vid.mat.app.error.NetworkHealthMonitor
import vid.mat.app.error.PlaybackErrorFactory
import vid.mat.app.error.recovery.SmartRecoveryManager
import vid.mat.app.network.OkHttpDataSourceHelper
import vid.mat.app.player.capability.DecoderFailureType
import vid.mat.app.player.capability.DeviceCodecCapabilityDetector
import vid.mat.app.player.intelligence.NetworkQualitySnapshot
import vid.mat.app.util.AppLogger
import vid.mat.app.util.SensitiveDataSanitizer
import vid.mat.app.util.StreamErrorMapper
import kotlin.random.Random

class PlaybackRecoveryManager(
    private val playerProvider: () -> ExoPlayer?,
    private val liveEdgeController: LiveEdgeController,
    private val failoverEngine: StreamFailoverEngine,
    private val smartRecoveryManager: SmartRecoveryManager,
    private val networkMonitor: NetworkHealthMonitor,
    private val codecDetector: DeviceCodecCapabilityDetector,
    private val errorFlow: MutableSharedFlow<Exception>,
    private val rescueProvider: () -> suspend (
        item: String?,
        mediaItem: MediaItem?,
        exception: Exception,
        repository: MediaRepository
    ) -> StreamExceptionResponse,
    private val repository: MediaRepository,
    private val sessionUpdate: (PlaybackSession) -> Unit = {},
    private val onActiveStreamChanged: (PlaybackSession, MediaItem?) -> Unit = { _, _ -> },
    private val onTerminalFailure: () -> Unit = {}
) {

    private val TAG = "PlaybackRecoveryManager"
    private val scope = CoroutineScope(SupervisorJob() + Dispatchers.Main)

    private val _recoveryState = MutableStateFlow<PlaybackRecoveryState>(PlaybackRecoveryState.Idle)
    val recoveryState: StateFlow<PlaybackRecoveryState> = _recoveryState.asStateFlow()

    @Volatile
    private var activeSession: PlaybackSession? = null

    @Volatile
    private var activeRequestId: PlaybackRequestId = PlaybackRequestId.NONE

    private var activeRetryPolicy: RetryPolicy = RetryPolicy()
    private var autoRetryCount = 0
    private var hasAttemptedDecoderCeilingFallback = false
    private var recoveryJob: Job? = null

    private data class PlaybackSnapshot(
        val positionMs: Long = 0L,
        val liveOffsetMs: Long = C.TIME_UNSET,
        val wasLive: Boolean = false
    )

    companion object {
        const val MAX_AUTO_RETRIES = 3
        const val BASE_RETRY_DELAY_MS = 1_000L
        private const val OFFLINE_WAIT_TIMEOUT_MS = 12_000L
    }

    fun requestProactiveFailover(
        targetCandidateId: String?,
        currentMediaItem: MediaItem?,
        playerWrapper: NewPlayer
    ) {
        val currentSession = activeSession ?: return
        if (_recoveryState.value.isTerminal || recoveryJob?.isActive == true) return
        val currentReqId = activeRequestId

        recoveryJob = scope.launch {
            val next = failoverEngine.selectNextCandidate(
                channelKey = currentSession.mediaId,
                failedCandidateId = currentSession.endpointId ?: currentSession.mediaId,
                failureReason = StreamFailureReason.BUFFER_STARVATION,
                networkQuality = (playerWrapper as? NewPlayerImpl)?.playbackIntelligence?.networkQualityFlow?.value
                    ?: NetworkQualitySnapshot.UNKNOWN
            ) ?: return@launch

            if (currentReqId != activeRequestId || currentSession.sessionId != activeSession?.sessionId) return@launch
            val savedPos = withContext(Dispatchers.Main) { playerProvider()?.currentPosition?.coerceAtLeast(0L) ?: 0L }
            withContext(Dispatchers.Main) {
                val player = playerProvider() ?: return@withContext
                val builder = (currentMediaItem ?: player.currentMediaItem)?.buildUpon()
                    ?: MediaItem.Builder().setMediaId(currentSession.mediaId)
                val newItem = builder.setMediaId(currentSession.mediaId).setUri(next.uri).build()
                OkHttpDataSourceHelper.registerStreamContext(
                    next.uri.toString(),
                    OkHttpDataSourceHelper.StreamContext(headers = next.headers)
                )
                val updated = currentSession.copy(endpointId = next.endpointId, resolvedUri = next.uri, resumePosition = savedPos)
                activeSession = updated
                sessionUpdate(updated)
                onActiveStreamChanged(updated, newItem)
                player.setMediaItem(newItem)
                player.prepare()
                if (currentSession.playbackType != PlaybackType.LIVE && savedPos > 0L) player.seekTo(savedPos)
                player.play()
            }
        }
    }

    private fun extractDecoderFailure(
        error: PlaybackException,
        fallbackMime: String,
        fallbackWidth: Int,
        fallbackHeight: Int
    ): DecoderFailureInfo {
        var cursor: Throwable? = error
        var decoderInitFound = false
        var decoderQueryFound = false
        while (cursor != null) {
            when (cursor) {
                is MediaCodecRenderer.DecoderInitializationException -> decoderInitFound = true
                is MediaCodecUtil.DecoderQueryException -> decoderQueryFound = true
            }
            cursor = cursor.cause
        }
        return DecoderFailureInfo(
            mimeType = fallbackMime,
            width = fallbackWidth,
            height = fallbackHeight,
            initializationExceptionFound = decoderInitFound,
            queryExceptionFound = decoderQueryFound
        )
    }

    private data class DecoderFailureInfo(
        val mimeType: String,
        val width: Int,
        val height: Int,
        val initializationExceptionFound: Boolean,
        val queryExceptionFound: Boolean
    )

    fun startSession(
        session: PlaybackSession,
        requestId: PlaybackRequestId = PlaybackRequestId(session.requestId),
        retryPolicy: RetryPolicy = RetryPolicy(MAX_AUTO_RETRIES, BASE_RETRY_DELAY_MS)
    ): PlaybackSession {
        cancel()
        activeSession = session
        activeRequestId = requestId
        activeRetryPolicy = retryPolicy
        hasAttemptedDecoderCeilingFallback = false
        _recoveryState.value = PlaybackRecoveryState.Preparing(session)
        AppLogger.d(TAG, "Initialized recovery session: ${session.sessionId} (Request: ${requestId.value}) for ${SensitiveDataSanitizer.sanitizeUrl(session.mediaId)}")
        return session
    }

    fun startSession(item: String): PlaybackSession {
        return startSession(PlaybackSession(mediaId = item))
    }

    fun getActiveSession(): PlaybackSession? = activeSession

    fun onPlaybackReady(session: PlaybackSession) {
        if (session.sessionId != activeSession?.sessionId) return
        autoRetryCount = 0
        smartRecoveryManager.dismiss()
        _recoveryState.value = PlaybackRecoveryState.Playing(session)
        session.endpointId?.let { epId ->
            failoverEngine.recordSuccess(session.mediaId, epId)
        }
    }

    fun onPlaybackPaused(session: PlaybackSession) {
        if (session.sessionId != activeSession?.sessionId) return
        _recoveryState.value = PlaybackRecoveryState.Paused(session)
    }

    fun handle(
        error: PlaybackException,
        currentMediaItem: MediaItem?,
        playerWrapper: NewPlayer
    ) {
        val currentSession = activeSession
        if (currentSession == null) {
            errorFlow.tryEmit(error)
            return
        }
        val currentReqId = activeRequestId
        val rawItem = currentMediaItem?.mediaId ?: currentSession.mediaId

        if (_recoveryState.value.isTerminal) {
            AppLogger.d(TAG, "Ignoring error on terminal session: ${currentSession.sessionId}")
            return
        }

        recoveryJob?.cancel()
        recoveryJob = scope.launch {
            val failureReason = failoverEngine.classifyError(error)
            val httpStatus = StreamErrorMapper.extractHttpStatus(error)
            val playbackSnapshot = withContext(Dispatchers.Main) {
                playerProvider()?.let { player ->
                    PlaybackSnapshot(
                        positionMs = player.currentPosition.coerceAtLeast(0L),
                        liveOffsetMs = player.currentLiveOffset,
                        wasLive = player.isCurrentMediaItemLive
                    )
                } ?: PlaybackSnapshot()
            }
            val savedPositionMs = playbackSnapshot.positionMs
            val savedLiveOffsetMs = playbackSnapshot.liveOffsetMs

            fun isSessionStillCurrent(): Boolean =
                currentReqId == activeRequestId && currentSession.sessionId == activeSession?.sessionId

            suspend fun executeCandidateFailover(candidate: StreamCandidate): Boolean {
                if (!isSessionStillCurrent()) {
                    AppLogger.d(TAG, "Cancelling failover for stale request ${currentReqId.value}")
                    return false
                }
                AppLogger.i(TAG, "Executing seamless candidate failover to: ${candidate.id}")
                return withContext(Dispatchers.Main) {
                    if (!isSessionStillCurrent()) return@withContext false
                    val player = playerProvider() ?: return@withContext false

                    currentSession.resolvedUri?.toString()?.let { oldKey ->
                        if (oldKey != candidate.uri.toString()) {
                            OkHttpDataSourceHelper.unregisterStreamContext(oldKey)
                        }
                    }

                    val streamKey = candidate.uri.toString()
                    OkHttpDataSourceHelper.registerStreamContext(
                        streamKey,
                        OkHttpDataSourceHelper.StreamContext(headers = candidate.headers)
                    )

                    val candidateUri = candidate.uri.toString().lowercase()
                    val candidateMime = when {
                        candidate.protocol.equals("m3u8", ignoreCase = true) || candidateUri.contains(".m3u8") || candidateUri.contains("/hls/") -> MimeTypes.APPLICATION_M3U8
                        candidate.protocol.equals("mpd", ignoreCase = true) || candidateUri.contains(".mpd") || candidateUri.contains("/dash/") -> MimeTypes.APPLICATION_MPD
                        candidate.protocol.equals("ts", ignoreCase = true) || candidateUri.endsWith(".ts") || candidateUri.endsWith(".m2ts") -> MimeTypes.VIDEO_MP2T
                        candidate.protocol.equals("mp4", ignoreCase = true) || candidateUri.endsWith(".mp4") || candidateUri.endsWith(".m4v") -> MimeTypes.VIDEO_MP4
                        candidate.protocol.equals("mkv", ignoreCase = true) || candidateUri.endsWith(".mkv") -> MimeTypes.VIDEO_MATROSKA
                        candidate.protocol.equals("webm", ignoreCase = true) || candidateUri.endsWith(".webm") -> MimeTypes.VIDEO_WEBM
                        else -> null
                    }
                    val builder = (currentMediaItem ?: player.currentMediaItem)?.buildUpon()
                        ?: MediaItem.Builder().setMediaId(currentSession.mediaId)
                    builder.setMediaId(currentSession.mediaId)
                    builder.setUri(candidate.uri)
                    builder.setMimeType(candidateMime)
                    if (savedLiveOffsetMs != C.TIME_UNSET && savedLiveOffsetMs > 0L) {
                        builder.setLiveConfiguration(
                            MediaItem.LiveConfiguration.Builder()
                                .setTargetOffsetMs(savedLiveOffsetMs)
                                .build()
                        )
                    }
                    val newItem = builder.build()

                    val updatedSession = currentSession.copy(
                        endpointId = candidate.endpointId,
                        resolvedUri = candidate.uri,
                        resumePosition = savedPositionMs,
                        liveOffset = if (savedLiveOffsetMs != C.TIME_UNSET) savedLiveOffsetMs else currentSession.liveOffset
                    )
                    activeSession = updatedSession
                    autoRetryCount = 0
                    hasAttemptedDecoderCeilingFallback = false
                    sessionUpdate(updatedSession)
                    _recoveryState.value = PlaybackRecoveryState.Recovering(
                        session = updatedSession,
                        attempt = 1,
                        maxAttempts = 1,
                        delayMs = 0L
                    )

                    player.setMediaItem(newItem)
                    onActiveStreamChanged(updatedSession, playerProvider()?.currentMediaItem)
                    player.prepare()
                    if (currentSession.playbackType == PlaybackType.LIVE) {
                        player.seekToDefaultPosition()
                    } else if (savedPositionMs > 0L) {
                        player.seekTo(savedPositionMs)
                    }
                    player.play()
                    true
                }
            }

            val rescueResponse = runCatching {
                rescueProvider().invoke(rawItem, currentMediaItem, error, repository)
            }.getOrDefault(NoResponse)

            if (!isSessionStillCurrent()) return@launch

            when (rescueResponse) {
                is ReplaceStream -> {
                    val rescuedCandidate = StreamCandidate(
                        endpointId = rescueResponse.newUri.toString(),
                        uri = rescueResponse.newUri
                    )
                    if (executeCandidateFailover(rescuedCandidate)) return@launch
                }
                is SkipItem -> {
                    withContext(Dispatchers.Main) {
                        val player = playerProvider()
                        if (player != null && player.hasNextMediaItem()) {
                            player.seekToNextMediaItem()
                            player.prepare()
                            player.play()
                        }
                    }
                    return@launch
                }
                is NoResponse -> Unit
            }

            if (error.errorCode == PlaybackException.ERROR_CODE_BEHIND_LIVE_WINDOW) {
                if (!isSessionStillCurrent()) return@launch
                AppLogger.w(TAG, "BehindLiveWindowException detected -> resynchronizing to live edge")
                liveEdgeController.recoverFromLiveWindowLoss()
                return@launch
            }

            val isDecoderError = error.errorCode == PlaybackException.ERROR_CODE_DECODER_INIT_FAILED ||
                error.errorCode == PlaybackException.ERROR_CODE_DECODER_QUERY_FAILED ||
                error.errorCode == PlaybackException.ERROR_CODE_DECODING_FAILED ||
                error.errorCode == PlaybackException.ERROR_CODE_DECODING_FORMAT_EXCEEDS_CAPABILITIES ||
                error.errorCode == PlaybackException.ERROR_CODE_DECODING_FORMAT_UNSUPPORTED

            if (isDecoderError) {
                val videoFormat = playerProvider()?.videoFormat
                val audioFormat = playerProvider()?.audioFormat
                val decoderInfo = extractDecoderFailure(
                    error = error,
                    fallbackMime = videoFormat?.sampleMimeType ?: audioFormat?.sampleMimeType ?: "video/unknown",
                    fallbackWidth = videoFormat?.width ?: 0,
                    fallbackHeight = videoFormat?.height ?: 0
                )
                val failingMime = decoderInfo.mimeType
                val width = decoderInfo.width
                val height = decoderInfo.height

                AppLogger.w(
                    TAG,
                    "Decoder failure on $failingMime (${width}x${height}) " +
                        "initCause=${decoderInfo.initializationExceptionFound}, " +
                        "queryCause=${decoderInfo.queryExceptionFound}"
                )

                val failureType = if (error.errorCode == PlaybackException.ERROR_CODE_DECODING_FORMAT_EXCEEDS_CAPABILITIES) {
                    DecoderFailureType.CAPABILITY_EXCEEDED
                } else {
                    DecoderFailureType.INITIALIZATION_FAILED
                }
                codecDetector.recordRuntimeDecoderFailure(failingMime, width, height, failureType)

                if (height > 720 && !hasAttemptedDecoderCeilingFallback) {
                    hasAttemptedDecoderCeilingFallback = true
                    val constrained = withContext(Dispatchers.Main) {
                        val player = playerProvider() ?: return@withContext false
                        val impl = playerWrapper as? NewPlayerImpl
                        if (impl != null) {
                            impl.playbackIntelligence.abrController.setHardwareCeiling(720, player)
                        } else {
                            player.trackSelectionParameters = player.trackSelectionParameters.buildUpon()
                                .setMaxVideoSize(1280, 720)
                                .build()
                        }
                        player.prepare()
                        player.play()
                        true
                    }
                    if (constrained) {
                        AppLogger.i(TAG, "Decoder fallback: Applied one-time 720p hardware ceiling")
                        return@launch
                    }
                }

                val alternateCandidate = failoverEngine.selectNextCandidate(
                    channelKey = rawItem,
                    failedCandidateId = currentSession.endpointId ?: rawItem,
                    failureReason = StreamFailureReason.DECODER_FAILURE,
                    networkQuality = (playerWrapper as? NewPlayerImpl)?.playbackIntelligence?.networkQualityFlow?.value
                        ?: NetworkQualitySnapshot.UNKNOWN
                )
                if (alternateCandidate != null && executeCandidateFailover(alternateCandidate)) {
                    AppLogger.i(TAG, "Decoder fallback: Switched to compatible candidate ${alternateCandidate.id}")
                    return@launch
                }
            }

            if (!networkMonitor.isConnected && !isDecoderError) {
                _recoveryState.value = PlaybackRecoveryState.Recovering(
                    session = currentSession,
                    attempt = 1,
                    maxAttempts = 1,
                    delayMs = OFFLINE_WAIT_TIMEOUT_MS
                )
                val reconnected = withTimeoutOrNull(OFFLINE_WAIT_TIMEOUT_MS) {
                    networkMonitor.networkState.first {
                        it == NetworkHealthMonitor.NetworkState.CONNECTED ||
                            it == NetworkHealthMonitor.NetworkState.METERED
                    }
                } != null
                if (!isSessionStillCurrent()) return@launch
                if (reconnected) {
                    withContext(Dispatchers.Main) {
                        playerProvider()?.let { player ->
                            player.prepare()
                            if (currentSession.playbackType == PlaybackType.LIVE) {
                                player.seekToDefaultPosition()
                            } else if (savedPositionMs > 0L) {
                                player.seekTo(savedPositionMs)
                            }
                            player.play()
                        }
                    }
                    return@launch
                }
            }

            val maxRetries = activeRetryPolicy.maxRetries.coerceAtLeast(0)
            val baseDelayMs = activeRetryPolicy.baseDelayMs.coerceAtLeast(250L)
            if (failureReason.isRetryableInPlace && autoRetryCount < maxRetries && !isDecoderError && networkMonitor.isConnected) {
                autoRetryCount++
                val nextAttempt = autoRetryCount
                val backoffMs = baseDelayMs * (1 shl (nextAttempt - 1))
                val jitterMs = Random.nextLong(100, 400)
                val totalDelay = backoffMs + jitterMs

                _recoveryState.value = PlaybackRecoveryState.Recovering(
                    session = currentSession,
                    attempt = nextAttempt,
                    maxAttempts = maxRetries,
                    delayMs = totalDelay
                )

                AppLogger.d(TAG, "Reconnecting attempt $nextAttempt/$maxRetries in ${totalDelay}ms")
                delay(totalDelay)

                if (!isSessionStillCurrent()) {
                    AppLogger.d(TAG, "Aborting retry for cancelled request ${currentReqId.value}")
                    return@launch
                }

                withContext(Dispatchers.Main) {
                    if (!isSessionStillCurrent()) return@withContext
                    playerProvider()?.let { player ->
                        player.prepare()
                        if (currentSession.playbackType == PlaybackType.LIVE) {
                            player.seekToDefaultPosition()
                        } else if (savedPositionMs > 0L) {
                            player.seekTo(savedPositionMs)
                        }
                        player.play()
                    }
                }
                return@launch
            }

            autoRetryCount = 0
            if (!isSessionStillCurrent()) return@launch

            val typedError = PlaybackErrorFactory.from(error, httpStatus)
            _recoveryState.value = PlaybackRecoveryState.Recovering(
                session = currentSession,
                attempt = maxRetries,
                maxAttempts = maxRetries,
                delayMs = 0L
            )

            try {
                smartRecoveryManager.handlePlaybackFault(
                    error = error,
                    channelKey = rawItem,
                    currentCandidateId = currentSession.endpointId ?: rawItem,
                    networkMonitor = networkMonitor,
                    player = playerWrapper,
                    onAlternateElected = { candidate ->
                        scope.launch {
                            val switched = executeCandidateFailover(candidate)
                            if (!switched && isSessionStillCurrent()) {
                                publishTerminalFailure(currentSession, typedError)
                            }
                        }
                    },
                    onTerminalFailure = {
                        scope.launch {
                            if (isSessionStillCurrent()) {
                                publishTerminalFailure(currentSession, typedError)
                            }
                        }
                    }
                )
            } catch (recoveryError: Exception) {
                AppLogger.e(TAG, "Smart recovery failed; publishing terminal playback error", recoveryError)
                if (isSessionStillCurrent()) {
                    publishTerminalFailure(currentSession, typedError)
                }
            }
        }
    }

    private suspend fun publishTerminalFailure(
        session: PlaybackSession,
        typedError: vid.mat.app.error.PlaybackError
    ) {
        onTerminalFailure()
        _recoveryState.value = PlaybackRecoveryState.RecoveryFailed(session, typedError)
        errorFlow.emit(typedError)
    }

    fun cancel() {
        val current = activeSession
        if (current != null) {
            _recoveryState.value = PlaybackRecoveryState.Terminated(current)
        } else {
            _recoveryState.value = PlaybackRecoveryState.Idle
        }
        activeSession = null
        activeRequestId = PlaybackRequestId.NONE
        autoRetryCount = 0
        hasAttemptedDecoderCeilingFallback = false
        recoveryJob?.cancel()
        recoveryJob = null
        smartRecoveryManager.dismiss()
    }

    fun destroy() {
        cancel()
        scope.cancel()
    }
}
""",
}


def main() -> None:
    if not BASE_DIR.exists():
        raise SystemExit(f"Target directory does not exist: {BASE_DIR}")

    for rel_path, content in FILES.items():
        target = BASE_DIR / rel_path
        target.parent.mkdir(parents=True, exist_ok=True)
        target.write_text(content, encoding="utf-8")
        print(f"Updated: {target}")

    print(f"Done. Wrote {len(FILES)} files.")


if __name__ == "__main__":
    main()
```

May I proceed with **Compile-Fix Batch 2 (5 files)**: `data/intelligence/PlaylistIntelligenceEngine.kt`, `dvr/engine/hls/HlsDvrPipeline.kt`, `dvr/ui/ScheduleRecordingDialogFragment.kt`, `player/abr/IntelligentAbrController.kt`, `playlist/M3uParser.kt`?

I did not generate a response in the previous turn. Let me output **Compile-Fix Batch 1 (5 files)** now:

1. `catchup/UnifiedCatchupEngine.kt` — Adds `xtreamDateFormat` in `companion object`.
2. `di/AppModule.kt` — Fixes `provideStreamResolver(...)` argument order (`okHttpClient, unifiedCatchupEngine`).
3. `error/recovery/SmartRecoveryManager.kt` — Adds missing `import vid.mat.app.player.StreamFailureReason`.
4. `network/OkHttpDataSourceHelper.kt` — Closes `DynamicStreamInterceptor` before `buildFactory(...)`.
5. `player/PlaybackRecoveryManager.kt` — Fixes `onActiveStreamChanged` default lambda (`= { _, _ -> }`).

```python
#!/usr/bin/env python3
from pathlib import Path

BASE_DIR = Path("/home/username/AndroidStudioProjects/Ytvmate/app/src/main/java/vid/mat/app")

FILES = {
    "catchup/UnifiedCatchupEngine.kt": """package vid.mat.app.catchup

import android.net.Uri
import vid.mat.app.data.dao.ChannelDao
import vid.mat.app.data.dao.PlaylistSourceDao
import vid.mat.app.data.dao.StreamEndpointDao
import vid.mat.app.data.model.CatchupMetadata
import vid.mat.app.data.model.CatchupType
import vid.mat.app.data.model.StreamEndpoint
import vid.mat.app.epg.model.EpgProgram
import vid.mat.app.provider.StreamDescriptor
import vid.mat.app.security.XtreamAccountManager
import java.text.SimpleDateFormat
import java.util.Date
import java.util.Locale
import java.util.TimeZone
import javax.inject.Inject
import javax.inject.Singleton

@Singleton
class UnifiedCatchupEngine @Inject constructor(
    private val channelDao: ChannelDao,
    private val endpointDao: StreamEndpointDao,
    private val sourceDao: PlaylistSourceDao,
    private val xtreamAccountManager: XtreamAccountManager
) {
    companion object {
        private const val TAG = "UnifiedCatchupEngine"
        private const val MS_PER_DAY = 86_400_000L
        private val xtreamDateFormat = SimpleDateFormat("yyyy-MM-dd:HH-mm", Locale.US).apply {
            timeZone = TimeZone.getTimeZone("UTC")
        }
    }

    suspend fun resolveCatchupDescriptor(
        channelId: Long,
        startMs: Long,
        stopMs: Long
    ): Result<StreamDescriptor> {
        if (channelId <= 0L || startMs <= 0L || stopMs <= startMs) {
            return Result.failure(IllegalStateException("Catch-up is not available for this time range"))
        }

        val channel = channelDao.getById(channelId)
            ?: return Result.failure(IllegalStateException("Channel not found"))

        val nowMs = System.currentTimeMillis()
        val ageMs = nowMs - stopMs
        val retentionMs = channel.catchupDays.toLong() * MS_PER_DAY
        if (channel.catchupDays <= 0 || ageMs >= retentionMs) {
            return Result.failure(IllegalStateException("Catch-up is not available or has expired for this program"))
        }

        val durationSec = (stopMs - startMs) / 1000L
        if (durationSec <= 0L) {
            return Result.failure(IllegalStateException("Invalid program duration for catch-up"))
        }

        val endpoints = endpointDao.getEndpointsForChannel(channelId)
            .sortedWith(
                compareByDescending<StreamEndpoint> { it.isWorking }
                    .thenByDescending { it.priority }
                    .thenBy { it.failureCount }
            )
        val primaryEndpoint = endpoints.firstOrNull()
            ?: return Result.failure(IllegalStateException("No endpoints available for channel '${channel.title}'"))

        val source = sourceDao.getById(channel.sourceId)
        val accountId = source?.xtreamAccountId
        if (accountId != null) {
            return when (val creds = xtreamAccountManager.resolveCredentials(accountId)) {
                is XtreamAccountManager.CredentialsResolution.Success -> {
                    val formattedDate = synchronized(xtreamDateFormat) { xtreamDateFormat.format(Date(startMs)) }
                    val durationMin = (durationSec / 60L).coerceAtLeast(1L)
                    val cleanServer = creds.serverUrl.trimEnd('/')
                    val streamId = channel.externalId?.takeIf { it.isNotBlank() }
                        ?: primaryEndpoint.url.substringAfterLast('/').substringBefore('.')
                            .takeIf { it.isNotBlank() }
                        ?: return Result.failure(IllegalStateException("Missing stream ID for Xtream catch-up"))

                    val uri = "$cleanServer/timeshift/X-AUTH-INJECT/$durationMin/$formattedDate/$streamId.m3u8"
                    Result.success(
                        StreamDescriptor(
                            streamUri = uri,
                            containerFormat = "m3u8",
                            headers = buildMap {
                                put("User-Agent", primaryEndpoint.userAgent ?: "IPTVSmartersPlayer")
                                primaryEndpoint.referer?.value?.let { put("Referer", it) }
                                primaryEndpoint.cookie?.value?.let { put("Cookie", it) }
                                primaryEndpoint.authorization?.value?.let { put("Authorization", it) }
                            },
                            userAgent = primaryEndpoint.userAgent ?: "IPTVSmartersPlayer",
                            isLive = false,
                            xtreamUser = creds.username,
                            xtreamPass = creds.pass
                        )
                    )
                }
                XtreamAccountManager.CredentialsResolution.NeedsReauthentication ->
                    Result.failure(IllegalStateException("Xtream credentials require re-authentication"))
                is XtreamAccountManager.CredentialsResolution.Error ->
                    Result.failure(IllegalStateException(creds.message))
            }
        }

        val inferredType = inferCatchupType(primaryEndpoint.url, channel.catchupSource)
        val metadata = CatchupMetadata(
            channelId = channelId,
            catchupType = inferredType,
            days = channel.catchupDays,
            sourceTemplate = channel.catchupSource
        )
        val resolvedUrl = CatchupUrlBuilder.buildCatchupUrl(
            streamUrl = primaryEndpoint.url,
            metadata = metadata,
            startEpochMs = startMs,
            durationSeconds = durationSec
        )

        return Result.success(
            StreamDescriptor(
                streamUri = resolvedUrl,
                containerFormat = if (resolvedUrl.contains(".ts", ignoreCase = true)) "ts" else "m3u8",
                headers = buildMap {
                    primaryEndpoint.userAgent?.let { put("User-Agent", it) }
                    primaryEndpoint.referer?.value?.let { put("Referer", it) }
                    primaryEndpoint.cookie?.value?.let { put("Cookie", it) }
                    primaryEndpoint.authorization?.value?.let { put("Authorization", it) }
                },
                userAgent = primaryEndpoint.userAgent,
                isLive = false
            )
        )
    }

    suspend fun resolveCatchupStream(channelId: Long, program: EpgProgram): String? {
        val descriptor = resolveCatchupDescriptor(channelId, program.startMs, program.stopMs).getOrNull()
            ?: return null
        return if (!descriptor.xtreamUser.isNullOrBlank() && !descriptor.xtreamPass.isNullOrBlank()) {
            descriptor.streamUri.replace(
                "/X-AUTH-INJECT/",
                "/${Uri.encode(descriptor.xtreamUser)}/${Uri.encode(descriptor.xtreamPass)}/"
            )
        } else {
            descriptor.streamUri
        }
    }

    private fun inferCatchupType(streamUrl: String, sourceTemplate: String?): CatchupType {
        val tmpl = sourceTemplate?.lowercase().orEmpty()
        return when {
            tmpl.contains("timeshift_abs") || tmpl.contains("video-") || streamUrl.contains("flussonic", ignoreCase = true) ->
                CatchupType.FLUSSONIC
            tmpl.contains("timeshift=") -> CatchupType.SHIFT
            tmpl.isNotBlank() -> CatchupType.APPEND
            else -> CatchupType.DEFAULT
        }
    }
}
""",

    "di/AppModule.kt": """package vid.mat.app.di

import android.content.Context
import dagger.Module
import dagger.Provides
import dagger.hilt.InstallIn
import dagger.hilt.android.qualifiers.ApplicationContext
import dagger.hilt.components.SingletonComponent
import okhttp3.OkHttpClient
import vid.mat.app.ads.promo.PromoAccountingDao
import vid.mat.app.catchup.UnifiedCatchupEngine
import vid.mat.app.data.dao.*
import vid.mat.app.data.database.VideoUrlDatabase
import vid.mat.app.data.repository.*
import vid.mat.app.dvr.dao.DvrDao
import vid.mat.app.epg.EpgAlignmentEngine
import vid.mat.app.epg.EpgDao
import vid.mat.app.epg.EpgRepository
import vid.mat.app.favorites.ProfileFavoriteSynchronizer
import vid.mat.app.favorites.UnifiedFavoriteRepository
import vid.mat.app.favorites.dao.UnifiedFavoriteDao
import vid.mat.app.history.WatchHistoryDao
import vid.mat.app.history.WatchHistoryRepository
import vid.mat.app.network.OkHttpStreamingClient
import vid.mat.app.player.NewPlayer
import vid.mat.app.player.NewPlayerImpl
import vid.mat.app.player.StreamFailoverEngine
import vid.mat.app.player.abr.IntelligentAbrController
import vid.mat.app.player.capability.DeviceCodecCapabilityDetector
import vid.mat.app.player.intelligence.BandwidthEstimator
import vid.mat.app.player.intelligence.PlaybackIntelligence
import vid.mat.app.player.intelligence.StreamHealthStore
import vid.mat.app.player.performance.StreamPerformanceMonitor
import vid.mat.app.player.quality.SmartStreamSelector
import vid.mat.app.player.sync.AudioDelayAudioProcessor
import vid.mat.app.player.sync.PlayerSynchronizationController
import vid.mat.app.player.sync.PlayerSynchronizationControllerImpl
import vid.mat.app.profile.ActiveProfileProvider
import vid.mat.app.profile.dao.ProfileDao
import vid.mat.app.provider.IptvProviderFactory
import vid.mat.app.provider.StreamResolver
import vid.mat.app.security.ParentalControlManager
import vid.mat.app.security.ParentalGatekeeper
import vid.mat.app.security.XtreamAccountManager
import javax.inject.Singleton

@Module
@InstallIn(SingletonComponent::class)
object AppModule {
    @Provides
    @Singleton
    fun provideDatabase(@ApplicationContext context: Context): VideoUrlDatabase =
        VideoUrlDatabase.getInstance(context)

    @Provides fun provideVideoUrlDao(db: VideoUrlDatabase): VideoUrlDao = db.videoUrlDao()
    @Provides fun providePlaylistSourceDao(db: VideoUrlDatabase): PlaylistSourceDao = db.playlistSourceDao()
    @Provides fun provideChannelDao(db: VideoUrlDatabase): ChannelDao = db.channelDao()
    @Provides fun provideStreamEndpointDao(db: VideoUrlDatabase): StreamEndpointDao = db.streamEndpointDao()
    @Provides fun provideEpgDao(db: VideoUrlDatabase): EpgDao = db.epgDao()
    @Provides fun provideWatchHistoryDao(db: VideoUrlDatabase): WatchHistoryDao = db.watchHistoryDao()
    @Provides fun provideXtreamAccountDao(db: VideoUrlDatabase): XtreamAccountDao = db.xtreamAccountDao()
    @Provides fun provideDvrDao(db: VideoUrlDatabase): DvrDao = db.dvrDao()
    @Provides fun provideProfileDao(db: VideoUrlDatabase): ProfileDao = db.profileDao()
    @Provides fun provideChannelUserOverrideDao(db: VideoUrlDatabase): ChannelUserOverrideDao = db.channelUserOverrideDao()
    @Provides fun provideCustomGroupDao(db: VideoUrlDatabase): CustomGroupDao = db.customGroupDao()
    @Provides fun provideWatchProgressDao(db: VideoUrlDatabase): WatchProgressDao = db.watchProgressDao()
    @Provides fun provideUnifiedFavoriteDao(db: VideoUrlDatabase): UnifiedFavoriteDao = db.unifiedFavoriteDao()
    @Provides fun provideVodMovieDao(db: VideoUrlDatabase): VodMovieDao = db.vodMovieDao()
    @Provides fun provideSeriesDao(db: VideoUrlDatabase): SeriesDao = db.seriesDao()
    @Provides fun provideSearchDao(db: VideoUrlDatabase): SearchDao = db.searchDao()
    @Provides fun provideUnifiedCatalogDao(db: VideoUrlDatabase): UnifiedCatalogDao = db.unifiedCatalogDao()
    @Provides fun providePromoAccountingDao(db: VideoUrlDatabase): PromoAccountingDao = db.promoAccountingDao()

    @Provides
    @Singleton
    fun provideOkHttpClient(): OkHttpClient = OkHttpStreamingClient.epgClient

    @Provides
    @Singleton
    fun provideVideoUrlRepository(dao: VideoUrlDao, favoriteRepository: UnifiedFavoriteRepository): VideoUrlRepository =
        VideoUrlRepository(dao, favoriteRepository)

    @Provides
    @Singleton
    fun provideChannelRepository(channelDao: ChannelDao, endpointDao: StreamEndpointDao, favoriteRepository: UnifiedFavoriteRepository): ChannelRepository =
        ChannelRepository(channelDao, endpointDao, favoriteRepository)

    @Provides
    @Singleton
    fun providePlaylistSourceRepository(sourceDao: PlaylistSourceDao): PlaylistSourceRepository =
        PlaylistSourceRepository(sourceDao)

    @Provides
    @Singleton
    fun provideEpgRepository(dao: EpgDao, okHttpClient: OkHttpClient): EpgRepository =
        EpgRepository(dao, okHttpClient)

    @Provides
    @Singleton
    fun provideWatchHistoryRepository(
        dao: WatchHistoryDao,
        activeProfileProvider: ActiveProfileProvider,
        @ApplicationContext context: Context
    ): WatchHistoryRepository =
        WatchHistoryRepository(dao, activeProfileProvider, context)

    @Provides
    @Singleton
    fun provideXtreamAccountManager(dao: XtreamAccountDao): XtreamAccountManager =
        XtreamAccountManager(dao)

    @Provides
    @Singleton
    fun provideParentalControlManager(@ApplicationContext context: Context): ParentalControlManager =
        ParentalControlManager(context)

    @Provides
    @Singleton
    fun provideParentalGatekeeper(
        parentalControlManager: ParentalControlManager,
        activeProfileProvider: ActiveProfileProvider
    ): ParentalGatekeeper = ParentalGatekeeper(parentalControlManager, activeProfileProvider)

    @Provides
    @Singleton
    fun provideUnifiedFavoriteRepository(
        favoriteDao: UnifiedFavoriteDao,
        activeProfileProvider: ActiveProfileProvider,
        parentalGatekeeper: ParentalGatekeeper,
        synchronizer: ProfileFavoriteSynchronizer
    ): UnifiedFavoriteRepository =
        UnifiedFavoriteRepository(favoriteDao, activeProfileProvider, parentalGatekeeper, synchronizer)

    @Provides
    @Singleton
    fun provideAudioDelayAudioProcessor(): AudioDelayAudioProcessor = AudioDelayAudioProcessor()

    @Provides
    @Singleton
    fun providePlayerSynchronizationController(
        @ApplicationContext context: Context,
        audioDelayProcessor: AudioDelayAudioProcessor
    ): PlayerSynchronizationController =
        PlayerSynchronizationControllerImpl(context, audioDelayProcessor)

    @Provides
    @Singleton
    fun provideDeviceCodecCapabilityDetector(): DeviceCodecCapabilityDetector =
        DeviceCodecCapabilityDetector()

    @Provides
    @Singleton
    fun provideStreamHealthStore(): StreamHealthStore =
        StreamHealthStore()

    @Provides
    @Singleton
    fun provideStreamPerformanceMonitor(): StreamPerformanceMonitor =
        StreamPerformanceMonitor()

    @Provides
    @Singleton
    fun provideBandwidthEstimator(): BandwidthEstimator =
        BandwidthEstimator()

    @Provides
    @Singleton
    fun provideIntelligentAbrController(
        bandwidthEstimator: BandwidthEstimator,
        codecDetector: DeviceCodecCapabilityDetector
    ): IntelligentAbrController =
        IntelligentAbrController(bandwidthEstimator, codecDetector)

    @Provides
    @Singleton
    fun provideSmartStreamSelector(
        codecDetector: DeviceCodecCapabilityDetector,
        healthStore: StreamHealthStore,
        bandwidthEstimator: BandwidthEstimator
    ): SmartStreamSelector = SmartStreamSelector(codecDetector, healthStore, bandwidthEstimator)

    @Provides
    @Singleton
    fun providePlaybackIntelligence(
        @ApplicationContext context: Context,
        failoverEngine: StreamFailoverEngine,
        performanceMonitor: StreamPerformanceMonitor,
        codecDetector: DeviceCodecCapabilityDetector,
        healthStore: StreamHealthStore,
        bandwidthEstimator: BandwidthEstimator,
        abrController: IntelligentAbrController
    ): PlaybackIntelligence =
        PlaybackIntelligence(
            context,
            failoverEngine,
            performanceMonitor,
            codecDetector,
            healthStore,
            bandwidthEstimator,
            abrController
        )

    @Provides
    @Singleton
    fun provideIptvProviderFactory(
        @ApplicationContext context: Context,
        httpClient: OkHttpClient,
        xtreamAccountManager: XtreamAccountManager
    ): IptvProviderFactory = IptvProviderFactory(context, httpClient, xtreamAccountManager)

    @Provides
    @Singleton
    fun provideUnifiedCatchupEngine(
        channelDao: ChannelDao,
        endpointDao: StreamEndpointDao,
        sourceDao: PlaylistSourceDao,
        xtreamAccountManager: XtreamAccountManager
    ): UnifiedCatchupEngine = UnifiedCatchupEngine(channelDao, endpointDao, sourceDao, xtreamAccountManager)

    @Provides
    @Singleton
    fun provideEpgAlignmentEngine(
        epgDao: EpgDao,
        sourceDao: PlaylistSourceDao
    ): EpgAlignmentEngine = EpgAlignmentEngine(epgDao, sourceDao)

    @Provides
    @Singleton
    fun provideStreamResolver(
        @ApplicationContext context: Context,
        channelDao: ChannelDao,
        endpointDao: StreamEndpointDao,
        sourceDao: PlaylistSourceDao,
        videoUrlDao: VideoUrlDao,
        vodMovieDao: VodMovieDao,
        seriesDao: SeriesDao,
        providerFactory: IptvProviderFactory,
        xtreamAccountManager: XtreamAccountManager,
        parentalGatekeeper: ParentalGatekeeper,
        okHttpClient: OkHttpClient,
        unifiedCatchupEngine: UnifiedCatchupEngine
    ): StreamResolver = StreamResolver(
        context, channelDao, endpointDao, sourceDao, videoUrlDao,
        vodMovieDao, seriesDao, providerFactory, xtreamAccountManager,
        parentalGatekeeper, okHttpClient, unifiedCatchupEngine
    )

    @Provides
    @Singleton
    fun provideMediaRepository(
        channelDao: ChannelDao,
        endpointDao: StreamEndpointDao,
        videoUrlDao: VideoUrlDao
    ): MediaRepository = DefaultMediaRepository(channelDao, endpointDao, videoUrlDao)

    @Provides
    @Singleton
    fun provideNewPlayer(impl: NewPlayerImpl): NewPlayer = impl
}
""",

    "error/recovery/SmartRecoveryManager.kt": """package vid.mat.app.error.recovery

import androidx.media3.common.PlaybackException
import kotlinx.coroutines.CoroutineScope
import kotlinx.coroutines.Dispatchers
import kotlinx.coroutines.Job
import kotlinx.coroutines.SupervisorJob
import kotlinx.coroutines.delay
import kotlinx.coroutines.flow.MutableStateFlow
import kotlinx.coroutines.flow.StateFlow
import kotlinx.coroutines.flow.asStateFlow
import kotlinx.coroutines.launch
import vid.mat.app.error.ErrorClassifier
import vid.mat.app.error.NetworkHealthMonitor
import vid.mat.app.player.NewPlayer
import vid.mat.app.player.StreamCandidate
import vid.mat.app.player.StreamFailoverEngine
import vid.mat.app.player.StreamFailureReason
import javax.inject.Inject
import javax.inject.Singleton

@Singleton
class SmartRecoveryManager @Inject constructor(
    private val failoverEngine: StreamFailoverEngine
) {
    private val scope = CoroutineScope(SupervisorJob() + Dispatchers.Main)
    private var recoveryJob: Job? = null

    private val _recoveryState = MutableStateFlow(PlaybackTroubleshooterState())
    val recoveryState: StateFlow<PlaybackTroubleshooterState> = _recoveryState.asStateFlow()

    fun dismiss() {
        recoveryJob?.cancel()
        recoveryJob = null
        _recoveryState.value = PlaybackTroubleshooterState()
    }

    fun handlePlaybackFault(
        error: PlaybackException,
        channelKey: String,
        currentCandidateId: String,
        networkMonitor: NetworkHealthMonitor,
        player: NewPlayer,
        onAlternateElected: (StreamCandidate) -> Unit,
        onTerminalFailure: (PlaybackTroubleshooterState) -> Unit = {}
    ) {
        recoveryJob?.cancel()
        recoveryJob = scope.launch {
            val classification = ErrorClassifier.classify(error)
            val initialSteps = listOf(
                TroubleshooterStep(RecoveryStepType.CHECK_NETWORK, "Verify Network Connectivity", StepExecutionStatus.IN_PROGRESS),
                TroubleshooterStep(RecoveryStepType.TRY_ALTERNATE_ENDPOINT, "Test Alternate Stream Endpoints", StepExecutionStatus.PENDING),
                TroubleshooterStep(RecoveryStepType.TEST_DECODER_FALLBACK, "Check Hardware Codec Compatibility", StepExecutionStatus.PENDING),
                TroubleshooterStep(RecoveryStepType.RETRY_STREAM_CONNECTION, "Re-establish Stream Handshake", StepExecutionStatus.PENDING)
            )

            _recoveryState.value = PlaybackTroubleshooterState(
                isTroubleshooting = true,
                steps = initialSteps,
                rootCauseSummary = classification.userMessage,
                recommendedAction = "Diagnosing stream failure and evaluating recovery routes...",
                channelKey = channelKey
            )

            delay(350L)

            val isNetworkOk = networkMonitor.isConnected
            updateStep(
                type = RecoveryStepType.CHECK_NETWORK,
                status = if (isNetworkOk) StepExecutionStatus.SUCCESS else StepExecutionStatus.FAILED,
                detail = if (isNetworkOk) "Internet connection online (${networkMonitor.networkType.value.name})" else "No active internet connection"
            )

            if (!isNetworkOk) {
                updateStep(RecoveryStepType.TRY_ALTERNATE_ENDPOINT, StepExecutionStatus.SKIPPED, "Network offline")
                updateStep(RecoveryStepType.TEST_DECODER_FALLBACK, StepExecutionStatus.SKIPPED, "Network offline")
                updateStep(RecoveryStepType.RETRY_STREAM_CONNECTION, StepExecutionStatus.FAILED, "Waiting for network")
                val terminalState = _recoveryState.value.copy(
                    isTroubleshooting = false,
                    isTerminalFailure = true,
                    canForceRetry = false,
                    recommendedAction = "Please check your Wi-Fi or mobile data connection and try again."
                )
                _recoveryState.value = terminalState
                onTerminalFailure(terminalState)
                return@launch
            }

            delay(250L)
            updateStep(RecoveryStepType.TRY_ALTERNATE_ENDPOINT, StepExecutionStatus.IN_PROGRESS)
            val failureReason = failoverEngine.classifyError(error)
            val alternate = failoverEngine.selectNextCandidate(channelKey, currentCandidateId, failureReason)

            if (alternate != null) {
                updateStep(
                    RecoveryStepType.TRY_ALTERNATE_ENDPOINT,
                    StepExecutionStatus.SUCCESS,
                    "Switched to backup source (${alternate.resolutionLabel})"
                )
                updateStep(RecoveryStepType.TEST_DECODER_FALLBACK, StepExecutionStatus.SKIPPED, "Backup source elected")
                updateStep(RecoveryStepType.RETRY_STREAM_CONNECTION, StepExecutionStatus.SUCCESS, "Handshaking backup endpoint")

                _recoveryState.value = _recoveryState.value.copy(
                    isTroubleshooting = false,
                    recommendedAction = "Connecting to alternate stream endpoint..."
                )
                onAlternateElected(alternate)
                delay(900L)
                if (!_recoveryState.value.isTerminalFailure) {
                    _recoveryState.value = PlaybackTroubleshooterState()
                }
            } else {
                updateStep(
                    RecoveryStepType.TRY_ALTERNATE_ENDPOINT,
                    StepExecutionStatus.FAILED,
                    "No working alternate endpoints available"
                )

                val isDecoderFault = failureReason == StreamFailureReason.DECODER_FAILURE ||
                    failureReason == StreamFailureReason.UNSUPPORTED_CODEC
                updateStep(
                    type = RecoveryStepType.TEST_DECODER_FALLBACK,
                    status = if (isDecoderFault) StepExecutionStatus.FAILED else StepExecutionStatus.SUCCESS,
                    detail = if (isDecoderFault) "Device decoder incompatible with stream format" else "Hardware decoder OK"
                )

                updateStep(
                    type = RecoveryStepType.RETRY_STREAM_CONNECTION,
                    status = StepExecutionStatus.FAILED,
                    detail = "Origin stream unreachable after retries"
                )

                val terminalState = _recoveryState.value.copy(
                    isTroubleshooting = false,
                    isTerminalFailure = true,
                    canForceRetry = true,
                    recommendedAction = "All stream sources exhausted. Tap Force Retry to clear cooldowns or Retry Now."
                )
                _recoveryState.value = terminalState
                onTerminalFailure(terminalState)
            }
        }
    }

    private fun updateStep(
        type: RecoveryStepType,
        status: StepExecutionStatus,
        detail: String? = null
    ) {
        val updated = _recoveryState.value.steps.map { step ->
            if (step.type == type) step.copy(status = status, detailMessage = detail ?: step.detailMessage)
            else step
        }
        _recoveryState.value = _recoveryState.value.copy(steps = updated)
    }
}
""",

    "network/OkHttpDataSourceHelper.kt": """package vid.mat.app.network

import android.content.Context
import android.net.Uri
import androidx.annotation.OptIn
import androidx.media3.common.util.UnstableApi
import androidx.media3.datasource.DataSource
import androidx.media3.datasource.DefaultDataSource
import androidx.media3.datasource.okhttp.OkHttpDataSource
import okhttp3.HttpUrl.Companion.toHttpUrlOrNull
import okhttp3.Interceptor
import okhttp3.OkHttpClient
import okhttp3.Response
import vid.mat.app.security.NetworkSecurityContext
import vid.mat.app.security.NetworkSecurityValidator
import java.util.concurrent.ConcurrentHashMap

@OptIn(UnstableApi::class)
object OkHttpDataSourceHelper {

    private const val DEFAULT_USER_AGENT =
        "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/120.0.0.0 Safari/537.36"

    data class StreamContext(
        val headers: OkHttpStreamingClient.StreamHeaders = OkHttpStreamingClient.StreamHeaders(),
        val resolvedUrlOverride: String? = null,
        val host: String? = null,
        val securityContext: NetworkSecurityContext = NetworkSecurityContext.IPTV_STREAM,
        val creationTimeMs: Long = System.currentTimeMillis()
    )

    private val streamContextRegistry = ConcurrentHashMap<String, StreamContext>()

    fun clearStreamContexts() {
        streamContextRegistry.clear()
    }

    fun registerStreamContext(mediaId: String, context: StreamContext) {
        streamContextRegistry[mediaId] = context
    }

    fun unregisterStreamContext(mediaId: String) {
        streamContextRegistry.remove(mediaId)
    }

    private fun directoryPrefix(url: String): String {
        val noQuery = url.substringBefore('?').substringBefore('#')
        val lastSlash = noQuery.lastIndexOf('/')
        return if (lastSlash > "https://".length) noQuery.substring(0, lastSlash + 1) else noQuery
    }

    private fun originKey(url: String): String? {
        val parsed = url.toHttpUrlOrNull() ?: return null
        return "${parsed.scheme.lowercase()}://${parsed.host.lowercase()}:${parsed.port}"
    }

    private fun findDeterministicContext(requestUrl: String): StreamContext? {
        streamContextRegistry[requestUrl]?.let { return it }

        val cleanRequestUrl = requestUrl.substringBefore('?').substringBefore('#')
        val prefixMatches = streamContextRegistry.entries
            .filter { (key, ctx) ->
                val keyPrefix = directoryPrefix(key)
                val overridePrefix = ctx.resolvedUrlOverride?.let(::directoryPrefix)
                (keyPrefix.isNotBlank() && (requestUrl.startsWith(keyPrefix) || cleanRequestUrl.startsWith(keyPrefix))) ||
                    (!overridePrefix.isNullOrBlank() && (requestUrl.startsWith(overridePrefix) || cleanRequestUrl.startsWith(overridePrefix)))
            }
            .sortedByDescending { it.key.length }

        if (prefixMatches.isNotEmpty()) {
            return prefixMatches.maxByOrNull { it.value.creationTimeMs }?.value
        }

        val reqOrigin = originKey(requestUrl)
        if (reqOrigin != null) {
            val originMatches = streamContextRegistry.entries.filter { (key, ctx) ->
                originKey(key) == reqOrigin || (ctx.resolvedUrlOverride != null && originKey(ctx.resolvedUrlOverride) == reqOrigin)
            }
            if (originMatches.isNotEmpty()) {
                return originMatches.maxByOrNull { it.value.creationTimeMs }?.value
            }
        }

        return null
    }

    private fun resolveSecurityContextForUrl(url: String): NetworkSecurityContext {
        findDeterministicContext(url)?.let { return it.securityContext }
        return NetworkSecurityValidator.resolveStreamContext(url)
    }

    class DynamicStreamInterceptor : Interceptor {
        override fun intercept(chain: Interceptor.Chain): Response {
            val originalRequest = chain.request()
            var requestUrl = originalRequest.url.toString()
            val requestBuilder = originalRequest.newBuilder()

            val matchedContext = findDeterministicContext(requestUrl)

            if (matchedContext != null) {
                if (matchedContext.headers.xtreamUser != null && matchedContext.headers.xtreamPass != null) {
                    if (requestUrl.contains("/X-AUTH-INJECT/")) {
                        val encodedUser = Uri.encode(matchedContext.headers.xtreamUser)
                        val encodedPass = Uri.encode(matchedContext.headers.xtreamPass)
                        val rewrittenUrl = requestUrl.replace("/X-AUTH-INJECT/", "/$encodedUser/$encodedPass/")
                        requestUrl = rewrittenUrl
                        requestBuilder.url(rewrittenUrl)
                        streamContextRegistry[rewrittenUrl] = matchedContext.copy(resolvedUrlOverride = rewrittenUrl)
                    }
                } else if (matchedContext.resolvedUrlOverride != null && requestUrl.startsWith("xtream://")) {
                    requestBuilder.url(matchedContext.resolvedUrlOverride)
                }

                val ua = NetworkSecurityValidator.sanitizeHeaderValue(matchedContext.headers.userAgent?.takeIf { it.isNotBlank() })
                    ?: DEFAULT_USER_AGENT
                if (originalRequest.header("User-Agent") == null) requestBuilder.header("User-Agent", ua)

                NetworkSecurityValidator.sanitizeHeaderValue(matchedContext.headers.referer)?.takeIf { it.isNotBlank() }?.let {
                    if (originalRequest.header("Referer") == null) requestBuilder.header("Referer", it)
                }
                NetworkSecurityValidator.sanitizeHeaderValue(matchedContext.headers.cookie)?.takeIf { it.isNotBlank() }?.let {
                    if (originalRequest.header("Cookie") == null) requestBuilder.header("Cookie", it)
                }
                NetworkSecurityValidator.sanitizeHeaderValue(matchedContext.headers.authorization)?.takeIf { it.isNotBlank() }?.let {
                    if (originalRequest.header("Authorization") == null) requestBuilder.header("Authorization", it)
                }
            } else {
                if (originalRequest.header("User-Agent") == null) {
                    requestBuilder.header("User-Agent", DEFAULT_USER_AGENT)
                }
            }

            val response = chain.proceed(requestBuilder.build())
            val finalUrl = response.request.url.toString()
            if (matchedContext != null && finalUrl != requestUrl && !streamContextRegistry.containsKey(finalUrl)) {
                val sameOrigin = originKey(finalUrl) == originKey(requestUrl)
                val preservedHeaders = if (sameOrigin) {
                    matchedContext.headers
                } else {
                    matchedContext.headers.copy(cookie = null, authorization = null, xtreamUser = null, xtreamPass = null)
                }
                streamContextRegistry[finalUrl] = matchedContext.copy(headers = preservedHeaders, resolvedUrlOverride = finalUrl)
            }
            return response
        }
    }

    fun buildFactory(
        context: Context,
        diagnosticsListener: okhttp3.EventListener.Factory? = null
    ): DataSource.Factory {
        val baseOkHttp: OkHttpClient = OkHttpStreamingClient
            .buildDynamicClient(DynamicStreamInterceptor(), diagnosticsListener)
            .newBuilder()
            .dns(NetworkSecurityValidator.ContextDispatchingDns(NetworkSecurityContext.IPTV_STREAM))
            .addInterceptor(
                NetworkSecurityValidator.RedirectGuardInterceptor { url ->
                    resolveSecurityContextForUrl(url.toString())
                }
            )
            .followRedirects(false)
            .followSslRedirects(false)
            .build()
        val okHttpFactory = OkHttpDataSource.Factory(baseOkHttp)
        return DefaultDataSource.Factory(context, okHttpFactory)
    }
}
""",

    "player/PlaybackRecoveryManager.kt": """package vid.mat.app.player

import androidx.media3.common.C
import androidx.media3.common.MediaItem
import androidx.media3.common.MimeTypes
import androidx.media3.common.PlaybackException
import androidx.media3.exoplayer.ExoPlayer
import androidx.media3.exoplayer.mediacodec.MediaCodecRenderer
import androidx.media3.exoplayer.mediacodec.MediaCodecUtil
import kotlinx.coroutines.CoroutineScope
import kotlinx.coroutines.Dispatchers
import kotlinx.coroutines.Job
import kotlinx.coroutines.SupervisorJob
import kotlinx.coroutines.cancel
import kotlinx.coroutines.delay
import kotlinx.coroutines.flow.MutableSharedFlow
import kotlinx.coroutines.flow.MutableStateFlow
import kotlinx.coroutines.flow.StateFlow
import kotlinx.coroutines.flow.asStateFlow
import kotlinx.coroutines.flow.first
import kotlinx.coroutines.launch
import kotlinx.coroutines.withContext
import kotlinx.coroutines.withTimeoutOrNull
import vid.mat.app.data.NoResponse
import vid.mat.app.data.ReplaceStream
import vid.mat.app.data.SkipItem
import vid.mat.app.data.StreamExceptionResponse
import vid.mat.app.data.repository.MediaRepository
import vid.mat.app.error.NetworkHealthMonitor
import vid.mat.app.error.PlaybackErrorFactory
import vid.mat.app.error.recovery.SmartRecoveryManager
import vid.mat.app.network.OkHttpDataSourceHelper
import vid.mat.app.player.capability.DecoderFailureType
import vid.mat.app.player.capability.DeviceCodecCapabilityDetector
import vid.mat.app.player.intelligence.NetworkQualitySnapshot
import vid.mat.app.util.AppLogger
import vid.mat.app.util.SensitiveDataSanitizer
import vid.mat.app.util.StreamErrorMapper
import kotlin.random.Random

class PlaybackRecoveryManager(
    private val playerProvider: () -> ExoPlayer?,
    private val liveEdgeController: LiveEdgeController,
    private val failoverEngine: StreamFailoverEngine,
    private val smartRecoveryManager: SmartRecoveryManager,
    private val networkMonitor: NetworkHealthMonitor,
    private val codecDetector: DeviceCodecCapabilityDetector,
    private val errorFlow: MutableSharedFlow<Exception>,
    private val rescueProvider: () -> suspend (
        item: String?,
        mediaItem: MediaItem?,
        exception: Exception,
        repository: MediaRepository
    ) -> StreamExceptionResponse,
    private val repository: MediaRepository,
    private val sessionUpdate: (PlaybackSession) -> Unit = {},
    private val onActiveStreamChanged: (PlaybackSession, MediaItem?) -> Unit = { _, _ -> },
    private val onTerminalFailure: () -> Unit = {}
) {

    private val TAG = "PlaybackRecoveryManager"
    private val scope = CoroutineScope(SupervisorJob() + Dispatchers.Main)

    private val _recoveryState = MutableStateFlow<PlaybackRecoveryState>(PlaybackRecoveryState.Idle)
    val recoveryState: StateFlow<PlaybackRecoveryState> = _recoveryState.asStateFlow()

    @Volatile
    private var activeSession: PlaybackSession? = null

    @Volatile
    private var activeRequestId: PlaybackRequestId = PlaybackRequestId.NONE

    private var activeRetryPolicy: RetryPolicy = RetryPolicy()
    private var autoRetryCount = 0
    private var hasAttemptedDecoderCeilingFallback = false
    private var recoveryJob: Job? = null

    private data class PlaybackSnapshot(
        val positionMs: Long = 0L,
        val liveOffsetMs: Long = C.TIME_UNSET,
        val wasLive: Boolean = false
    )

    companion object {
        const val MAX_AUTO_RETRIES = 3
        const val BASE_RETRY_DELAY_MS = 1_000L
        private const val OFFLINE_WAIT_TIMEOUT_MS = 12_000L
    }

    fun requestProactiveFailover(
        targetCandidateId: String?,
        currentMediaItem: MediaItem?,
        playerWrapper: NewPlayer
    ) {
        val currentSession = activeSession ?: return
        if (_recoveryState.value.isTerminal || recoveryJob?.isActive == true) return
        val currentReqId = activeRequestId

        recoveryJob = scope.launch {
            val next = failoverEngine.selectNextCandidate(
                channelKey = currentSession.mediaId,
                failedCandidateId = currentSession.endpointId ?: currentSession.mediaId,
                failureReason = StreamFailureReason.BUFFER_STARVATION,
                networkQuality = (playerWrapper as? NewPlayerImpl)?.playbackIntelligence?.networkQualityFlow?.value
                    ?: NetworkQualitySnapshot.UNKNOWN
            ) ?: return@launch

            if (currentReqId != activeRequestId || currentSession.sessionId != activeSession?.sessionId) return@launch
            val savedPos = withContext(Dispatchers.Main) { playerProvider()?.currentPosition?.coerceAtLeast(0L) ?: 0L }
            withContext(Dispatchers.Main) {
                val player = playerProvider() ?: return@withContext
                val builder = (currentMediaItem ?: player.currentMediaItem)?.buildUpon()
                    ?: MediaItem.Builder().setMediaId(currentSession.mediaId)
                val newItem = builder.setMediaId(currentSession.mediaId).setUri(next.uri).build()
                OkHttpDataSourceHelper.registerStreamContext(
                    next.uri.toString(),
                    OkHttpDataSourceHelper.StreamContext(headers = next.headers)
                )
                val updated = currentSession.copy(endpointId = next.endpointId, resolvedUri = next.uri, resumePosition = savedPos)
                activeSession = updated
                sessionUpdate(updated)
                onActiveStreamChanged(updated, newItem)
                player.setMediaItem(newItem)
                player.prepare()
                if (currentSession.playbackType != PlaybackType.LIVE && savedPos > 0L) player.seekTo(savedPos)
                player.play()
            }
        }
    }

    private fun extractDecoderFailure(
        error: PlaybackException,
        fallbackMime: String,
        fallbackWidth: Int,
        fallbackHeight: Int
    ): DecoderFailureInfo {
        var cursor: Throwable? = error
        var decoderInitFound = false
        var decoderQueryFound = false
        while (cursor != null) {
            when (cursor) {
                is MediaCodecRenderer.DecoderInitializationException -> decoderInitFound = true
                is MediaCodecUtil.DecoderQueryException -> decoderQueryFound = true
            }
            cursor = cursor.cause
        }
        return DecoderFailureInfo(
            mimeType = fallbackMime,
            width = fallbackWidth,
            height = fallbackHeight,
            initializationExceptionFound = decoderInitFound,
            queryExceptionFound = decoderQueryFound
        )
    }

    private data class DecoderFailureInfo(
        val mimeType: String,
        val width: Int,
        val height: Int,
        val initializationExceptionFound: Boolean,
        val queryExceptionFound: Boolean
    )

    fun startSession(
        session: PlaybackSession,
        requestId: PlaybackRequestId = PlaybackRequestId(session.requestId),
        retryPolicy: RetryPolicy = RetryPolicy(MAX_AUTO_RETRIES, BASE_RETRY_DELAY_MS)
    ): PlaybackSession {
        cancel()
        activeSession = session
        activeRequestId = requestId
        activeRetryPolicy = retryPolicy
        hasAttemptedDecoderCeilingFallback = false
        _recoveryState.value = PlaybackRecoveryState.Preparing(session)
        AppLogger.d(TAG, "Initialized recovery session: ${session.sessionId} (Request: ${requestId.value}) for ${SensitiveDataSanitizer.sanitizeUrl(session.mediaId)}")
        return session
    }

    fun startSession(item: String): PlaybackSession {
        return startSession(PlaybackSession(mediaId = item))
    }

    fun getActiveSession(): PlaybackSession? = activeSession

    fun onPlaybackReady(session: PlaybackSession) {
        if (session.sessionId != activeSession?.sessionId) return
        autoRetryCount = 0
        smartRecoveryManager.dismiss()
        _recoveryState.value = PlaybackRecoveryState.Playing(session)
        session.endpointId?.let { epId ->
            failoverEngine.recordSuccess(session.mediaId, epId)
        }
    }

    fun onPlaybackPaused(session: PlaybackSession) {
        if (session.sessionId != activeSession?.sessionId) return
        _recoveryState.value = PlaybackRecoveryState.Paused(session)
    }

    fun handle(
        error: PlaybackException,
        currentMediaItem: MediaItem?,
        playerWrapper: NewPlayer
    ) {
        val currentSession = activeSession
        if (currentSession == null) {
            errorFlow.tryEmit(error)
            return
        }
        val currentReqId = activeRequestId
        val rawItem = currentMediaItem?.mediaId ?: currentSession.mediaId

        if (_recoveryState.value.isTerminal) {
            AppLogger.d(TAG, "Ignoring error on terminal session: ${currentSession.sessionId}")
            return
        }

        recoveryJob?.cancel()
        recoveryJob = scope.launch {
            val failureReason = failoverEngine.classifyError(error)
            val httpStatus = StreamErrorMapper.extractHttpStatus(error)
            val playbackSnapshot = withContext(Dispatchers.Main) {
                playerProvider()?.let { player ->
                    PlaybackSnapshot(
                        positionMs = player.currentPosition.coerceAtLeast(0L),
                        liveOffsetMs = player.currentLiveOffset,
                        wasLive = player.isCurrentMediaItemLive
                    )
                } ?: PlaybackSnapshot()
            }
            val savedPositionMs = playbackSnapshot.positionMs
            val savedLiveOffsetMs = playbackSnapshot.liveOffsetMs

            fun isSessionStillCurrent(): Boolean =
                currentReqId == activeRequestId && currentSession.sessionId == activeSession?.sessionId

            suspend fun executeCandidateFailover(candidate: StreamCandidate): Boolean {
                if (!isSessionStillCurrent()) {
                    AppLogger.d(TAG, "Cancelling failover for stale request ${currentReqId.value}")
                    return false
                }
                AppLogger.i(TAG, "Executing seamless candidate failover to: ${candidate.id}")
                return withContext(Dispatchers.Main) {
                    if (!isSessionStillCurrent()) return@withContext false
                    val player = playerProvider() ?: return@withContext false

                    currentSession.resolvedUri?.toString()?.let { oldKey ->
                        if (oldKey != candidate.uri.toString()) {
                            OkHttpDataSourceHelper.unregisterStreamContext(oldKey)
                        }
                    }

                    val streamKey = candidate.uri.toString()
                    OkHttpDataSourceHelper.registerStreamContext(
                        streamKey,
                        OkHttpDataSourceHelper.StreamContext(headers = candidate.headers)
                    )

                    val candidateUri = candidate.uri.toString().lowercase()
                    val candidateMime = when {
                        candidate.protocol.equals("m3u8", ignoreCase = true) || candidateUri.contains(".m3u8") || candidateUri.contains("/hls/") -> MimeTypes.APPLICATION_M3U8
                        candidate.protocol.equals("mpd", ignoreCase = true) || candidateUri.contains(".mpd") || candidateUri.contains("/dash/") -> MimeTypes.APPLICATION_MPD
                        candidate.protocol.equals("ts", ignoreCase = true) || candidateUri.endsWith(".ts") || candidateUri.endsWith(".m2ts") -> MimeTypes.VIDEO_MP2T
                        candidate.protocol.equals("mp4", ignoreCase = true) || candidateUri.endsWith(".mp4") || candidateUri.endsWith(".m4v") -> MimeTypes.VIDEO_MP4
                        candidate.protocol.equals("mkv", ignoreCase = true) || candidateUri.endsWith(".mkv") -> MimeTypes.VIDEO_MATROSKA
                        candidate.protocol.equals("webm", ignoreCase = true) || candidateUri.endsWith(".webm") -> MimeTypes.VIDEO_WEBM
                        else -> null
                    }
                    val builder = (currentMediaItem ?: player.currentMediaItem)?.buildUpon()
                        ?: MediaItem.Builder().setMediaId(currentSession.mediaId)
                    builder.setMediaId(currentSession.mediaId)
                    builder.setUri(candidate.uri)
                    builder.setMimeType(candidateMime)
                    if (savedLiveOffsetMs != C.TIME_UNSET && savedLiveOffsetMs > 0L) {
                        builder.setLiveConfiguration(
                            MediaItem.LiveConfiguration.Builder()
                                .setTargetOffsetMs(savedLiveOffsetMs)
                                .build()
                        )
                    }
                    val newItem = builder.build()

                    val updatedSession = currentSession.copy(
                        endpointId = candidate.endpointId,
                        resolvedUri = candidate.uri,
                        resumePosition = savedPositionMs,
                        liveOffset = if (savedLiveOffsetMs != C.TIME_UNSET) savedLiveOffsetMs else currentSession.liveOffset
                    )
                    activeSession = updatedSession
                    autoRetryCount = 0
                    hasAttemptedDecoderCeilingFallback = false
                    sessionUpdate(updatedSession)
                    _recoveryState.value = PlaybackRecoveryState.Recovering(
                        session = updatedSession,
                        attempt = 1,
                        maxAttempts = 1,
                        delayMs = 0L
                    )

                    player.setMediaItem(newItem)
                    onActiveStreamChanged(updatedSession, playerProvider()?.currentMediaItem)
                    player.prepare()
                    if (currentSession.playbackType == PlaybackType.LIVE) {
                        player.seekToDefaultPosition()
                    } else if (savedPositionMs > 0L) {
                        player.seekTo(savedPositionMs)
                    }
                    player.play()
                    true
                }
            }

            val rescueResponse = runCatching {
                rescueProvider().invoke(rawItem, currentMediaItem, error, repository)
            }.getOrDefault(NoResponse)

            if (!isSessionStillCurrent()) return@launch

            when (rescueResponse) {
                is ReplaceStream -> {
                    val rescuedCandidate = StreamCandidate(
                        endpointId = rescueResponse.newUri.toString(),
                        uri = rescueResponse.newUri
                    )
                    if (executeCandidateFailover(rescuedCandidate)) return@launch
                }
                is SkipItem -> {
                    withContext(Dispatchers.Main) {
                        val player = playerProvider()
                        if (player != null && player.hasNextMediaItem()) {
                            player.seekToNextMediaItem()
                            player.prepare()
                            player.play()
                        }
                    }
                    return@launch
                }
                is NoResponse -> Unit
            }

            if (error.errorCode == PlaybackException.ERROR_CODE_BEHIND_LIVE_WINDOW) {
                if (!isSessionStillCurrent()) return@launch
                AppLogger.w(TAG, "BehindLiveWindowException detected -> resynchronizing to live edge")
                liveEdgeController.recoverFromLiveWindowLoss()
                return@launch
            }

            val isDecoderError = error.errorCode == PlaybackException.ERROR_CODE_DECODER_INIT_FAILED ||
                error.errorCode == PlaybackException.ERROR_CODE_DECODER_QUERY_FAILED ||
                error.errorCode == PlaybackException.ERROR_CODE_DECODING_FAILED ||
                error.errorCode == PlaybackException.ERROR_CODE_DECODING_FORMAT_EXCEEDS_CAPABILITIES ||
                error.errorCode == PlaybackException.ERROR_CODE_DECODING_FORMAT_UNSUPPORTED

            if (isDecoderError) {
                val videoFormat = playerProvider()?.videoFormat
                val audioFormat = playerProvider()?.audioFormat
                val decoderInfo = extractDecoderFailure(
                    error = error,
                    fallbackMime = videoFormat?.sampleMimeType ?: audioFormat?.sampleMimeType ?: "video/unknown",
                    fallbackWidth = videoFormat?.width ?: 0,
                    fallbackHeight = videoFormat?.height ?: 0
                )
                val failingMime = decoderInfo.mimeType
                val width = decoderInfo.width
                val height = decoderInfo.height

                AppLogger.w(
                    TAG,
                    "Decoder failure on $failingMime (${width}x${height}) " +
                        "initCause=${decoderInfo.initializationExceptionFound}, " +
                        "queryCause=${decoderInfo.queryExceptionFound}"
                )

                val failureType = if (error.errorCode == PlaybackException.ERROR_CODE_DECODING_FORMAT_EXCEEDS_CAPABILITIES) {
                    DecoderFailureType.CAPABILITY_EXCEEDED
                } else {
                    DecoderFailureType.INITIALIZATION_FAILED
                }
                codecDetector.recordRuntimeDecoderFailure(failingMime, width, height, failureType)

                if (height > 720 && !hasAttemptedDecoderCeilingFallback) {
                    hasAttemptedDecoderCeilingFallback = true
                    val constrained = withContext(Dispatchers.Main) {
                        val player = playerProvider() ?: return@withContext false
                        val impl = playerWrapper as? NewPlayerImpl
                        if (impl != null) {
                            impl.playbackIntelligence.abrController.setHardwareCeiling(720, player)
                        } else {
                            player.trackSelectionParameters = player.trackSelectionParameters.buildUpon()
                                .setMaxVideoSize(1280, 720)
                                .build()
                        }
                        player.prepare()
                        player.play()
                        true
                    }
                    if (constrained) {
                        AppLogger.i(TAG, "Decoder fallback: Applied one-time 720p hardware ceiling")
                        return@launch
                    }
                }

                val alternateCandidate = failoverEngine.selectNextCandidate(
                    channelKey = rawItem,
                    failedCandidateId = currentSession.endpointId ?: rawItem,
                    failureReason = StreamFailureReason.DECODER_FAILURE,
                    networkQuality = (playerWrapper as? NewPlayerImpl)?.playbackIntelligence?.networkQualityFlow?.value
                        ?: NetworkQualitySnapshot.UNKNOWN
                )
                if (alternateCandidate != null && executeCandidateFailover(alternateCandidate)) {
                    AppLogger.i(TAG, "Decoder fallback: Switched to compatible candidate ${alternateCandidate.id}")
                    return@launch
                }
            }

            if (!networkMonitor.isConnected && !isDecoderError) {
                _recoveryState.value = PlaybackRecoveryState.Recovering(
                    session = currentSession,
                    attempt = 1,
                    maxAttempts = 1,
                    delayMs = OFFLINE_WAIT_TIMEOUT_MS
                )
                val reconnected = withTimeoutOrNull(OFFLINE_WAIT_TIMEOUT_MS) {
                    networkMonitor.networkState.first {
                        it == NetworkHealthMonitor.NetworkState.CONNECTED ||
                            it == NetworkHealthMonitor.NetworkState.METERED
                    }
                } != null
                if (!isSessionStillCurrent()) return@launch
                if (reconnected) {
                    withContext(Dispatchers.Main) {
                        playerProvider()?.let { player ->
                            player.prepare()
                            if (currentSession.playbackType == PlaybackType.LIVE) {
                                player.seekToDefaultPosition()
                            } else if (savedPositionMs > 0L) {
                                player.seekTo(savedPositionMs)
                            }
                            player.play()
                        }
                    }
                    return@launch
                }
            }

            val maxRetries = activeRetryPolicy.maxRetries.coerceAtLeast(0)
            val baseDelayMs = activeRetryPolicy.baseDelayMs.coerceAtLeast(250L)
            if (failureReason.isRetryableInPlace && autoRetryCount < maxRetries && !isDecoderError && networkMonitor.isConnected) {
                autoRetryCount++
                val nextAttempt = autoRetryCount
                val backoffMs = baseDelayMs * (1 shl (nextAttempt - 1))
                val jitterMs = Random.nextLong(100, 400)
                val totalDelay = backoffMs + jitterMs

                _recoveryState.value = PlaybackRecoveryState.Recovering(
                    session = currentSession,
                    attempt = nextAttempt,
                    maxAttempts = maxRetries,
                    delayMs = totalDelay
                )

                AppLogger.d(TAG, "Reconnecting attempt $nextAttempt/$maxRetries in ${totalDelay}ms")
                delay(totalDelay)

                if (!isSessionStillCurrent()) {
                    AppLogger.d(TAG, "Aborting retry for cancelled request ${currentReqId.value}")
                    return@launch
                }

                withContext(Dispatchers.Main) {
                    if (!isSessionStillCurrent()) return@withContext
                    playerProvider()?.let { player ->
                        player.prepare()
                        if (currentSession.playbackType == PlaybackType.LIVE) {
                            player.seekToDefaultPosition()
                        } else if (savedPositionMs > 0L) {
                            player.seekTo(savedPositionMs)
                        }
                        player.play()
                    }
                }
                return@launch
            }

            autoRetryCount = 0
            if (!isSessionStillCurrent()) return@launch

            val typedError = PlaybackErrorFactory.from(error, httpStatus)
            _recoveryState.value = PlaybackRecoveryState.Recovering(
                session = currentSession,
                attempt = maxRetries,
                maxAttempts = maxRetries,
                delayMs = 0L
            )

            try {
                smartRecoveryManager.handlePlaybackFault(
                    error = error,
                    channelKey = rawItem,
                    currentCandidateId = currentSession.endpointId ?: rawItem,
                    networkMonitor = networkMonitor,
                    player = playerWrapper,
                    onAlternateElected = { candidate ->
                        scope.launch {
                            val switched = executeCandidateFailover(candidate)
                            if (!switched && isSessionStillCurrent()) {
                                publishTerminalFailure(currentSession, typedError)
                            }
                        }
                    },
                    onTerminalFailure = {
                        scope.launch {
                            if (isSessionStillCurrent()) {
                                publishTerminalFailure(currentSession, typedError)
                            }
                        }
                    }
                )
            } catch (recoveryError: Exception) {
                AppLogger.e(TAG, "Smart recovery failed; publishing terminal playback error", recoveryError)
                if (isSessionStillCurrent()) {
                    publishTerminalFailure(currentSession, typedError)
                }
            }
        }
    }

    private suspend fun publishTerminalFailure(
        session: PlaybackSession,
        typedError: vid.mat.app.error.PlaybackError
    ) {
        onTerminalFailure()
        _recoveryState.value = PlaybackRecoveryState.RecoveryFailed(session, typedError)
        errorFlow.emit(typedError)
    }

    fun cancel() {
        val current = activeSession
        if (current != null) {
            _recoveryState.value = PlaybackRecoveryState.Terminated(current)
        } else {
            _recoveryState.value = PlaybackRecoveryState.Idle
        }
        activeSession = null
        activeRequestId = PlaybackRequestId.NONE
        autoRetryCount = 0
        hasAttemptedDecoderCeilingFallback = false
        recoveryJob?.cancel()
        recoveryJob = null
        smartRecoveryManager.dismiss()
    }

    fun destroy() {
        cancel()
        scope.cancel()
    }
}
"""
}


def main() -> None:
    if not BASE_DIR.exists():
        raise SystemExit(f"Target directory does not exist: {BASE_DIR}")

    for rel_path, content in FILES.items():
        target = BASE_DIR / rel_path
        target.parent.mkdir(parents=True, exist_ok=True)
        target.write_text(content, encoding="utf-8")
        print(f"Updated: {target}")

    print(f"Done. Wrote {len(FILES)} files.")


if __name__ == "__main__":
    main()
```

May I proceed with **Compile-Fix Batch 2 (5 files)**: `data/intelligence/PlaylistIntelligenceEngine.kt`, `dvr/engine/hls/HlsDvrPipeline.kt`, `dvr/ui/ScheduleRecordingDialogFragment.kt`, `player/abr/IntelligentAbrController.kt`, and `playlist/M3uParser.kt`?
