Here is the Python script for **Batch 5 (5 classes — Channel Organization & Android TV Presenter/VOD Repo)**:

1. `data/repository/ChannelOrganizationRepository.kt` — Adds deterministic tie-breakers (`displayNumber`, `displayTitle`) in `getOrganizedChannels()` and ignores blank custom group names in `createCustomGroup()`.
2. `viewmodel/ChannelOrganizationViewModel.kt` — Adds `deleteCustomGroup(name)` and synchronizes in-memory `userOrderIndex` overrides after drag-and-drop reordering.
3. `ui/organization/ChannelOrganizationFragment.kt` — Cleans up `btnIncludeHidden` listener in `setupListeners()`.
4. `tv/TvChannelPresenter.kt` — Fixes missing imports (`VodMovieEntity`, `SeriesEntity`, `SearchItem`) and binds Now/Next EPG subtitles onto `Channel` cards.
5. `data/repository/VodRepository.kt` — Exposes `getMoviesByCategory(...)` for Android TV category rows.

```python
#!/usr/bin/env python3
from pathlib import Path

BASE_DIR = Path("/home/username/AndroidStudioProjects/Ytvmate/app/src/main/java/vid/mat/app")

FILES = {
    "data/repository/ChannelOrganizationRepository.kt": """package vid.mat.app.data.repository

import androidx.room.withTransaction
import kotlinx.coroutines.Dispatchers
import kotlinx.coroutines.flow.Flow
import kotlinx.coroutines.withContext
import vid.mat.app.data.dao.ChannelDao
import vid.mat.app.data.dao.ChannelUserOverrideDao
import vid.mat.app.data.dao.CustomGroupDao
import vid.mat.app.data.database.VideoUrlDatabase
import vid.mat.app.data.model.ChannelUserOverrideEntity
import vid.mat.app.data.model.CustomGroupEntity
import vid.mat.app.data.model.OrganizedChannel
import vid.mat.app.util.AppLogger
import javax.inject.Inject
import javax.inject.Singleton

@Singleton
class ChannelOrganizationRepository @Inject constructor(
    private val database: VideoUrlDatabase,
    private val channelDao: ChannelDao,
    private val overrideDao: ChannelUserOverrideDao,
    private val groupDao: CustomGroupDao
) {

    companion object {
        private const val TAG = "ChannelOrganizationRepository"
    }

    val customGroups: Flow<List<CustomGroupEntity>> = groupDao.getAllFlow()

    suspend fun getOrganizedChannels(
        groupFilter: String? = null,
        includeHidden: Boolean = false
    ): List<OrganizedChannel> = withContext(Dispatchers.IO) {
        val rawChannels = channelDao.getAllOrdered(100_000)
        val overridesMap = overrideDao.getAll().associateBy { it.channelIdentity }

        rawChannels.map { ch ->
            val override = overridesMap[ch.channelIdentity]
            OrganizedChannel(channel = ch, override = override)
        }.filter { item ->
            (includeHidden || !item.isHidden) &&
                (groupFilter == null || item.displayGroup.equals(groupFilter, ignoreCase = true))
        }.sortedWith(
            compareBy<OrganizedChannel> { it.orderIndex }
                .thenBy { it.displayNumber ?: Int.MAX_VALUE }
                .thenBy { it.displayTitle }
        )
    }

    suspend fun saveChannelOverride(
        channelIdentity: String,
        customTitle: String?,
        customGroup: String?,
        customLogoUrl: String?,
        customNumber: Int?
    ) = withContext(Dispatchers.IO) {
        AppLogger.i(TAG, "Saving user overrides for channel identity '$channelIdentity'")
        val existing = overrideDao.getByIdentity(channelIdentity)
        val updated = (existing ?: ChannelUserOverrideEntity(channelIdentity = channelIdentity)).copy(
            customTitle = customTitle?.trim()?.takeIf { it.isNotBlank() },
            customGroupTitle = customGroup?.trim()?.takeIf { it.isNotBlank() },
            customLogoUrl = customLogoUrl?.trim()?.takeIf { it.isNotBlank() },
            customChannelNumber = customNumber,
            updatedAtMs = System.currentTimeMillis()
        )
        overrideDao.upsert(updated)
    }

    suspend fun reorderChannels(orderedIdentities: List<String>) = withContext(Dispatchers.IO) {
        AppLogger.i(TAG, "Persisting user drag-and-drop order for ${orderedIdentities.size} channels")
        val now = System.currentTimeMillis()
        database.withTransaction {
            orderedIdentities.forEachIndexed { index, identity ->
                val existing = overrideDao.getByIdentity(identity)
                if (existing != null) {
                    overrideDao.updateOrderIndex(identity, index, now)
                } else {
                    overrideDao.upsert(
                        ChannelUserOverrideEntity(
                            channelIdentity = identity,
                            userOrderIndex = index,
                            updatedAtMs = now
                        )
                    )
                }
            }
        }
    }

    suspend fun setChannelHidden(channelIdentity: String, isHidden: Boolean) = withContext(Dispatchers.IO) {
        val now = System.currentTimeMillis()
        val existing = overrideDao.getByIdentity(channelIdentity)
        if (existing != null) {
            overrideDao.setHidden(channelIdentity, isHidden, now)
        } else {
            overrideDao.upsert(
                ChannelUserOverrideEntity(
                    channelIdentity = channelIdentity,
                    isHidden = isHidden,
                    updatedAtMs = now
                )
            )
        }
    }

    suspend fun createCustomGroup(name: String) = withContext(Dispatchers.IO) {
        val trimmed = name.trim()
        if (trimmed.isNotBlank()) {
            groupDao.insert(CustomGroupEntity(name = trimmed))
        }
    }

    suspend fun deleteCustomGroup(name: String) = withContext(Dispatchers.IO) {
        groupDao.deleteByName(name.trim())
    }

    suspend fun restoreOriginalChannel(channelIdentity: String) = withContext(Dispatchers.IO) {
        AppLogger.i(TAG, "Restoring channel '$channelIdentity' to raw provider defaults")
        overrideDao.deleteByIdentity(channelIdentity)
    }

    suspend fun resetAllOverrides() = withContext(Dispatchers.IO) {
        AppLogger.w(TAG, "Resetting all user channel overrides to provider defaults")
        overrideDao.deleteAll()
    }
}
""",

    "viewmodel/ChannelOrganizationViewModel.kt": """package vid.mat.app.viewmodel

import androidx.lifecycle.ViewModel
import androidx.lifecycle.viewModelScope
import dagger.hilt.android.lifecycle.HiltViewModel
import kotlinx.coroutines.flow.MutableStateFlow
import kotlinx.coroutines.flow.SharingStarted
import kotlinx.coroutines.flow.StateFlow
import kotlinx.coroutines.flow.asStateFlow
import kotlinx.coroutines.flow.stateIn
import kotlinx.coroutines.launch
import vid.mat.app.data.model.ChannelUserOverrideEntity
import vid.mat.app.data.model.CustomGroupEntity
import vid.mat.app.data.model.OrganizedChannel
import vid.mat.app.data.repository.ChannelOrganizationRepository
import javax.inject.Inject

@HiltViewModel
class ChannelOrganizationViewModel @Inject constructor(
    private val organizationRepo: ChannelOrganizationRepository
) : ViewModel() {

    private val _organizedChannels = MutableStateFlow<List<OrganizedChannel>>(emptyList())
    val organizedChannels: StateFlow<List<OrganizedChannel>> = _organizedChannels.asStateFlow()

    private val _selectedGroup = MutableStateFlow<String?>(null)
    val selectedGroup: StateFlow<String?> = _selectedGroup.asStateFlow()

    private val _includeHidden = MutableStateFlow(false)
    val includeHidden: StateFlow<Boolean> = _includeHidden.asStateFlow()

    private val _isLoading = MutableStateFlow(false)
    val isLoading: StateFlow<Boolean> = _isLoading.asStateFlow()

    val customGroups: StateFlow<List<CustomGroupEntity>> = organizationRepo.customGroups
        .stateIn(viewModelScope, SharingStarted.Lazily, emptyList())

    init {
        loadChannels()
    }

    fun loadChannels() = viewModelScope.launch {
        _isLoading.value = true
        _organizedChannels.value = organizationRepo.getOrganizedChannels(
            groupFilter = _selectedGroup.value,
            includeHidden = _includeHidden.value
        )
        _isLoading.value = false
    }

    fun selectGroup(group: String?) {
        _selectedGroup.value = group
        loadChannels()
    }

    fun toggleIncludeHidden() {
        _includeHidden.value = !_includeHidden.value
        loadChannels()
    }

    fun moveChannel(fromPosition: Int, toPosition: Int) {
        val currentList = _organizedChannels.value.toMutableList()
        if (fromPosition in currentList.indices && toPosition in currentList.indices) {
            val moved = currentList.removeAt(fromPosition)
            currentList.add(toPosition, moved)
            _organizedChannels.value = currentList
        }
    }

    fun saveReorderedChannels() = viewModelScope.launch {
        val current = _organizedChannels.value
        val identities = current.map { it.channel.channelIdentity }
        organizationRepo.reorderChannels(identities)
        _organizedChannels.value = current.mapIndexed { idx, item ->
            val updatedOverride = (item.override ?: ChannelUserOverrideEntity(
                channelIdentity = item.channel.channelIdentity
            )).copy(userOrderIndex = idx)
            item.copy(override = updatedOverride)
        }
    }

    fun saveChannelOverride(
        channelIdentity: String,
        title: String?,
        group: String?,
        logo: String?,
        number: Int?
    ) = viewModelScope.launch {
        organizationRepo.saveChannelOverride(channelIdentity, title, group, logo, number)
        loadChannels()
    }

    fun toggleHideChannel(channel: OrganizedChannel) = viewModelScope.launch {
        organizationRepo.setChannelHidden(channel.channel.channelIdentity, !channel.isHidden)
        loadChannels()
    }

    fun restoreOriginalChannel(channel: OrganizedChannel) = viewModelScope.launch {
        organizationRepo.restoreOriginalChannel(channel.channel.channelIdentity)
        loadChannels()
    }

    fun resetAllOverrides() = viewModelScope.launch {
        organizationRepo.resetAllOverrides()
        loadChannels()
    }

    fun createCustomGroup(name: String) = viewModelScope.launch {
        organizationRepo.createCustomGroup(name)
    }

    fun deleteCustomGroup(name: String) = viewModelScope.launch {
        organizationRepo.deleteCustomGroup(name)
        if (_selectedGroup.value == name) {
            _selectedGroup.value = null
        }
        loadChannels()
    }
}
""",

    "ui/organization/ChannelOrganizationFragment.kt": """package vid.mat.app.ui.organization

import android.os.Bundle
import android.view.LayoutInflater
import android.view.View
import android.view.ViewGroup
import androidx.fragment.app.Fragment
import androidx.fragment.app.viewModels
import androidx.lifecycle.Lifecycle
import androidx.lifecycle.lifecycleScope
import androidx.lifecycle.repeatOnLifecycle
import androidx.recyclerview.widget.ItemTouchHelper
import androidx.recyclerview.widget.LinearLayoutManager
import androidx.recyclerview.widget.RecyclerView
import com.google.android.material.dialog.MaterialAlertDialogBuilder
import dagger.hilt.android.AndroidEntryPoint
import kotlinx.coroutines.launch
import vid.mat.app.R
import vid.mat.app.data.model.OrganizedChannel
import vid.mat.app.databinding.FragmentChannelOrganizationBinding
import vid.mat.app.databinding.ItemChannelOrganizeRowBinding
import vid.mat.app.image.LogoImageLoader
import vid.mat.app.viewmodel.ChannelOrganizationViewModel

@AndroidEntryPoint
class ChannelOrganizationFragment : Fragment() {

    private var _binding: FragmentChannelOrganizationBinding? = null
    private val binding get() = _binding!!
    private val viewModel: ChannelOrganizationViewModel by viewModels()

    private lateinit var organizeAdapter: ChannelOrganizeAdapter

    override fun onCreateView(inflater: LayoutInflater, container: ViewGroup?, savedInstanceState: Bundle?): View {
        _binding = FragmentChannelOrganizationBinding.inflate(inflater, container, false)
        return binding.root
    }

    override fun onViewCreated(view: View, savedInstanceState: Bundle?) {
        super.onViewCreated(view, savedInstanceState)
        setupRecycler()
        setupListeners()
        observeData()
    }

    private fun setupRecycler() {
        organizeAdapter = ChannelOrganizeAdapter(
            onEdit = { channel -> showEditDialog(channel) },
            onToggleHide = { channel -> viewModel.toggleHideChannel(channel) }
        )

        val touchHelper = ItemTouchHelper(object : ItemTouchHelper.SimpleCallback(
            ItemTouchHelper.UP or ItemTouchHelper.DOWN, 0
        ) {
            override fun onMove(rv: RecyclerView, vh: RecyclerView.ViewHolder, target: RecyclerView.ViewHolder): Boolean {
                viewModel.moveChannel(vh.adapterPosition, target.adapterPosition)
                organizeAdapter.notifyItemMoved(vh.adapterPosition, target.adapterPosition)
                return true
            }

            override fun onSwiped(vh: RecyclerView.ViewHolder, direction: Int) {}
            override fun clearView(rv: RecyclerView, vh: RecyclerView.ViewHolder) {
                super.clearView(rv, vh)
                viewModel.saveReorderedChannels()
            }
        })

        binding.rvChannels.apply {
            layoutManager = LinearLayoutManager(requireContext())
            adapter = organizeAdapter
            touchHelper.attachToRecyclerView(this)
        }
    }

    private fun setupListeners() {
        binding.btnIncludeHidden.setOnCheckedChangeListener { _, _ ->
            viewModel.toggleIncludeHidden()
        }

        binding.btnResetAll.setOnClickListener {
            MaterialAlertDialogBuilder(requireContext())
                .setTitle("Reset All Channel Overrides?")
                .setMessage("This will remove all custom names, drag-and-drop orders, and custom channel numbers, restoring provider defaults.")
                .setPositiveButton("Reset All") { _, _ -> viewModel.resetAllOverrides() }
                .setNegativeButton("Cancel", null)
                .show()
        }
    }

    private fun observeData() {
        viewLifecycleOwner.lifecycleScope.launch {
            viewLifecycleOwner.repeatOnLifecycle(Lifecycle.State.STARTED) {
                viewModel.organizedChannels.collect { channels ->
                    organizeAdapter.items = channels
                    organizeAdapter.notifyDataSetChanged()
                }
            }
        }
    }

    private fun showEditDialog(channel: OrganizedChannel) {
        EditChannelOverrideDialogFragment.newInstance(
            channel = channel,
            onSave = { title, group, logo, num ->
                viewModel.saveChannelOverride(channel.channel.channelIdentity, title, group, logo, num)
            },
            onReset = {
                viewModel.restoreOriginalChannel(channel)
            }
        ).show(parentFragmentManager, EditChannelOverrideDialogFragment.TAG)
    }

    override fun onDestroyView() {
        super.onDestroyView()
        _binding = null
    }

    private class ChannelOrganizeAdapter(
        private val onEdit: (OrganizedChannel) -> Unit,
        private val onToggleHide: (OrganizedChannel) -> Unit
    ) : RecyclerView.Adapter<ChannelOrganizeAdapter.VH>() {

        var items: List<OrganizedChannel> = emptyList()

        override fun getItemCount(): Int = items.size

        override fun onCreateViewHolder(parent: ViewGroup, viewType: Int): VH {
            val binding = ItemChannelOrganizeRowBinding.inflate(LayoutInflater.from(parent.context), parent, false)
            return VH(binding)
        }

        override fun onBindViewHolder(holder: VH, position: Int) {
            val item = items[position]
            holder.binding.tvNumber.text = item.displayNumber?.toString() ?: "#${position + 1}"
            holder.binding.tvTitle.text = item.displayTitle
            holder.binding.tvGroup.text = item.displayGroup

            holder.binding.tvCustomBadge.visibility = if (item.hasOverride) View.VISIBLE else View.GONE
            holder.binding.btnHide.setImageResource(
                if (item.isHidden) R.drawable.ic_close else R.drawable.ic_notification
            )

            LogoImageLoader.load(
                context = holder.binding.ivLogo.context,
                title = item.displayTitle,
                logoUrl = item.displayLogo,
                target = holder.binding.ivLogo,
                cornerDp = 6f
            )

            holder.binding.btnEdit.setOnClickListener { onEdit(item) }
            holder.binding.btnHide.setOnClickListener { onToggleHide(item) }
        }

        class VH(val binding: ItemChannelOrganizeRowBinding) : RecyclerView.ViewHolder(binding.root)
    }
}
""",

    "tv/TvChannelPresenter.kt": """package vid.mat.app.tv

import android.content.Context
import android.view.ViewGroup
import android.widget.ImageView
import androidx.leanback.widget.ImageCardView
import androidx.leanback.widget.Presenter
import vid.mat.app.data.model.Channel
import vid.mat.app.data.model.SeriesEntity
import vid.mat.app.data.model.VideoUrl
import vid.mat.app.data.model.VodMovieEntity
import vid.mat.app.epg.EpgViewModel
import vid.mat.app.history.WatchHistory
import vid.mat.app.image.LogoImageLoader
import vid.mat.app.search.model.SearchItem
import vid.mat.app.util.performance.DeviceCapabilityProfile

class TvChannelPresenter(
    private val epgViewModel: EpgViewModel? = null
) : Presenter() {

    companion object {
        private const val CARD_WIDTH_DP = 220
        private const val CARD_HEIGHT_DP = 124
    }

    override fun onCreateViewHolder(parent: ViewGroup): ViewHolder {
        val isLowEnd = DeviceCapabilityProfile.isLowEndHardware(parent.context)
        val card = ImageCardView(parent.context).apply {
            isFocusable = true
            isFocusableInTouchMode = true
            setMainImageDimensions(
                dpToPx(parent.context, if (isLowEnd) 180 else CARD_WIDTH_DP),
                dpToPx(parent.context, if (isLowEnd) 100 else CARD_HEIGHT_DP)
            )
            setMainImageScaleType(ImageView.ScaleType.CENTER_CROP)
        }
        return ViewHolder(card)
    }

    override fun onBindViewHolder(viewHolder: ViewHolder, item: Any?) {
        val card = viewHolder.view as ImageCardView

        when (item) {
            is Channel -> {
                card.titleText = item.title
                val epgText = item.tvgId?.let { tvgId ->
                    val cached = epgViewModel?.getCachedNowNext(tvgId)
                    if (cached != null && !cached.isEmpty) cached.toDisplayString()
                    else {
                        epgViewModel?.requestNowAndNext(tvgId)
                        null
                    }
                }
                card.contentText = epgText ?: item.groupTitle ?: ""
                loadLogo(card, item.title, item.logoUrl)
            }
            is VideoUrl -> {
                card.titleText = item.title
                card.contentText = epgSubtitle(item)
                loadLogo(card, item.title, item.logoUrl)
            }
            is WatchHistory -> {
                card.titleText = item.title
                card.contentText = "${item.completedPercent}% watched"
                loadLogo(card, item.title, item.logoUrl)
            }
            is VodMovieEntity -> {
                card.titleText = item.title
                card.contentText = item.year ?: item.categoryName
                loadLogo(card, item.title, item.posterUrl ?: item.backdropUrl)
            }
            is SeriesEntity -> {
                card.titleText = item.title
                card.contentText = item.categoryName
                loadLogo(card, item.title, item.coverUrl ?: item.backdropUrl)
            }
            is SearchItem -> {
                card.titleText = item.title
                card.contentText = item.subtitle ?: item.groupTitle ?: ""
                loadLogo(card, item.title, item.logoUrl)
            }
        }
    }

    override fun onUnbindViewHolder(viewHolder: ViewHolder) {
        val card = viewHolder.view as ImageCardView
        card.mainImageView?.setImageDrawable(null)
        card.badgeImage = null
    }

    private fun loadLogo(card: ImageCardView, title: String, logoUrl: String?) {
        val imageView = card.mainImageView ?: return
        LogoImageLoader.load(
            context = imageView.context,
            title = title,
            logoUrl = logoUrl,
            target = imageView,
            cornerDp = 8f
        )
    }

    private fun epgSubtitle(channel: VideoUrl): String {
        val tvgId = channel.tvgId ?: return ""
        val cached = epgViewModel?.getCachedNowNext(tvgId)
        if (cached != null && !cached.isEmpty) return cached.toDisplayString()
        epgViewModel?.requestNowAndNext(tvgId)
        return ""
    }

    private fun dpToPx(context: Context, dp: Int): Int =
        (dp * context.resources.displayMetrics.density).toInt()
}
""",

    "data/repository/VodRepository.kt": """package vid.mat.app.data.repository

import androidx.paging.Pager
import androidx.paging.PagingConfig
import androidx.paging.PagingData
import kotlinx.coroutines.Dispatchers
import kotlinx.coroutines.flow.Flow
import kotlinx.coroutines.withContext
import vid.mat.app.data.dao.VodMovieDao
import vid.mat.app.data.model.VodCategoryEntity
import vid.mat.app.data.model.VodMovieEntity
import vid.mat.app.favorites.FavoriteTargetIds
import vid.mat.app.favorites.UnifiedFavoriteRepository
import vid.mat.app.favorites.model.FavoriteType
import vid.mat.app.provider.IptvProviderFactory
import vid.mat.app.security.XtreamAccountManager
import vid.mat.app.util.AppLogger
import javax.inject.Inject
import javax.inject.Singleton

@Singleton
class VodRepository @Inject constructor(
    private val vodDao: VodMovieDao,
    private val playlistSourceRepository: PlaylistSourceRepository,
    private val providerFactory: IptvProviderFactory,
    private val favoriteRepository: UnifiedFavoriteRepository,
    private val xtreamAccountManager: XtreamAccountManager
) {
    companion object {
        private const val TAG = "VodRepository"
        const val PAGE_SIZE = 30
        const val PREFETCH = 60
    }

    val continueWatching: Flow<List<VodMovieEntity>> = vodDao.getContinueWatching()
    val favorites: Flow<List<VodMovieEntity>> = vodDao.getFavorites()

    fun getPagedMovies(
        sourceId: Long? = null,
        categoryId: String? = null,
        query: String = "",
        sortMode: Int = 0,
        favOnly: Boolean = false
    ): Flow<PagingData<VodMovieEntity>> = Pager(
        config = PagingConfig(pageSize = PAGE_SIZE, prefetchDistance = PREFETCH, enablePlaceholders = false),
        pagingSourceFactory = { vodDao.getPagedMovies(sourceId, categoryId, query, sortMode, favOnly) }
    ).flow

    fun getCategories(sourceId: Long? = null): Flow<List<VodCategoryEntity>> = vodDao.getDistinctCategories(sourceId)

    fun countMovies(sourceId: Long? = null): Flow<Int> = vodDao.countMovies(sourceId)

    suspend fun getMoviesByCategory(sourceId: Long?, categoryId: String, limit: Int = 25): List<VodMovieEntity> =
        vodDao.getMoviesByCategory(sourceId, categoryId, limit)

    suspend fun getMovieById(movieId: Long): VodMovieEntity? = vodDao.getById(movieId)

    suspend fun toggleFavorite(movie: VodMovieEntity) {
        favoriteRepository.toggleFavorite(
            targetId = FavoriteTargetIds.movie(movie.id),
            type = FavoriteType.VOD_MOVIE,
            title = movie.title,
            subtitle = movie.year,
            logoUrl = movie.posterUrl,
            streamUrl = "vod_${movie.id}",
            groupTitle = movie.categoryName
        )
    }

    suspend fun updateWatchProgress(movieId: Long, positionMs: Long, durationMs: Long) {
        val percent = if (durationMs > 0L) ((positionMs.toDouble() / durationMs) * 100).toInt().coerceIn(0, 100) else 0
        vodDao.updateWatchProgress(movieId, positionMs, durationMs, percent, System.currentTimeMillis())
    }

    suspend fun fetchAndCacheFullMovieDetails(movie: VodMovieEntity): Result<VodMovieEntity> = withContext(Dispatchers.IO) {
        try {
            val source = playlistSourceRepository.getSource(movie.sourceId)
                ?: return@withContext Result.failure(IllegalStateException("Provider source not found for VOD"))
            val provider = providerFactory.createProvider(source)
            val details = provider.fetchVodDetails(movie.streamId).getOrThrow()

            val updated = movie.copy(
                plot = details.description?.takeIf { it.isNotBlank() } ?: movie.plot,
                cast = details.cast?.takeIf { it.isNotBlank() } ?: movie.cast,
                director = details.director?.takeIf { it.isNotBlank() } ?: movie.director,
                genre = details.genre?.takeIf { it.isNotBlank() } ?: movie.genre,
                durationSecs = details.durationSecs?.takeIf { it > 0L } ?: movie.durationSecs,
                backdropUrl = details.movieImage?.takeIf { it.isNotBlank() } ?: movie.backdropUrl,
                containerExtension = details.containerExtension?.takeIf { it.isNotBlank() } ?: movie.containerExtension,
                rating = details.rating?.takeIf { it.isNotBlank() } ?: movie.rating,
                rating5Based = details.rating?.toFloatOrNull()?.let { it / 2f } ?: movie.rating5Based,
                hasFullDetailsLoaded = true
            )
            vodDao.update(updated)
            Result.success(updated)
        } catch (e: kotlinx.coroutines.CancellationException) {
            throw e
        } catch (e: Exception) {
            AppLogger.w(TAG, "Failed fetching deep VOD metadata for movie ${movie.id}: ${e.message}")
            Result.success(movie)
        }
    }

    suspend fun resolvePlayableUrl(movie: VodMovieEntity): String? {
        val creds = xtreamAccountManager.resolveCredentials(movie.accountId)
        if (creds !is XtreamAccountManager.CredentialsResolution.Success) {
            return null
        }
        return "vod_${movie.id}"
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

May I proceed with **Batch 6 (5 classes — Android TV Browse & Search)** (`data/repository/SeriesRepository.kt`, `tv/TvVodBrowseFragment.kt`, `tv/TvSeriesBrowseFragment.kt`, `tv/TvSearchFragment.kt`, `tv/TvBrowseFragment.kt`)?

Here is the Python script for **Batch 6 (5 classes — Android TV Browse & Search)**:

1. `data/repository/SeriesRepository.kt` — Exposes `getSeriesByCategory(...)` for Android TV category rows.
2. `tv/TvVodBrowseFragment.kt` — Populates VOD category rows from `vodRepository.getMoviesByCategory(...)` and enforces `ParentalGatekeeper`.
3. `tv/TvSeriesBrowseFragment.kt` — Populates Series category rows from `seriesRepository.getSeriesByCategory(...)` and enforces `ParentalGatekeeper`.
4. `tv/TvSearchFragment.kt` — Routes `LiveChannelItem` by `channelIdentity`, `VodMovieItem` to `VodDetailsActivity`, and `SeriesItem` to `SeriesDetailsActivity`.
5. `tv/TvBrowseFragment.kt` — Applies `ChannelOrganizationRepository` overrides, `ParentalGatekeeper` filtering, and `TvFocusCoordinator` focus restoration.

```python
#!/usr/bin/env python3
from pathlib import Path

BASE_DIR = Path("/home/username/AndroidStudioProjects/Ytvmate/app/src/main/java/vid/mat/app")

FILES = {
    "data/repository/SeriesRepository.kt": """package vid.mat.app.data.repository

import androidx.paging.Pager
import androidx.paging.PagingConfig
import androidx.paging.PagingData
import kotlinx.coroutines.CancellationException
import kotlinx.coroutines.Dispatchers
import kotlinx.coroutines.flow.Flow
import kotlinx.coroutines.withContext
import vid.mat.app.data.dao.SeriesDao
import vid.mat.app.data.model.EpisodeEntity
import vid.mat.app.data.model.SeasonEntity
import vid.mat.app.data.model.SeriesCategoryEntity
import vid.mat.app.data.model.SeriesEntity
import vid.mat.app.favorites.FavoriteTargetIds
import vid.mat.app.favorites.UnifiedFavoriteRepository
import vid.mat.app.favorites.model.FavoriteType
import vid.mat.app.provider.IptvProviderFactory
import vid.mat.app.security.XtreamAccountManager
import vid.mat.app.util.AppLogger
import javax.inject.Inject
import javax.inject.Singleton

@Singleton
class SeriesRepository @Inject constructor(
    private val seriesDao: SeriesDao,
    private val playlistSourceRepository: PlaylistSourceRepository,
    private val providerFactory: IptvProviderFactory,
    private val favoriteRepository: UnifiedFavoriteRepository,
    private val xtreamAccountManager: XtreamAccountManager
) {
    companion object {
        private const val TAG = "SeriesRepository"
        const val PAGE_SIZE = 30
        const val PREFETCH = 60
    }

    val continueWatching: Flow<List<SeriesEntity>> = seriesDao.getContinueWatchingSeries()
    val favorites: Flow<List<SeriesEntity>> = seriesDao.getFavorites()

    suspend fun getSeriesById(seriesDbId: Long): SeriesEntity? = seriesDao.getById(seriesDbId)

    fun getPagedSeries(
        sourceId: Long? = null,
        categoryId: String? = null,
        query: String = "",
        sortMode: Int = 0,
        favOnly: Boolean = false
    ): Flow<PagingData<SeriesEntity>> = Pager(
        config = PagingConfig(pageSize = PAGE_SIZE, prefetchDistance = PREFETCH, enablePlaceholders = false),
        pagingSourceFactory = { seriesDao.getPagedSeries(sourceId, categoryId, query, sortMode, favOnly) }
    ).flow

    fun getCategories(sourceId: Long? = null): Flow<List<SeriesCategoryEntity>> = seriesDao.getDistinctCategories(sourceId)
    fun countSeries(sourceId: Long? = null): Flow<Int> = seriesDao.countSeries(sourceId)

    suspend fun getSeriesByCategory(sourceId: Long?, categoryId: String, limit: Int = 25): List<SeriesEntity> =
        seriesDao.getSeriesByCategory(sourceId, categoryId, limit)

    suspend fun toggleFavorite(series: SeriesEntity) {
        favoriteRepository.toggleFavorite(
            targetId = FavoriteTargetIds.series(series.id),
            type = FavoriteType.SERIES,
            title = series.title,
            subtitle = series.categoryName,
            logoUrl = series.coverUrl,
            streamUrl = "series_${series.id}",
            groupTitle = series.categoryName
        )
    }

    suspend fun getSeasons(seriesDbId: Long): List<SeasonEntity> = seriesDao.getSeasonsForSeries(seriesDbId)

    suspend fun getEpisodes(seriesDbId: Long, seasonNumber: Int): List<EpisodeEntity> =
        seriesDao.getEpisodesForSeason(seriesDbId, seasonNumber)

    suspend fun getNextEpisode(seriesDbId: Long, currentSeason: Int, currentEpisode: Int): EpisodeEntity? =
        seriesDao.getNextEpisode(seriesDbId, currentSeason, currentEpisode)

    suspend fun updateEpisodeProgress(seriesDbId: Long, episode: EpisodeEntity, positionMs: Long, durationMs: Long) {
        val percent = if (durationMs > 0L) ((positionMs.toDouble() / durationMs) * 100).toInt().coerceIn(0, 100) else 0
        val isWatched = percent >= 90
        val now = System.currentTimeMillis()
        seriesDao.updateEpisodeProgress(episode.id, positionMs, durationMs, percent, isWatched, now)
        seriesDao.updateSeriesLastWatched(seriesDbId, episode.episodeIdStr, episode.seasonNumber, episode.episodeNum, now)
    }

    suspend fun fetchAndCacheFullSeriesDetails(series: SeriesEntity): Result<SeriesEntity> = withContext(Dispatchers.IO) {
        try {
            val source = playlistSourceRepository.getSource(series.sourceId)
                ?: return@withContext Result.failure(IllegalStateException("Provider source not found for series"))
            val provider = providerFactory.createProvider(source)
            val details = provider.fetchSeriesDetails(series.seriesId).getOrThrow()
            val seasonsList = details.seasons
            val episodesMap = details.episodes

            val updatedSeries = series.copy(
                plot = details.plot?.takeIf { it.isNotBlank() } ?: series.plot,
                cast = details.cast?.takeIf { it.isNotBlank() } ?: series.cast,
                director = details.director?.takeIf { it.isNotBlank() } ?: series.director,
                genre = details.genre?.takeIf { it.isNotBlank() } ?: series.genre,
                rating = details.rating?.takeIf { it.isNotBlank() } ?: series.rating,
                rating5Based = details.rating?.toFloatOrNull()?.let { it / 2f } ?: series.rating5Based,
                backdropUrl = details.coverUrl?.takeIf { it.isNotBlank() } ?: series.backdropUrl,
                totalSeasons = if (seasonsList.isNotEmpty()) seasonsList.size else episodesMap.keys.size,
                hasFullDetailsLoaded = true
            )
            seriesDao.update(updatedSeries)

            val seasonEntities = if (seasonsList.isNotEmpty()) {
                seasonsList.map { s ->
                    SeasonEntity(
                        seriesDbId = series.id,
                        seriesId = series.seriesId,
                        seasonNumber = s.seasonNumber,
                        name = s.name.ifBlank { "Season ${s.seasonNumber}" },
                        airDate = s.airDate,
                        episodeCount = s.episodeCount,
                        coverUrl = s.coverUrl,
                        overview = s.overview
                    )
                }
            } else {
                episodesMap.keys.mapNotNull { it.toIntOrNull() }.sorted().map { seasonNum ->
                    SeasonEntity(
                        seriesDbId = series.id,
                        seriesId = series.seriesId,
                        seasonNumber = seasonNum,
                        name = "Season $seasonNum",
                        episodeCount = episodesMap[seasonNum.toString()]?.size ?: 0
                    )
                }
            }
            if (seasonEntities.isNotEmpty()) {
                seriesDao.insertSeasons(seasonEntities)
            }

            val existingEpisodes = seriesDao.getAllEpisodesForSeries(series.id)
                .associateBy { it.seasonNumber to it.episodeNum }
            val seenKeys = HashSet<Pair<Int, Int>>()
            val toInsert = mutableListOf<EpisodeEntity>()
            val toUpdate = mutableListOf<EpisodeEntity>()

            episodesMap.forEach { (seasonNumStr, epList) ->
                val seasonNum = seasonNumStr.toIntOrNull() ?: 1
                epList.forEach { ep ->
                    val key = seasonNum to ep.episodeNumber
                    if (!seenKeys.add(key)) return@forEach
                    val existing = existingEpisodes[key]
                    if (existing == null) {
                        toInsert.add(
                            EpisodeEntity(
                                seriesDbId = series.id,
                                seriesId = series.seriesId,
                                episodeIdStr = ep.id,
                                seasonNumber = seasonNum,
                                episodeNum = ep.episodeNumber,
                                title = ep.title.ifBlank { "Episode ${ep.episodeNumber}" },
                                containerExtension = ep.containerExtension?.ifBlank { "mp4" } ?: "mp4",
                                durationSecs = ep.durationSecs ?: 0L,
                                plot = ep.plot,
                                movieImage = ep.movieImage,
                                directSource = ep.directSource,
                                customSid = ep.customSid
                            )
                        )
                    } else {
                        toUpdate.add(
                            existing.copy(
                                episodeIdStr = ep.id.ifBlank { existing.episodeIdStr },
                                title = ep.title.ifBlank { existing.title },
                                containerExtension = ep.containerExtension?.ifBlank { existing.containerExtension } ?: existing.containerExtension,
                                durationSecs = ep.durationSecs?.takeIf { it > 0L } ?: existing.durationSecs,
                                plot = ep.plot?.takeIf { it.isNotBlank() } ?: existing.plot,
                                movieImage = ep.movieImage?.takeIf { it.isNotBlank() } ?: existing.movieImage,
                                directSource = ep.directSource?.takeIf { it.isNotBlank() } ?: existing.directSource,
                                customSid = ep.customSid ?: existing.customSid
                            )
                        )
                    }
                }
            }

            if (toInsert.isNotEmpty()) seriesDao.insertEpisodes(toInsert)
            if (toUpdate.isNotEmpty()) seriesDao.updateEpisodes(toUpdate)

            if (seenKeys.isNotEmpty()) {
                val droppedIds = existingEpisodes.filterKeys { it !in seenKeys }.values.map { it.id }
                if (droppedIds.isNotEmpty()) {
                    droppedIds.chunked(250).forEach { seriesDao.deleteEpisodesByIds(series.id, it) }
                }
            }

            AppLogger.i(TAG, "Cached ${seasonEntities.size} seasons (${toInsert.size} new, ${toUpdate.size} updated episodes) for ${series.title}")
            Result.success(updatedSeries)
        } catch (e: CancellationException) {
            throw e
        } catch (e: Exception) {
            AppLogger.w(TAG, "Failed fetching deep series metadata: ${e.message}")
            Result.success(series)
        }
    }

    suspend fun getEpisodeById(episodeId: Long): EpisodeEntity? = seriesDao.getEpisodeById(episodeId)

    suspend fun resolvePlayableEpisodeUrl(series: SeriesEntity, episode: EpisodeEntity): String? {
        if (!episode.directSource.isNullOrBlank()) return "series_${episode.id}"
        val creds = xtreamAccountManager.resolveCredentials(series.accountId)
        if (creds !is XtreamAccountManager.CredentialsResolution.Success) return null
        return "series_${episode.id}"
    }
}
""",

    "tv/TvVodBrowseFragment.kt": """package vid.mat.app.tv

import android.content.Intent
import android.os.Bundle
import android.view.View
import androidx.core.content.ContextCompat
import androidx.fragment.app.viewModels
import androidx.leanback.app.BrowseSupportFragment
import androidx.leanback.widget.ArrayObjectAdapter
import androidx.leanback.widget.HeaderItem
import androidx.leanback.widget.ListRow
import androidx.leanback.widget.ListRowPresenter
import androidx.leanback.widget.OnItemViewClickedListener
import androidx.lifecycle.Lifecycle
import androidx.lifecycle.lifecycleScope
import androidx.lifecycle.repeatOnLifecycle
import dagger.hilt.android.AndroidEntryPoint
import kotlinx.coroutines.flow.collectLatest
import kotlinx.coroutines.launch
import vid.mat.app.R
import vid.mat.app.data.model.VodMovieEntity
import vid.mat.app.data.repository.VodRepository
import vid.mat.app.security.ParentalGatekeeper
import vid.mat.app.ui.vod.VodDetailsActivity
import vid.mat.app.viewmodel.VodCatalogViewModel
import javax.inject.Inject

@AndroidEntryPoint
class TvVodBrowseFragment : BrowseSupportFragment() {

    private val viewModel: VodCatalogViewModel by viewModels()
    private val rowsAdapter = ArrayObjectAdapter(ListRowPresenter())

    @Inject
    lateinit var vodRepository: VodRepository

    @Inject
    lateinit var parentalGatekeeper: ParentalGatekeeper

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        headersState = HEADERS_ENABLED
        isHeadersTransitionOnBackEnabled = true
        title = "Movies & VOD"
        brandColor = ContextCompat.getColor(requireContext(), R.color.tv_brand_color)
        adapter = rowsAdapter

        onItemViewClickedListener = OnItemViewClickedListener { _, item, _, _ ->
            if (item is VodMovieEntity) {
                val intent = Intent(requireContext(), VodDetailsActivity::class.java).apply {
                    putExtra(VodDetailsActivity.EXTRA_MOVIE_ID, item.id)
                }
                startActivity(intent)
            }
        }
    }

    override fun onViewCreated(view: View, savedInstanceState: Bundle?) {
        super.onViewCreated(view, savedInstanceState)
        observeData()
    }

    private fun observeData() {
        viewLifecycleOwner.lifecycleScope.launch {
            viewLifecycleOwner.repeatOnLifecycle(Lifecycle.State.STARTED) {
                viewModel.continueWatching.collectLatest { list ->
                    val existingFirst = rowsAdapter.size() > 0 && (rowsAdapter.get(0) as? ListRow)?.id == 1L
                    if (existingFirst) {
                        rowsAdapter.removeItems(0, 1)
                    }
                    val allowed = list.filterNot { parentalGatekeeper.isMovieRestricted(it) }
                    if (allowed.isNotEmpty()) {
                        val presenter = TvChannelPresenter(null)
                        val itemAdapter = ArrayObjectAdapter(presenter).apply { allowed.forEach { add(it) } }
                        rowsAdapter.add(0, ListRow(HeaderItem(1L, "Continue Watching"), itemAdapter))
                    }
                }
            }
        }

        viewLifecycleOwner.lifecycleScope.launch {
            viewLifecycleOwner.repeatOnLifecycle(Lifecycle.State.STARTED) {
                viewModel.categories.collectLatest { categories ->
                    val hasContinue = rowsAdapter.size() > 0 && (rowsAdapter.get(0) as? ListRow)?.id == 1L
                    val startIndex = if (hasContinue) 1 else 0
                    if (rowsAdapter.size() > startIndex) {
                        rowsAdapter.removeItems(startIndex, rowsAdapter.size() - startIndex)
                    }

                    val presenter = TvChannelPresenter(null)
                    categories.take(15).forEachIndexed { index, category ->
                        if (parentalGatekeeper.isCategoryRestricted(category.categoryName)) return@forEachIndexed
                        val rowId = (index + 10).toLong()
                        val movies = vodRepository.getMoviesByCategory(category.sourceId, category.categoryId, 25)
                            .filterNot { parentalGatekeeper.isMovieRestricted(it) }
                        if (movies.isNotEmpty()) {
                            val itemAdapter = ArrayObjectAdapter(presenter).apply { movies.forEach { add(it) } }
                            rowsAdapter.add(ListRow(HeaderItem(rowId, category.categoryName), itemAdapter))
                        }
                    }
                }
            }
        }
    }
}
""",

    "tv/TvSeriesBrowseFragment.kt": """package vid.mat.app.tv

import android.content.Intent
import android.os.Bundle
import android.view.View
import androidx.core.content.ContextCompat
import androidx.fragment.app.viewModels
import androidx.leanback.app.BrowseSupportFragment
import androidx.leanback.widget.ArrayObjectAdapter
import androidx.leanback.widget.HeaderItem
import androidx.leanback.widget.ListRow
import androidx.leanback.widget.ListRowPresenter
import androidx.leanback.widget.OnItemViewClickedListener
import androidx.lifecycle.Lifecycle
import androidx.lifecycle.lifecycleScope
import androidx.lifecycle.repeatOnLifecycle
import dagger.hilt.android.AndroidEntryPoint
import kotlinx.coroutines.flow.collectLatest
import kotlinx.coroutines.launch
import vid.mat.app.R
import vid.mat.app.data.model.SeriesEntity
import vid.mat.app.data.repository.SeriesRepository
import vid.mat.app.security.ParentalGatekeeper
import vid.mat.app.ui.series.SeriesDetailsActivity
import vid.mat.app.viewmodel.SeriesCatalogViewModel
import javax.inject.Inject

@AndroidEntryPoint
class TvSeriesBrowseFragment : BrowseSupportFragment() {

    private val viewModel: SeriesCatalogViewModel by viewModels()
    private val rowsAdapter = ArrayObjectAdapter(ListRowPresenter())

    @Inject
    lateinit var seriesRepository: SeriesRepository

    @Inject
    lateinit var parentalGatekeeper: ParentalGatekeeper

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        headersState = HEADERS_ENABLED
        isHeadersTransitionOnBackEnabled = true
        title = "TV Series & Shows"
        brandColor = ContextCompat.getColor(requireContext(), R.color.tv_brand_color)
        adapter = rowsAdapter

        onItemViewClickedListener = OnItemViewClickedListener { _, item, _, _ ->
            if (item is SeriesEntity) {
                val intent = Intent(requireContext(), SeriesDetailsActivity::class.java).apply {
                    putExtra(SeriesDetailsActivity.EXTRA_SERIES_ID, item.id)
                }
                startActivity(intent)
            }
        }
    }

    override fun onViewCreated(view: View, savedInstanceState: Bundle?) {
        super.onViewCreated(view, savedInstanceState)
        observeData()
    }

    private fun observeData() {
        viewLifecycleOwner.lifecycleScope.launch {
            viewLifecycleOwner.repeatOnLifecycle(Lifecycle.State.STARTED) {
                viewModel.continueWatching.collectLatest { list ->
                    val existingFirst = rowsAdapter.size() > 0 && (rowsAdapter.get(0) as? ListRow)?.id == 1L
                    if (existingFirst) {
                        rowsAdapter.removeItems(0, 1)
                    }
                    val allowed = list.filterNot { parentalGatekeeper.isSeriesRestricted(it) }
                    if (allowed.isNotEmpty()) {
                        val presenter = TvChannelPresenter(null)
                        val itemAdapter = ArrayObjectAdapter(presenter).apply { allowed.forEach { add(it) } }
                        rowsAdapter.add(0, ListRow(HeaderItem(1L, "Continue Watching"), itemAdapter))
                    }
                }
            }
        }

        viewLifecycleOwner.lifecycleScope.launch {
            viewLifecycleOwner.repeatOnLifecycle(Lifecycle.State.STARTED) {
                viewModel.categories.collectLatest { categories ->
                    val hasContinue = rowsAdapter.size() > 0 && (rowsAdapter.get(0) as? ListRow)?.id == 1L
                    val startIndex = if (hasContinue) 1 else 0
                    if (rowsAdapter.size() > startIndex) {
                        rowsAdapter.removeItems(startIndex, rowsAdapter.size() - startIndex)
                    }

                    val presenter = TvChannelPresenter(null)
                    categories.take(15).forEachIndexed { index, category ->
                        if (parentalGatekeeper.isCategoryRestricted(category.categoryName)) return@forEachIndexed
                        val rowId = (index + 20).toLong()
                        val seriesList = seriesRepository.getSeriesByCategory(category.sourceId, category.categoryId, 25)
                            .filterNot { parentalGatekeeper.isSeriesRestricted(it) }
                        if (seriesList.isNotEmpty()) {
                            val itemAdapter = ArrayObjectAdapter(presenter).apply { seriesList.forEach { add(it) } }
                            rowsAdapter.add(ListRow(HeaderItem(rowId, category.categoryName), itemAdapter))
                        }
                    }
                }
            }
        }
    }
}
""",

    "tv/TvSearchFragment.kt": """package vid.mat.app.tv

import android.content.Intent
import android.os.Bundle
import android.view.View
import androidx.fragment.app.viewModels
import androidx.leanback.app.SearchSupportFragment
import androidx.leanback.widget.ArrayObjectAdapter
import androidx.leanback.widget.HeaderItem
import androidx.leanback.widget.ListRow
import androidx.leanback.widget.ListRowPresenter
import androidx.leanback.widget.ObjectAdapter
import androidx.leanback.widget.OnItemViewClickedListener
import androidx.lifecycle.Lifecycle
import androidx.lifecycle.lifecycleScope
import androidx.lifecycle.repeatOnLifecycle
import dagger.hilt.android.AndroidEntryPoint
import kotlinx.coroutines.launch
import vid.mat.app.search.model.SearchItem
import vid.mat.app.ui.series.SeriesDetailsActivity
import vid.mat.app.ui.vod.VodDetailsActivity
import vid.mat.app.viewmodel.SearchViewModel

@AndroidEntryPoint
class TvSearchFragment : SearchSupportFragment(), SearchSupportFragment.SearchResultProvider {

    private val viewModel: SearchViewModel by viewModels()
    private val rowsAdapter = ArrayObjectAdapter(ListRowPresenter())

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setSearchResultProvider(this)
        setOnItemViewClickedListener(OnItemViewClickedListener { _, item, _, _ ->
            if (item is SearchItem) {
                when (item) {
                    is SearchItem.LiveChannelItem ->
                        playMedia(item.id, item.title, item.logoUrl, item.groupTitle, 0L)
                    is SearchItem.VodMovieItem -> {
                        val movieDbId = item.movieDbId
                        if (movieDbId != null && movieDbId > 0L) {
                            startActivity(
                                Intent(requireContext(), VodDetailsActivity::class.java).apply {
                                    putExtra(VodDetailsActivity.EXTRA_MOVIE_ID, movieDbId)
                                }
                            )
                        } else {
                            playMedia(item.streamUrl, item.title, item.logoUrl, item.groupTitle, 0L)
                        }
                    }
                    is SearchItem.SeriesItem -> {
                        val seriesDbId = if (item.seriesDbId > 0L) item.seriesDbId else item.seriesId.toLong()
                        if (seriesDbId > 0L) {
                            startActivity(
                                Intent(requireContext(), SeriesDetailsActivity::class.java).apply {
                                    putExtra(SeriesDetailsActivity.EXTRA_SERIES_ID, seriesDbId)
                                }
                            )
                        }
                    }
                    is SearchItem.HistorySearchItem ->
                        playMedia(item.streamUrl, item.title, item.logoUrl, item.groupTitle, item.resumePositionMs)
                    else -> Unit
                }
            }
        })
    }

    override fun onViewCreated(view: View, savedInstanceState: Bundle?) {
        super.onViewCreated(view, savedInstanceState)
        observeResults()
    }

    private fun observeResults() {
        viewLifecycleOwner.lifecycleScope.launch {
            viewLifecycleOwner.repeatOnLifecycle(Lifecycle.State.STARTED) {
                viewModel.uiState.collect { state ->
                    rowsAdapter.clear()
                    if (state is SearchViewModel.SearchUiState.Success) {
                        val presenter = TvChannelPresenter(null)

                        if (state.results.liveChannels.isNotEmpty()) {
                            val channelAdapter = ArrayObjectAdapter(presenter).apply {
                                state.results.liveChannels.forEach { add(it) }
                            }
                            rowsAdapter.add(ListRow(HeaderItem(0, "Live Channels"), channelAdapter))
                        }

                        if (state.results.movies.isNotEmpty()) {
                            val movieAdapter = ArrayObjectAdapter(presenter).apply {
                                state.results.movies.forEach { add(it) }
                            }
                            rowsAdapter.add(ListRow(HeaderItem(1, "Movies (VOD)"), movieAdapter))
                        }

                        if (state.results.series.isNotEmpty()) {
                            val seriesAdapter = ArrayObjectAdapter(presenter).apply {
                                state.results.series.forEach { add(it) }
                            }
                            rowsAdapter.add(ListRow(HeaderItem(2, "Series"), seriesAdapter))
                        }
                    }
                }
            }
        }
    }

    override fun getResultsAdapter(): ObjectAdapter = rowsAdapter

    override fun onQueryTextChange(newQuery: String?): Boolean {
        viewModel.onQueryChanged(newQuery.orEmpty())
        return true
    }

    override fun onQueryTextSubmit(query: String?): Boolean {
        viewModel.onQueryChanged(query.orEmpty())
        return true
    }

    private fun playMedia(url: String, title: String, logo: String?, group: String?, resumeMs: Long) {
        startActivity(Intent(requireContext(), TvPlaybackActivity::class.java).apply {
            putExtra(TvPlaybackActivity.EXTRA_URL, url)
            putExtra(TvPlaybackActivity.EXTRA_TITLE, title)
            putExtra(TvPlaybackActivity.EXTRA_LOGO, logo)
            putExtra(TvPlaybackActivity.EXTRA_RESUME_MS, resumeMs)
            putExtra("group_title", group)
        })
    }
}
""",

    "tv/TvBrowseFragment.kt": """package vid.mat.app.tv

import android.content.Intent
import android.os.Bundle
import android.view.View
import androidx.core.content.ContextCompat
import androidx.fragment.app.viewModels
import androidx.leanback.app.BrowseSupportFragment
import androidx.leanback.widget.ArrayObjectAdapter
import androidx.leanback.widget.HeaderItem
import androidx.leanback.widget.ListRow
import androidx.leanback.widget.ListRowPresenter
import androidx.leanback.widget.OnItemViewClickedListener
import androidx.lifecycle.Lifecycle
import androidx.lifecycle.lifecycleScope
import androidx.lifecycle.repeatOnLifecycle
import dagger.hilt.android.AndroidEntryPoint
import kotlinx.coroutines.flow.collectLatest
import kotlinx.coroutines.launch
import vid.mat.app.R
import vid.mat.app.data.model.Channel
import vid.mat.app.data.repository.ChannelOrganizationRepository
import vid.mat.app.data.repository.ChannelRepository
import vid.mat.app.epg.EpgViewModel
import vid.mat.app.history.WatchHistory
import vid.mat.app.history.WatchHistoryViewModel
import vid.mat.app.security.ParentalGatekeeper
import vid.mat.app.tv.navigation.TvFocusCoordinator
import vid.mat.app.util.performance.DeviceCapabilityProfile
import javax.inject.Inject

@AndroidEntryPoint
class TvBrowseFragment : BrowseSupportFragment() {

    private val rowsAdapter = ArrayObjectAdapter(ListRowPresenter())
    private val historyViewModel: WatchHistoryViewModel by viewModels()
    private val epgViewModel: EpgViewModel by viewModels()
    private val focusCoordinator = TvFocusCoordinator()

    @Inject
    lateinit var channelRepository: ChannelRepository

    @Inject
    lateinit var parentalGatekeeper: ParentalGatekeeper

    @Inject
    lateinit var channelOrganizationRepository: ChannelOrganizationRepository

    private val rowMap = mutableMapOf<Long, Int>()

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        headersState = HEADERS_ENABLED
        isHeadersTransitionOnBackEnabled = true
        title = getString(R.string.app_name)
        brandColor = ContextCompat.getColor(requireContext(), R.color.tv_brand_color)
        adapter = rowsAdapter
    }

    override fun onViewCreated(view: View, savedInstanceState: Bundle?) {
        super.onViewCreated(view, savedInstanceState)

        onItemViewClickedListener = OnItemViewClickedListener { itemViewHolder, item, _, _ ->
            itemViewHolder?.view?.let { focusCoordinator.recordFocus(it) }
            when (item) {
                is Channel -> launchPlayer(item.channelIdentity, item.title, item.logoUrl, item.groupTitle, 0L)
                is WatchHistory -> launchPlayer(item.url, item.title, item.logoUrl, item.groupTitle, item.resumePositionMs)
            }
        }

        observeRows()
    }

    override fun onResume() {
        super.onResume()
        focusCoordinator.restoreFocus(view)
    }

    private fun observeRows() {
        val isLowEnd = DeviceCapabilityProfile.isLowEndHardware(requireContext())
        val maxChannelPerGroup = if (isLowEnd) 15 else 25
        val maxGroups = if (isLowEnd) 8 else 15

        viewLifecycleOwner.lifecycleScope.launch {
            viewLifecycleOwner.repeatOnLifecycle(Lifecycle.State.STARTED) {
                historyViewModel.continueWatching.collectLatest { items ->
                    val allowed = items.filterNot {
                        parentalGatekeeper.isCategoryRestricted(it.groupTitle) ||
                            parentalGatekeeper.isTargetRestricted(it.url)
                    }
                    if (allowed.isNotEmpty()) {
                        upsertRow(ROW_CONTINUE, "Continue Watching", allowed.take(10))
                    } else {
                        removeRow(ROW_CONTINUE)
                    }
                }
            }
        }

        viewLifecycleOwner.lifecycleScope.launch {
            viewLifecycleOwner.repeatOnLifecycle(Lifecycle.State.STARTED) {
                channelRepository.favorites.collectLatest { items ->
                    val allowed = items.filterNot { parentalGatekeeper.isChannelRestricted(it) }
                    if (allowed.isNotEmpty()) {
                        upsertRow(ROW_FAVORITES, "Favourites", allowed.take(maxChannelPerGroup))
                    } else {
                        removeRow(ROW_FAVORITES)
                    }
                }
            }
        }

        viewLifecycleOwner.lifecycleScope.launch {
            viewLifecycleOwner.repeatOnLifecycle(Lifecycle.State.STARTED) {
                channelRepository.recents.collectLatest { items ->
                    val allowed = items.filterNot { parentalGatekeeper.isChannelRestricted(it) }
                    if (allowed.isNotEmpty()) {
                        upsertRow(ROW_RECENTS, "Recently Watched", allowed.take(maxChannelPerGroup))
                    } else {
                        removeRow(ROW_RECENTS)
                    }
                }
            }
        }

        viewLifecycleOwner.lifecycleScope.launch {
            viewLifecycleOwner.repeatOnLifecycle(Lifecycle.State.STARTED) {
                channelRepository.distinctGroups.collectLatest { groups ->
                    val visibleGroups = groups.filterNot { parentalGatekeeper.isCategoryRestricted(it) }.take(maxGroups)
                    visibleGroups.forEachIndexed { i, groupName ->
                        val rowId = CATEGORY_ROW_BASE + i
                        val organized = channelOrganizationRepository.getOrganizedChannels(
                            groupFilter = groupName,
                            includeHidden = false
                        ).map { org ->
                            org.channel.copy(
                                title = org.displayTitle,
                                groupTitle = org.displayGroup,
                                logoUrl = org.displayLogo,
                                channelNumber = org.displayNumber
                            )
                        }.filterNot { parentalGatekeeper.isChannelRestricted(it) }
                            .take(maxChannelPerGroup)

                        if (organized.isNotEmpty()) {
                            upsertRow(rowId, groupName, organized)
                        } else {
                            removeRow(rowId)
                        }
                    }
                }
            }
        }
    }

    private fun upsertRow(rowId: Long, header: String, items: List<Any>) {
        val presenter = TvChannelPresenter(epgViewModel)
        val itemAdapter = ArrayObjectAdapter(presenter).also { a ->
            items.forEach { a.add(it) }
        }
        val row = ListRow(rowId, HeaderItem(rowId, header), itemAdapter)
        val existing = rowMap[rowId]
        if (existing != null && existing < rowsAdapter.size()) {
            rowsAdapter.replace(existing, row)
        } else {
            val pos = when (rowId) {
                ROW_CONTINUE  -> 0
                ROW_FAVORITES -> minOf(1, rowsAdapter.size())
                ROW_RECENTS   -> minOf(2, rowsAdapter.size())
                else          -> rowsAdapter.size()
            }
            rowsAdapter.add(pos, row)
            rebuildRowMap()
        }
    }

    private fun removeRow(rowId: Long) {
        val pos = rowMap[rowId] ?: return
        if (pos < rowsAdapter.size()) {
            rowsAdapter.removeItems(pos, 1)
            rebuildRowMap()
        }
    }

    private fun rebuildRowMap() {
        rowMap.clear()
        (0 until rowsAdapter.size()).forEach { i ->
            (rowsAdapter.get(i) as? ListRow)?.let { rowMap[it.id] = i }
        }
    }

    private fun launchPlayer(url: String, title: String, logo: String?, group: String?, resumeMs: Long) {
        startActivity(Intent(requireContext(), TvPlaybackActivity::class.java).apply {
            putExtra(TvPlaybackActivity.EXTRA_URL, url)
            putExtra(TvPlaybackActivity.EXTRA_TITLE, title)
            putExtra(TvPlaybackActivity.EXTRA_LOGO, logo)
            putExtra(TvPlaybackActivity.EXTRA_RESUME_MS, resumeMs)
            putExtra("group_title", group)
        })
    }

    companion object {
        const val ROW_CONTINUE      = 1L
        const val ROW_FAVORITES     = 2L
        const val ROW_RECENTS       = 3L
        const val CATEGORY_ROW_BASE = 100L
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

May I proceed with **Batch 7 (5 classes — Android TV Playback/Surfing & Encrypted Backup)** (`player/surfing/ChannelSurfingCoordinator.kt`, `tv/TvPlaybackActivity.kt`, `tv/dialog/TvChannelActionMenuDialog.kt`, `backup/model/BackupModels.kt`, `backup/ui/BackupCreateDialogFragment.kt`)?

Here is the Python script for **Batch 7 (5 classes — TV Playback/Surfing & Backup Models/Dialog)**:

1. `player/surfing/ChannelSurfingCoordinator.kt` — Syncs `navController.updateChannelList` on channel switches and prevents `FastChannelZapper` from pre-warming unresolved `/X-AUTH-INJECT/` or `xtream://` URLs.
2. `tv/TvPlaybackActivity.kt` — Wires `ChannelSurfingCoordinator` and `TvKeyNavigationHandler` for D-pad/number-pad channel surfing and uses safe `stopService(...)` in `onDestroy()`.
3. `tv/dialog/TvChannelActionMenuDialog.kt` — Uses a rotation-safe no-arg constructor with `Bundle` arguments.
4. `backup/model/BackupModels.kt` — Adds `isFavorite` to `ChannelOverrideBackupDto` and `isUserCreated` to `CustomGroupBackupDto`.
5. `backup/ui/BackupCreateDialogFragment.kt` — Preserves `targetPassphrase` across configuration changes via `onSaveInstanceState`.

```python
#!/usr/bin/env python3
from pathlib import Path

BASE_DIR = Path("/home/username/AndroidStudioProjects/Ytvmate/app/src/main/java/vid/mat/app")

FILES = {
    "player/surfing/ChannelSurfingCoordinator.kt": """package vid.mat.app.player.surfing

import android.view.KeyEvent
import android.view.ViewGroup
import androidx.appcompat.app.AppCompatActivity
import kotlinx.coroutines.CoroutineScope
import kotlinx.coroutines.Dispatchers
import kotlinx.coroutines.Job
import kotlinx.coroutines.SupervisorJob
import kotlinx.coroutines.cancel
import kotlinx.coroutines.delay
import kotlinx.coroutines.flow.MutableStateFlow
import kotlinx.coroutines.flow.StateFlow
import kotlinx.coroutines.flow.asStateFlow
import kotlinx.coroutines.launch
import vid.mat.app.data.PlayMode
import vid.mat.app.data.model.Channel
import vid.mat.app.data.repository.ChannelOrganizationRepository
import vid.mat.app.data.repository.ChannelRepository
import vid.mat.app.epg.EpgViewModel
import vid.mat.app.navigation.ChannelNavigationController
import vid.mat.app.network.OkHttpStreamingClient
import vid.mat.app.player.FastChannelZapper
import vid.mat.app.player.NewPlayer
import vid.mat.app.ui.player.LiveTvOsdController
import vid.mat.app.util.AppLogger

class ChannelSurfingCoordinator(
    private val activity: AppCompatActivity,
    private val player: NewPlayer,
    private val channelRepository: ChannelRepository,
    private val organizationRepository: ChannelOrganizationRepository?,
    private val epgViewModel: EpgViewModel?,
    private val osdContainer: ViewGroup
) {

    companion object {
        private const val TAG = "ChannelSurfingCoordinator"
    }

    private val scope = CoroutineScope(SupervisorJob() + Dispatchers.Main)
    private val zapper = FastChannelZapper(activity.applicationContext)
    private val navController = ChannelNavigationController(
        player, channelRepository, organizationRepository,
        scope = scope
    )

    private val osdController = LiveTvOsdController(
        activity = activity,
        container = osdContainer,
        onChannelSelected = { selected -> switchChannel(selected) },
        epgViewModel = epgViewModel
    )

    private var activeChannelsList: List<Channel> = emptyList()
    private var currentChannelIndex: Int = -1
    private var preWarmJob: Job? = null

    private val _currentChannel = MutableStateFlow<Channel?>(null)
    val currentChannel: StateFlow<Channel?> = _currentChannel.asStateFlow()

    fun initialize(initialChannelIdentity: String?) {
        scope.launch {
            val organized = organizationRepository?.getOrganizedChannels(includeHidden = false)
            val channels = if (!organized.isNullOrEmpty()) {
                organized.map { it.channel }
            } else {
                channelRepository.getAllChannels(50_000)
            }

            activeChannelsList = channels
            val initialIndex = channels.indexOfFirst { it.channelIdentity == initialChannelIdentity }
            currentChannelIndex = if (initialIndex >= 0) initialIndex else 0

            navController.updateChannelList(channels, channels.getOrNull(currentChannelIndex)?.channelId)

            val active = channels.getOrNull(currentChannelIndex)
            _currentChannel.value = active

            AppLogger.i(TAG, "Surfing coordinator initialized with ${channels.size} channels. Active: ${active?.title}")
            preWarmAdjacentChannels()
        }
    }

    fun nextChannel() {
        if (activeChannelsList.isEmpty()) return
        val nextIdx = (currentChannelIndex + 1) % activeChannelsList.size
        switchToIndex(nextIdx)
    }

    fun previousChannel() {
        if (activeChannelsList.isEmpty()) return
        val prevIdx = if (currentChannelIndex - 1 < 0) activeChannelsList.size - 1 else currentChannelIndex - 1
        switchToIndex(prevIdx)
    }

    fun toggleLastChannel() {
        navController.toggleLastChannel()
    }

    fun showQuickChannelOsd() {
        if (activeChannelsList.isNotEmpty()) {
            osdController.showOsd(activeChannelsList, currentChannelIndex)
        }
    }

    fun handleKeyEvent(event: KeyEvent): Boolean {
        if (event.action != KeyEvent.ACTION_DOWN) return false

        return when (event.keyCode) {
            KeyEvent.KEYCODE_DPAD_UP,
            KeyEvent.KEYCODE_CHANNEL_UP -> {
                nextChannel()
                true
            }
            KeyEvent.KEYCODE_DPAD_DOWN,
            KeyEvent.KEYCODE_CHANNEL_DOWN -> {
                previousChannel()
                true
            }
            KeyEvent.KEYCODE_0, KeyEvent.KEYCODE_1, KeyEvent.KEYCODE_2,
            KeyEvent.KEYCODE_3, KeyEvent.KEYCODE_4, KeyEvent.KEYCODE_5,
            KeyEvent.KEYCODE_6, KeyEvent.KEYCODE_7, KeyEvent.KEYCODE_8,
            KeyEvent.KEYCODE_9 -> {
                val digit = event.keyCode - KeyEvent.KEYCODE_0
                osdController.handleDigitKey(digit, activeChannelsList)
                navController.onDigitKey(digit)
                true
            }
            KeyEvent.KEYCODE_LAST_CHANNEL,
            KeyEvent.KEYCODE_MEDIA_PREVIOUS -> {
                toggleLastChannel()
                true
            }
            KeyEvent.KEYCODE_DPAD_LEFT,
            KeyEvent.KEYCODE_MENU,
            KeyEvent.KEYCODE_TV_CONTENTS_MENU -> {
                showQuickChannelOsd()
                true
            }
            else -> false
        }
    }

    private fun switchToIndex(index: Int) {
        val target = activeChannelsList.getOrNull(index) ?: return
        currentChannelIndex = index
        switchChannel(target)
    }

    private fun switchChannel(channel: Channel) {
        AppLogger.i(TAG, "Zapping to channel: ${channel.title} (#${channel.channelNumber ?: (currentChannelIndex + 1)})")
        _currentChannel.value = channel
        navController.updateChannelList(activeChannelsList, channel.channelId)
        player.playStream(channel.channelIdentity, PlayMode.FULLSCREEN_VIDEO)
        preWarmAdjacentChannels()
    }

    private fun preWarmAdjacentChannels() {
        preWarmJob?.cancel()
        preWarmJob = scope.launch(Dispatchers.IO) {
            delay(250)

            if (activeChannelsList.size < 2 || currentChannelIndex < 0) return@launch

            val nextIdx = (currentChannelIndex + 1) % activeChannelsList.size
            val nextChannel = activeChannelsList.getOrNull(nextIdx) ?: return@launch

            val endpoints = channelRepository.getEndpointsForChannel(nextChannel.channelId)
            val primaryEndpoint = endpoints.firstOrNull() ?: return@launch
            if (primaryEndpoint.url.contains("/X-AUTH-INJECT/") || primaryEndpoint.url.startsWith("xtream://")) {
                return@launch
            }

            val headers = OkHttpStreamingClient.StreamHeaders(
                userAgent = primaryEndpoint.userAgent,
                referer = primaryEndpoint.referer?.value,
                cookie = primaryEndpoint.cookie?.value,
                authorization = primaryEndpoint.authorization?.value
            )

            AppLogger.d(TAG, "FastZapper pre-warming next adjacent channel: ${nextChannel.title}")
            zapper.preWarm(primaryEndpoint.url, headers)
        }
    }

    fun destroy() {
        preWarmJob?.cancel()
        zapper.release()
        scope.cancel()
    }
}
""",

    "tv/TvPlaybackActivity.kt": """package vid.mat.app.tv

import android.content.Intent
import android.os.Build
import android.os.Bundle
import android.view.KeyEvent
import android.view.ViewGroup
import android.view.WindowManager
import androidx.activity.OnBackPressedCallback
import androidx.annotation.OptIn
import androidx.appcompat.app.AppCompatActivity
import androidx.lifecycle.Lifecycle
import androidx.lifecycle.SavedStateHandle
import androidx.lifecycle.ViewModelProvider
import androidx.lifecycle.lifecycleScope
import androidx.lifecycle.repeatOnLifecycle
import androidx.media3.common.Player
import androidx.media3.common.util.UnstableApi
import dagger.hilt.android.AndroidEntryPoint
import kotlinx.coroutines.Dispatchers
import kotlinx.coroutines.Job
import kotlinx.coroutines.delay
import kotlinx.coroutines.isActive
import kotlinx.coroutines.launch
import vid.mat.app.data.repository.ChannelOrganizationRepository
import vid.mat.app.data.repository.ChannelRepository
import vid.mat.app.history.WatchHistoryRepository
import vid.mat.app.lifecycle.PlayerSavedStateRestorer
import vid.mat.app.player.AutoFrameRateManager
import vid.mat.app.player.NewPlayerImpl
import vid.mat.app.player.surfing.ChannelSurfingCoordinator
import vid.mat.app.profile.ActiveProfileProvider
import vid.mat.app.ui.player.PlayerMediaData
import vid.mat.app.uimodel.NewPlayerViewModel
import vid.mat.app.util.PlaybackService
import vid.mat.app.util.SensitiveDataSanitizer
import javax.inject.Inject

@AndroidEntryPoint
@OptIn(UnstableApi::class)
class TvPlaybackActivity : AppCompatActivity() {

    companion object {
        private const val TAG            = "TvPlaybackActivity"
        const val EXTRA_URL       = "tv_url"
        const val EXTRA_TITLE     = "tv_title"
        const val EXTRA_LOGO      = "tv_logo"
        const val EXTRA_RESUME_MS = "tv_resume_ms"
    }

    private lateinit var viewModel: NewPlayerViewModel
    private var currentMediaData: PlayerMediaData? = null
    private var hasSentStartIntent = false
    private var lastAttachedPlayer: Player? = null
    private var positionSaveJob: Job? = null
    private var surfingCoordinator: ChannelSurfingCoordinator? = null

    @Inject
    lateinit var activeProfileProvider: ActiveProfileProvider

    @Inject
    lateinit var watchHistoryRepo: WatchHistoryRepository

    @Inject
    lateinit var channelRepository: ChannelRepository

    @Inject
    lateinit var organizationRepository: ChannelOrganizationRepository

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        window.addFlags(WindowManager.LayoutParams.FLAG_KEEP_SCREEN_ON)

        viewModel = ViewModelProvider(this)[NewPlayerViewModel::class.java]

        if (savedInstanceState != null) {
            val handle = SavedStateHandle()
            for (key in savedInstanceState.keySet()) {
                handle[key] = savedInstanceState.get(key)
            }
            val restored = PlayerSavedStateRestorer.restorePlayerState(handle)
            if (restored != null) {
                currentMediaData = restored.mediaData
                viewModel.setAudioDelay(restored.audioDelayMs)
                viewModel.setSubtitleDelay(restored.subtitleDelayMs)
            }
        }

        if (currentMediaData == null) {
            val rawUrl = intent.getStringExtra(EXTRA_URL) ?: run { finish(); return }
            val title = SensitiveDataSanitizer.sanitizeText(intent.getStringExtra(EXTRA_TITLE) ?: "Stream")
            val logo = intent.getStringExtra(EXTRA_LOGO)
            val resume = intent.getLongExtra(EXTRA_RESUME_MS, 0L)
            currentMediaData = PlayerMediaData(
                mediaPath = rawUrl,
                mediaTitle = title,
                thumbnailUrl = logo.orEmpty(),
                resumePosition = resume,
                groupTitle = intent.getStringExtra("group_title")
            )
        }

        val data = currentMediaData ?: run { finish(); return }

        val videoFragment = TvVideoFragment.newInstance(data.mediaTitle, data.thumbnailUrl)
        supportFragmentManager.beginTransaction()
            .replace(android.R.id.content, videoFragment)
            .commit()

        val rootContainer = findViewById<ViewGroup>(android.R.id.content)
        surfingCoordinator = ChannelSurfingCoordinator(
            activity = this,
            player = viewModel.newPlayer,
            channelRepository = channelRepository,
            organizationRepository = organizationRepository,
            epgViewModel = null,
            osdContainer = rootContainer
        ).also {
            it.initialize(data.mediaPath)
        }

        onBackPressedDispatcher.addCallback(this, object : OnBackPressedCallback(true) {
            override fun handleOnBackPressed() {
                finish()
            }
        })

        lifecycleScope.launch {
            repeatOnLifecycle(Lifecycle.State.STARTED) {
                viewModel.newPlayer.exoPlayer.collect { player ->
                    if (player != null && player !== lastAttachedPlayer) {
                        lastAttachedPlayer = player
                        videoFragment.attachPlayer(player, data.mediaTitle)
                    }
                }
            }
        }

        if (!hasSentStartIntent) {
            hasSentStartIntent = true
            val activeMediaId = viewModel.newPlayer.currentSession.value?.mediaId
            if (activeMediaId != data.mediaPath || viewModel.newPlayer.exoPlayer.value == null) {
                startPlaybackService(data)
            }
        }
    }

    override fun dispatchKeyEvent(event: KeyEvent): Boolean {
        if (TvKeyNavigationHandler.dispatchKeyEvent(event, surfingCoordinator)) {
            return true
        }
        return super.dispatchKeyEvent(event)
    }

    override fun onSaveInstanceState(outState: Bundle) {
        super.onSaveInstanceState(outState)
        val handle = SavedStateHandle()
        val pos = viewModel.newPlayer.exoPlayer.value?.currentPosition ?: currentMediaData?.resumePosition ?: 0L
        val audioDelay = viewModel.uiState.value.audioDelayMs
        val subDelay = viewModel.uiState.value.subtitleDelayMs

        PlayerSavedStateRestorer.savePlayerState(
            handle = handle,
            session = viewModel.newPlayer.currentSession.value,
            data = currentMediaData,
            currentPositionMs = pos,
            audioDelayMs = audioDelay,
            subtitleDelayMs = subDelay,
            profileId = activeProfileProvider.activeProfile.value.id
        )
        for (key in handle.keys()) {
            val value = handle.get<Any>(key)
            when (value) {
                is String -> outState.putString(key, value)
                is Long -> outState.putLong(key, value)
                is Int -> outState.putInt(key, value)
            }
        }
    }

    private fun startPlaybackService(data: PlayerMediaData) {
        val serviceIntent = Intent(this, PlaybackService::class.java).apply {
            action = PlaybackService.ACTION_START_PLAYBACK
            putExtra(PlaybackService.EXTRA_MEDIA_PATH, data.mediaPath)
            putExtra(PlaybackService.EXTRA_MEDIA_TITLE, data.mediaTitle)
            putExtra(PlaybackService.EXTRA_THUMBNAIL_URI, data.thumbnailUrl)
            putExtra(PlaybackService.EXTRA_RESUME_POSITION, data.resumePosition)
        }

        if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.O) {
            startForegroundService(serviceIntent)
        } else {
            startService(serviceIntent)
        }
    }

    override fun onResume() {
        super.onResume()
        setupPositionSaver()
        AutoFrameRateManager.onActivityResumed(this)
        (viewModel.newPlayer as? NewPlayerImpl)?.playbackIntelligence?.bindActivity(this)
    }

    override fun onPause() {
        super.onPause()
        positionSaveJob?.cancel()
        saveWatchProgress()
        (viewModel.newPlayer as? NewPlayerImpl)?.playbackIntelligence?.bindActivity(null)
        AutoFrameRateManager.onActivityPaused(this)
    }

    private fun setupPositionSaver() {
        positionSaveJob?.cancel()
        positionSaveJob = lifecycleScope.launch {
            while (isActive) {
                delay(15_000L)
                saveWatchProgress()
            }
        }
    }

    private fun saveWatchProgress() {
        val player = viewModel.newPlayer.exoPlayer.value ?: return
        val data = currentMediaData ?: return
        if (data.mediaPath.isNotBlank()) {
            val pos = player.currentPosition
            val dur = player.duration.takeIf { it > 0L } ?: 0L
            lifecycleScope.launch(Dispatchers.IO) {
                watchHistoryRepo.record(
                    url        = data.mediaPath,
                    title      = data.mediaTitle,
                    logoUrl    = data.thumbnailUrl.takeIf { it.isNotBlank() },
                    groupTitle = data.groupTitle,
                    positionMs = pos,
                    durationMs = dur
                )
            }
        }
    }

    override fun onStop() {
        super.onStop()
        AutoFrameRateManager.onActivityStopped(this)
    }

    override fun onDestroy() {
        super.onDestroy()
        positionSaveJob?.cancel()
        surfingCoordinator?.destroy()
        surfingCoordinator = null
        lastAttachedPlayer = null
        AutoFrameRateManager.cleanupActivity(this)

        if (isFinishing) {
            stopService(Intent(this, PlaybackService::class.java))
        }
    }
}
""",

    "tv/dialog/TvChannelActionMenuDialog.kt": """package vid.mat.app.tv.dialog

import android.app.Dialog
import android.os.Bundle
import androidx.fragment.app.DialogFragment
import com.google.android.material.dialog.MaterialAlertDialogBuilder
import vid.mat.app.data.model.Channel

class TvChannelActionMenuDialog : DialogFragment() {

    var onPlay: (() -> Unit)? = null
    var onToggleFavorite: (() -> Unit)? = null
    var onOpenEpg: (() -> Unit)? = null
    var onShowDetails: (() -> Unit)? = null

    override fun onCreateDialog(savedInstanceState: Bundle?): Dialog {
        val args = arguments
        val title = args?.getString(ARG_TITLE) ?: "Channel"
        val isFav = args?.getBoolean(ARG_IS_FAV, false) ?: false

        val options = arrayOf(
            "▶ Play Live Channel",
            if (isFav) "★ Remove from Favorites" else "☆ Add to Favorites",
            "📅 View TV Guide Schedule",
            "ℹ Channel Technical Details"
        )

        return MaterialAlertDialogBuilder(requireContext())
            .setTitle(title)
            .setItems(options) { _, which ->
                when (which) {
                    0 -> onPlay?.invoke()
                    1 -> onToggleFavorite?.invoke()
                    2 -> onOpenEpg?.invoke()
                    3 -> onShowDetails?.invoke()
                }
                dismiss()
            }
            .create()
    }

    companion object {
        const val TAG = "TvChannelActionMenuDialog"
        private const val ARG_TITLE = "arg_title"
        private const val ARG_IS_FAV = "arg_is_fav"

        fun newInstance(
            channel: Channel,
            onPlay: () -> Unit,
            onToggleFavorite: () -> Unit,
            onOpenEpg: () -> Unit,
            onShowDetails: () -> Unit
        ) = TvChannelActionMenuDialog().apply {
            arguments = Bundle().apply {
                putString(ARG_TITLE, channel.title)
                putBoolean(ARG_IS_FAV, channel.isFavorite)
            }
            this.onPlay = onPlay
            this.onToggleFavorite = onToggleFavorite
            this.onOpenEpg = onOpenEpg
            this.onShowDetails = onShowDetails
        }
    }
}
""",

    "backup/model/BackupModels.kt": """package vid.mat.app.backup.model

import com.google.gson.annotations.SerializedName

data class BackupHeader(
    @SerializedName("magic") val magic: String = "IPTV_BACKUP_V1",
    @SerializedName("version") val version: Int = 2,
    @SerializedName("created_at_ms") val createdAtMs: Long = System.currentTimeMillis(),
    @SerializedName("app_version") val appVersion: String = "1.0.0",
    @SerializedName("is_encrypted") val isEncrypted: Boolean = true,
    @SerializedName("salt_base64") val saltBase64: String = "",
    @SerializedName("iv_base64") val ivBase64: String = ""
)

data class BackupPayload(
    @SerializedName("sources") val sources: List<PlaylistSourceBackupDto> = emptyList(),
    @SerializedName("channels") val channels: List<ChannelBackupDto> = emptyList(),
    @SerializedName("overrides") val overrides: List<ChannelOverrideBackupDto> = emptyList(),
    @SerializedName("custom_groups") val customGroups: List<CustomGroupBackupDto> = emptyList(),
    @SerializedName("profiles") val profiles: List<ProfileBackupDto> = emptyList(),
    @SerializedName("history") val history: List<WatchHistoryBackupDto> = emptyList(),
    @SerializedName("favorites") val favorites: List<FavoriteBackupDto> = emptyList(),
    @SerializedName("xtream_accounts") val xtreamAccounts: List<XtreamAccountBackupDto> = emptyList()
)

data class PlaylistSourceBackupDto(
    val name: String,
    val type: String,
    val sourceUrl: String,
    val epgUrl: String? = null,
    val xtreamAccountKey: String? = null
)

data class ChannelOverrideBackupDto(
    val channelIdentity: String,
    val customTitle: String? = null,
    val customGroupTitle: String? = null,
    val customLogoUrl: String? = null,
    val customChannelNumber: Int? = null,
    val userOrderIndex: Int = 0,
    val isHidden: Boolean = false,
    val isFavorite: Boolean = false
)

data class CustomGroupBackupDto(
    val name: String,
    val sortOrder: Int = 0,
    val isHidden: Boolean = false,
    val isUserCreated: Boolean = true
)

data class ProfileBackupDto(
    val sourceProfileId: Long = 0L,
    val name: String,
    val avatarColorHex: String,
    val avatarIconName: String = "ic_profile_default",
    val isKidsProfile: Boolean,
    val isProtected: Boolean,
    val pinHash: String? = null,
    val isDefault: Boolean = false
)

data class FavoriteBackupDto(
    val targetId: String,
    val profileId: Long,
    val type: String,
    val title: String,
    val subtitle: String? = null,
    val groupTitle: String? = null,
    val logoUrl: String? = null,
    val streamUrl: String,
    val customFavoriteGroup: String? = null,
    val userOrder: Int = 0,
    val createdAtMs: Long = 0L
)

data class WatchHistoryBackupDto(
    val mediaKey: String,
    val mediaTitle: String,
    val streamUrl: String,
    val positionMs: Long,
    val durationMs: Long,
    val completionPercent: Int,
    val profileId: Long = 1L,
    val contentType: String = "LIVE_TV",
    val mediaSubtitle: String? = null,
    val thumbnailUrl: String? = null,
    val seriesId: Long? = null,
    val seasonNumber: Int? = null,
    val episodeNumber: Int? = null,
    val lastWatchedTimestampMs: Long = 0L,
    val isCompleted: Boolean = false
)

data class XtreamAccountBackupDto(
    val serverUrl: String,
    val username: String,
    val encryptedPassBase64: String = "",
    val ivBase64: String = "",
    val password: String? = null
)

data class BackupExportSummary(
    val sourcesCount: Int,
    val channelsCount: Int,
    val overridesCount: Int,
    val historyCount: Int,
    val profilesCount: Int,
    val totalBytesWritten: Long
)

data class BackupRestoreSummary(
    val restoredSources: Int,
    val restoredChannels: Int,
    val restoredOverrides: Int,
    val restoredProfiles: Int,
    val restoredHistory: Int,
    val restoredFavorites: Int = 0
)

data class StreamEndpointBackupDto(
    val url: String,
    val priority: Int = 0,
    val containerFormat: String? = "m3u8",
    val resolutionLabel: String? = "Auto",
    val userAgent: String? = null,
    val referer: String? = null,
    val cookie: String? = null,
    val authorization: String? = null
)

data class ChannelBackupDto(
    val channelIdentity: String,
    val sourceId: Long = 0L,
    val sourceUrl: String = "",
    val sourceType: String = "M3U_REMOTE",
    val externalId: String? = null,
    val channelNumber: Int? = null,
    val title: String,
    val logoUrl: String? = null,
    val groupTitle: String? = null,
    val tvgId: String? = null,
    val tvgName: String? = null,
    val isFavorite: Boolean = false,
    val userOrder: Int = 0,
    val catchupDays: Int = 0,
    val catchupSource: String? = null,
    val lastPlayedAtMs: Long? = null,
    val createdAtMs: Long = 0L,
    val updatedAtMs: Long = 0L,
    val endpoints: List<StreamEndpointBackupDto> = emptyList()
)
""",

    "backup/ui/BackupCreateDialogFragment.kt": """package vid.mat.app.backup.ui

import android.app.Activity
import android.content.Intent
import android.net.Uri
import android.os.Bundle
import android.view.LayoutInflater
import android.view.View
import android.view.ViewGroup
import android.widget.Toast
import androidx.activity.result.contract.ActivityResultContracts
import androidx.fragment.app.DialogFragment
import androidx.lifecycle.lifecycleScope
import dagger.hilt.android.AndroidEntryPoint
import kotlinx.coroutines.launch
import vid.mat.app.backup.EncryptedBackupManager
import vid.mat.app.databinding.DialogBackupCreateBinding
import java.text.SimpleDateFormat
import java.util.Date
import java.util.Locale
import javax.inject.Inject

@AndroidEntryPoint
class BackupCreateDialogFragment : DialogFragment() {

    private var _binding: DialogBackupCreateBinding? = null
    private val binding get() = _binding!!

    @Inject
    lateinit var backupManager: EncryptedBackupManager

    private var targetPassphrase: String? = null

    private val saveFileLauncher = registerForActivityResult(
        ActivityResultContracts.StartActivityForResult()
    ) { result ->
        if (result.resultCode == Activity.RESULT_OK) {
            result.data?.data?.let { uri ->
                performExport(uri)
            }
        }
    }

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        targetPassphrase = savedInstanceState?.getString(KEY_PASSPHRASE)
    }

    override fun onSaveInstanceState(outState: Bundle) {
        super.onSaveInstanceState(outState)
        outState.putString(KEY_PASSPHRASE, targetPassphrase)
    }

    override fun onCreateView(inflater: LayoutInflater, container: ViewGroup?, savedInstanceState: Bundle?): View {
        _binding = DialogBackupCreateBinding.inflate(inflater, container, false)
        return binding.root
    }

    override fun onViewCreated(view: View, savedInstanceState: Bundle?) {
        super.onViewCreated(view, savedInstanceState)

        binding.btnExport.setOnClickListener {
            val pass = binding.etPassword.text.toString().trim()
            val confirm = binding.etConfirmPassword.text.toString().trim()

            if (pass.isBlank()) {
                Toast.makeText(requireContext(), "A backup passphrase is required", Toast.LENGTH_SHORT).show()
                return@setOnClickListener
            }
            if (pass != confirm) {
                Toast.makeText(requireContext(), "Passwords do not match", Toast.LENGTH_SHORT).show()
                return@setOnClickListener
            }

            targetPassphrase = pass

            val timestamp = SimpleDateFormat("yyyyMMdd_HHmm", Locale.US).format(Date())
            val intent = Intent(Intent.ACTION_CREATE_DOCUMENT).apply {
                addCategory(Intent.CATEGORY_OPENABLE)
                type = "application/octet-stream"
                putExtra(Intent.EXTRA_TITLE, "iptv_backup_$timestamp.ytvbk")
            }
            saveFileLauncher.launch(intent)
        }

        binding.btnCancel.setOnClickListener { dismiss() }
    }

    private fun performExport(uri: Uri) {
        lifecycleScope.launch {
            binding.btnExport.isEnabled = false
            binding.btnExport.text = "Exporting..."

            val outputStream = requireContext().contentResolver.openOutputStream(uri)
            if (outputStream != null) {
                val result = backupManager.exportBackup(outputStream, targetPassphrase)
                result.fold(
                    onSuccess = { summary ->
                        Toast.makeText(
                            requireContext(),
                            "Backup created: ${summary.channelsCount} channels saved",
                            Toast.LENGTH_LONG
                        ).show()
                        dismiss()
                    },
                    onFailure = { err ->
                        Toast.makeText(
                            requireContext(),
                            "Export failed: ${err.message}",
                            Toast.LENGTH_LONG
                        ).show()
                        binding.btnExport.isEnabled = true
                        binding.btnExport.text = "Create Encrypted Backup"
                    }
                )
            } else {
                Toast.makeText(
                    requireContext(),
                    "Unable to open the selected file for writing",
                    Toast.LENGTH_LONG
                ).show()
                binding.btnExport.isEnabled = true
                binding.btnExport.text = "Create Encrypted Backup"
            }
        }
    }

    override fun onDestroyView() {
        super.onDestroyView()
        _binding = null
    }

    companion object {
        const val TAG = "BackupCreateDialogFragment"
        private const val KEY_PASSPHRASE = "state_target_passphrase"
        fun newInstance() = BackupCreateDialogFragment()
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

Note: `TvPlaybackActivity.kt` extends `AppCompatActivity()` (compatible with `ChannelSurfingCoordinator(activity: AppCompatActivity, ...)`). Make sure `TvPlaybackActivity` uses an AppCompat theme in `AndroidManifest.xml` (or note that for manifest review).

May I proceed with **Batch 8 (5 classes — Encrypted Backup & Parental Enforcement)** (`backup/EncryptedBackupManager.kt`, `security/ParentalControlManager.kt`, `ui/epg/EpgProgramDetailsBottomSheet.kt`, `favorites/ui/FavoritesFragment.kt`, `history/ui/UnifiedHistoryFragment.kt`)?
Here is the Python script for **Batch 8 (5 classes — Backup Restoration & Parental Gate Enforcement)**:

1. `backup/EncryptedBackupManager.kt` — Exports hidden custom groups and override `isFavorite`, merges existing profiles by name cleanly, and skips non-portable local VOD/Series IDs during favorite restore.
2. `security/ParentalControlManager.kt` — Enforces `isAdultPattern(cat)` inside `isCategoryRestricted`.
3. `ui/epg/EpgProgramDetailsBottomSheet.kt` — Enforces `ParentalGatekeeper` + PIN challenge before tuning live or catch-up playback, and validates catch-up availability before launching the player.
4. `favorites/ui/FavoritesFragment.kt` — Enforces `ParentalGatekeeper` + PIN challenge before opening or playing favorites.
5. `history/ui/UnifiedHistoryFragment.kt` — Enforces `ParentalGatekeeper` + PIN challenge before resuming or restarting history items.

```python
#!/usr/bin/env python3
from pathlib import Path

BASE_DIR = Path("/home/username/AndroidStudioProjects/Ytvmate/app/src/main/java/vid/mat/app")

FILES = {
    "backup/EncryptedBackupManager.kt": """package vid.mat.app.backup

import android.util.Base64
import androidx.room.withTransaction
import com.google.gson.Gson
import kotlinx.coroutines.Dispatchers
import kotlinx.coroutines.flow.first
import kotlinx.coroutines.withContext
import vid.mat.app.backup.model.BackupExportSummary
import vid.mat.app.backup.model.BackupHeader
import vid.mat.app.backup.model.BackupPayload
import vid.mat.app.backup.model.BackupRestoreSummary
import vid.mat.app.backup.model.ChannelBackupDto
import vid.mat.app.backup.model.ChannelOverrideBackupDto
import vid.mat.app.backup.model.CustomGroupBackupDto
import vid.mat.app.backup.model.FavoriteBackupDto
import vid.mat.app.backup.model.PlaylistSourceBackupDto
import vid.mat.app.backup.model.ProfileBackupDto
import vid.mat.app.backup.model.StreamEndpointBackupDto
import vid.mat.app.backup.model.WatchHistoryBackupDto
import vid.mat.app.backup.model.XtreamAccountBackupDto
import vid.mat.app.data.dao.ChannelDao
import vid.mat.app.data.dao.ChannelUserOverrideDao
import vid.mat.app.data.dao.CustomGroupDao
import vid.mat.app.data.dao.PlaylistSourceDao
import vid.mat.app.data.dao.StreamEndpointDao
import vid.mat.app.data.dao.WatchProgressDao
import vid.mat.app.data.dao.XtreamAccountDao
import vid.mat.app.data.database.VideoUrlDatabase
import vid.mat.app.data.model.Channel
import vid.mat.app.data.model.ChannelUserOverrideEntity
import vid.mat.app.data.model.CustomGroupEntity
import vid.mat.app.data.model.MediaContentType
import vid.mat.app.data.model.PlaylistSource
import vid.mat.app.data.model.SourceType
import vid.mat.app.data.model.StreamEndpoint
import vid.mat.app.data.model.WatchProgressEntity
import vid.mat.app.data.model.XtreamAccount
import vid.mat.app.data.sync.PlaylistFingerprint
import vid.mat.app.favorites.ProfileFavoriteSynchronizer
import vid.mat.app.favorites.dao.UnifiedFavoriteDao
import vid.mat.app.favorites.model.FavoriteType
import vid.mat.app.favorites.model.UnifiedFavoriteEntity
import vid.mat.app.profile.ActiveProfileProvider
import vid.mat.app.profile.dao.ProfileDao
import vid.mat.app.profile.model.ProfileEntity
import vid.mat.app.provider.normalization.ChannelNameNormalizer
import vid.mat.app.security.CryptoUtils
import vid.mat.app.security.EncryptedField
import vid.mat.app.util.AppLogger
import java.io.ByteArrayOutputStream
import java.io.InputStream
import java.io.OutputStream
import java.nio.ByteBuffer
import java.nio.charset.StandardCharsets
import java.security.SecureRandom
import javax.crypto.AEADBadTagException
import javax.crypto.Cipher
import javax.crypto.SecretKeyFactory
import javax.crypto.spec.GCMParameterSpec
import javax.crypto.spec.PBEKeySpec
import javax.crypto.spec.SecretKeySpec
import javax.inject.Inject
import javax.inject.Singleton

@Singleton
class EncryptedBackupManager @Inject constructor(
    private val database: VideoUrlDatabase,
    private val sourceDao: PlaylistSourceDao,
    private val channelDao: ChannelDao,
    private val endpointDao: StreamEndpointDao,
    private val overrideDao: ChannelUserOverrideDao,
    private val groupDao: CustomGroupDao,
    private val historyDao: WatchProgressDao,
    private val profileDao: ProfileDao,
    private val favoriteDao: UnifiedFavoriteDao,
    private val xtreamDao: XtreamAccountDao,
    private val activeProfileProvider: ActiveProfileProvider,
    private val favoriteSynchronizer: ProfileFavoriteSynchronizer
) {
    companion object {
        private const val TAG = "EncryptedBackupManager"
        private const val MAGIC_HEADER = "IPTV_BACKUP_V2"
        private const val LEGACY_MAGIC_HEADER = "IPTV_BACKUP_V1"
        private const val MAX_HEADER_BYTES = 65_536
        private const val MAX_BACKUP_BYTES = 256L * 1024L * 1024L
        private const val PBKDF2_ITERATIONS = 65_536
        private const val KEY_LENGTH_BITS = 256
        private const val GCM_TAG_LENGTH = 128
        private const val GCM_IV_LENGTH_BYTES = 12
        private const val SALT_LENGTH_BYTES = 16
    }

    private val gson = Gson()

    suspend fun exportBackup(outputStream: OutputStream, passphrase: String?): Result<BackupExportSummary> = withContext(Dispatchers.IO) {
        try {
            require(!passphrase.isNullOrBlank()) { "A non-empty backup passphrase is required for encrypted exports" }
            val accounts = xtreamDao.getAll()
            val accountById = accounts.associateBy { it.accountId }
            val allSources = sourceDao.getAll()
            val sources = allSources.map { source ->
                val account = source.xtreamAccountId?.let { accountById[it] }
                PlaylistSourceBackupDto(
                    source.name, source.type.name, source.sourceUrl, source.epgUrl,
                    account?.let { "${it.serverUrl}\u0000${it.username}" }
                )
            }
            val sourceById = allSources.associateBy { it.sourceId }
            val channels = channelDao.getAll().map { c ->
                val source = sourceById[c.sourceId]
                    ?: throw IllegalStateException("Channel ${c.channelIdentity} references missing source ${c.sourceId}")
                val endpoints = endpointDao.getEndpointsForChannel(c.channelId).map { ep ->
                    StreamEndpointBackupDto(
                        url = ep.url,
                        priority = ep.priority,
                        containerFormat = ep.containerFormat,
                        resolutionLabel = ep.resolutionLabel,
                        userAgent = ep.userAgent,
                        referer = ep.referer?.value,
                        cookie = ep.cookie?.value,
                        authorization = ep.authorization?.value
                    )
                }
                ChannelBackupDto(
                    channelIdentity = c.channelIdentity,
                    sourceId = c.sourceId,
                    sourceUrl = source.sourceUrl,
                    sourceType = source.type.name,
                    externalId = c.externalId,
                    channelNumber = c.channelNumber,
                    title = c.title,
                    logoUrl = c.logoUrl,
                    groupTitle = c.groupTitle,
                    tvgId = c.tvgId,
                    tvgName = c.tvgName,
                    isFavorite = c.isFavorite,
                    userOrder = c.userOrder,
                    catchupDays = c.catchupDays,
                    catchupSource = c.catchupSource,
                    lastPlayedAtMs = c.lastPlayedAtMs,
                    createdAtMs = c.createdAtMs,
                    updatedAtMs = c.updatedAtMs,
                    endpoints = endpoints
                )
            }
            val overrides = overrideDao.getAll().map {
                ChannelOverrideBackupDto(
                    it.channelIdentity,
                    it.customTitle,
                    it.customGroupTitle,
                    it.customLogoUrl,
                    it.customChannelNumber,
                    it.userOrderIndex,
                    it.isHidden,
                    it.isFavorite
                )
            }
            val customGroups = runCatching { groupDao.getAllFlow().first() }.getOrDefault(emptyList())
                .map { CustomGroupBackupDto(it.name, it.sortOrder, it.isHidden, it.isUserCreated) }
            val profileEntities = profileDao.getAllProfiles()
            val profiles = profileEntities.map { profile ->
                if (profile.isProtected && !isValidProfileVerifier(profile.pinHash)) {
                    throw IllegalStateException("Protected profile ${profile.id} cannot be backed up because its PIN verifier is invalid")
                }
                ProfileBackupDto(profile.id, profile.name, profile.avatarColorHex, profile.avatarIconName, profile.isKidsProfile, profile.isProtected, profile.pinHash, profile.isDefault)
            }
            val history = buildList {
                for (profile in profileEntities) {
                    historyDao.getRecentHistory(profile.id, 500).first().forEach { progress ->
                        add(WatchHistoryBackupDto(progress.mediaKey, progress.mediaTitle, progress.streamUrl, progress.positionMs, progress.durationMs, progress.completionPercent, progress.profileId, progress.contentType.name, progress.mediaSubtitle, progress.thumbnailUrl, progress.seriesId, progress.seasonNumber, progress.episodeNumber, progress.lastWatchedTimestampMs, progress.isCompleted))
                    }
                }
            }
            val favorites = buildList {
                for (profile in profileEntities) {
                    favoriteDao.getFavoritesFlow(profile.id, null, null, "").first().forEach { favorite ->
                        add(
                            FavoriteBackupDto(
                                targetId = favorite.targetId, profileId = favorite.profileId, type = favorite.type.name,
                                title = favorite.title, subtitle = favorite.subtitle, groupTitle = favorite.groupTitle,
                                logoUrl = favorite.logoUrl, streamUrl = favorite.streamUrl,
                                customFavoriteGroup = favorite.customFavoriteGroup, userOrder = favorite.userOrder,
                                createdAtMs = favorite.createdAtMs
                            )
                        )
                    }
                }
            }

            val xtreamAccounts = accounts.map { xa ->
                val password = when (val decrypted = CryptoUtils.decrypt(xa.encryptedPassword, xa.iv)) {
                    is CryptoUtils.DecryptionResult.Success -> decrypted.plaintext
                    CryptoUtils.DecryptionResult.KeyInvalidated -> throw IllegalStateException("Xtream credentials key is invalidated; re-authentication is required before backup")
                    CryptoUtils.DecryptionResult.Corrupted -> throw IllegalStateException("Xtream credentials are corrupted and cannot be backed up")
                    is CryptoUtils.DecryptionResult.Failure -> throw IllegalStateException("Xtream credentials could not be decrypted for backup", decrypted.cause)
                }
                XtreamAccountBackupDto(xa.serverUrl, xa.username, "", "", password)
            }

            val payloadBytes = gson.toJson(BackupPayload(sources, channels, overrides, customGroups, profiles, history, favorites, xtreamAccounts))
                .toByteArray(StandardCharsets.UTF_8)
            val salt = ByteArray(SALT_LENGTH_BYTES).also { SecureRandom().nextBytes(it) }
            val iv = ByteArray(GCM_IV_LENGTH_BYTES).also { SecureRandom().nextBytes(it) }
            val cipher = Cipher.getInstance("AES/GCM/NoPadding")
            cipher.init(Cipher.ENCRYPT_MODE, deriveKey(passphrase, salt), GCMParameterSpec(GCM_TAG_LENGTH, iv))
            val ciphertext = cipher.doFinal(payloadBytes)
            val headerBytes = gson.toJson(
                BackupHeader(
                    MAGIC_HEADER, 2, System.currentTimeMillis(), "1.0.0", true,
                    Base64.encodeToString(salt, Base64.NO_WRAP), Base64.encodeToString(iv, Base64.NO_WRAP)
                )
            ).toByteArray(StandardCharsets.UTF_8)

            require(headerBytes.size <= MAX_HEADER_BYTES)
            outputStream.use { os ->
                os.write(ByteBuffer.allocate(4).putInt(headerBytes.size).array())
                os.write(headerBytes)
                os.write(ciphertext)
                os.flush()
            }
            Result.success(BackupExportSummary(sources.size, channels.size, overrides.size, history.size, profiles.size, 4L + headerBytes.size + ciphertext.size))
        } catch (e: Exception) {
            AppLogger.e(TAG, "Export failed", e)
            Result.failure(e)
        }
    }

    suspend fun importBackup(inputStream: InputStream, passphrase: String?): Result<BackupRestoreSummary> = withContext(Dispatchers.IO) {
        try {
            val (header, ciphertext) = inputStream.use { stream ->
                val lengthBytes = readExactly(stream, 4) ?: throw IllegalStateException("Invalid backup file structure")
                val headerLength = ByteBuffer.wrap(lengthBytes).int
                require(headerLength in 1..MAX_HEADER_BYTES) { "Invalid backup header length" }
                val headerBytes = readExactly(stream, headerLength) ?: throw IllegalStateException("Malformed backup header")
                val parsed = gson.fromJson(String(headerBytes, StandardCharsets.UTF_8), BackupHeader::class.java)
                require(parsed.magic == MAGIC_HEADER || parsed.magic == LEGACY_MAGIC_HEADER) { "Unsupported backup format" }
                val remaining = stream.readBytesLimited(MAX_BACKUP_BYTES - 4L - headerLength)
                Pair(parsed, remaining)
            }
            val salt = Base64.decode(header.saltBase64, Base64.NO_WRAP)
            val iv = Base64.decode(header.ivBase64, Base64.NO_WRAP)
            require(salt.size == SALT_LENGTH_BYTES && iv.size == GCM_IV_LENGTH_BYTES) { "Invalid backup cryptographic parameters" }
            val effectivePassphrase = passphrase ?: if (header.magic == LEGACY_MAGIC_HEADER) "DEFAULT_IPTV_KEY" else throw IllegalArgumentException("Backup passphrase is required")
            val cipher = Cipher.getInstance("AES/GCM/NoPadding")
            cipher.init(Cipher.DECRYPT_MODE, deriveKey(effectivePassphrase, salt), GCMParameterSpec(GCM_TAG_LENGTH, iv))
            val payload = gson.fromJson(String(cipher.doFinal(ciphertext), StandardCharsets.UTF_8), BackupPayload::class.java)
                ?: throw IllegalArgumentException("Backup payload is empty")

            var restoredSources = 0
            var restoredChannels = 0
            var restoredOverrides = 0
            var restoredProfiles = 0
            var restoredHistory = 0
            var restoredFavorites = 0
            var restoredDefaultProfileId: Long? = null

            database.withTransaction {
                val restoredProfileIds = HashMap<Long, Long>()
                val accountIds = HashMap<String, Long>()
                val identityRemap = HashMap<String, String>()

                payload.xtreamAccounts.forEach { xa ->
                    val passwordResult = xa.password?.let { CryptoUtils.encrypt(it) }
                    val encrypted = when (passwordResult) {
                        is CryptoUtils.EncryptionResult.Success -> passwordResult.result
                        CryptoUtils.EncryptionResult.KeyInvalidated -> throw IllegalStateException("Device credential key is invalidated; re-authentication is required")
                        is CryptoUtils.EncryptionResult.Failure -> throw IllegalStateException("Unable to securely restore Xtream credentials", passwordResult.cause)
                        null -> null
                    }
                    val existing = xtreamDao.getByServerAndUsername(xa.serverUrl, xa.username)
                    val id = if (existing != null && encrypted == null) existing.accountId else xtreamDao.insert(
                        XtreamAccount(
                            accountId = existing?.accountId ?: 0L,
                            serverUrl = xa.serverUrl,
                            username = xa.username,
                            encryptedPassword = encrypted?.ciphertextBase64 ?: xa.encryptedPassBase64,
                            iv = encrypted?.ivBase64 ?: xa.ivBase64
                        )
                    )
                    accountIds["${xa.serverUrl}\u0000${xa.username}"] = id
                }

                val sourceIds = HashMap<String, Long>()
                payload.sources.forEach { src ->
                    val type = runCatching { SourceType.valueOf(src.type) }.getOrDefault(SourceType.M3U_REMOTE)
                    val existing = sourceDao.getByUrlAndType(src.sourceUrl, type)
                    val accountId = src.xtreamAccountKey?.let { accountIds[it] }
                    val id = existing?.sourceId ?: sourceDao.insert(
                        PlaylistSource(name = src.name, type = type, sourceUrl = src.sourceUrl, epgUrl = src.epgUrl, xtreamAccountId = accountId)
                    )
                    sourceIds["${type.name}\u0000${src.sourceUrl}"] = id
                    restoredSources++
                }

                payload.channels.forEach { c ->
                    if (c.sourceUrl.isBlank()) return@forEach
                    val type = runCatching { SourceType.valueOf(c.sourceType) }.getOrDefault(SourceType.M3U_REMOTE)
                    val sourceId = sourceIds["${type.name}\u0000${c.sourceUrl}"]
                        ?: sourceDao.getByUrlAndType(c.sourceUrl, type)?.sourceId
                        ?: return@forEach

                    val targetIdentity = if (c.sourceId > 0L && c.sourceId != sourceId) {
                        val canonicalTitle = ChannelNameNormalizer.normalize(c.title).ifBlank { c.title }
                        PlaylistFingerprint.computeIdentity(sourceId, c.externalId, c.tvgId, canonicalTitle)
                    } else {
                        c.channelIdentity
                    }
                    identityRemap[c.channelIdentity] = targetIdentity

                    val existingChannel = channelDao.getByIdentity(targetIdentity)
                    val channelId = if (existingChannel != null) {
                        existingChannel.channelId
                    } else {
                        val inserted = channelDao.insert(
                            Channel(
                                sourceId = sourceId,
                                channelIdentity = targetIdentity,
                                externalId = c.externalId,
                                channelNumber = c.channelNumber,
                                title = c.title,
                                logoUrl = c.logoUrl,
                                groupTitle = c.groupTitle,
                                tvgId = c.tvgId,
                                tvgName = c.tvgName,
                                isFavorite = c.isFavorite,
                                userOrder = c.userOrder,
                                catchupDays = c.catchupDays,
                                catchupSource = c.catchupSource,
                                lastPlayedAtMs = c.lastPlayedAtMs,
                                createdAtMs = c.createdAtMs.takeIf { it > 0 } ?: System.currentTimeMillis(),
                                updatedAtMs = c.updatedAtMs.takeIf { it > 0 } ?: System.currentTimeMillis()
                            )
                        )
                        if (inserted != -1L) {
                            restoredChannels++
                            inserted
                        } else {
                            channelDao.getByIdentity(targetIdentity)?.channelId ?: -1L
                        }
                    }

                    if (channelId > 0L && c.endpoints.isNotEmpty()) {
                        val existingUrls = endpointDao.getEndpointsForChannel(channelId).map { it.url }.toSet()
                        val toInsert = c.endpoints.filterNot { it.url in existingUrls }.map { ep ->
                            StreamEndpoint(
                                channelId = channelId,
                                url = ep.url,
                                priority = ep.priority,
                                containerFormat = ep.containerFormat,
                                resolutionLabel = ep.resolutionLabel,
                                userAgent = ep.userAgent,
                                referer = EncryptedField.from(ep.referer),
                                cookie = EncryptedField.from(ep.cookie),
                                authorization = EncryptedField.from(ep.authorization)
                            )
                        }
                        if (toInsert.isNotEmpty()) endpointDao.insertBatch(toInsert)
                    }
                }

                payload.overrides.forEach { o ->
                    val mappedIdentity = identityRemap[o.channelIdentity] ?: o.channelIdentity
                    overrideDao.upsert(
                        ChannelUserOverrideEntity(
                            channelIdentity = mappedIdentity,
                            customTitle = o.customTitle,
                            customGroupTitle = o.customGroupTitle,
                            customLogoUrl = o.customLogoUrl,
                            customChannelNumber = o.customChannelNumber,
                            userOrderIndex = o.userOrderIndex,
                            isHidden = o.isHidden,
                            isFavorite = o.isFavorite
                        )
                    )
                    restoredOverrides++
                }
                payload.customGroups.forEach { g ->
                    groupDao.insert(
                        CustomGroupEntity(
                            name = g.name,
                            sortOrder = g.sortOrder,
                            isHidden = g.isHidden,
                            isUserCreated = g.isUserCreated
                        )
                    )
                }

                val existingProfilesByName = profileDao.getAllProfiles().associateBy { it.name.trim().lowercase() }
                if (payload.profiles.any { it.isDefault }) {
                    profileDao.clearDefaultFlags()
                }
                payload.profiles.forEach { p ->
                    if (p.isProtected && !isValidProfileVerifier(p.pinHash)) {
                        throw IllegalArgumentException("Backup contains an invalid verifier for protected profile '${p.name}'")
                    }
                    val existing = existingProfilesByName[p.name.trim().lowercase()]
                    val restoredId = if (existing != null) {
                        profileDao.update(
                            existing.copy(
                                avatarColorHex = p.avatarColorHex,
                                avatarIconName = p.avatarIconName,
                                isKidsProfile = p.isKidsProfile,
                                isProtected = p.isProtected,
                                pinHash = p.pinHash,
                                isDefault = p.isDefault
                            )
                        )
                        existing.id
                    } else {
                        profileDao.insert(
                            ProfileEntity(
                                name = p.name,
                                avatarColorHex = p.avatarColorHex,
                                avatarIconName = p.avatarIconName,
                                isKidsProfile = p.isKidsProfile,
                                isProtected = p.isProtected,
                                pinHash = p.pinHash,
                                isDefault = p.isDefault
                            )
                        )
                    }
                    if (restoredId > 0L) {
                        restoredProfileIds[p.sourceProfileId] = restoredId
                        if (p.sourceProfileId <= 0L) restoredProfileIds.putIfAbsent(1L, restoredId)
                        if (p.isDefault || restoredDefaultProfileId == null) {
                            restoredDefaultProfileId = restoredId
                        }
                    }
                    restoredProfiles++
                }
                payload.favorites.forEach { f ->
                    val restoredProfileId = restoredProfileIds[f.profileId] ?: return@forEach
                    val type = runCatching { FavoriteType.valueOf(f.type) }.getOrNull() ?: return@forEach
                    if ((type == FavoriteType.VOD_MOVIE || type == FavoriteType.SERIES) &&
                        (f.targetId.startsWith("movie_") || f.targetId.startsWith("series_"))
                    ) {
                        return@forEach
                    }
                    val remappedTarget = if (f.targetId.startsWith("live_identity:")) {
                        val oldId = f.targetId.removePrefix("live_identity:")
                        "live_identity:${identityRemap[oldId] ?: oldId}"
                    } else {
                        f.targetId
                    }
                    favoriteDao.upsert(
                        UnifiedFavoriteEntity(
                            targetId = remappedTarget,
                            profileId = restoredProfileId,
                            type = type,
                            title = f.title,
                            subtitle = f.subtitle,
                            groupTitle = f.groupTitle,
                            logoUrl = f.logoUrl,
                            streamUrl = f.streamUrl,
                            customFavoriteGroup = f.customFavoriteGroup,
                            userOrder = f.userOrder,
                            createdAtMs = f.createdAtMs.takeIf { it > 0 } ?: System.currentTimeMillis()
                        )
                    )
                    restoredFavorites++
                }
                payload.history.forEach { h ->
                    val restoredProfileId = restoredProfileIds[h.profileId] ?: return@forEach
                    val remappedKey = identityRemap[h.mediaKey] ?: h.mediaKey
                    val remappedStreamUrl = identityRemap[h.streamUrl] ?: h.streamUrl
                    historyDao.upsert(
                        WatchProgressEntity(
                            mediaKey = remappedKey,
                            profileId = restoredProfileId,
                            contentType = runCatching { MediaContentType.valueOf(h.contentType) }.getOrDefault(MediaContentType.LIVE_TV),
                            mediaTitle = h.mediaTitle,
                            mediaSubtitle = h.mediaSubtitle,
                            thumbnailUrl = h.thumbnailUrl,
                            streamUrl = remappedStreamUrl,
                            positionMs = h.positionMs,
                            durationMs = h.durationMs,
                            completionPercent = h.completionPercent,
                            isCompleted = h.isCompleted,
                            seriesId = h.seriesId,
                            seasonNumber = h.seasonNumber,
                            episodeNumber = h.episodeNumber,
                            lastWatchedTimestampMs = h.lastWatchedTimestampMs.takeIf { it > 0 } ?: System.currentTimeMillis()
                        )
                    )
                    restoredHistory++
                }
            }

            val profileToActivate = restoredDefaultProfileId ?: activeProfileProvider.awaitInitializedProfile().id
            if (!activeProfileProvider.switchProfile(profileToActivate)) {
                favoriteSynchronizer.syncProfile(activeProfileProvider.activeProfile.value.id)
            }

            Result.success(BackupRestoreSummary(restoredSources, restoredChannels, restoredOverrides, restoredProfiles, restoredHistory, restoredFavorites))
        } catch (_: AEADBadTagException) {
            Result.failure(IllegalArgumentException("Incorrect passphrase or corrupted backup file"))
        } catch (e: Exception) {
            AppLogger.e(TAG, "Import failed", e)
            Result.failure(e)
        }
    }

    private fun isValidProfileVerifier(verifier: String?): Boolean {
        if (verifier.isNullOrBlank()) return false
        if (verifier.startsWith("PBKDF2v1:")) {
            val parts = verifier.split(':')
            return parts.size == 3 && runCatching {
                val salt = Base64.decode(parts[1], Base64.NO_WRAP)
                val derived = Base64.decode(parts[2], Base64.NO_WRAP)
                salt.size == 16 && derived.size == 32
            }.getOrDefault(false)
        }
        return verifier.matches(Regex("^[0-9a-fA-F]{64}$"))
    }

    private fun readExactly(input: InputStream, length: Int): ByteArray? {
        val result = ByteArray(length)
        var offset = 0
        while (offset < length) {
            val read = input.read(result, offset, length - offset)
            if (read < 0) return null
            if (read == 0) continue
            offset += read
        }
        return result
    }

    private fun InputStream.readBytesLimited(maxBytes: Long): ByteArray {
        require(maxBytes >= 0)
        val out = ByteArrayOutputStream()
        val buffer = ByteArray(32 * 1024)
        var total = 0L
        while (true) {
            val n = read(buffer)
            if (n < 0) break
            if (n == 0) continue
            total += n
            if (total > maxBytes) throw IllegalArgumentException("Backup file exceeds maximum supported size")
            out.write(buffer, 0, n)
        }
        return out.toByteArray()
    }

    private fun deriveKey(passphrase: String, salt: ByteArray): SecretKeySpec {
        val spec = PBEKeySpec(passphrase.toCharArray(), salt, PBKDF2_ITERATIONS, KEY_LENGTH_BITS)
        return SecretKeySpec(SecretKeyFactory.getInstance("PBKDF2WithHmacSHA256").generateSecret(spec).encoded, "AES")
    }
}
""",

    "security/ParentalControlManager.kt": """package vid.mat.app.security

import android.content.Context
import android.util.Base64
import androidx.core.content.edit
import androidx.security.crypto.EncryptedSharedPreferences
import androidx.security.crypto.MasterKey
import dagger.hilt.android.qualifiers.ApplicationContext
import java.security.SecureRandom
import javax.crypto.SecretKeyFactory
import javax.crypto.spec.PBEKeySpec
import javax.inject.Inject
import javax.inject.Singleton

@Singleton
class ParentalControlManager @Inject constructor(
    @ApplicationContext context: Context
) {

    companion object {
        private const val PREFS_NAME = "parental_control_secure_prefs"
        private const val KEY_PIN_HASH = "parental_pin_hash"
        private const val KEY_ENABLED = "parental_protection_enabled"
        private const val KEY_LOCKED_CATEGORIES = "locked_category_names"
        private const val KEY_LOCKED_CHANNELS = "locked_channel_identities"
        private const val KEY_CUSTOM_BLOCKED_WORDS = "custom_blocked_words"
        private const val KEY_SESSION_TIMEOUT_MINUTES = "session_timeout_minutes"
        private const val KEY_FAILED_ATTEMPTS = "failed_pin_attempts"
        private const val KEY_LOCKOUT_TIMESTAMP = "lockout_until_timestamp"
        private const val KEY_PIN_SALT = "parental_pin_salt"
        private const val PBKDF2_ITERATIONS = 120_000
        private const val PBKDF2_KEY_BITS = 256
    }

    private val masterKey = MasterKey.Builder(context)
        .setKeyScheme(MasterKey.KeyScheme.AES256_GCM)
        .build()

    private val prefs = EncryptedSharedPreferences.create(
        context,
        PREFS_NAME,
        masterKey,
        EncryptedSharedPreferences.PrefKeyEncryptionScheme.AES256_SIV,
        EncryptedSharedPreferences.PrefValueEncryptionScheme.AES256_GCM
    )

    private var sessionAuthenticatedTime = 0L

    val isParentalControlActive: Boolean
        get() = prefs.getBoolean(KEY_ENABLED, false)

    var sessionTimeoutMinutes: Int
        get() = prefs.getInt(KEY_SESSION_TIMEOUT_MINUTES, 15)
        set(value) { prefs.edit { putInt(KEY_SESSION_TIMEOUT_MINUTES, value) } }

    fun setParentalControlEnabled(enabled: Boolean) {
        prefs.edit { putBoolean(KEY_ENABLED, enabled) }
    }

    fun setPin(pin: String) {
        require(pin.length in 4..12) { "PIN must contain 4 to 12 characters" }
        require(pin.all { it.isDigit() }) { "PIN must contain digits only" }
        val salt = ByteArray(16).also { SecureRandom().nextBytes(it) }
        val hash = hashPin(pin, salt)
        prefs.edit {
            putString(KEY_PIN_HASH, hash)
            putString(KEY_PIN_SALT, Base64.encodeToString(salt, Base64.NO_WRAP))
            putBoolean(KEY_ENABLED, true)
            putInt(KEY_FAILED_ATTEMPTS, 0)
            putLong(KEY_LOCKOUT_TIMESTAMP, 0L)
        }
    }

    fun isLockedOut(): Boolean {
        val lockoutUntil = prefs.getLong(KEY_LOCKOUT_TIMESTAMP, 0L)
        return System.currentTimeMillis() < lockoutUntil
    }

    fun getRemainingLockoutSeconds(): Int {
        val lockoutUntil = prefs.getLong(KEY_LOCKOUT_TIMESTAMP, 0L)
        val remainingMs = lockoutUntil - System.currentTimeMillis()
        return if (remainingMs > 0) (remainingMs / 1000).toInt() else 0
    }

    fun verifyPin(pin: String): Boolean {
        if (isLockedOut()) return false
        val storedHash = prefs.getString(KEY_PIN_HASH, null) ?: return false
        val saltB64 = prefs.getString(KEY_PIN_SALT, null)
        val matches = if (!saltB64.isNullOrBlank()) {
            val salt = runCatching { Base64.decode(saltB64, Base64.NO_WRAP) }.getOrNull()
            salt != null && java.security.MessageDigest.isEqual(
                Base64.decode(storedHash, Base64.NO_WRAP),
                Base64.decode(hashPin(pin, salt), Base64.NO_WRAP)
            )
        } else {
            val legacy = java.security.MessageDigest.getInstance("SHA-256")
                .digest(pin.toByteArray(Charsets.UTF_8))
                .joinToString("") { "%02x".format(it) }
            if (storedHash == legacy) {
                setPin(pin)
                true
            } else false
        }
        if (matches) {
            sessionAuthenticatedTime = System.currentTimeMillis()
            prefs.edit {
                putInt(KEY_FAILED_ATTEMPTS, 0)
                putLong(KEY_LOCKOUT_TIMESTAMP, 0L)
            }
        } else {
            val failedCount = prefs.getInt(KEY_FAILED_ATTEMPTS, 0) + 1
            val lockoutDelayMs = when {
                failedCount >= 5 -> 300_000L
                failedCount >= 3 -> 30_000L
                else -> 0L
            }
            prefs.edit {
                putInt(KEY_FAILED_ATTEMPTS, failedCount)
                if (lockoutDelayMs > 0) {
                    putLong(KEY_LOCKOUT_TIMESTAMP, System.currentTimeMillis() + lockoutDelayMs)
                }
            }
        }
        return matches
    }

    fun isSessionUnlocked(): Boolean {
        if (!isParentalControlActive) return true
        val timeoutMs = sessionTimeoutMinutes * 60_000L
        return (System.currentTimeMillis() - sessionAuthenticatedTime) < timeoutMs
    }

    fun lockSessionNow() {
        sessionAuthenticatedTime = 0L
    }

    fun lockCategory(categoryName: String) {
        val locked = getLockedCategories().toMutableSet()
        locked.add(categoryName.trim().lowercase())
        prefs.edit { putStringSet(KEY_LOCKED_CATEGORIES, locked) }
    }

    fun unlockCategory(categoryName: String) {
        val locked = getLockedCategories().toMutableSet()
        locked.remove(categoryName.trim().lowercase())
        prefs.edit { putStringSet(KEY_LOCKED_CATEGORIES, locked) }
    }

    fun lockChannel(channelIdentity: String) {
        val locked = getLockedChannels().toMutableSet()
        locked.add(channelIdentity)
        prefs.edit { putStringSet(KEY_LOCKED_CHANNELS, locked) }
    }

    fun unlockChannel(channelIdentity: String) {
        val locked = getLockedChannels().toMutableSet()
        locked.remove(channelIdentity)
        prefs.edit { putStringSet(KEY_LOCKED_CHANNELS, locked) }
    }

    fun isChannelLocked(channelIdentity: String): Boolean {
        return getLockedChannels().contains(channelIdentity)
    }

    fun getCustomBlockedKeywords(): Set<String> =
        prefs.getStringSet(KEY_CUSTOM_BLOCKED_WORDS, emptySet()) ?: emptySet()

    fun addBlockedKeyword(keyword: String) {
        val blocked = getCustomBlockedKeywords().toMutableSet()
        blocked.add(keyword.trim().lowercase())
        prefs.edit { putStringSet(KEY_CUSTOM_BLOCKED_WORDS, blocked) }
    }

    fun removeBlockedKeyword(keyword: String) {
        val blocked = getCustomBlockedKeywords().toMutableSet()
        blocked.remove(keyword.trim().lowercase())
        prefs.edit { putStringSet(KEY_CUSTOM_BLOCKED_WORDS, blocked) }
    }

    fun isCategoryRestricted(categoryName: String): Boolean {
        val cat = categoryName.trim().lowercase()
        if (cat.isBlank()) return false
        val locked = getLockedCategories()
        if (locked.contains(cat)) return true

        val customBlocked = getCustomBlockedKeywords()
        if (customBlocked.any { cat.contains(it) }) return true

        return isAdultPattern(cat)
    }

    fun getLockedCategories(): Set<String> =
        prefs.getStringSet(KEY_LOCKED_CATEGORIES, emptySet()) ?: emptySet()

    fun getLockedChannels(): Set<String> =
        prefs.getStringSet(KEY_LOCKED_CHANNELS, emptySet()) ?: emptySet()

    private fun isAdultPattern(name: String): Boolean {
        val adultKeywords = listOf("adult", "xxx", "porn", "18+", "erotic", "nsfw", "for adults")
        return adultKeywords.any { name.contains(it) }
    }

    private fun hashPin(pin: String, salt: ByteArray): String {
        val spec = PBEKeySpec(pin.toCharArray(), salt, PBKDF2_ITERATIONS, PBKDF2_KEY_BITS)
        return try {
            val factory = SecretKeyFactory.getInstance("PBKDF2WithHmacSHA256")
            Base64.encodeToString(factory.generateSecret(spec).encoded, Base64.NO_WRAP)
        } finally {
            spec.clearPassword()
        }
    }
}
""",

    "ui/epg/EpgProgramDetailsBottomSheet.kt": """package vid.mat.app.ui.epg

import android.content.Intent
import android.os.Bundle
import android.view.LayoutInflater
import android.view.View
import android.view.ViewGroup
import android.widget.Toast
import androidx.lifecycle.lifecycleScope
import com.google.android.material.bottomsheet.BottomSheetDialogFragment
import dagger.hilt.android.AndroidEntryPoint
import kotlinx.coroutines.launch
import vid.mat.app.R
import vid.mat.app.catchup.UnifiedCatchupEngine
import vid.mat.app.data.model.Channel
import vid.mat.app.databinding.BottomSheetEpgProgramDetailsBinding
import vid.mat.app.dvr.ui.ScheduleRecordingDialogFragment
import vid.mat.app.epg.model.EpgProgram
import vid.mat.app.epg.reminder.EpgReminderScheduler
import vid.mat.app.image.LogoImageLoader
import vid.mat.app.security.ParentalGatekeeper
import vid.mat.app.ui.MediaPlayerActivity
import vid.mat.app.ui.security.ParentalPinChallengeDialog
import vid.mat.app.util.Utils.Companion.formatTime
import java.text.SimpleDateFormat
import java.util.Date
import java.util.Locale
import javax.inject.Inject

@AndroidEntryPoint
class EpgProgramDetailsBottomSheet : BottomSheetDialogFragment() {

    private var _binding: BottomSheetEpgProgramDetailsBinding? = null
    private val binding get() = _binding!!

    @Inject
    lateinit var unifiedCatchupEngine: UnifiedCatchupEngine

    @Inject
    lateinit var parentalGatekeeper: ParentalGatekeeper

    private var channel: Channel? = null
    private var program: EpgProgram? = null
    private var pendingParentalAction: (() -> Unit)? = null

    private val timeFormat = SimpleDateFormat("EEEE, d MMM    HH:mm", Locale.getDefault())
    private val endTimeFormat = SimpleDateFormat("HH:mm", Locale.getDefault())

    override fun onCreateView(
        inflater: LayoutInflater,
        container: ViewGroup?,
        savedInstanceState: Bundle?
    ): View {
        _binding = BottomSheetEpgProgramDetailsBinding.inflate(inflater, container, false)
        return binding.root
    }

    override fun onViewCreated(view: View, savedInstanceState: Bundle?) {
        super.onViewCreated(view, savedInstanceState)

        childFragmentManager.setFragmentResultListener(
            ParentalPinChallengeDialog.RESULT_KEY,
            viewLifecycleOwner
        ) { _, bundle ->
            if (bundle.getBoolean(ParentalPinChallengeDialog.KEY_UNLOCKED, false)) {
                childFragmentManager.clearFragmentResult(ParentalPinChallengeDialog.RESULT_KEY)
                val action = pendingParentalAction
                pendingParentalAction = null
                action?.invoke()
            }
        }

        val ch = readChannelArgument() ?: return
        val prog = readProgramArgument() ?: return

        val nowMs = System.currentTimeMillis()
        val isAiringNow = prog.startMs <= nowMs && prog.stopMs > nowMs
        val isPast = prog.stopMs <= nowMs

        val catchupWindowMs = ch.catchupDays.toLong() * 86_400_000L
        val catchupAvailable = isPast &&
            ch.catchupDays > 0 &&
            (nowMs - prog.stopMs) < catchupWindowMs

        binding.tvProgramTitle.text = prog.title
        binding.tvChannelName.text = ch.title
        binding.tvCategory.text = prog.category ?: "General"
        binding.tvTimeSlot.text = "${timeFormat.format(Date(prog.startMs))} - ${endTimeFormat.format(Date(prog.stopMs))}"

        val durationMs = prog.stopMs - prog.startMs
        binding.tvDuration.text = formatTime(durationMs)
        binding.tvDescription.text =
            prog.description ?: "No detailed synopsis available for this broadcast."

        LogoImageLoader.load(requireContext(), ch.title, ch.logoUrl, binding.ivChannelLogo, 8f)

        when {
            isAiringNow -> {
                binding.btnPrimaryAction.text = "Watch Live Now"
                binding.btnPrimaryAction.setIconResource(R.drawable.ic_play)
                binding.btnPrimaryAction.setOnClickListener { tuneChannel(ch) }

                binding.btnSecondaryAction.visibility = View.VISIBLE
                binding.btnSecondaryAction.text = "Record"
                binding.btnSecondaryAction.setOnClickListener { openDvrDialog(ch, prog) }
            }

            catchupAvailable -> {
                binding.btnPrimaryAction.text = "Play Catch-Up"
                binding.btnPrimaryAction.setIconResource(R.drawable.ic_replay)
                binding.btnPrimaryAction.setOnClickListener { playCatchup(ch, prog) }
                binding.btnSecondaryAction.visibility = View.GONE
            }

            isPast && !catchupAvailable -> {
                binding.btnPrimaryAction.text = "Watch Live Channel"
                binding.btnPrimaryAction.setIconResource(R.drawable.ic_play)
                binding.btnPrimaryAction.setOnClickListener { tuneChannel(ch) }

                binding.btnSecondaryAction.visibility = View.GONE
            }

            else -> {
                binding.btnPrimaryAction.text = "Set Reminder"
                binding.btnPrimaryAction.setIconResource(R.drawable.ic_notification)
                binding.btnPrimaryAction.setOnClickListener {
                    EpgReminderScheduler.scheduleReminder(
                        requireContext(), prog, ch.title, ch.channelIdentity
                    )
                    Toast.makeText(
                        requireContext(),
                        "Reminder set for '${prog.title}'",
                        Toast.LENGTH_SHORT
                    ).show()
                    dismiss()
                }

                binding.btnSecondaryAction.visibility = View.VISIBLE
                binding.btnSecondaryAction.text = "Schedule DVR"
                binding.btnSecondaryAction.setOnClickListener { openDvrDialog(ch, prog) }
            }
        }
    }

    private fun tuneChannel(ch: Channel) {
        if (parentalGatekeeper.isChannelRestricted(ch)) {
            pendingParentalAction = { executeTuneChannel(ch) }
            ParentalPinChallengeDialog.newInstance(
                title = "Restricted Channel",
                message = "'${ch.title}' is locked by parental controls."
            ).show(childFragmentManager, ParentalPinChallengeDialog.TAG)
            return
        }
        executeTuneChannel(ch)
    }

    private fun executeTuneChannel(ch: Channel) {
        val intent = Intent(requireContext(), MediaPlayerActivity::class.java).apply {
            putExtra(MediaPlayerActivity.EXTRA_MEDIA_PATH, ch.channelIdentity)
            putExtra(MediaPlayerActivity.EXTRA_MEDIA_TITLE, ch.title)
            putExtra(MediaPlayerActivity.EXTRA_MEDIA_THUMBNAIL, ch.logoUrl)
            putExtra("group_title", ch.groupTitle)
        }
        startActivity(intent)
        dismiss()
    }

    private fun playCatchup(ch: Channel, prog: EpgProgram) {
        if (parentalGatekeeper.isChannelRestricted(ch)) {
            pendingParentalAction = { executePlayCatchup(ch, prog) }
            ParentalPinChallengeDialog.newInstance(
                title = "Restricted Channel",
                message = "'${ch.title}' is locked by parental controls."
            ).show(childFragmentManager, ParentalPinChallengeDialog.TAG)
            return
        }
        executePlayCatchup(ch, prog)
    }

    private fun executePlayCatchup(ch: Channel, prog: EpgProgram) {
        lifecycleScope.launch {
            val result = unifiedCatchupEngine.resolveCatchupDescriptor(ch.channelId, prog.startMs, prog.stopMs)
            if (result.isFailure) {
                Toast.makeText(
                    requireContext(),
                    result.exceptionOrNull()?.message ?: "Catch-up is unavailable for this program.",
                    Toast.LENGTH_LONG
                ).show()
                return@launch
            }

            val catchupId = "catchup_${ch.channelId}_${prog.startMs}_${prog.stopMs}"
            val intent = Intent(requireContext(), MediaPlayerActivity::class.java).apply {
                putExtra(MediaPlayerActivity.EXTRA_MEDIA_PATH, catchupId)
                putExtra(MediaPlayerActivity.EXTRA_MEDIA_TITLE, "${ch.title}: ${prog.title}")
                putExtra(MediaPlayerActivity.EXTRA_MEDIA_THUMBNAIL, ch.logoUrl)
                putExtra("group_title", ch.groupTitle)
                putExtra(MediaPlayerActivity.EXTRA_RESUME_POSITION, 0L)
            }
            startActivity(intent)
            dismiss()
        }
    }

    private fun openDvrDialog(ch: Channel, prog: EpgProgram) {
        ScheduleRecordingDialogFragment.newInstance(ch, prog)
            .show(parentFragmentManager, ScheduleRecordingDialogFragment.TAG)
        dismiss()
    }

    override fun onDestroyView() {
        super.onDestroyView()
        _binding = null
    }

    @Suppress("DEPRECATION")
    private fun readChannelArgument(): Channel? =
        (arguments?.getParcelable(ARG_CHANNEL) as? Channel) ?: channel

    @Suppress("DEPRECATION")
    private fun readProgramArgument(): EpgProgram? =
        (arguments?.getParcelable(ARG_PROGRAM) as? EpgProgram) ?: program

    companion object {
        const val TAG = "EpgProgramDetailsBottomSheet"
        private const val ARG_CHANNEL = "arg_channel"
        private const val ARG_PROGRAM = "arg_program"

        fun newInstance(channel: Channel, program: EpgProgram): EpgProgramDetailsBottomSheet {
            return EpgProgramDetailsBottomSheet().apply {
                arguments = Bundle().apply {
                    putParcelable(ARG_CHANNEL, channel)
                    putParcelable(ARG_PROGRAM, program)
                }
            }
        }
    }
}
""",

    "favorites/ui/FavoritesFragment.kt": """package vid.mat.app.favorites.ui

import android.content.Intent
import android.os.Bundle
import android.view.LayoutInflater
import android.view.View
import android.view.ViewGroup
import androidx.appcompat.widget.SearchView
import androidx.fragment.app.Fragment
import androidx.fragment.app.viewModels
import androidx.lifecycle.Lifecycle
import androidx.lifecycle.lifecycleScope
import androidx.lifecycle.repeatOnLifecycle
import androidx.recyclerview.widget.GridLayoutManager
import com.google.android.material.chip.Chip
import dagger.hilt.android.AndroidEntryPoint
import kotlinx.coroutines.launch
import vid.mat.app.databinding.FragmentUnifiedFavoritesBinding
import vid.mat.app.favorites.model.FavoriteType
import vid.mat.app.favorites.model.UnifiedFavoriteEntity
import vid.mat.app.security.ParentalGatekeeper
import vid.mat.app.ui.MediaPlayerActivity
import vid.mat.app.ui.security.ParentalPinChallengeDialog
import vid.mat.app.ui.series.SeriesDetailsActivity
import vid.mat.app.ui.vod.VodDetailsActivity
import javax.inject.Inject

@AndroidEntryPoint
class FavoritesFragment : Fragment() {

    private var _binding: FragmentUnifiedFavoritesBinding? = null
    private val binding get() = _binding!!
    private val viewModel: FavoritesViewModel by viewModels()

    @Inject
    lateinit var parentalGatekeeper: ParentalGatekeeper

    private lateinit var adapter: UnifiedFavoriteAdapter
    private var pendingParentalAction: (() -> Unit)? = null

    override fun onCreateView(inflater: LayoutInflater, container: ViewGroup?, savedInstanceState: Bundle?): View {
        _binding = FragmentUnifiedFavoritesBinding.inflate(inflater, container, false)
        return binding.root
    }

    override fun onViewCreated(view: View, savedInstanceState: Bundle?) {
        super.onViewCreated(view, savedInstanceState)

        parentFragmentManager.setFragmentResultListener(
            ParentalPinChallengeDialog.RESULT_KEY,
            viewLifecycleOwner
        ) { _, bundle ->
            if (bundle.getBoolean(ParentalPinChallengeDialog.KEY_UNLOCKED, false)) {
                parentFragmentManager.clearFragmentResult(ParentalPinChallengeDialog.RESULT_KEY)
                val action = pendingParentalAction
                pendingParentalAction = null
                action?.invoke()
            }
        }

        setupRecycler()
        setupSearch()
        setupTypeChips()
        observeFavorites()
    }

    private fun setupRecycler() {
        adapter = UnifiedFavoriteAdapter(
            onItemClick = { item -> playFavorite(item) },
            onRemoveClick = { item -> viewModel.removeFavorite(item) }
        )
        binding.rvFavorites.apply {
            layoutManager = GridLayoutManager(requireContext(), 2)
            this.adapter = this@FavoritesFragment.adapter
            setHasFixedSize(true)
        }
    }

    private fun setupSearch() {
        binding.searchView.setOnQueryTextListener(object : SearchView.OnQueryTextListener {
            override fun onQueryTextSubmit(query: String?): Boolean = false
            override fun onQueryTextChange(newText: String?): Boolean {
                viewModel.setSearchQuery(newText.orEmpty())
                return true
            }
        })
    }

    private fun setupTypeChips() {
        binding.chipGroupType.removeAllViews()
        FavoriteType.entries.forEach { type ->
            val chip = Chip(requireContext()).apply {
                text = type.displayName
                isCheckable = true
                isChecked = type == FavoriteType.ALL
                setOnCheckedChangeListener { _, isChecked ->
                    if (isChecked) viewModel.selectType(type)
                }
            }
            binding.chipGroupType.addView(chip)
        }
    }

    private fun observeFavorites() {
        viewLifecycleOwner.lifecycleScope.launch {
            viewLifecycleOwner.repeatOnLifecycle(Lifecycle.State.STARTED) {
                viewModel.favoritesList.collect { list ->
                    adapter.submitList(list)
                    binding.layoutEmpty.visibility = if (list.isEmpty()) View.VISIBLE else View.GONE
                }
            }
        }
    }

    private fun checkParentalGateAndRun(
        title: String,
        groupTitle: String?,
        targetId: String?,
        action: () -> Unit
    ) {
        val restricted = parentalGatekeeper.isCategoryRestricted(groupTitle) ||
            parentalGatekeeper.isTargetRestricted(targetId)
        if (restricted) {
            pendingParentalAction = action
            ParentalPinChallengeDialog.newInstance(
                title = "Restricted Content",
                message = "'$title' is protected by parental controls."
            ).show(parentFragmentManager, ParentalPinChallengeDialog.TAG)
        } else {
            action()
        }
    }

    private fun playFavorite(item: UnifiedFavoriteEntity) {
        checkParentalGateAndRun(item.title, item.groupTitle, item.targetId) {
            when (item.type) {
                FavoriteType.SERIES -> {
                    val seriesDbId = item.targetId.removePrefix("series_").toLongOrNull() ?: return@checkParentalGateAndRun
                    startActivity(
                        Intent(requireContext(), SeriesDetailsActivity::class.java).apply {
                            putExtra(SeriesDetailsActivity.EXTRA_SERIES_ID, seriesDbId)
                        }
                    )
                }
                FavoriteType.VOD_MOVIE -> {
                    val movieDbId = item.targetId.removePrefix("movie_").toLongOrNull()
                    if (movieDbId != null) {
                        startActivity(
                            Intent(requireContext(), VodDetailsActivity::class.java).apply {
                                putExtra(VodDetailsActivity.EXTRA_MOVIE_ID, movieDbId)
                            }
                        )
                    } else {
                        launchPlayer(item.streamUrl, item)
                    }
                }
                FavoriteType.LIVE_CHANNEL -> {
                    val mediaKey = if (item.targetId.startsWith("live_identity:")) {
                        item.targetId.removePrefix("live_identity:")
                    } else {
                        item.streamUrl.ifBlank { item.targetId }
                    }
                    launchPlayer(mediaKey, item)
                }
                else -> {
                    if (item.streamUrl.isNotBlank()) launchPlayer(item.streamUrl, item)
                }
            }
        }
    }

    private fun launchPlayer(mediaPath: String, item: UnifiedFavoriteEntity) {
        if (mediaPath.isBlank()) return
        val intent = Intent(requireContext(), MediaPlayerActivity::class.java).apply {
            putExtra(MediaPlayerActivity.EXTRA_MEDIA_PATH, mediaPath)
            putExtra(MediaPlayerActivity.EXTRA_MEDIA_TITLE, item.title)
            putExtra(MediaPlayerActivity.EXTRA_MEDIA_THUMBNAIL, item.logoUrl)
            putExtra("group_title", item.groupTitle)
        }
        startActivity(intent)
    }

    override fun onDestroyView() {
        super.onDestroyView()
        _binding = null
    }
}
""",

    "history/ui/UnifiedHistoryFragment.kt": """package vid.mat.app.history.ui

import android.content.Intent
import android.os.Bundle
import android.view.LayoutInflater
import android.view.View
import android.view.ViewGroup
import androidx.appcompat.widget.SearchView
import androidx.fragment.app.Fragment
import androidx.fragment.app.viewModels
import androidx.lifecycle.Lifecycle
import androidx.lifecycle.lifecycleScope
import androidx.lifecycle.repeatOnLifecycle
import androidx.recyclerview.widget.ItemTouchHelper
import androidx.recyclerview.widget.LinearLayoutManager
import androidx.recyclerview.widget.RecyclerView
import com.google.android.material.chip.Chip
import com.google.android.material.dialog.MaterialAlertDialogBuilder
import com.google.android.material.snackbar.Snackbar
import dagger.hilt.android.AndroidEntryPoint
import kotlinx.coroutines.launch
import vid.mat.app.data.model.WatchProgressEntity
import vid.mat.app.databinding.FragmentUnifiedHistoryBinding
import vid.mat.app.security.ParentalGatekeeper
import vid.mat.app.ui.MediaPlayerActivity
import vid.mat.app.ui.security.ParentalPinChallengeDialog
import javax.inject.Inject

@AndroidEntryPoint
class UnifiedHistoryFragment : Fragment() {

    private var _binding: FragmentUnifiedHistoryBinding? = null
    private val binding get() = _binding!!
    private val viewModel: UnifiedHistoryViewModel by viewModels()

    @Inject
    lateinit var parentalGatekeeper: ParentalGatekeeper

    private lateinit var adapter: UnifiedHistoryAdapter
    private var pendingParentalAction: (() -> Unit)? = null

    override fun onCreateView(inflater: LayoutInflater, container: ViewGroup?, savedInstanceState: Bundle?): View {
        _binding = FragmentUnifiedHistoryBinding.inflate(inflater, container, false)
        return binding.root
    }

    override fun onViewCreated(view: View, savedInstanceState: Bundle?) {
        super.onViewCreated(view, savedInstanceState)

        parentFragmentManager.setFragmentResultListener(
            ParentalPinChallengeDialog.RESULT_KEY,
            viewLifecycleOwner
        ) { _, bundle ->
            if (bundle.getBoolean(ParentalPinChallengeDialog.KEY_UNLOCKED, false)) {
                parentFragmentManager.clearFragmentResult(ParentalPinChallengeDialog.RESULT_KEY)
                val action = pendingParentalAction
                pendingParentalAction = null
                action?.invoke()
            }
        }

        setupRecycler()
        setupSearch()
        setupTypeChips()
        setupActions()
        observeHistory()
    }

    private fun setupRecycler() {
        adapter = UnifiedHistoryAdapter(
            onItemClick = { item -> resumePlayback(item) },
            onRestartClick = { item -> restartPlayback(item) },
            onRemoveClick = { item -> viewModel.removeEntry(item.mediaKey) }
        )

        val touchHelper = ItemTouchHelper(object : ItemTouchHelper.SimpleCallback(0, ItemTouchHelper.LEFT or ItemTouchHelper.RIGHT) {
            override fun onMove(rv: RecyclerView, vh: RecyclerView.ViewHolder, target: RecyclerView.ViewHolder): Boolean = false
            override fun onSwiped(vh: RecyclerView.ViewHolder, direction: Int) {
                val item = (adapter.currentList.getOrNull(vh.adapterPosition) as? HistoryDisplayItem.Entry)?.progress ?: return
                viewModel.removeEntry(item.mediaKey)
                Snackbar.make(binding.root, "Removed from history", Snackbar.LENGTH_SHORT).show()
            }
        })

        binding.rvHistory.apply {
            layoutManager = LinearLayoutManager(requireContext())
            this.adapter = this@UnifiedHistoryFragment.adapter
            touchHelper.attachToRecyclerView(this)
            setHasFixedSize(true)
        }
    }

    private fun setupSearch() {
        binding.searchView.setOnQueryTextListener(object : SearchView.OnQueryTextListener {
            override fun onQueryTextSubmit(query: String?): Boolean = false
            override fun onQueryTextChange(newText: String?): Boolean {
                viewModel.setSearchQuery(newText.orEmpty())
                return true
            }
        })
    }

    private fun setupTypeChips() {
        binding.chipGroupFilter.removeAllViews()
        HistoryTypeFilter.entries.forEach { filter ->
            val chip = Chip(requireContext()).apply {
                text = filter.displayName
                isCheckable = true
                isChecked = filter == HistoryTypeFilter.ALL
                setOnCheckedChangeListener { _, isChecked ->
                    if (isChecked) viewModel.setTypeFilter(filter)
                }
            }
            binding.chipGroupFilter.addView(chip)
        }
    }

    private fun setupActions() {
        binding.btnClearAll.setOnClickListener {
            MaterialAlertDialogBuilder(requireContext())
                .setTitle("Clear Watch History?")
                .setMessage("All watch history and resume positions for this profile will be cleared.")
                .setPositiveButton("Clear All") { _, _ -> viewModel.clearAllHistory() }
                .setNegativeButton("Cancel", null)
                .show()
        }

        binding.btnPauseHistory.setOnClickListener {
            viewModel.togglePauseHistory()
        }
    }

    private fun observeHistory() {
        viewLifecycleOwner.lifecycleScope.launch {
            viewLifecycleOwner.repeatOnLifecycle(Lifecycle.State.STARTED) {
                viewModel.historyItems.collect { items ->
                    adapter.submitList(items)
                    binding.layoutEmpty.visibility = if (items.isEmpty()) View.VISIBLE else View.GONE
                }
            }
        }

        viewLifecycleOwner.lifecycleScope.launch {
            viewLifecycleOwner.repeatOnLifecycle(Lifecycle.State.STARTED) {
                viewModel.isHistoryPaused.collect { isPaused ->
                    binding.btnPauseHistory.text = if (isPaused) "Resume History" else "Pause History"
                }
            }
        }
    }

    private fun checkParentalGateAndRun(
        title: String,
        groupTitle: String?,
        targetId: String?,
        action: () -> Unit
    ) {
        val restricted = parentalGatekeeper.isCategoryRestricted(groupTitle) ||
            parentalGatekeeper.isTargetRestricted(targetId)
        if (restricted) {
            pendingParentalAction = action
            ParentalPinChallengeDialog.newInstance(
                title = "Restricted Content",
                message = "'$title' is protected by parental controls."
            ).show(parentFragmentManager, ParentalPinChallengeDialog.TAG)
        } else {
            action()
        }
    }

    private fun resumePlayback(item: WatchProgressEntity) {
        checkParentalGateAndRun(item.mediaTitle, item.mediaSubtitle, item.mediaKey) {
            val intent = Intent(requireContext(), MediaPlayerActivity::class.java).apply {
                putExtra(MediaPlayerActivity.EXTRA_MEDIA_PATH, item.streamUrl)
                putExtra(MediaPlayerActivity.EXTRA_MEDIA_TITLE, item.mediaTitle)
                putExtra(MediaPlayerActivity.EXTRA_MEDIA_THUMBNAIL, item.thumbnailUrl)
                putExtra(MediaPlayerActivity.EXTRA_RESUME_POSITION, item.positionMs)
                putExtra("group_title", item.mediaSubtitle)
            }
            startActivity(intent)
        }
    }

    private fun restartPlayback(item: WatchProgressEntity) {
        checkParentalGateAndRun(item.mediaTitle, item.mediaSubtitle, item.mediaKey) {
            val intent = Intent(requireContext(), MediaPlayerActivity::class.java).apply {
                putExtra(MediaPlayerActivity.EXTRA_MEDIA_PATH, item.streamUrl)
                putExtra(MediaPlayerActivity.EXTRA_MEDIA_TITLE, item.mediaTitle)
                putExtra(MediaPlayerActivity.EXTRA_MEDIA_THUMBNAIL, item.thumbnailUrl)
                putExtra(MediaPlayerActivity.EXTRA_RESUME_POSITION, 0L)
                putExtra("group_title", item.mediaSubtitle)
            }
            startActivity(intent)
        }
    }

    override fun onDestroyView() {
        super.onDestroyView()
        _binding = null
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

May I proceed with **Batch 9 (5 classes — Home Dashboard, Crypto/Network Hardening & Parental Settings)** (`home/ui/HomeDashboardFragment.kt`, `home/SmartHomeDashboardRepository.kt`, `home/ui/HomeDashboardSectionAdapter.kt`, `security/EncryptedField.kt`, `security/NetworkSecurityPolicy.kt`)?
Here is the Python script for **Batch 9 (5 classes — Home Dashboard & Security Hardening)**:

1. `home/ui/HomeDashboardFragment.kt` — Fixes `binding.layoutEmpty.root` visibility bug, passes `EXTRA_MEDIA_THUMBNAIL` on Continue cards, and enforces `ParentalGatekeeper` + PIN challenge on card clicks.
2. `home/SmartHomeDashboardRepository.kt` — Uses the Room `SeriesEntity.id` primary key (instead of provider `seriesId`) when constructing `HomeCardItem.SeriesCard`.
3. `home/ui/HomeDashboardSectionAdapter.kt` — Reuses `MediaCardAdapter` and shares `OptimizedViewPoolFactory.getSharedViewPool()` across horizontal rows.
4. `security/EncryptedField.kt` — Returns `null` instead of leaking `ENC:` ciphertext strings when decryption fails or format is malformed.
5. `security/NetworkSecurityPolicy.kt` — Re-evaluates `resolveStreamContext(nextUrl.toString())` during stream redirects in `executeSafeHttpCall`.

```python
#!/usr/bin/env python3
from pathlib import Path

BASE_DIR = Path("/home/username/AndroidStudioProjects/Ytvmate/app/src/main/java/vid/mat/app")

FILES = {
    "home/ui/HomeDashboardFragment.kt": """package vid.mat.app.home.ui

import android.content.Intent
import android.os.Bundle
import android.view.LayoutInflater
import android.view.View
import android.view.ViewGroup
import androidx.fragment.app.Fragment
import androidx.fragment.app.viewModels
import androidx.lifecycle.Lifecycle
import androidx.lifecycle.lifecycleScope
import androidx.lifecycle.repeatOnLifecycle
import androidx.recyclerview.widget.LinearLayoutManager
import dagger.hilt.android.AndroidEntryPoint
import kotlinx.coroutines.launch
import vid.mat.app.databinding.FragmentHomeDashboardBinding
import vid.mat.app.home.model.HomeCardItem
import vid.mat.app.security.ParentalGatekeeper
import vid.mat.app.ui.MediaPlayerActivity
import vid.mat.app.ui.security.ParentalPinChallengeDialog
import vid.mat.app.ui.series.SeriesDetailsActivity
import vid.mat.app.ui.vod.VodDetailsActivity
import javax.inject.Inject

@AndroidEntryPoint
class HomeDashboardFragment : Fragment() {

    private var _binding: FragmentHomeDashboardBinding? = null
    private val binding get() = _binding!!
    private val viewModel: HomeDashboardViewModel by viewModels()

    @Inject
    lateinit var parentalGatekeeper: ParentalGatekeeper

    private lateinit var sectionAdapter: HomeDashboardSectionAdapter
    private var pendingParentalAction: (() -> Unit)? = null

    override fun onCreateView(
        inflater: LayoutInflater,
        container: ViewGroup?,
        savedInstanceState: Bundle?
    ): View {
        _binding = FragmentHomeDashboardBinding.inflate(inflater, container, false)
        return binding.root
    }

    override fun onViewCreated(view: View, savedInstanceState: Bundle?) {
        super.onViewCreated(view, savedInstanceState)

        parentFragmentManager.setFragmentResultListener(
            ParentalPinChallengeDialog.RESULT_KEY,
            viewLifecycleOwner
        ) { _, bundle ->
            if (bundle.getBoolean(ParentalPinChallengeDialog.KEY_UNLOCKED, false)) {
                parentFragmentManager.clearFragmentResult(ParentalPinChallengeDialog.RESULT_KEY)
                val action = pendingParentalAction
                pendingParentalAction = null
                action?.invoke()
            }
        }

        setupRecycler()
        observeDashboard()
    }

    private fun setupRecycler() {
        sectionAdapter = HomeDashboardSectionAdapter { card ->
            handleCardClick(card)
        }
        binding.rvDashboardSections.apply {
            layoutManager = LinearLayoutManager(requireContext())
            adapter = sectionAdapter
            setHasFixedSize(true)
        }
    }

    private fun observeDashboard() {
        viewLifecycleOwner.lifecycleScope.launch {
            viewLifecycleOwner.repeatOnLifecycle(Lifecycle.State.STARTED) {
                viewModel.dashboardState.collect { state ->
                    sectionAdapter.submitList(state.sections)
                    binding.layoutEmpty.root.visibility =
                        if (state.sections.isEmpty() && !state.isLoading) View.VISIBLE else View.GONE
                }
            }
        }
    }

    private fun checkParentalGateAndRun(
        title: String,
        groupTitle: String?,
        targetId: String?,
        action: () -> Unit
    ) {
        val restricted = parentalGatekeeper.isCategoryRestricted(groupTitle) ||
            parentalGatekeeper.isTargetRestricted(targetId)
        if (restricted) {
            pendingParentalAction = action
            ParentalPinChallengeDialog.newInstance(
                title = "Restricted Content",
                message = "'$title' is protected by parental controls."
            ).show(parentFragmentManager, ParentalPinChallengeDialog.TAG)
        } else {
            action()
        }
    }

    private fun handleCardClick(card: HomeCardItem) {
        when (card) {
            is HomeCardItem.LiveChannelCard -> {
                checkParentalGateAndRun(card.title, card.subtitle, card.channelIdentity) {
                    val intent = Intent(requireContext(), MediaPlayerActivity::class.java).apply {
                        putExtra(MediaPlayerActivity.EXTRA_MEDIA_PATH, card.channelIdentity)
                        putExtra(MediaPlayerActivity.EXTRA_MEDIA_TITLE, card.title)
                        putExtra(MediaPlayerActivity.EXTRA_MEDIA_THUMBNAIL, card.imageUrl)
                        putExtra("group_title", card.subtitle)
                    }
                    startActivity(intent)
                }
            }
            is HomeCardItem.MovieCard -> {
                checkParentalGateAndRun(card.title, card.subtitle, "movie_${card.movieId}") {
                    val intent = Intent(requireContext(), VodDetailsActivity::class.java).apply {
                        putExtra(VodDetailsActivity.EXTRA_MOVIE_ID, card.movieId)
                    }
                    startActivity(intent)
                }
            }
            is HomeCardItem.ContinueWatchingCard -> {
                checkParentalGateAndRun(card.title, card.subtitle, card.id) {
                    val intent = Intent(requireContext(), MediaPlayerActivity::class.java).apply {
                        putExtra(MediaPlayerActivity.EXTRA_MEDIA_PATH, card.streamUrl)
                        putExtra(MediaPlayerActivity.EXTRA_MEDIA_TITLE, card.title)
                        putExtra(MediaPlayerActivity.EXTRA_MEDIA_THUMBNAIL, card.imageUrl)
                        putExtra(MediaPlayerActivity.EXTRA_RESUME_POSITION, card.resumePositionMs)
                        putExtra("group_title", card.subtitle)
                    }
                    startActivity(intent)
                }
            }
            is HomeCardItem.SeriesCard -> {
                checkParentalGateAndRun(card.title, card.subtitle, "series_${card.seriesId}") {
                    val intent = Intent(requireContext(), SeriesDetailsActivity::class.java).apply {
                        putExtra(SeriesDetailsActivity.EXTRA_SERIES_ID, card.seriesId)
                    }
                    startActivity(intent)
                }
            }
        }
    }

    override fun onDestroyView() {
        super.onDestroyView()
        _binding = null
    }
}
""",

    "home/SmartHomeDashboardRepository.kt": """package vid.mat.app.home

import kotlinx.coroutines.Dispatchers
import kotlinx.coroutines.flow.Flow
import kotlinx.coroutines.flow.combine
import kotlinx.coroutines.flow.flowOn
import vid.mat.app.data.model.Channel
import vid.mat.app.data.model.SeriesEntity
import vid.mat.app.data.model.VodMovieEntity
import vid.mat.app.data.model.WatchProgressEntity
import vid.mat.app.data.repository.ChannelRepository
import vid.mat.app.data.repository.SeriesRepository
import vid.mat.app.data.repository.VodRepository
import vid.mat.app.history.WatchProgressManager
import vid.mat.app.home.model.HomeCardItem
import vid.mat.app.home.model.HomeDashboardSection
import vid.mat.app.home.model.HomeSectionData
import vid.mat.app.profile.ActiveProfileProvider
import vid.mat.app.security.ParentalGatekeeper
import javax.inject.Inject
import javax.inject.Singleton

@Singleton
class SmartHomeDashboardRepository @Inject constructor(
    private val watchProgressManager: WatchProgressManager,
    private val channelRepository: ChannelRepository,
    private val vodRepository: VodRepository,
    private val seriesRepository: SeriesRepository,
    private val activeProfileProvider: ActiveProfileProvider,
    private val parentalGatekeeper: ParentalGatekeeper
) {

    private data class DashboardInputs(
        val continueItems: List<WatchProgressEntity>,
        val favChannels: List<Channel>,
        val recentChannels: List<Channel>,
        val favMovies: List<VodMovieEntity>,
        val favSeries: List<SeriesEntity>
    )

    fun getDashboardSections(): Flow<List<HomeSectionData>> = combine(
        watchProgressManager.continueWatching,
        channelRepository.favorites,
        channelRepository.recents,
        vodRepository.favorites,
        seriesRepository.favorites
    ) { continueItems, favChannels, recentChannels, favMovies, favSeries ->
        DashboardInputs(
            continueItems = continueItems,
            favChannels = favChannels,
            recentChannels = recentChannels,
            favMovies = favMovies,
            favSeries = favSeries
        )
    }.combine(activeProfileProvider.activeProfile) { inputs, _ ->
        buildSections(inputs)
    }.flowOn(Dispatchers.IO)

    private fun buildSections(inputs: DashboardInputs): List<HomeSectionData> {
        val sections = mutableListOf<HomeSectionData>()

        val filteredContinue = inputs.continueItems.filterNot { item ->
            parentalGatekeeper.isCategoryRestricted(item.mediaSubtitle) ||
                parentalGatekeeper.isTargetRestricted(item.mediaKey)
        }
        if (filteredContinue.isNotEmpty()) {
            val cards = filteredContinue.map { c ->
                HomeCardItem.ContinueWatchingCard(
                    id = c.mediaKey,
                    title = c.mediaTitle,
                    subtitle = c.mediaSubtitle,
                    imageUrl = c.thumbnailUrl,
                    streamUrl = c.streamUrl,
                    resumePositionMs = c.positionMs,
                    progressPercent = c.completionPercent
                )
            }
            sections.add(HomeSectionData(HomeDashboardSection.CONTINUE_WATCHING, "Continue Watching", cards))
        }

        val filteredFavChannels = inputs.favChannels.filterNot { ch -> parentalGatekeeper.isChannelRestricted(ch) }
        val filteredFavMovies   = inputs.favMovies.filterNot   { m  -> parentalGatekeeper.isMovieRestricted(m) }
        val filteredFavSeries   = inputs.favSeries.filterNot   { s  -> parentalGatekeeper.isSeriesRestricted(s) }

        val favoriteCards = mutableListOf<HomeCardItem>()

        filteredFavChannels.forEach { ch ->
            favoriteCards.add(
                HomeCardItem.LiveChannelCard(
                    id = "fav_ch_${ch.channelId}",
                    title = ch.title,
                    subtitle = ch.groupTitle,
                    imageUrl = ch.logoUrl,
                    channelIdentity = ch.channelIdentity
                )
            )
        }
        filteredFavMovies.forEach { m ->
            favoriteCards.add(
                HomeCardItem.MovieCard(
                    id = "fav_mov_${m.id}",
                    title = m.title,
                    subtitle = m.categoryName,
                    imageUrl = m.posterUrl ?: m.backdropUrl,
                    movieId = m.id
                )
            )
        }
        filteredFavSeries.forEach { s ->
            favoriteCards.add(
                HomeCardItem.SeriesCard(
                    id = "fav_series_${s.id}",
                    title = s.title,
                    subtitle = s.genre ?: s.categoryName,
                    imageUrl = s.coverUrl ?: s.backdropUrl,
                    seriesId = s.id
                )
            )
        }
        if (favoriteCards.isNotEmpty()) {
            sections.add(HomeSectionData(HomeDashboardSection.FAVORITES, "Favorites", favoriteCards))
        }

        val filteredRecents = inputs.recentChannels.filterNot { ch -> parentalGatekeeper.isChannelRestricted(ch) }
        if (filteredRecents.isNotEmpty()) {
            val recentsCards = filteredRecents.map { ch ->
                HomeCardItem.LiveChannelCard(
                    id = "rec_${ch.channelId}",
                    title = ch.title,
                    subtitle = ch.groupTitle,
                    imageUrl = ch.logoUrl,
                    channelIdentity = ch.channelIdentity
                )
            }
            sections.add(HomeSectionData(HomeDashboardSection.RECENTLY_WATCHED, "Recently Watched", recentsCards))
        }

        return sections.sortedByDescending { it.section.basePriority }
    }
}
""",

    "home/ui/HomeDashboardSectionAdapter.kt": """package vid.mat.app.home.ui

import android.view.LayoutInflater
import android.view.View
import android.view.ViewGroup
import android.widget.ProgressBar
import android.widget.TextView
import androidx.recyclerview.widget.DiffUtil
import androidx.recyclerview.widget.LinearLayoutManager
import androidx.recyclerview.widget.ListAdapter
import androidx.recyclerview.widget.RecyclerView
import vid.mat.app.R
import vid.mat.app.databinding.ItemHomeMediaCardBinding
import vid.mat.app.databinding.ItemHomeSectionRowBinding
import vid.mat.app.home.model.HomeCardItem
import vid.mat.app.home.model.HomeSectionData
import vid.mat.app.image.LogoImageLoader
import vid.mat.app.util.performance.OptimizedViewPoolFactory

class HomeDashboardSectionAdapter(
    private val onCardClick: (HomeCardItem) -> Unit
) : ListAdapter<HomeSectionData, HomeDashboardSectionAdapter.SectionViewHolder>(DiffCallback()) {

    private val sharedCardPool = OptimizedViewPoolFactory.getSharedViewPool()

    override fun onCreateViewHolder(parent: ViewGroup, viewType: Int): SectionViewHolder {
        val binding = ItemHomeSectionRowBinding.inflate(LayoutInflater.from(parent.context), parent, false)
        return SectionViewHolder(binding, onCardClick, sharedCardPool)
    }

    override fun onBindViewHolder(holder: SectionViewHolder, position: Int) {
        holder.bind(getItem(position))
    }

    class SectionViewHolder(
        private val binding: ItemHomeSectionRowBinding,
        onCardClick: (HomeCardItem) -> Unit,
        sharedPool: RecyclerView.RecycledViewPool
    ) : RecyclerView.ViewHolder(binding.root) {

        private val cardAdapter = MediaCardAdapter(onCardClick)

        init {
            binding.rvSectionItems.apply {
                layoutManager = LinearLayoutManager(binding.root.context, LinearLayoutManager.HORIZONTAL, false)
                adapter = cardAdapter
                setRecycledViewPool(sharedPool)
                setHasFixedSize(true)
            }
        }

        fun bind(section: HomeSectionData) {
            binding.tvSectionTitle.text = section.title
            cardAdapter.submitList(section.items)
        }
    }

    class MediaCardAdapter(
        private val onClick: (HomeCardItem) -> Unit
    ) : ListAdapter<HomeCardItem, MediaCardAdapter.CardVH>(CardDiff()) {

        override fun onCreateViewHolder(parent: ViewGroup, viewType: Int): CardVH {
            val binding = ItemHomeMediaCardBinding.inflate(LayoutInflater.from(parent.context), parent, false)
            return CardVH(binding)
        }

        override fun onBindViewHolder(holder: CardVH, position: Int) {
            holder.bind(getItem(position), onClick)
        }

        class CardVH(val binding: ItemHomeMediaCardBinding) : RecyclerView.ViewHolder(binding.root) {
            fun bind(item: HomeCardItem, onClick: (HomeCardItem) -> Unit) {
                binding.tvTitle.text = item.title
                binding.tvSubtitle.text = item.subtitle ?: ""

                val tvBadge = binding.root.findViewById<TextView?>(R.id.tvBadge)
                val progressBar = binding.root.findViewById<ProgressBar?>(R.id.progressBar)

                if (item is HomeCardItem.LiveChannelCard) {
                    tvBadge?.visibility = View.VISIBLE
                } else {
                    tvBadge?.visibility = View.GONE
                }

                if (item is HomeCardItem.ContinueWatchingCard) {
                    progressBar?.visibility = View.VISIBLE
                    progressBar?.progress = item.progressPercent
                } else {
                    progressBar?.visibility = View.GONE
                }

                LogoImageLoader.load(binding.ivPoster.context, item.title, item.imageUrl, binding.ivPoster, 10f)
                binding.root.setOnClickListener { onClick(item) }
            }
        }

        class CardDiff : DiffUtil.ItemCallback<HomeCardItem>() {
            override fun areItemsTheSame(a: HomeCardItem, b: HomeCardItem) = a.id == b.id
            override fun areContentsTheSame(a: HomeCardItem, b: HomeCardItem) = a == b
        }
    }

    class DiffCallback : DiffUtil.ItemCallback<HomeSectionData>() {
        override fun areItemsTheSame(a: HomeSectionData, b: HomeSectionData) = a.section == b.section
        override fun areContentsTheSame(a: HomeSectionData, b: HomeSectionData) = a == b
    }
}
""",

    "security/EncryptedField.kt": """package vid.mat.app.security

import android.os.Parcelable
import androidx.room.TypeConverter
import kotlinx.parcelize.Parcelize
import vid.mat.app.util.AppLogger

/**
 * Strongly-typed wrapper for credential-bearing database columns.
 * Redacts value in toString() to prevent accidental plaintext leaks in logs.
 */
@Parcelize
data class EncryptedField(val value: String) : Parcelable {
    override fun toString(): String = "[REDACTED]"

    companion object {
        fun from(value: String?): EncryptedField? =
            if (!value.isNullOrBlank()) EncryptedField(value.trim()) else null
    }
}

class EncryptedFieldConverter {

    @TypeConverter
    fun toEncryptedField(storedValue: String?): EncryptedField? {
        if (storedValue.isNullOrBlank()) return null
        if (storedValue.startsWith("ENC:")) {
            val parts = storedValue.removePrefix("ENC:").split(":")
            if (parts.size == 2) {
                val iv = parts[0]
                val cipher = parts[1]
                return when (val dec = CryptoUtils.decrypt(cipher, iv)) {
                    is CryptoUtils.DecryptionResult.Success -> EncryptedField(dec.plaintext)
                    else -> {
                        AppLogger.w("EncryptedFieldConverter", "Failed decrypting encrypted field (${dec::class.simpleName})")
                        null
                    }
                }
            }
            return null
        }
        return EncryptedField(storedValue)
    }

    @TypeConverter
    fun fromEncryptedField(field: EncryptedField?): String? {
        val plaintext = field?.value ?: return null
        if (plaintext.isBlank()) return null
        return when (val enc = CryptoUtils.encrypt(plaintext)) {
            is CryptoUtils.EncryptionResult.Success -> "ENC:${enc.result.ivBase64}:${enc.result.ciphertextBase64}"
            is CryptoUtils.EncryptionResult.Failure -> null
            else -> null
        }
    }
}
""",

    "security/NetworkSecurityPolicy.kt": """package vid.mat.app.security

import android.net.Uri
import okhttp3.Dns
import okhttp3.HttpUrl
import okhttp3.Interceptor
import okhttp3.OkHttpClient
import okhttp3.Request
import okhttp3.Response
import org.json.JSONObject
import vid.mat.app.network.BoundedResponseReader
import vid.mat.app.player.timeshift.TimeshiftLocalServer
import vid.mat.app.util.AppLogger
import vid.mat.app.util.SensitiveDataSanitizer
import java.io.File
import java.io.IOException
import java.net.Inet4Address
import java.net.Inet6Address
import java.net.InetAddress
import java.net.URI
import java.net.URLEncoder
import java.net.UnknownHostException
import java.nio.charset.StandardCharsets
import java.util.concurrent.ConcurrentHashMap
import java.util.concurrent.TimeUnit

enum class NetworkSecurityContext {
    REMOTE_PLAYLIST,
    REMOTE_EPG,
    XTREAM_API,
    PUBLIC_WEB,
    IPTV_STREAM,
    LOCAL_IPTV_GATEWAY,
    LOCAL_TIMESHIFT_LOOPBACK,
    LOCAL_FILE_SYSTEM
}

object NetworkSecurityValidator {

    private const val TAG = "NetworkSecurityValidator"
    private const val MAX_URL_LENGTH = 8_192
    private const val MAX_REDIRECT_DEPTH = 5

    private val HTTP_SCHEMES = setOf("http", "https")
    private val STREAMING_SCHEMES = setOf("rtmp", "rtmps", "rtsp", "udp", "rtp", "mms", "mmsh")
    private val LOCAL_MEDIA_SCHEMES = setOf("content", "file", "asset", "android.resource")
    private val STREAM_CONTEXTS = setOf(
        NetworkSecurityContext.IPTV_STREAM,
        NetworkSecurityContext.LOCAL_IPTV_GATEWAY,
        NetworkSecurityContext.LOCAL_TIMESHIFT_LOOPBACK
    )

    private val CLOUD_METADATA_IPS = setOf(
        "169.254.169.254",
        "169.254.170.2",
        "100.100.100.200"
    )

    private val CLOUD_METADATA_HOSTS = setOf(
        "metadata.google.internal",
        "instance-data",
        "metadata.packet.net",
        "169.254.169.254"
    )

    private val INJECTION_PATTERNS = listOf(
        " ", "%00", "<script", "javascript:", "vbscript:", "data:"
    )

    private val LOOPBACK_HOST_LITERALS = setOf("localhost", "127.0.0.1", "::1")
    private val NUMERIC_HOST_REGEX = Regex("""^[0-9a-fA-F:.\[\]]+$""")

    private val trustedLoopbackPorts: MutableSet<Int> = ConcurrentHashMap.newKeySet()

    fun sanitizeHeaderValue(value: String?): String? {
        if (value == null) return null
        return value.replace("\r", "").replace("\n", "").trim()
    }

    fun registerTrustedLoopbackOrigin(port: Int) {
        if (port in 1..65_535) {
            trustedLoopbackPorts.add(port)
            AppLogger.i(TAG, "Registered trusted loopback origin on port $port")
        } else {
            AppLogger.w(TAG, "Attempt to register invalid loopback port $port (ignored)")
        }
    }

    fun unregisterTrustedLoopbackOrigin(port: Int) {
        trustedLoopbackPorts.remove(port)
        AppLogger.d(TAG, "Unregistered trusted loopback origin on port $port")
    }

    fun hasActiveTrustedLoopbackOrigin(): Boolean {
        val active = trustedLoopbackPorts.isNotEmpty()
        AppLogger.d(TAG, "hasActiveTrustedLoopbackOrigin=$active (ports=${trustedLoopbackPorts.joinToString()})")
        return active
    }

    fun isTrustedLoopbackPort(port: Int): Boolean {
        val trusted = trustedLoopbackPorts.contains(port)
        AppLogger.d(TAG, "isTrustedLoopbackPort($port)=$trusted")
        return trusted
    }

    fun isLoopbackHostLiteral(host: String?): Boolean {
        if (host.isNullOrBlank()) {
            AppLogger.d(TAG, "isLoopbackHostLiteral: host blank -> false")
            return false
        }
        val normalized = host.lowercase().removeSurrounding("[", "]")
        val result = LOOPBACK_HOST_LITERALS.contains(normalized)
        AppLogger.d(TAG, "isLoopbackHostLiteral($host) -> $result")
        return result
    }

    fun parseNumericAddress(host: String?): InetAddress? {
        if (host.isNullOrBlank()) {
            AppLogger.d(TAG, "parseNumericAddress: host blank -> null")
            return null
        }
        val candidate = host.removeSurrounding("[", "]")
        if (!NUMERIC_HOST_REGEX.matches(candidate)) {
            AppLogger.d(TAG, "parseNumericAddress: $host doesn't match numeric regex -> null")
            return null
        }
        if (!candidate.contains('.') && !candidate.contains(':')) {
            AppLogger.d(TAG, "parseNumericAddress: $host lacks dot or colon -> null")
            return null
        }
        val address = runCatching { InetAddress.getByName(candidate) }.getOrNull()
        AppLogger.d(TAG, "parseNumericAddress($host) -> ${address?.hostAddress}")
        return address
    }

    fun isPrivateAddressLiteral(host: String?): Boolean {
        val address = parseNumericAddress(host) ?: return false
        if (address.isLoopbackAddress || address.isLinkLocalAddress ||
            address.isAnyLocalAddress || address.isMulticastAddress
        ) {
            AppLogger.d(TAG, "isPrivateAddressLiteral($host): special address -> false")
            return false
        }

        val result = when (address) {
            is Inet4Address -> {
                val b = address.address
                val b0 = b[0].toInt() and 0xFF
                val b1 = b[1].toInt() and 0xFF
                (b0 == 10) || (b0 == 172 && b1 in 16..31) ||
                    (b0 == 192 && b1 == 168) || (b0 == 100 && b1 in 64..127)
            }
            is Inet6Address -> {
                (address.address[0].toInt() and 0xFE) == 0xFC
            }
            else -> false
        }
        AppLogger.d(TAG, "isPrivateAddressLiteral($host) -> $result")
        return result
    }

    fun resolveStreamContext(rawUrl: String?): NetworkSecurityContext {
        if (rawUrl == null) {
            AppLogger.d(TAG, "resolveStreamContext(null) -> IPTV_STREAM")
            return NetworkSecurityContext.IPTV_STREAM
        }
        val trimmed = rawUrl.trim()

        if (trimmed.startsWith("/") || LOCAL_MEDIA_SCHEMES.any { trimmed.startsWith("$it:", ignoreCase = true) }) {
            AppLogger.d(TAG, "resolveStreamContext(${SensitiveDataSanitizer.sanitizeUrl(rawUrl)}) -> LOCAL_FILE_SYSTEM (local path or media scheme)")
            return NetworkSecurityContext.LOCAL_FILE_SYSTEM
        }

        val host = extractHost(trimmed)
        if (isLoopbackHostLiteral(host)) {
            val port = extractPort(trimmed)
            val context = if (port != -1 && isTrustedLoopbackPort(port)) {
                NetworkSecurityContext.LOCAL_TIMESHIFT_LOOPBACK
            } else {
                NetworkSecurityContext.IPTV_STREAM
            }
            AppLogger.d(TAG, "resolveStreamContext(${SensitiveDataSanitizer.sanitizeUrl(rawUrl)}) -> $context (loopback host, port=$port)")
            return context
        }

        if (isPrivateAddressLiteral(host)) {
            AppLogger.d(TAG, "resolveStreamContext(${SensitiveDataSanitizer.sanitizeUrl(rawUrl)}) -> LOCAL_IPTV_GATEWAY (private address)")
            return NetworkSecurityContext.LOCAL_IPTV_GATEWAY
        }

        AppLogger.d(TAG, "resolveStreamContext(${SensitiveDataSanitizer.sanitizeUrl(rawUrl)}) -> IPTV_STREAM (default)")
        return NetworkSecurityContext.IPTV_STREAM
    }

    sealed interface UrlValidationResult {
        data class Valid(val sanitizedUrl: String, val uri: Uri? = null) : UrlValidationResult
        data class Invalid(val reason: String) : UrlValidationResult
    }

    fun validateUri(
        rawUrl: String?,
        context: NetworkSecurityContext,
        timeshiftServer: TimeshiftLocalServer? = null
    ): UrlValidationResult {
        AppLogger.d(TAG, "validateUri: url='${SensitiveDataSanitizer.sanitizeUrl(rawUrl)}', context=$context, timeshiftServer=${timeshiftServer != null}")

        if (rawUrl.isNullOrBlank()) {
            AppLogger.w(TAG, "validateUri: URL is blank")
            return UrlValidationResult.Invalid("URL is blank")
        }

        val trimmed = rawUrl.trim()
        if (trimmed.length > MAX_URL_LENGTH) {
            AppLogger.w(TAG, "validateUri: URL length ${trimmed.length} exceeds $MAX_URL_LENGTH")
            return UrlValidationResult.Invalid("URL length exceeds $MAX_URL_LENGTH chars")
        }

        val isLocalInput = context == NetworkSecurityContext.LOCAL_FILE_SYSTEM ||
            trimmed.startsWith("/") ||
            LOCAL_MEDIA_SCHEMES.any { trimmed.startsWith("$it:", ignoreCase = true) }

        val lower = trimmed.lowercase()
        for (pattern in INJECTION_PATTERNS) {
            if (pattern == " " && isLocalInput) continue
            if (lower.contains(pattern)) {
                AppLogger.w(TAG, "validateUri: injection pattern '$pattern' found in URL")
                return UrlValidationResult.Invalid("Forbidden injection sequence: $pattern")
            }
        }

        if (trimmed.startsWith("/")) {
            val fileUri = runCatching { Uri.fromFile(File(trimmed)) }.getOrNull()
            val result = UrlValidationResult.Valid(trimmed, fileUri ?: createSafeUri(trimmed))
            AppLogger.d(TAG, "validateUri: local file path -> valid")
            return result
        }

        val scheme = extractScheme(trimmed)
        if (scheme == null) {
            AppLogger.w(TAG, "validateUri: URI scheme missing")
            return UrlValidationResult.Invalid("URI scheme missing")
        }

        val allowedSchemes = when (context) {
            NetworkSecurityContext.REMOTE_PLAYLIST,
            NetworkSecurityContext.REMOTE_EPG,
            NetworkSecurityContext.XTREAM_API,
            NetworkSecurityContext.PUBLIC_WEB -> HTTP_SCHEMES
            NetworkSecurityContext.LOCAL_TIMESHIFT_LOOPBACK -> setOf("http")
            NetworkSecurityContext.IPTV_STREAM,
            NetworkSecurityContext.LOCAL_IPTV_GATEWAY -> HTTP_SCHEMES + STREAMING_SCHEMES
            NetworkSecurityContext.LOCAL_FILE_SYSTEM -> LOCAL_MEDIA_SCHEMES + HTTP_SCHEMES
        }

        if (scheme !in allowedSchemes) {
            AppLogger.w(TAG, "validateUri: scheme '$scheme' disallowed for context $context")
            return UrlValidationResult.Invalid("Scheme '$scheme' disallowed for context $context")
        }

        if (context == NetworkSecurityContext.LOCAL_FILE_SYSTEM || scheme in LOCAL_MEDIA_SCHEMES) {
            AppLogger.d(TAG, "validateUri: local file system or local scheme -> valid")
            return UrlValidationResult.Valid(trimmed, createSafeUri(trimmed))
        }

        val host = extractHost(trimmed)
        if (host.isNullOrBlank()) {
            AppLogger.w(TAG, "validateUri: URI host missing")
            return UrlValidationResult.Invalid("URI host missing")
        }
        val lowerHost = host.lowercase()

        if (CLOUD_METADATA_HOSTS.any { lowerHost == it || lowerHost.endsWith(".$it") } || CLOUD_METADATA_IPS.contains(lowerHost)) {
            AppLogger.w(TAG, "validateUri: cloud metadata access blocked for host '$host'")
            return UrlValidationResult.Invalid("Cloud metadata access is strictly blocked")
        }

        val port = extractPort(trimmed)
        if (port != -1 && (port < 1 || port > 65_535)) {
            AppLogger.w(TAG, "validateUri: port $port out of range")
            return UrlValidationResult.Invalid("Port $port out of valid range (1..65535)")
        }

        val isLoopbackHost = isLoopbackHostLiteral(lowerHost)
        if (isLoopbackHost) {
            if (context != NetworkSecurityContext.LOCAL_TIMESHIFT_LOOPBACK && !isTrustedLoopbackPort(port)) {
                AppLogger.w(TAG, "validateUri: loopback restricted (context=$context, port=$port)")
                return UrlValidationResult.Invalid("Loopback restricted to internal Timeshift only")
            }

            if (timeshiftServer != null && port != timeshiftServer.serverPort) {
                AppLogger.w(TAG, "validateUri: loopback port $port does not match timeshift server port ${timeshiftServer.serverPort}")
                return UrlValidationResult.Invalid("Loopback port mismatch")
            } else if (!isTrustedLoopbackPort(port)) {
                AppLogger.w(TAG, "validateUri: loopback port $port not registered as trusted")
                return UrlValidationResult.Invalid("Loopback port $port not registered as trusted")
            }
        }

        val sanitized = trimmed.substringBefore('#')
        AppLogger.d(TAG, "validateUri: valid -> sanitized='${SensitiveDataSanitizer.sanitizeUrl(sanitized)}'")
        return UrlValidationResult.Valid(sanitized, createSafeUri(sanitized))
    }

    private fun extractScheme(url: String): String? {
        val colonIdx = url.indexOf(':')
        if (colonIdx <= 0) return null
        val schemePart = url.substring(0, colonIdx).lowercase()
        val isValid = schemePart.all { it.isLetterOrDigit() || it == '+' || it == '-' || it == '.' }
        val result = if (isValid) schemePart else null
        AppLogger.d(TAG, "extractScheme(${SensitiveDataSanitizer.sanitizeUrl(url)}) -> $result")
        return result
    }

    private fun extractHost(url: String?): String? {
        if (url == null) return null
        val standardHost = runCatching { URI(url).host }.getOrNull()
        if (standardHost != null) {
            AppLogger.d(TAG, "extractHost(${SensitiveDataSanitizer.sanitizeUrl(url)}) -> $standardHost (URI parser)")
            return standardHost
        }

        val result = runCatching {
            val noScheme = if (url.contains("://")) url.substringAfter("://") else url
            val authority = noScheme.substringBefore('/').substringBefore('?').substringBefore('#')
            val hostPort = if (authority.contains('@')) authority.substringAfter('@') else authority
            if (hostPort.startsWith('[')) {
                hostPort.substringBefore(']') + "]"
            } else {
                hostPort.substringBefore(':')
            }
        }.getOrNull()?.takeIf { it.isNotBlank() }
        AppLogger.d(TAG, "extractHost(${SensitiveDataSanitizer.sanitizeUrl(url)}) -> $result (manual)")
        return result
    }

    private fun extractPort(url: String?): Int {
        if (url == null) return -1
        val javaUriPort = runCatching { URI(url).port }.getOrDefault(-1)
        if (javaUriPort != -1) {
            AppLogger.d(TAG, "extractPort(${SensitiveDataSanitizer.sanitizeUrl(url)}) -> $javaUriPort (URI parser)")
            return javaUriPort
        }

        val result = runCatching {
            val noScheme = if (url.contains("://")) url.substringAfter("://") else url
            val authority = noScheme.substringBefore('/').substringBefore('?').substringBefore('#')
            val hostPort = if (authority.contains('@')) authority.substringAfter('@') else authority
            val portStr = if (hostPort.startsWith('[')) {
                hostPort.substringAfter(']', "").substringAfter(':', "")
            } else {
                hostPort.substringAfter(':', "")
            }
            portStr.toIntOrNull() ?: -1
        }.getOrDefault(-1)
        AppLogger.d(TAG, "extractPort(${SensitiveDataSanitizer.sanitizeUrl(url)}) -> $result (manual)")
        return result
    }

    private fun createSafeUri(url: String): Uri? {
        return try {
            Uri.parse(url)
        } catch (_: Throwable) {
            null
        }
    }

    fun isAddressForbidden(address: InetAddress, context: NetworkSecurityContext): Boolean {
        AppLogger.d(TAG, "isAddressForbidden: address=${address.hostAddress}, context=$context")

        if (context == NetworkSecurityContext.LOCAL_FILE_SYSTEM) {
            AppLogger.d(TAG, "isAddressForbidden: LOCAL_FILE_SYSTEM -> false")
            return false
        }

        if (address.isLoopbackAddress) {
            val forbidden = context != NetworkSecurityContext.LOCAL_TIMESHIFT_LOOPBACK
            AppLogger.d(TAG, "isAddressForbidden: loopback -> $forbidden (context=$context)")
            return forbidden
        }

        if (address.isMulticastAddress || address.isAnyLocalAddress ||
            address.isLinkLocalAddress
        ) {
            AppLogger.d(TAG, "isAddressForbidden: multicast/any/link-local -> true")
            return true
        }

        if (address.isSiteLocalAddress &&
            context != NetworkSecurityContext.IPTV_STREAM &&
            context != NetworkSecurityContext.LOCAL_IPTV_GATEWAY
        ) {
            AppLogger.d(TAG, "isAddressForbidden: site-local address in non-streaming context -> true")
            return true
        }

        val effectiveIpv4: Inet4Address? = when {
            address is Inet4Address -> address
            address is Inet6Address && isIpv4MappedAddress(address) -> extractIpv4FromMapped(address)
            address is Inet6Address && is6to4Address(address) -> extractIpv4From6to4(address)
            else -> null
        }

        if (effectiveIpv4 != null) {
            val ipStr = effectiveIpv4.hostAddress ?: ""
            if (CLOUD_METADATA_IPS.contains(ipStr)) {
                AppLogger.w(TAG, "isAddressForbidden: cloud metadata IP $ipStr -> true")
                return true
            }

            val bytes = effectiveIpv4.address
            val b0 = bytes[0].toInt() and 0xFF
            val b1 = bytes[1].toInt() and 0xFF
            val b2 = bytes[2].toInt() and 0xFF

            if (b0 == 0) return true
            if (b0 == 127) return context != NetworkSecurityContext.LOCAL_TIMESHIFT_LOOPBACK
            if (b0 == 169 && b1 == 254) return true
            if (b0 == 192 && b1 == 0 && b2 == 2) return true
            if (b0 == 198 && (b1 in 18..19)) return true
            if (b0 == 198 && b1 == 51 && b2 == 100) return true
            if (b0 == 203 && b1 == 0 && b2 == 113) return true
            if (b0 >= 224) return true

            val isPrivateSubnet = (b0 == 10) || (b0 == 172 && b1 in 16..31) ||
                (b0 == 192 && b1 == 168) || (b0 == 100 && b1 in 64..127)

            if (isPrivateSubnet) {
                val isStreamingContext = context == NetworkSecurityContext.IPTV_STREAM ||
                    context == NetworkSecurityContext.LOCAL_IPTV_GATEWAY
                return !isStreamingContext
            }
            return false
        }

        if (address is Inet6Address) {
            val rawBytes = address.address

            if ((rawBytes[0].toInt() and 0xFE) == 0xFC) {
                val isStreamingContext = context == NetworkSecurityContext.IPTV_STREAM ||
                    context == NetworkSecurityContext.LOCAL_IPTV_GATEWAY
                return !isStreamingContext
            }

            if (address.isLoopbackAddress) {
                return context != NetworkSecurityContext.LOCAL_TIMESHIFT_LOOPBACK
            }

            return false
        }

        return false
    }

    private fun isIpv4MappedAddress(ipv6: Inet6Address): Boolean {
        val bytes = ipv6.address
        for (i in 0..9) if (bytes[i] != 0.toByte()) return false
        return bytes[10] == 0xFF.toByte() && bytes[11] == 0xFF.toByte()
    }

    private fun extractIpv4FromMapped(ipv6: Inet6Address): Inet4Address? {
        val bytes = ipv6.address
        val ipv4Bytes = byteArrayOf(bytes[12], bytes[13], bytes[14], bytes[15])
        return runCatching { InetAddress.getByAddress(ipv4Bytes) as? Inet4Address }.getOrNull()
    }

    private fun is6to4Address(ipv6: Inet6Address): Boolean {
        val bytes = ipv6.address
        return bytes[0] == 0x20.toByte() && bytes[1] == 0x02.toByte()
    }

    private fun extractIpv4From6to4(ipv6: Inet6Address): Inet4Address? {
        val bytes = ipv6.address
        val ipv4Bytes = byteArrayOf(bytes[2], bytes[3], bytes[4], bytes[5])
        return runCatching { InetAddress.getByAddress(ipv4Bytes) as? Inet4Address }.getOrNull()
    }

    object ResilientDns {
        private val dohClient by lazy {
            OkHttpClient.Builder()
                .connectTimeout(4, TimeUnit.SECONDS)
                .readTimeout(4, TimeUnit.SECONDS)
                .callTimeout(5, TimeUnit.SECONDS)
                .build()
        }

        fun lookup(hostname: String): List<InetAddress> {
            AppLogger.d(TAG, "ResilientDns.lookup: resolving host '$hostname'")
            try {
                val results = Dns.SYSTEM.lookup(hostname)
                if (results.isNotEmpty()) {
                    return results
                }
            } catch (_: UnknownHostException) {
            }

            val encodedHost = runCatching {
                URLEncoder.encode(hostname, StandardCharsets.UTF_8.name())
            }.getOrElse { hostname }

            resolveDoH("https://1.1.1.1/dns-query?name=$encodedHost&type=A")?.let { aRecords ->
                if (aRecords.isNotEmpty()) {
                    val aaaa = resolveDoH("https://1.1.1.1/dns-query?name=$encodedHost&type=AAAA").orEmpty()
                    return (aRecords + aaaa).distinctBy { address -> address.hostAddress }
                }
            }

            resolveDoH("https://dns.google/resolve?name=$encodedHost&type=A")?.let { aRecords ->
                if (aRecords.isNotEmpty()) {
                    val aaaa = resolveDoH("https://dns.google/resolve?name=$encodedHost&type=AAAA").orEmpty()
                    return (aRecords + aaaa).distinctBy { address -> address.hostAddress }
                }
            }

            resolveDoH("https://1.1.1.1/dns-query?name=$encodedHost&type=AAAA")?.let {
                if (it.isNotEmpty()) return it.distinctBy { address -> address.hostAddress }
            }
            resolveDoH("https://dns.google/resolve?name=$encodedHost&type=AAAA")?.let {
                if (it.isNotEmpty()) return it.distinctBy { address -> address.hostAddress }
            }

            throw UnknownHostException("Unable to resolve host \"$hostname\": No address associated with hostname")
        }

        private fun resolveDoH(url: String): List<InetAddress>? {
            return try {
                val request = Request.Builder()
                    .url(url)
                    .header("Accept", "application/dns-json")
                    .build()

                dohClient.newCall(request).execute().use { response ->
                    if (!response.isSuccessful) return null
                    val jsonStr = response.body?.let { BoundedResponseReader.readUtf8(it, 1L * 1024L * 1024L) } ?: return null
                    val json = JSONObject(jsonStr)
                    val answers = json.optJSONArray("Answer") ?: return null

                    val ips = mutableListOf<InetAddress>()
                    for (i in 0 until answers.length()) {
                        val obj = answers.optJSONObject(i) ?: continue
                        when (obj.optInt("type", -1)) {
                            1, 28 -> {
                                val ip = obj.optString("data").trim()
                                if (ip.isNotBlank()) {
                                    runCatching { InetAddress.getByName(ip) }.getOrNull()?.let(ips::add)
                                }
                            }
                        }
                    }
                    ips.distinctBy { it.hostAddress }.takeIf { it.isNotEmpty() }
                }
            } catch (_: Exception) {
                null
            }
        }
    }

    class SafeDns(private val context: NetworkSecurityContext) : Dns {
        @Throws(IOException::class)
        override fun lookup(hostname: String): List<InetAddress> {
            val addresses = ResilientDns.lookup(hostname)
            val forbiddenAddresses = addresses.filter { isAddressForbidden(it, context) }
            if (forbiddenAddresses.isNotEmpty()) {
                AppLogger.w(TAG, "SafeDns: SSRF blocked for '$hostname' -> ${forbiddenAddresses.joinToString { it.hostAddress }}")
                throw IOException("SSRF Blocked: Host '$hostname' resolves to restricted destination")
            }
            return addresses
        }
    }

    class ContextDispatchingDns(
        private val fallbackContext: NetworkSecurityContext = NetworkSecurityContext.IPTV_STREAM
    ) : Dns {
        @Throws(IOException::class)
        override fun lookup(hostname: String): List<InetAddress> {
            val effectiveContext = when {
                isLoopbackHostLiteral(hostname) -> {
                    if (hasActiveTrustedLoopbackOrigin()) NetworkSecurityContext.LOCAL_TIMESHIFT_LOOPBACK
                    else fallbackContext
                }
                isPrivateAddressLiteral(hostname) -> NetworkSecurityContext.LOCAL_IPTV_GATEWAY
                else -> fallbackContext
            }

            val addresses = ResilientDns.lookup(hostname)
            val forbiddenAddresses = addresses.filter { isAddressForbidden(it, effectiveContext) }
            if (forbiddenAddresses.isNotEmpty()) {
                AppLogger.w(TAG, "ContextDispatchingDns: SSRF blocked for '$hostname' -> ${forbiddenAddresses.joinToString { it.hostAddress }}")
                throw IOException("SSRF Blocked: Host '$hostname' resolves to restricted destination")
            }
            return addresses
        }
    }

    class RedirectGuardInterceptor(
        private val contextResolver: (HttpUrl) -> NetworkSecurityContext
    ) : Interceptor {
        @Throws(IOException::class)
        override fun intercept(chain: Interceptor.Chain): Response {
            var currentRequest = chain.request()
            var redirectCount = 0

            while (true) {
                val currentContext = contextResolver(currentRequest.url)
                val validation = validateUri(currentRequest.url.toString(), currentContext)
                if (validation !is UrlValidationResult.Valid) {
                    val reason = (validation as? UrlValidationResult.Invalid)?.reason ?: "policy violation"
                    throw IOException("Network policy blocked '${currentRequest.url.host}': $reason")
                }

                val response = chain.proceed(currentRequest)
                if (!response.isRedirect) {
                    return response
                }

                val location = response.header("Location")
                response.close()
                if (location.isNullOrBlank()) {
                    throw IOException("Redirect missing Location header")
                }

                val nextUrl = currentRequest.url.resolve(location)
                    ?: throw IOException("Malformed redirect location: $location")

                redirectCount++
                if (redirectCount > MAX_REDIRECT_DEPTH) {
                    throw IOException("SSRF Violation: Max redirect depth exceeded ($MAX_REDIRECT_DEPTH)")
                }

                val nextContext = contextResolver(nextUrl)
                val nextValidation = validateUri(nextUrl.toString(), nextContext)
                if (nextValidation !is UrlValidationResult.Valid) {
                    throw IOException("SSRF Redirect Blocked: '$nextUrl' rejected by policy")
                }

                val isCrossOrigin = currentRequest.url.host != nextUrl.host ||
                    currentRequest.url.port != nextUrl.port ||
                    currentRequest.url.scheme != nextUrl.scheme

                val requestBuilder = currentRequest.newBuilder().url(nextValidation.sanitizedUrl)
                if (isCrossOrigin) {
                    requestBuilder.removeHeader("Authorization")
                    requestBuilder.removeHeader("Cookie")
                    requestBuilder.removeHeader("Proxy-Authorization")
                }
                currentRequest = requestBuilder.build()
            }
        }
    }

    @Throws(IOException::class)
    fun executeSafeHttpCall(
        client: OkHttpClient,
        initialRequest: Request,
        context: NetworkSecurityContext
    ): Response {
        val dns: Dns = if (context in STREAM_CONTEXTS) {
            ContextDispatchingDns(NetworkSecurityContext.IPTV_STREAM)
        } else {
            SafeDns(context)
        }

        val safeClient = client.newBuilder()
            .dns(dns)
            .followRedirects(false)
            .followSslRedirects(false)
            .build()

        var currentRequest = initialRequest
        var redirectCount = 0

        while (true) {
            val response = safeClient.newCall(currentRequest).execute()
            if (!response.isRedirect) {
                return response
            }

            redirectCount++
            if (redirectCount > MAX_REDIRECT_DEPTH) {
                response.close()
                throw IOException("SSRF Violation: Max redirect depth exceeded ($MAX_REDIRECT_DEPTH)")
            }

            val location = response.header("Location")
            response.close()
            if (location.isNullOrBlank()) {
                throw IOException("Redirect missing Location header")
            }

            val nextUrl = currentRequest.url.resolve(location)
                ?: throw IOException("Malformed redirect location: $location")

            val effectiveRedirectContext = if (context in STREAM_CONTEXTS) {
                resolveStreamContext(nextUrl.toString())
            } else {
                context
            }
            val validation = validateUri(nextUrl.toString(), effectiveRedirectContext)
            if (validation !is UrlValidationResult.Valid) {
                AppLogger.w(TAG, "executeSafeHttpCall: redirect to ${SensitiveDataSanitizer.sanitizeUrl(nextUrl.toString())} blocked: ${(validation as? UrlValidationResult.Invalid)?.reason}")
                throw IOException("SSRF Redirect Blocked: target rejected by policy")
            }

            val isCrossOrigin = currentRequest.url.host != nextUrl.host ||
                currentRequest.url.port != nextUrl.port ||
                currentRequest.url.scheme != nextUrl.scheme
            val requestBuilder = currentRequest.newBuilder().url(nextUrl)

            if (isCrossOrigin) {
                requestBuilder.removeHeader("Authorization")
                requestBuilder.removeHeader("Cookie")
                requestBuilder.removeHeader("Proxy-Authorization")
            }

            currentRequest = requestBuilder.build()
        }
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

May I proceed with **Batch 10 (5 classes — Parental PIN, Casting & Boot/Lifecycle)** (`ui/security/ParentalControlSettingsActivity.kt`, `cast/CastManager.kt`, `ui/player/PlayerCastController.kt`, `notification/BootCompletedReceiver.kt`, `lifecycle/AppLifecycleSupervisor.kt`)?
thought
Let's carefully verify the 5 classes for **Batch 10**:
1. `ui/security/ParentalControlSettingsActivity.kt`
   - Phase 14 change:
     - `showSetPinDialog`: enforces 4–12 numeric digits (`pin.length in 4..12 && pin.all(Char::isDigit)`).
     - Notice in `ParentalControlSettingsActivity.kt` line 41: if user disables or changes PIN when `parentalControlManager.isParentalControlActive` is true, wait—did Phase 14 only show `showSetPinDialog`? Let's check the Phase 14 paste: yes, `showSetPinDialog` checks `pin.length in 4..12 && pin.all(Char::isDigit)`. Let's keep the rest intact and clean.
2. `cast/CastManager.kt`
   - Phase 15 change:
     - Add `val isLive: Boolean = false` to `StreamInfo`.
     - In `loadOnSession`: set `.setStreamType(if (stream.isLive) MediaInfo.STREAM_TYPE_LIVE else MediaInfo.STREAM_TYPE_BUFFERED)` and `.setCurrentTime(if (stream.isLive) 0L else stream.positionMs)`.
3. `ui/player/PlayerCastController.kt`
   - Phase 15 change:
     - Reject local/loopback URIs (`content://`, `file://`, `/`, `http://127.0.0.1`, `http://localhost`).
     - Expand `/X-AUTH-INJECT/` credentials via `expandCastUrlIfNeeded` using `(activity as? MediaPlayerActivity)?.failoverEngine?.getActiveCandidate(session.mediaId)`.
     - Pass `isLive = session?.playbackType == PlaybackType.LIVE` and `positionMs = if (isLive) 0L else viewModel.newPlayer.currentPosition.coerceAtLeast(0L)`.
4. `notification/BootCompletedReceiver.kt`
   - Phase 15 change:
     - Inject `database: VideoUrlDatabase`.
     - Use `goAsync()` + `CoroutineScope(Dispatchers.IO).launch` to reschedule `EpgRefreshWorker.schedule(context, ...)` for manual EPG URL and all distinct playlist source EPG URLs on boot.
5. `lifecycle/AppLifecycleSupervisor.kt`
   - Phase 15 change:
     - Wrap `workManager.getWorkInfosForUniqueWork("dvr_work_${recording.id}").get()` in `runCatching { ... }.getOrDefault(emptyList())`.
     - Filter `allValidRecordings` to only `DvrStatus.COMPLETED` or `DvrStatus.PARTIAL` when building `validPaths` for `storageManager.cleanupOrphanedFiles(validPaths)`.

Let's double-check all imports and generate the Python script for Batch 10.Here is the Python script for **Batch 10 (5 classes — Parental PIN, Google Cast & Background Lifecycle)**:

1. `ui/security/ParentalControlSettingsActivity.kt` — Enforces 4–12 numeric digits in `showSetPinDialog` matching `ParentalControlManager.setPin`.
2. `cast/CastManager.kt` — Adds `isLive: Boolean = false` on `StreamInfo` and sets `MediaInfo.STREAM_TYPE_LIVE` for Live TV casting.
3. `ui/player/PlayerCastController.kt` — Rejects uncastable local/loopback URIs, expands `/X-AUTH-INJECT/` credentials before casting, and sets `isLive`.
4. `notification/BootCompletedReceiver.kt` — Uses `goAsync()` to reschedule `EpgRefreshWorker` for manual and playlist source EPG URLs after reboot.
5. `lifecycle/AppLifecycleSupervisor.kt` — Safely wraps `WorkManager.getWorkInfosForUniqueWork().get()` and retains only `COMPLETED`/`PARTIAL` recording paths during orphan file cleanup.

```python
#!/usr/bin/env python3
from pathlib import Path

BASE_DIR = Path("/home/username/AndroidStudioProjects/Ytvmate/app/src/main/java/vid/mat/app")

FILES = {
    "ui/security/ParentalControlSettingsActivity.kt": """package vid.mat.app.ui.security

import android.os.Bundle
import android.text.InputType
import android.view.LayoutInflater
import android.view.View
import android.view.ViewGroup
import android.widget.EditText
import android.widget.TextView
import android.widget.Toast
import androidx.activity.enableEdgeToEdge
import androidx.appcompat.app.AppCompatActivity
import androidx.lifecycle.lifecycleScope
import androidx.recyclerview.widget.LinearLayoutManager
import androidx.recyclerview.widget.RecyclerView
import com.google.android.material.dialog.MaterialAlertDialogBuilder
import dagger.hilt.android.AndroidEntryPoint
import kotlinx.coroutines.launch
import vid.mat.app.R
import vid.mat.app.data.repository.ChannelRepository
import vid.mat.app.databinding.ActivityParentalSettingsBinding
import vid.mat.app.security.ParentalControlManager
import javax.inject.Inject

@AndroidEntryPoint
class ParentalControlSettingsActivity : AppCompatActivity() {

    private lateinit var binding: ActivityParentalSettingsBinding

    @Inject
    lateinit var parentalControlManager: ParentalControlManager

    @Inject
    lateinit var channelRepository: ChannelRepository

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        enableEdgeToEdge()
        binding = ActivityParentalSettingsBinding.inflate(layoutInflater)
        setContentView(binding.root)

        binding.btnBack.setOnClickListener { finish() }

        binding.switchParentalEnabled.isChecked = parentalControlManager.isParentalControlActive
        binding.switchParentalEnabled.setOnCheckedChangeListener { _, isChecked ->
            if (isChecked && !parentalControlManager.isParentalControlActive) {
                showSetPinDialog {
                    parentalControlManager.setParentalControlEnabled(true)
                }
            } else {
                parentalControlManager.setParentalControlEnabled(isChecked)
            }
        }

        binding.btnChangePin.setOnClickListener {
            showSetPinDialog {
                Toast.makeText(this, "PIN updated successfully", Toast.LENGTH_SHORT).show()
            }
        }

        setupCategoriesList()
    }

    private fun setupCategoriesList() {
        binding.rvLockedCategories.layoutManager = LinearLayoutManager(this)
        lifecycleScope.launch {
            channelRepository.distinctGroups.collect { groups ->
                val lockedSet = parentalControlManager.getLockedCategories()
                binding.rvLockedCategories.adapter = CategoryLockAdapter(groups, lockedSet) { category, isLocked ->
                    if (isLocked) parentalControlManager.lockCategory(category)
                    else parentalControlManager.unlockCategory(category)
                }
            }
        }
    }

    private fun showSetPinDialog(onSuccess: () -> Unit) {
        val input = EditText(this).apply {
            inputType = InputType.TYPE_CLASS_NUMBER or InputType.TYPE_NUMBER_VARIATION_PASSWORD
            setPadding(48, 32, 48, 16)
        }

        MaterialAlertDialogBuilder(this)
            .setTitle("Set Security PIN (4–12 Digits)")
            .setMessage("Enter a numeric PIN to restrict access to locked content.")
            .setView(input)
            .setPositiveButton("Save PIN") { _, _ ->
                val pin = input.text.toString().trim()
                if (pin.length in 4..12 && pin.all(Char::isDigit)) {
                    parentalControlManager.setPin(pin)
                    onSuccess()
                } else {
                    Toast.makeText(this, "PIN must be 4 to 12 digits", Toast.LENGTH_SHORT).show()
                }
            }
            .setNegativeButton("Cancel", null)
            .show()
    }

    private class CategoryLockAdapter(
        private val categories: List<String>,
        private val lockedSet: Set<String>,
        private val onToggle: (String, Boolean) -> Unit
    ) : RecyclerView.Adapter<CategoryLockAdapter.VH>() {

        override fun getItemCount(): Int = categories.size

        override fun onCreateViewHolder(parent: ViewGroup, viewType: Int): VH {
            val view = LayoutInflater.from(parent.context).inflate(R.layout.item_category_drawer, parent, false)
            return VH(view)
        }

        override fun onBindViewHolder(holder: VH, position: Int) {
            val cat = categories[position]
            holder.tvName.text = cat
            val isLocked = lockedSet.contains(cat.lowercase().trim())
            holder.tvCount.text = if (isLocked) "LOCKED" else "UNLOCKED"
            holder.tvCount.setTextColor(if (isLocked) 0xFFEF4444.toInt() else 0xFF10B981.toInt())

            holder.itemView.setOnClickListener {
                onToggle(cat, !isLocked)
                notifyItemChanged(position)
            }
        }

        class VH(v: View) : RecyclerView.ViewHolder(v) {
            val tvName: TextView = v.findViewById(R.id.tvCategoryName)
            val tvCount: TextView = v.findViewById(R.id.tvCategoryCount)
        }
    }
}
""",

    "cast/CastManager.kt": """package vid.mat.app.cast

import android.content.Context
import androidx.core.net.toUri
import com.google.android.gms.cast.MediaInfo
import com.google.android.gms.cast.MediaLoadRequestData
import com.google.android.gms.cast.MediaMetadata
import com.google.android.gms.cast.framework.CastContext
import com.google.android.gms.cast.framework.CastSession
import com.google.android.gms.cast.framework.SessionManagerListener
import com.google.android.gms.common.images.WebImage
import kotlinx.coroutines.flow.MutableStateFlow
import kotlinx.coroutines.flow.StateFlow
import kotlinx.coroutines.flow.asStateFlow
import vid.mat.app.util.AppLogger

class CastManager(private val context: Context) {

    enum class CastState {
        DISCONNECTED,
        AVAILABLE,
        CONNECTED,
        LOADING,
        ERROR
    }

    private val _castState = MutableStateFlow(CastState.DISCONNECTED)
    val castState: StateFlow<CastState> = _castState.asStateFlow()

    val isCasting: Boolean
        get() = _castState.value == CastState.CONNECTED || _castState.value == CastState.LOADING

    private var castContext: CastContext? = null
    private var currentSession: CastSession? = null
    private var pendingStream: StreamInfo? = null
    private var lastKnownPositionMs: Long = 0L
    private var _lastErrorMessage: String? = null

    val lastErrorMessage: String?
        get() = _lastErrorMessage

    data class StreamInfo(
        val url: String,
        val title: String,
        val thumbnailUrl: String? = null,
        val positionMs: Long = 0L,
        val mimeType: String = inferMimeType(url),
        val userAgent: String? = null,
        val isLive: Boolean = false
    )

    private val sessionListener = object : SessionManagerListener<CastSession> {
        override fun onSessionStarted(session: CastSession, sessionId: String) {
            currentSession = session
            val pending = pendingStream
            if (pending != null) {
                _castState.value = CastState.LOADING
                loadOnSession(session, pending)
            } else {
                _castState.value = CastState.CONNECTED
            }
        }

        override fun onSessionResumed(session: CastSession, wasSuspended: Boolean) {
            currentSession = session
            val pending = pendingStream
            if (pending != null) {
                _castState.value = CastState.LOADING
                if (!remoteMediaMatches(pending)) loadOnSession(session, pending)
                else _castState.value = CastState.CONNECTED
            } else {
                _castState.value = CastState.CONNECTED
            }
        }

        override fun onSessionEnded(session: CastSession, error: Int) {
            currentSession = null
            _castState.value = CastState.DISCONNECTED
        }

        override fun onSessionSuspended(session: CastSession, reason: Int) {
            _castState.value = CastState.AVAILABLE
        }

        override fun onSessionStartFailed(session: CastSession, error: Int) {
            currentSession = null
            _castState.value = CastState.AVAILABLE
        }

        override fun onSessionStarting(session: CastSession) {
            _castState.value = CastState.LOADING
        }

        override fun onSessionEnding(session: CastSession) {
            _castState.value = CastState.LOADING
        }

        override fun onSessionResumeFailed(session: CastSession, error: Int) {
            AppLogger.w("CastManager", "Cast session resume failed with error code: $error")
            currentSession = null
            _castState.value = CastState.AVAILABLE
        }

        override fun onSessionResuming(session: CastSession, sessionId: String) {
            _castState.value = CastState.LOADING
        }
    }

    fun register() {
        try {
            castContext = CastContext.getSharedInstance(context)
            castContext?.sessionManager?.addSessionManagerListener(
                sessionListener, CastSession::class.java
            )
            val existingSession = castContext?.sessionManager?.currentCastSession
            if (existingSession != null && existingSession.isConnected) {
                currentSession = existingSession
                _castState.value = CastState.CONNECTED
            } else {
                _castState.value = CastState.DISCONNECTED
            }
        } catch (e: Exception) {
            AppLogger.e("CastManager", "Google Play Services Cast framework unavailable", e)
            _castState.value = CastState.DISCONNECTED
        }
    }

    fun unregister() {
        try {
            castContext?.sessionManager?.removeSessionManagerListener(
                sessionListener, CastSession::class.java
            )
        } catch (e: Exception) {
            AppLogger.w("CastManager", "Failed to unregister session listener: ${e.message}")
        }
    }

    fun prepareStream(stream: StreamInfo) {
        pendingStream = stream
        val session = currentSession
        if (session != null && session.isConnected) {
            _castState.value = CastState.LOADING
            loadOnSession(session, stream)
        }
    }

    fun castStream(stream: StreamInfo) {
        prepareStream(stream)
    }

    val hasActiveMedia: Boolean
        get() = pendingStream != null && (currentSession?.isConnected == true)

    val currentStream: StreamInfo?
        get() = pendingStream

    val lastKnownRemotePositionMs: Long
        get() = try {
            val position = currentSession?.remoteMediaClient?.approximateStreamPosition ?: lastKnownPositionMs
            position.coerceAtLeast(0L).also { lastKnownPositionMs = it }
        } catch (_: Exception) {
            lastKnownPositionMs
        }

    fun stopCasting(): Long {
        val resumePos = lastKnownRemotePositionMs

        try {
            castContext?.sessionManager?.endCurrentSession(true)
        } catch (e: Exception) {
            AppLogger.w("CastManager", "Error ending cast session: ${e.message}")
        }

        currentSession = null
        pendingStream = null
        lastKnownPositionMs = resumePos
        _lastErrorMessage = null
        _castState.value = CastState.DISCONNECTED
        return resumePos
    }

    fun pause() { currentSession?.remoteMediaClient?.pause() }
    fun play() { currentSession?.remoteMediaClient?.play() }

    val remotePositionMs: Long
        get() = lastKnownRemotePositionMs

    private fun loadOnSession(session: CastSession, stream: StreamInfo) {
        if (!isCastSupported(stream)) {
            _lastErrorMessage = "This stream type is not supported for Cast playback."
            _castState.value = CastState.ERROR
            return
        }
        _lastErrorMessage = null
        if (remoteMediaMatches(stream)) {
            lastKnownPositionMs = lastKnownRemotePositionMs
            _castState.value = CastState.CONNECTED
            return
        }

        val metadata = MediaMetadata(MediaMetadata.MEDIA_TYPE_MOVIE).apply {
            putString(MediaMetadata.KEY_TITLE, stream.title)
            stream.thumbnailUrl?.let { thumb ->
                addImage(WebImage(thumb.toUri()))
            }
        }

        val mediaInfo = MediaInfo.Builder(stream.url)
            .setStreamType(if (stream.isLive) MediaInfo.STREAM_TYPE_LIVE else MediaInfo.STREAM_TYPE_BUFFERED)
            .setContentType(stream.mimeType)
            .setMetadata(metadata)
            .also { builder ->
                if (!stream.userAgent.isNullOrBlank()) {
                    val customData = org.json.JSONObject().apply {
                        put("userAgent", stream.userAgent)
                    }
                    builder.setCustomData(customData)
                }
            }
            .build()

        val loadRequest = MediaLoadRequestData.Builder()
            .setMediaInfo(mediaInfo)
            .setCurrentTime(if (stream.isLive) 0L else stream.positionMs)
            .setAutoplay(true)
            .build()

        session.remoteMediaClient?.load(loadRequest)
            ?.addStatusListener { status ->
                if (status.isSuccess) {
                    pendingStream = stream
                    lastKnownPositionMs = if (stream.isLive) 0L else stream.positionMs.coerceAtLeast(0L)
                    _castState.value = CastState.CONNECTED
                } else {
                    _lastErrorMessage = "The Cast device rejected the media."
                    _castState.value = CastState.ERROR
                }
            }
    }

    private fun remoteMediaMatches(stream: StreamInfo): Boolean = try {
        currentSession?.remoteMediaClient?.mediaInfo?.contentId == stream.url
    } catch (_: Exception) {
        false
    }

    private fun isCastSupported(stream: StreamInfo): Boolean {
        val lower = stream.url.lowercase()
        if (lower.startsWith("rtsp://") || lower.startsWith("rtsp:")) return false
        if (stream.url.isBlank()) return false
        return true
    }

    companion object {
        private fun inferMimeType(url: String): String = when {
            url.contains(".m3u8", ignoreCase = true) -> "application/x-mpegURL"
            url.contains(".mpd", ignoreCase = true) -> "application/dash+xml"
            url.contains(".mp4", ignoreCase = true) -> "video/mp4"
            url.contains(".ts", ignoreCase = true) -> "video/MP2T"
            url.contains("rtsp", ignoreCase = true) -> "video/rtsp"
            else -> "video/mp4"
        }
    }
}
""",

    "ui/player/PlayerCastController.kt": """package vid.mat.app.ui.player

import android.net.Uri
import android.view.View
import android.widget.ImageView
import android.widget.TextView
import android.widget.Toast
import androidx.appcompat.app.AppCompatActivity
import androidx.lifecycle.Lifecycle
import androidx.lifecycle.lifecycleScope
import androidx.lifecycle.repeatOnLifecycle
import androidx.media3.common.MediaItem
import com.google.android.gms.cast.framework.CastButtonFactory
import kotlinx.coroutines.Job
import kotlinx.coroutines.delay
import kotlinx.coroutines.isActive
import kotlinx.coroutines.launch
import vid.mat.app.R
import vid.mat.app.cast.CastManager
import vid.mat.app.databinding.ActivityMediaPlayerBinding
import vid.mat.app.player.PlaybackDestination
import vid.mat.app.player.PlaybackType
import vid.mat.app.ui.MediaPlayerActivity
import vid.mat.app.uimodel.NewPlayerViewModel
import vid.mat.app.util.AppLogger

class PlayerCastController(
    private val activity: AppCompatActivity,
    private val binding: ActivityMediaPlayerBinding,
    private val viewModel: NewPlayerViewModel
) {
    private val castManager = CastManager(activity.applicationContext)
    private var isCastPlaying = true
    private var localPlaybackPausedForCast = false
    private var wasCastingMedia = false
    private var castPositionJob: Job? = null

    @Suppress("UNCHECKED_CAST")
    private fun <T : View> findOptionalView(name: String): T? {
        val id = activity.resources.getIdentifier(name, "id", activity.packageName)
        return if (id != 0) binding.root.findViewById(id) as? T else null
    }

    private val mediaRouteButton: View? get() = findOptionalView("mediaRouteButton")
    private val castPlayPauseButton: ImageView? get() = findOptionalView("castPlayPauseButton")
    private val castStopButton: View? get() = findOptionalView("castStopButton")
    private val castMiniController: View? get() = findOptionalView("castMiniController")
    private val castTitleText: TextView? get() = findOptionalView("castTitleText")
    private val castStatusOverlay: TextView? get() = findOptionalView("castStatusOverlay")

    fun initialize(mediaTitleProvider: () -> String) {
        val routeButton = mediaRouteButton
        if (routeButton is androidx.mediarouter.app.MediaRouteButton) {
            try {
                CastButtonFactory.setUpMediaRouteButton(activity.applicationContext, routeButton)
                routeButton.visibility = View.VISIBLE
            } catch (e: Exception) {
                routeButton.visibility = View.GONE
                AppLogger.w("PlayerCastController", "Cast framework unavailable", e)
            }
        }

        castPlayPauseButton?.setOnClickListener {
            if (isCastPlaying) {
                castManager.pause()
                isCastPlaying = false
            } else {
                castManager.play()
                isCastPlaying = true
            }
            castPlayPauseButton?.setImageResource(
                if (isCastPlaying) R.drawable.ic_pause else R.drawable.ic_play
            )
        }

        castStopButton?.setOnClickListener {
            castManager.stopCasting()
        }

        activity.lifecycleScope.launch {
            activity.repeatOnLifecycle(Lifecycle.State.STARTED) {
                launch {
                    viewModel.newPlayer.currentlyPlaying.collect { mediaItem ->
                        prepareCurrentMedia(mediaItem, mediaTitleProvider)
                    }
                }
                launch {
                    castManager.castState.collect { state ->
                        when (state) {
                            CastManager.CastState.LOADING -> {
                                showCastStatus("Connecting to Cast device...")
                            }
                            CastManager.CastState.CONNECTED -> {
                                if (!castManager.hasActiveMedia) {
                                    binding.playerView.visibility = View.VISIBLE
                                    return@collect
                                }
                                wasCastingMedia = true
                                startRemotePositionTracking()
                                val position = castManager.lastKnownRemotePositionMs
                                viewModel.newPlayer.handoffPlaybackTo(PlaybackDestination.CAST, position)
                                localPlaybackPausedForCast = true
                                showMiniController(true, mediaTitleProvider())
                                showCastStatus("Now playing on TV")
                                binding.playerView.visibility = View.INVISIBLE
                            }
                            CastManager.CastState.DISCONNECTED -> {
                                stopRemotePositionTracking()
                                showMiniController(false, mediaTitleProvider())
                                binding.playerView.visibility = View.VISIBLE
                                if (wasCastingMedia || viewModel.newPlayer.playbackDestination.value == PlaybackDestination.CAST) {
                                    val resumePos = castManager.lastKnownRemotePositionMs
                                    viewModel.newPlayer.handoffPlaybackTo(PlaybackDestination.LOCAL, resumePos)
                                    showCastStatus("Cast ended; resumed locally")
                                }
                                wasCastingMedia = false
                                localPlaybackPausedForCast = false
                            }
                            CastManager.CastState.AVAILABLE -> {
                                showMiniController(false, mediaTitleProvider())
                            }
                            CastManager.CastState.ERROR -> {
                                stopRemotePositionTracking()
                                showMiniController(false, mediaTitleProvider())
                                binding.playerView.visibility = View.VISIBLE
                                if (wasCastingMedia || localPlaybackPausedForCast || viewModel.newPlayer.playbackDestination.value == PlaybackDestination.CAST) {
                                    viewModel.newPlayer.handoffPlaybackTo(PlaybackDestination.LOCAL, castManager.lastKnownRemotePositionMs)
                                }
                                showCastStatus(castManager.lastErrorMessage ?: "Unable to play this stream on the Cast device.")
                                localPlaybackPausedForCast = false
                                wasCastingMedia = false
                            }
                        }
                    }
                }
            }
        }
    }

    private fun prepareCurrentMedia(mediaItem: MediaItem?, titleProvider: () -> String) {
        val rawUri = mediaItem?.localConfiguration?.uri?.toString()?.trim().orEmpty()
        if (rawUri.isBlank()) return

        if (rawUri.startsWith("content://", ignoreCase = true) ||
            rawUri.startsWith("file://", ignoreCase = true) ||
            rawUri.startsWith("/") ||
            rawUri.startsWith("http://127.0.0.1", ignoreCase = true) ||
            rawUri.startsWith("http://localhost", ignoreCase = true)
        ) {
            return
        }

        val resolvedUrl = expandCastUrlIfNeeded(rawUri) ?: return
        val session = viewModel.newPlayer.currentSession.value
        val metadata = mediaItem?.mediaMetadata
        val title = metadata?.displayTitle?.toString()
            ?: metadata?.title?.toString()
            ?: titleProvider()
        val thumbnail = metadata?.artworkUri?.toString()
        val isLive = session?.playbackType == PlaybackType.LIVE

        castManager.prepareStream(
            CastManager.StreamInfo(
                url = resolvedUrl,
                title = title.ifBlank { "Stream" },
                thumbnailUrl = thumbnail,
                positionMs = if (isLive) 0L else viewModel.newPlayer.currentPosition.coerceAtLeast(0L),
                isLive = isLive
            )
        )
    }

    private fun expandCastUrlIfNeeded(rawUrl: String): String? {
        if (!rawUrl.contains("/X-AUTH-INJECT/")) return rawUrl
        val session = viewModel.newPlayer.currentSession.value ?: return null
        val candidate = (activity as? MediaPlayerActivity)?.failoverEngine?.getActiveCandidate(session.mediaId)
            ?: return null
        val user = candidate.headers.xtreamUser ?: return null
        val pass = candidate.headers.xtreamPass ?: return null
        return rawUrl.replace(
            "/X-AUTH-INJECT/",
            "/${Uri.encode(user)}/${Uri.encode(pass)}/"
        )
    }

    fun onResume() = castManager.register()

    fun onPause() {
        stopRemotePositionTracking()
        castManager.unregister()
    }

    private fun startRemotePositionTracking() {
        if (castPositionJob?.isActive == true) return
        castPositionJob = activity.lifecycleScope.launch {
            while (isActive && castManager.isCasting) {
                viewModel.newPlayer.updateExternalPlaybackPosition(castManager.lastKnownRemotePositionMs)
                delay(1_000L)
            }
        }
    }

    private fun stopRemotePositionTracking() {
        castPositionJob?.cancel()
        castPositionJob = null
    }

    private fun showMiniController(visible: Boolean, title: String) {
        castMiniController?.visibility = if (visible) View.VISIBLE else View.GONE
        castTitleText?.text = title.ifEmpty { "Casting to TV" }
    }

    private fun showCastStatus(message: String) {
        val overlay = castStatusOverlay
        if (overlay != null) {
            overlay.text = message
            overlay.visibility = View.VISIBLE
            overlay.animate().alpha(1f).setDuration(200).withEndAction {
                overlay.postDelayed({
                    overlay.animate().alpha(0f).setDuration(400)
                        .withEndAction { overlay.visibility = View.GONE }.start()
                }, 2000L)
            }.start()
        } else {
            Toast.makeText(activity, message, Toast.LENGTH_SHORT).show()
        }
    }
}
""",

    "notification/BootCompletedReceiver.kt": """package vid.mat.app.notification

import android.content.BroadcastReceiver
import android.content.Context
import android.content.Intent
import dagger.hilt.android.AndroidEntryPoint
import kotlinx.coroutines.CoroutineScope
import kotlinx.coroutines.Dispatchers
import kotlinx.coroutines.launch
import vid.mat.app.data.database.VideoUrlDatabase
import vid.mat.app.epg.EpgRefreshWorker
import vid.mat.app.util.AppLogger
import javax.inject.Inject

@AndroidEntryPoint
class BootCompletedReceiver : BroadcastReceiver() {

    @Inject
    lateinit var notificationManager: AppNotificationManager

    @Inject
    lateinit var database: VideoUrlDatabase

    override fun onReceive(context: Context, intent: Intent) {
        if (intent.action == Intent.ACTION_BOOT_COMPLETED || intent.action == "android.intent.action.QUICKBOOT_POWERON") {
            AppLogger.i("BootCompletedReceiver", "Device reboot detected -> Re-initializing channels and scheduled EPG/DVR workers")
            notificationManager.createAllNotificationChannels()

            val pendingResult = goAsync()
            CoroutineScope(Dispatchers.IO).launch {
                try {
                    val epgPrefs = context.getSharedPreferences("epg_prefs", Context.MODE_PRIVATE)
                    val manualEpg = epgPrefs.getString("epg_url", "").orEmpty().trim()
                    if (manualEpg.isNotBlank()) {
                        EpgRefreshWorker.schedule(context, manualEpg)
                    }
                    database.playlistSourceDao().getAll()
                        .mapNotNull { it.epgUrl?.trim()?.takeIf { url -> url.isNotBlank() } }
                        .distinct()
                        .forEach { url ->
                            EpgRefreshWorker.schedule(context, url)
                        }
                } catch (e: Exception) {
                    AppLogger.w("BootCompletedReceiver", "Boot worker rescheduling warning: ${e.message}")
                } finally {
                    pendingResult.finish()
                }
            }
        }
    }
}
""",

    "lifecycle/AppLifecycleSupervisor.kt": """package vid.mat.app.lifecycle

import android.content.Context
import androidx.lifecycle.DefaultLifecycleObserver
import androidx.lifecycle.LifecycleOwner
import androidx.work.WorkInfo
import androidx.work.WorkManager
import dagger.hilt.android.qualifiers.ApplicationContext
import kotlinx.coroutines.CoroutineScope
import kotlinx.coroutines.Dispatchers
import kotlinx.coroutines.SupervisorJob
import kotlinx.coroutines.flow.first
import kotlinx.coroutines.launch
import vid.mat.app.data.database.VideoUrlDatabase
import vid.mat.app.dvr.DvrRepository
import vid.mat.app.dvr.engine.ts.TsStreamSanitizer
import vid.mat.app.dvr.model.DvrStatus
import vid.mat.app.dvr.storage.DvrStorageManager
import vid.mat.app.image.LogoImageLoader
import vid.mat.app.util.AppLogger
import vid.mat.app.util.ReleaseReadinessVerifier
import java.io.File
import java.io.FileInputStream
import java.nio.ByteBuffer
import javax.inject.Inject
import javax.inject.Singleton

@Singleton
class AppLifecycleSupervisor @Inject constructor(
    @ApplicationContext private val context: Context,
    private val database: VideoUrlDatabase,
    private val dvrRepository: DvrRepository,
    private val storageManager: DvrStorageManager,
    private val readinessVerifier: ReleaseReadinessVerifier
) : DefaultLifecycleObserver {

    companion object {
        private const val TAG = "AppLifecycleSupervisor"
    }

    private val supervisorScope = CoroutineScope(SupervisorJob() + Dispatchers.IO)

    fun onApplicationCreated() {
        supervisorScope.launch {
            readinessVerifier.verifySystemHealth()
            reconcileInterruptedRecordings()
            reconcileOrphanedSyncStates()
        }
    }

    private suspend fun reconcileOrphanedSyncStates() {
        runCatching {
            database.playlistSourceDao().resetOrphanedSyncStates()
        }.onFailure { e ->
            AppLogger.e(TAG, "Failed to reconcile orphaned sync states", e)
        }
    }

    override fun onStop(owner: LifecycleOwner) {
        super.onStop(owner)
        AppLogger.i(TAG, "Application entered background. Trimming non-essential memory.")
        try {
            LogoImageLoader.get(context).memoryCache?.clear()
            database.openHelper.writableDatabase.execSQL("PRAGMA shrink_memory;")
        } catch (e: Exception) {
            AppLogger.w(TAG, "Background memory trim failed: ${e.message}")
        }
    }

    suspend fun reconcileInterruptedRecordings() {
        runCatching {
            val dvrDao = database.dvrDao()
            val workManager = WorkManager.getInstance(context)
            val activeRecordings = dvrDao.getActiveRecordings()

            for (recording in activeRecordings) {
                var isJobAlive = false
                if (!recording.workRequestId.isNullOrBlank()) {
                    val workInfos = runCatching {
                        workManager.getWorkInfosForUniqueWork("dvr_work_${recording.id}").get()
                    }.getOrDefault(emptyList())
                    isJobAlive = workInfos.any {
                        it.state == WorkInfo.State.RUNNING || it.state == WorkInfo.State.ENQUEUED
                    }
                }

                if (!isJobAlive) {
                    val finalFile = File(recording.outputFilePath)
                    val tempFile = storageManager.getTempWorkingFile(finalFile)

                    if (tempFile.exists() && tempFile.length() > 0L) {
                        val hasValidStructure = validateDiskFileStructure(tempFile)
                        if (hasValidStructure) {
                            val resolvedFinalFile = storageManager.resolveFinalFileForCapturedTemp(tempFile, finalFile)
                            val renamed = storageManager.finalizeAtomic(tempFile, resolvedFinalFile)
                            dvrDao.markRecordingFinished(
                                id = recording.id,
                                status = if (renamed) DvrStatus.PARTIAL else DvrStatus.FAILED,
                                stopMs = System.currentTimeMillis(),
                                fileSize = if (renamed) resolvedFinalFile.length() else 0L,
                                segments = recording.segmentsDownloaded,
                                failures = recording.segmentFailures,
                                reconnects = recording.reconnectCount,
                                error = "Interrupted by system process death. Salvaged.",
                                outputPath = if (renamed) resolvedFinalFile.absolutePath else null
                            )
                        } else {
                            tempFile.delete()
                            dvrDao.markRecordingFinished(
                                id = recording.id,
                                status = DvrStatus.FAILED,
                                stopMs = System.currentTimeMillis(),
                                fileSize = 0L,
                                segments = 0,
                                failures = 0,
                                reconnects = 0,
                                error = "Terminated due to corrupted stream data on process death"
                            )
                        }
                    } else if (finalFile.exists() && finalFile.length() > 0L) {
                        dvrDao.markRecordingFinished(
                            id = recording.id,
                            status = DvrStatus.PARTIAL,
                            stopMs = System.currentTimeMillis(),
                            fileSize = finalFile.length(),
                            segments = 0,
                            failures = 0,
                            reconnects = 0,
                            error = "Interrupted during finalization"
                        )
                    } else {
                        dvrDao.markRecordingFinished(
                            id = recording.id,
                            status = DvrStatus.FAILED,
                            stopMs = System.currentTimeMillis(),
                            fileSize = 0L,
                            segments = 0,
                            failures = 0,
                            reconnects = 0,
                            error = "Terminated due to system process death"
                        )
                    }

                    if (recording.isRecurring) {
                        dvrRepository.scheduleNextRecurrence(recording)
                    }
                }
            }

            val allValidRecordings = dvrDao.getAllRecordingsFlow().first()
            val validPaths = allValidRecordings
                .filter { it.status == DvrStatus.COMPLETED || it.status == DvrStatus.PARTIAL }
                .map { it.outputFilePath }
                .toSet()
            storageManager.cleanupOrphanedFiles(validPaths)
        }.onFailure { e ->
            AppLogger.e(TAG, "Error executing DVR reconciliation on startup", e)
        }
    }

    private fun validateDiskFileStructure(file: File): Boolean {
        if (!file.exists() || file.length() < 188L * 4) return false
        return try {
            FileInputStream(file).use { fis ->
                val header = ByteArray(188 * 16)
                val read = fis.read(header)
                if (read < 188 * 4) return false
                if (TsStreamSanitizer.hasValidMpegTsStructure(header)) return true

                val boxType = String(header, 4, 4, Charsets.US_ASCII)
                if (boxType in setOf("ftyp", "styp", "moov", "moof", "sidx")) {
                    return validateMp4Atoms(file)
                }
                false
            }
        } catch (_: Exception) {
            false
        }
    }

    private fun validateMp4Atoms(file: File): Boolean {
        var hasMoovOrMoof = false
        var hasMdat = false
        try {
            FileInputStream(file).use { fis ->
                var offset = 0L
                val fileLength = file.length()
                val buffer = ByteArray(8)
                while (offset < fileLength) {
                    val read = fis.read(buffer)
                    if (read < 8) break

                    val size32 = ByteBuffer.wrap(buffer, 0, 4).int.toLong() and 0xFFFFFFFFL
                    val type = String(buffer, 4, 4, Charsets.US_ASCII)

                    if (type == "moov" || type == "moof") hasMoovOrMoof = true
                    if (type == "mdat") hasMdat = true

                    val size = if (size32 == 1L) {
                        val extBuf = ByteArray(8)
                        fis.read(extBuf)
                        ByteBuffer.wrap(extBuf).long
                    } else size32

                    if (size < 8L) return false

                    val advance = if (size32 == 1L) size - 16L else size - 8L
                    if (advance < 0 || offset + size > fileLength) {
                        return false
                    }
                    fis.skip(advance)
                    offset += size
                }
            }
        } catch (_: Exception) {
            return false
        }
        return hasMoovOrMoof && hasMdat
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

May I proceed with the **final Batch 11 (7 remaining modified classes — PiP, Splash, Live Badge, Settings, Promo Coil/HMAC & Readiness Gate)** (`ui/player/PlaybackLifecycleCoordinator.kt`, `ui/SplashActivity.kt`, `ui/adapter/UrlListAdapter.kt`, `ui/settings/SettingsActivity.kt`, `ads/promo/PromoBannerImageLoader.kt`, `ads/promo/PromoSyncWorker.kt`, `util/ReleaseReadinessVerifier.kt`)? Or would you prefer me to split Batch 11 into 5 + 2?
Here is the Python script for the **final Batch 11 (7 remaining modified classes)**:

1. `ui/player/PlaybackLifecycleCoordinator.kt` — Adds `updatePiPState(isPlaying)` to refresh PiP play/pause `RemoteAction` icons and uses safe `stopService(...)` on `ACTION_STOP_PLAYBACK`.
2. `ui/SplashActivity.kt` — Routes Android TV devices to `TvMainActivity` and launches `OnboardingActivity` on mobile first-run.
3. `ui/adapter/UrlListAdapter.kt` — Recognizes unified `channelIdentity` Live TV channels via `!item.tvgId.isNullOrBlank()`.
4. `ui/settings/SettingsActivity.kt` — Wires optional cards/switches for Parental Controls, Profiles, Encrypted Backup/Restore, Auto Frame Rate, Auto-PiP, and Auto-Timeshift.
5. `ads/promo/PromoBannerImageLoader.kt` — Migrates promo banner loading from Glide to Coil (`LogoImageLoader`) with `NetworkSecurityValidator` URL validation.
6. `ads/promo/PromoSyncWorker.kt` — Uploads the real Keystore HMAC receipt signature via `CryptoUtils.signReceipt(...)`.
7. `util/ReleaseReadinessVerifier.kt` — Runs `DatabaseMigrationHelper.verifyDatabaseIntegrity(db)` during system health verification.

```python
#!/usr/bin/env python3
from pathlib import Path

BASE_DIR = Path("/home/username/AndroidStudioProjects/Ytvmate/app/src/main/java/vid/mat/app")

FILES = {
    "ui/player/PlaybackLifecycleCoordinator.kt": """package vid.mat.app.ui.player

import android.app.PendingIntent
import android.app.PictureInPictureParams
import android.app.RemoteAction
import android.content.BroadcastReceiver
import android.content.Context
import android.content.Intent
import android.content.IntentFilter
import android.graphics.drawable.Icon
import android.os.Build
import android.util.Rational
import android.widget.Toast
import androidx.annotation.RequiresApi
import androidx.appcompat.app.AppCompatActivity
import androidx.core.content.ContextCompat
import androidx.lifecycle.lifecycleScope
import androidx.preference.PreferenceManager
import kotlinx.coroutines.Job
import kotlinx.coroutines.launch
import vid.mat.app.R
import vid.mat.app.databinding.ActivityMediaPlayerBinding
import vid.mat.app.ui.MediaPlayerActivity
import vid.mat.app.uimodel.NewPlayerViewModel
import vid.mat.app.util.AppLogger
import vid.mat.app.util.PlaybackService
import vid.mat.app.util.SensitiveDataSanitizer
import java.util.concurrent.atomic.AtomicBoolean

class PlaybackLifecycleCoordinator(
    private val activity: AppCompatActivity,
    private val binding: ActivityMediaPlayerBinding,
    private val viewModel: NewPlayerViewModel,
    private val onError: (String) -> Unit
) {
    companion object {
        private const val TAG = "PlaybackLifecycleCoord"
    }

    private var playerObservationJob: Job? = null
    private var pipActionReceiver: BroadcastReceiver? = null
    private var playbackErrorReceiver: BroadcastReceiver? = null
    private val isReceiverRegistered = AtomicBoolean(false)

    fun startPlaybackService(mediaData: PlayerMediaData) {
        val serviceIntent = Intent(activity, PlaybackService::class.java).apply {
            action = PlaybackService.ACTION_START_PLAYBACK
            putExtra(PlaybackService.EXTRA_MEDIA_PATH, mediaData.mediaPath)
            putExtra(PlaybackService.EXTRA_MEDIA_TITLE, mediaData.mediaTitle)
            putExtra(PlaybackService.EXTRA_RESUME_POSITION, mediaData.resumePosition)
            putExtra(PlaybackService.EXTRA_THUMBNAIL_URI, mediaData.thumbnailUrl)
        }
        try {
            if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.O) {
