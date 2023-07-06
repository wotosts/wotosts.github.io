---
title: "[Android] MVI 의 구현"
categories:
- Android
tags:
- Architecture
- MVI
---

MVI 개념정리 2편입니다.<br><br>
[지난 포스트](http://https://wotosts.github.io/android/MVI-Android-Architecture-Blueprint-1/)에서 MVI 의 개념 및 Android 공식 Architecture blueprint를 MVI와 연관지어 살펴보았는데, <br>
이번에는 다른 방식의 MVI 구현 방법을 살펴봅니다. 
{: .notice--success}

<br>


# 두 가지 예시

많은 자료를 찾아본 것은 아니지만, 크게 두 가지 구현을 예시로 들 수 있을 것 같아요. 
1. 제가 생각하는 정석? 적인 방법과
2. 정석?을 좀 더 단순하게 구현한 방법입니다. 

## Delish

[https://github.com/Elbehiry/Delish](https://github.com/Elbehiry/Delish)

보기에 매우 복잡합니다. 하지만 제가 생각하기에 가장 정석, MVI 의 정의? 와 가까운 방법이 아닐까 싶습니다.
세부 구현은 물론 앱마다 다르겠지만 대략적인 흐름은 아래와 같습니다.

1. Intent(Event) 발생 (MviEvent)
2. Intent로 Partial state 생성 (MviViewResult)
3. Partial State + reducer를 통해 Ui state 생성 (MviViewState)
4. (옵션) Side Effect 발생
5. MviViewState 렌더링


MVI 구현을 위해 `MviEvent`, `MviViewResult`, `MviViewState`, `MviSideEffect` 4가지의 interface가 필요합니다. 

```kotlin
interface MviViewState
interface MviSideEffect
interface MviEvent
interface MviViewResult
```

이를 이용하여 만든 `MviViewModel`을 사용합니다.

```kotlin
    abstract class MviViewModel<Event : MviEvent, Result : MviViewResult, State : MviViewState, Effect : MviSideEffect>(
        initialState: State
    ) : ViewModel() {
        val states: StateFlow<State>
        val effects: Flow<Effect>
        private val events = MutableSharedFlow<Event>()
    
        init {
            events
                .share()
                .toResults()
                .share()
                .also { results ->
                    states = results.toStates(initialState)
                        .stateIn(
                            scope = viewModelScope,
                            started = SharingStarted.Lazily,
                            initialValue = initialState
                        )
                    effects = results.toEffects()
                }
        }
    
        // View(UI) 에서 호출하는 함수 - intent
        fun processEvent(event: Event) {
            viewModelScope.launch {
                events.emit(event)
            }
        }
    
        protected abstract fun Flow<Event>.toResults(): Flow<Result>
        protected abstract fun Result.reduce(state: State): State
        protected open fun Flow<Result>.toEffects(): Flow<Effect> = emptyFlow()
    
        private fun Flow<Result>.toStates(initialState: State): Flow<State> {
            return scan(initialState) { state, result -> result.reduce(state) }
        }
    
        private fun <T> Flow<T>.share(): Flow<T> {
            return shareIn(scope = viewModelScope, started = SharingStarted.Eagerly)
        }
    }
```


* `MviEvent`
	*  Intent 입니다.
	*  View에서 viewModel.processEvent()를 호출하면서 사용자와의 상호작용 또는 Side Effect로 인해 발생한 Intent를 ViewModel에 전달합니다.
        
        ```kotlin
        internal sealed interface ViewEvent : MviViewModel.MviEvent {
            object GetHomeContent : ViewEvent
            data class ToggleBookMark(val recipesItem: RecipesItem) : ViewEvent
            object OpenIngredients : ViewEvent
        }
        ```
        
*  `MviResult`
	*  MviViewModel은 init 블록에서 event flow를 구독하고 있습니다. 
	*  processEvent로부터 발생한 event에 비즈니스 로직을 적용하여 알맞은 Result로 변환합니다. (toResult())
	*  지난 포스트에서 언급했던 Partial State 입니다.
        
        ```kotlin
        internal sealed interface ViewResult : MviViewModel.MviViewResult {
            object ErrorResult : ViewResult
            data class HomeContent(
                val ingredientList: List<IngredientItem> = emptyList(),
                val cuisinesList: List<CuisineItem> = emptyList(),
                val randomRecipes: List<RecipesItem> = emptyList()
            ) : ViewResult
            data class OnIngredientsSheet(val ingredientList: List<IngredientItem>) : ViewResult
            object NoOpResult : ViewResult
        }
        ```
        
* `MviViewState`
	* Event -> Result로 변환 후, reduce를 통해 만들어지는 화면 state입니다.
        
        ```kotlin
        internal data class ViewState(
            val isLoading: Boolean = true,
            val hasError: Boolean = false,
            val ingredientList: List<IngredientItem> = emptyList(),
            val cuisinesList: List<CuisineItem> = emptyList(),
            val randomRecipes: List<RecipesItem> = emptyList()
        ) : MviViewModel.MviViewState
        ```
        
* `MviSideEffect`
	* Result의 결과로 발생하는 Side Effect 입니다. 
	* View에서 Effect를 처리하기도 합니다. View에서 아래 Effect를 확인하면 시트를 보여주는 동작을 하겠지요.
        
        ```kotlin
        internal interface ViewEffect : MviViewModel.MviSideEffect {
            data class OpenIngredientsSheet(val ingredients : List<IngredientItem>) : ViewEffect
        }
        ```
        
				
#### 예시			
 
 ```kotlin
 @HiltViewModel
internal class DetailsViewModel @Inject constructor(
    private val getRecipeInformationUseCase: GetRecipeInformationUseCase,
    private val toggleSavedRecipeUseCase: ToggleSavedRecipeUseCase,
    savedStateHandle: SavedStateHandle
) : MviViewModel<ViewEvent, ViewResult, ViewState, ViewEffect>(ViewState()) {

    private val recipeId: Int = savedStateHandle[RECIPE_ID] ?: DEFAULT_RECIPE_ID

    init {
        processEvent(GetRecipe(recipeId))
    }

    override fun Flow<ViewEvent>.toResults(): Flow<ViewResult> {
        return merge(
            filterIsInstance<GetRecipe>().toGetRecipeResult(),
            filterIsInstance<ToggleBookMark>().toToggleBookMarkResult()
        )
    }

    override fun ViewResult.reduce(state: ViewState): ViewState {
        return when (this) {
            is ErrorResult -> state.copy(isLoading = false, hasError = true)
            is RecipeItem -> state.copy(
                isLoading = false,
                hasError = false,
                recipe = recipe
            )
            else -> state
        }
    }

    private fun Flow<GetRecipe>.toGetRecipeResult(): Flow<ViewResult> {
        return mapLatest { getRecipeInformationUseCase(it.recipeId) }
            .map {
                if (it is Result.Success) {
                    RecipeItem(it.data)
                } else {
                    ErrorResult
                }
            }
    }

    private fun Flow<ToggleBookMark>.toToggleBookMarkResult(): Flow<ViewResult> {
        return mapLatest {
            toggleSavedRecipeUseCase(it.recipesItem)
            NoOpResult
        }
    }
}
 ```
   

## One of ProAndroidDev post
[Migrate from MVVM to MVI](https://proandroiddev.com/migrate-from-mvvm-to-mvi-f938c27c214f)

MVI 구현 방법을 이곳 저곳 찾아다니다 보면 가장 많이 마주하게 되는 구현 방법이라고 생각됩니다. 
저도 사이드 프로젝트를 진행할 때 이 방법을 참고하여 MVI를 구현하였습니다. 

이 방법도 Delish와 마찬가지로 3가지 필수 개념인 `STATE`, `EVENT`, `EFFECT`가 있으며,  이들을 이용하여 `UnidirectionalViewModel` 을 만들어 사용합니다.
UnidirectionalViewModel에는 View에 노출할 `state`, `effect`와 intent를 처리하는 `event()` 함수가 존재합니다. 
    
```kotlin
    interface UnidirectionalViewModel<STATE, EVENT, EFFECT> {
        val state: StateFlow<STATE>
        val effect: SharedFlow<EFFECT>
    
        fun event(event: EVENT)
    }
```

* `STATE`
	* 화면에 보여지는 state 입니다.

* `EVENT`
	* Intent 입니다.

* `EFFECT`
	* Side Effect 입니다.


    
3가지 개념을 UnidirectionalViewModel을 확장한 Contract 로 묶어서 한 곳에서 정의하고 있습니다. 
현재 화면에 보여줄 데이터와 발생 가능한 이벤트들을 한눈에 파악하기 용이해보입니다. 
    
```kotlin
    interface NewsListContract :
        UnidirectionalViewModel<NewsListContract.State, NewsListContract.Event,
            NewsListContract.Effect> {
    
        // 화면에 보여줄 state
        data class State(
           val news: List<News> = listOf(),
           val refreshing: Boolean = false,
           val showFavoriteList: Boolean = false,
        )
    
        // intent
        sealed class Event {
          data class OnFavoriteClick(val news: News) : Event()
          data class OnGetNewsList(val showFavoriteList: Boolean) : Event()
          data class OnSetShowFavoriteList(val showFavoriteList: Boolean) : Event()
          object OnRefresh: Event()
          object OnBackPressed : Event()
          data class ShowToast(val message: String) : Event()
        }
    
        // 그리고 side effect
        sealed class Effect {
          object OnBackPressed : Effect()
          data class ShowToast(val message: String) : Effect()
        }
    }
```
    

#### 예시
Delish보다 비교적 코드를 이해하기 쉽습니다. 
Event -> PartialState로 변환하는 과정이 생략되어있고, Event를 바로 UiState로 변환합니다.
event() 함수가 reducer 역할을 한다고 볼 수 있습니다. 

```kotlin
@HiltViewModel
class NewsListViewModel @Inject constructor(
    private val getNewsUseCase: GetNewsUseCase,
    private val getFavoriteNewsUseCase: GetFavoriteNewsUseCase,
    private val toggleFavoriteNewsUseCase: ToggleFavoriteNewsUseCase,
) : NewsListContract {

    private val mutableState = MutableStateFlow(NewsListContract.State())
    override val state: StateFlow<NewsListContract.State> = 
          mutableState.asStateFlow()

    private val effectFlow = MutableSharedFlow<NewsListContract.Effect>()
    override val effect: SharedFlow<NewsListContract.Effect> = 
          effectFlow.asSharedFlow()

    override fun event(event: NewsListContract.Event) = when (event) {
        is NewsListContract.Event.OnSetShowFavoriteList -> 
            onSetShowFavoriteList(showFavoriteList = event.showFavoriteList)
        is NewsListContract.Event.OnGetNewsList -> 
            getData(showFavoriteList = mutableState.value.showFavoriteList)
        is NewsListContract.Event.OnFavoriteClick -> 
            onFavoriteClick(news = event.news)
        NewsListContract.Event.OnRefresh -> getData(isRefreshing = true)
        NewsListContract.Event.OnBackPressed -> onBackPressed()
        is NewsListContract.Event.ShowToast -> showToast(event.message)
    }
}
```


## 공통점 
두 방법의 공통점으로는, 

- **View → ViewModel으로 Intent를 전달하는 함수는 오직 하나**
- 이름은 다르지만 Intent, Side Effect, State 개념 + Reducer가 있고, 
- Side Effect 개념을 좁혀, View에서 구독하여 ViewModel -> UI 로 필요한 동작을 하도록 알려주는 역할로 사용하고 있습니다. 
	- MVI 개념적으로는, 백그라운드와 API 호출 등의 작업도 Side Effect로 취급합니다.

#### 기타
두 방법 모두 ViewModel의 effect 자료형은 SharedFlow 입니다. 
UI에서 처리하는 이벤트는 보통 1번만 처리하고 말아야 하는 경우가 있어서 SharedFlow를 주로 사용하는 것 같은데,
effect 구독을 늦게 시작했으나, 이전에 발생한 effect를 사용해야 하는 경우에는 문제가 발생할 수 있습니다. 
(실제로 발생했습니다... 🥲)

이런 디테일한 부분들은 요구사항에 맞추어 적당히 수정하여 사용합니다.



---


처음 MVI를 접했을 때에는 생각보다 이해하기 어려웠지만, 익숙해지고 나면 단방향 플로우가 정말 깔끔하게 느껴지는 것 같아요.
