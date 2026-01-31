---
name: android-flow-architecture
description: AndroidプロジェクトでのViewModel、Flow、Repository、UIの実装パターンガイド。新規機能実装時やリファクタリング時に参照。Jetpack Compose + Hilt/Koin環境向け。
---

# ViewModel/Flow/Repository 実装ガイド

## 基本原則

- **initでAPIを呼ばない** - Flowの遅延評価を使用
- **UIからのトリガー不要** - `SharingStarted.WhileSubscribed()` でsubscribe時に自動取得
- **Screen/Content分離** - Screenで状態管理、Contentは表示のみ（プレビュー可能）

## ViewModel パターン

```kotlin
class MyViewModel : ViewModel() {
    private val _uiState = MutableStateFlow(UiState())
    val uiState = _uiState.stateIn(viewModelScope, SharingStarted.Lazily, UiState())

    // リロードトリガー
    private val reloadTrigger = MutableStateFlow(0L)

    @OptIn(ExperimentalCoroutinesApi::class)
    val items = reloadTrigger.flatMapLatest { repository.getItemsFlow() }
        .stateIn(viewModelScope, SharingStarted.WhileSubscribed(), emptyList())

    fun reload() { reloadTrigger.update { Clock.System.now().toEpochMilliseconds() } }
}
```

## Repository パターン

### キャッシュ付きFlow（遅延評価）

```kotlin
// subscribe時に自動取得、initやLaunchedEffect不要
fun pinnedPostIdFlow(fromApi: Boolean): Flow<Int?> = channelFlow {
    if (fromApi || _cache.value == null) {
        _cache.value = api.fetch()
    }
    send(_cache.value)
    launch { _cache.collect { send(it) } }
}
```

### LRUキャッシュ（IDでデータ取得）

```kotlin
private val cache = lruCache<Int, Feed>(maxSize = 100)

fun getFeedCache(postId: Int): Feed? = synchronized(this) { cache[postId] }

// loadFlow内でキャッシュ保存
override fun loadFlow() = channelFlow { /*...*/ }
    .onEach { result -> result.feeds.forEach { cache.put(it.id, it) } }
```

### アクション通知でリスト自動更新

```kotlin
private val action = MutableSharedFlow<Action>()

override fun loadFlow() = channelFlow {
    val feeds = mutableListOf<Feed>()
    // アクション監視
    action.onEach { act ->
        when (act) {
            is Action.Delete -> feeds.removeAll { it.id == act.id }
        }
        send(feeds.toList())
    }.launchIn(this)
}

override suspend fun delete(id: Int) {
    api.delete(id)
    action.emit(Action.Delete(id))
}
```

## Hilt EntryPoint（カスタムViewModel）

@HiltViewModelが使えない場合（動的引数）：

```kotlin
@EntryPoint
@InstallIn(SingletonComponent::class)
interface MenuEntryPoint {
    fun repository(): MyRepository
}

@Composable
fun MenuBottomSheet(itemId: Int) {
    val context = LocalContext.current
    val viewModel = viewModel<MenuViewModel>(
        key = "menu_$itemId",
        factory = object : ViewModelProvider.Factory {
            override fun <T : ViewModel> create(modelClass: Class<T>): T {
                val ep = EntryPointAccessors.fromApplication(context, MenuEntryPoint::class.java)
                return MenuViewModel(itemId, ep.repository()) as T
            }
        }
    )
}
```

## UI パターン

```kotlin
@Composable
fun MyScreen(viewModel: MyViewModel = hiltViewModel()) {
    val items by viewModel.items.collectAsStateWithLifecycle()
    var isRefreshing by rememberSaveable { mutableStateOf(false) }

    if (isRefreshing) {
        LaunchedEffect(isLoading) { isRefreshing = isLoading }
    }

    MyContent(
        items = items,
        isRefreshing = isRefreshing,
        onRefresh = { viewModel.reload(); isRefreshing = true }
    )
}
```

## DO / DON'T

| DO | DON'T |
|----|-------|
| `stateIn` + `SharingStarted.WhileSubscribed()` | initでAPI呼び出し |
| `collectAsStateWithLifecycle()` | `collectAsState()` |
| `runCatching` でエラー処理 | UIからLaunchedEffectでAPI呼び出し |
| `synchronized` でキャッシュアクセス | GlobalScope使用 |
| Screen/Content分離 | ContentでViewModel依存 |
