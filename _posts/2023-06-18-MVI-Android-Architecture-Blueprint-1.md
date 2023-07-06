---
title: "[Android] MVI와 Android Architecture Blueprint"
categories:
- Android
tags:
- Architecture
- MVI
---

MVI 개념정리 1편입니다.
{: .notice--success}

# MVI?

(함수형 프로그래밍의) 상태 지향 아키텍쳐.

### 구성

`Model`-`View`-`Intent` 로 구성되며, **단방향**으로 데이터가 이동하는 순환구조입니다.

💡 **view(model(intent()))** 이렇게 표현하기도 하네요. 


- `Model`
    - State. 상태(앱의 상태).
    - Intent에 의해서만 새로운 State가 생성될 수 있습니다.
- `View`
    - View. UI
    - intent 를 관찰하여 유저와 상호작용
- `Intent`
    - 사용자에 의해, 또는 앱 내에서 발생하는, 앱의 상태를 변화시키는 Action
    - Intent는 새로운 State를 생성합니다.

<br>

### MVI의 Model은 아래처럼..

```kotlin
sealed class MovieState {
  object LoadingState : MovieState()
  data class DataState(val data: List<Movie>) : MovieState()
  data class ErrorState(val data: String) : MovieState()
}
```

위 sealed class 내의 class들이 각각의 state를 나타내고, intent의 결과에 따라 state가 변화합니다.

```kotlin
// initial state
var state: MovieState = LoadingState

// change state
state = DataState(data)
```

<br>

### 이전 상태의 model이 필요한 경우에는?

위 같은 형태로 state를 작성할 경우, 이전 state의 데이터를 화면에 보여주고 싶을 때를 처리하기가 어렵습니다.

이 때 필요한 것이 Reducer로,
`Reducer` = 각 요소를 축약된 컴포넌트로 병합하는 단계를 제공하는 개념입니다.


```kotlin
val numbers = listOf(5, 2, 10, 4)

val simpleSum = numbers.reduce { sum, element -> 
  println("sum = $sum, element = $element")
  sum + element 
}
println(simpleSum)


sum = 5, element = 2
sum = 7, element = 10
sum = 17, element = 4
21
```

위 코드는 [코틀린이 제공하는 reduce 연산의 예시](https://kotlinlang.org/docs/collection-aggregate.html#fold-and-reduce)입니다.
State Reducer 또한 위와 같이 동작하며, 이전 state를 바탕으로 새로운 state를 만들어 냅니다. 

과정은

1. 새로운 state를 나타내는 PartialState
2. 이전 state가 필요한 새로운 Intent가 들어오면, 완료된 상태로부터 PartialState를 만듭니다.
3. reduce() 함수에서 이전 state와 PartialState를 이용하여 화면에 표시할 새 state를 만듭니다.
4. 이전 상태를 적용하고 새 상태를 반환합니다. 

💡 ViewState, PartialState 두 개의 state를 사용한 예시입니다.

```kotlin
// partial state
sealed class MovieState {
  object LoadingState : MovieState()
  data class DataState(val data: List<Movie>) : MovieState()
  data class ErrorState(val data: String) : MovieState()
}

// ui state
data class MovieUiState(
	isLoading: Boolean = false, 
	error: String = "",
	movies: List<Movie> = emptyList()
)

// reducer
var state: MovieState = LoadingState
var uiState: MoviewUiState = MovieUiState()

val reducer = { prevUiState, newState -> 
		uiState = when(newState) {
			LoadingState -> prevUiState.copy(isLoading = true)
			is DataState -> prevUiState.copy(isLoading = false, movies = newState.data)
			is ErrorState -> prevUiState.copy(isLoading = false, error = newState.data)
		}
}
```


<br>

### MVI의 이점

위에서도 언급했듯, MVI는 단방향의 순환구조를 가지며, 아래와 같은 이점을 가집니다.

- 단일 상태
    - 데이터를 한 곳에서 관리하고, 단일 상태를 보장하기 때문에
    - 유지보수가 편하며
    - 디버깅에 용이
    - View 생명주기 동안 일관성 있는 상태
- **단방향 데이터 흐름**
    - 앱의 로직을 예측가능하게 만듭니다.
- **Thread Safety**
    - 모델의 수정이 불가능하며, 모델이 만들어지는 곳은 단 한 곳 뿐입니다.
    - 다른 스레드에서 모델을 수정하게 되어 일어날 수 있는 여러 상황을 방지할 수 있습니다.
- 용이한 디버깅
    - 오류가 발생한 시점의 상태를 확인할 수 있어, 추적이 용이합니다.
- 테스트 유용성
    - 메소드의 결과가 예상되는 상태가 맞는지 확인하면 됩니다.

### MVI의 단점

- 상용구가 많습니다.... 정말...
- 단방향으로 데이터가 흐른다고는 하지만 상당히 복잡하게 느껴질 수 있으며
- 상태 변경 시 매번 객체를 생성하므로, 빈번한 GC 발생 가능성이 있습니다.



<br>
<br>

## MVI는 MVVM의 어떤 문제를 개선하는가?

MVI에서 State를 변경할 수 있는 방법은 Intent를 발생시켜 새로운 State를 생성하는, 단방향 흐름을 타는 것 뿐입니다. 

- 즉, **State는 불변**.
- 또한, View에 그려주는, View가 확인할 수 있는 **state는 단 하나.**

1. Multiple State → Single State
    
    안드로이드 MVVM 패턴의 구현은 보통 View(xml), DataBinding, ViewModel 등을 이용합니다.
    
    여기서, View와 비즈니스 로직(viewModel, domain, data layer 등)이 서로 다른 state를 가질 수 있기 때문에 개발자가 이들을 동기화 시켜주는 과정이 필요합니다.<br>
		(예: 유명한 예시로.. 분명 ViewModel은 로딩이 끝난 상태를 가지고 있으나, View는 로딩을 표시하는... 직접 격어본... 그것이 있습니다.)
    
    → MVI의 state는 단 하나로, **여러 state를 동기화 시킬 필요가 없어, 상태 제어가 간단해집니다.**
    
2. Side Effect
    
     Side effect(API 호출 등)는 결과를 예측하기가 쉽지 않은데, 
    
    → MVI 에서는 side effect의 결과로 intent로 발생시켜서 새로운 state를 생성할 수 있기 때문에, Side effect의 결과도 예측 가능해집니다. 
    

<br>

~~이 둘을 이렇게 비교한다는게 이상하지만~~

| - | MVVM | MVI |
| --- | --- | --- |
| View | * UI <br> * 사용자와 상호작용 | * UI <br>* 사용자와 상호작용 → Intent 발생<br>* 상태에 따라 UI 렌더링 |
| ViewModel | * View에 보여줄 데이터 가공을 위한 비즈니스 로직 포함<br>* 바인딩을 통해 UI 요소를 업데이트<br>* AAC ViewModel은 안드로이드 수명주기에 맞추어 View에 보여줄 데이터를 관리(홀더) | * state 홀더로 사용<br>  intent → 도메인 또는 데이터 레이어와 상호작용하면서 발생한 각각의 결과를 State로 맵핑 |
| Model | * 앱에서 사용되는 데이터와 데이터 처리 로직 | * 앱의 상태 자체 |

<br>

MVI 에는 ViewModel 이라는 개념이 없지만, 안드로이드 공식에서 상태 홀더로 ViewModel을 사용하는 방법도 안내하고 있기 때문에 포함하여 비교해보았습니다. 
MVI에서도 ViewModel에 데이터 가공 로직을 포함할 수 있습니다. 사용하기 나름인 것 같아요.



<br>

---

# 안드로이드에서의 MVI

안드로이드 앱은 기본적인 MVI 사이클과 같이 순수 함수만으로 구성되지 않습니다. 
때문에 `Side Effect` 라는 개념이 추가됩니다.

<img src= '/assets/images/2023-06-18/mvi.png'/>


- `Model(State)`
    - 앱의 현재 state
        
        ```kotlin
        // 화면에 보여줄 데이터(ui state)
        data class UiState(
        	val isLoading: Boolean,
        	val title: String,
        	val desc: String
        )
        
        // 현재 앱의 state
        sealed class State {
        	object Loading: State()
        	data class Idle(title: String, desc: String): State()
        }
        ```
        
- `View`
    - Model을 적절하게 렌더링하는 Composable이나 Activity, Fragment 등
- `Intent(Event)`
    - 사용자와의 상호작용 또는 ViewModel에서 발생하는, 상태를 변화시키는 Action
    - Intent의 결과는 Model을 변경하지만, 동시에 사이드 이펙트를 발생 시킬 수 있습니다.
- `Side Effect`
    - 네비게이팅이나 토스트, 백그라운드 작업, API 호출 등
    - 사이드 이펙트의 결과가 새로운 Intent가 되어 Model을 변경할 수 있습니다.


실제로 MVI를 적용했을 때 각 요소의 경계가 어디까지이며, 이건 이런 역할을 하니까 무조건 이거야! 라고 정의할 수는 없을 것 같아요.
회바회, 코바코...


<br>

## Android Blueprint

[architecture-samples/TasksViewModel.kt at main · android/architecture-samples](https://github.com/android/architecture-samples/blob/main/app/src/main/java/com/example/android/architecture/blueprints/todoapp/tasks/TasksViewModel.kt)

안드로이드 공식에서는 MVI라는 단어를 언급하지는 않지만, UDF와 State를 강조하고 있습니다. 
(가이드가 볼 때마다 업데이트 되는 것 같은데 기분탓이겠죠?)

MVI 또한 UDF, State가 중요한 패턴이므로 연관지어 보겠습니다.



코드 샘플을 간단하게 살펴보자면

- **UiState 는 현재 화면의 상태, 화면에 보여줄 데이터를 가집니다.**
    
    ```kotlin
    // ui state
    data class XXUiState(
    	val isLoading: Boolean = false,
    	val userMessage: Int = 0,
    	val item: List<Any> = emptyList()
    )
    ```
    

- **ViewModel은 Intent를 처리하여 UiState를 생성합니다.**
    
    샘플에서는 UiState의 프로퍼티들을 개별 MutableStateFlow로 선언하고, 이 값들이 바뀔 때마다 새로운 UiState를 만듭니다.  
	UiState 생성은 하나의 Reducer에서 담당합니다.
    
    결과적으로는 Reducer가 생성한 하나의 State만을 이용하게 됩니다.  
    
    ```kotlin
    // viewModel
    class XXViewModel() {
      // 얘네가 intent가 될 수도 있는 것. 
      // 얘네가 바뀔 때마다 uiState를 구성하기 때문(intent -> state)
    	val _isLoading = MutableStateFlow(false)
    	val _userMessage: MutableStateFlow<String?> = MutableStateFlow(null)
    	val _item = MutableStateFlow(emptyList())
    
    	val uiState: StateFlow<XState> = combine(
    		_isLoading, _userMessage, _item
    	) { isLoading, userMessage, item -> 
    			// do something
    			XState(
    				isLoading = isLoading, 
    				userMessage = userMessage, 
    				item = item 
    			)
    	}.stateIn(
    		scope = viewModelScope,
    		started = WhileSubscribed,
    		initialValue = XXState(isLoading = true)
    	)
    
    	// ...
    }
    ```
    


- **View는 UiState에 포함된 정보를 이용하여 UI를 그려줍니다.**
    
    UI는 Compose로 작성되어 있습니다. 
    
    Compose는 주어진 상태에 따라 UI를 렌더링하기 때문에 MVI와 합이 맞는 것 같아요. 
    
    ```kotlin
    // view
    @Composable
    fun XXScreen(viewModel: XXViewModel) {
    	//...
    
    	val uiState by viewModel.uiState.collectAsStateWithLifecycle()
    
    	Content(isLoading = uiState.isLoading, item = uiState.item)
    	
    	uiState.userMessage?.let {
    		LaunchedEffect(viewModel, it) {
    			// ..
    			// viewModel.stateXChange
    		}
    	}
    }
    
    @Composable
    fun Content(isLoading: Boolean, item: List<Any>) {
    	// ...
    }
    ```
    

- **Side Effect의 처리**
    
    Blueprint 코드상에서 side effect라고 명시된 부분은 없습니다.
    
    대신 state 내에 side effect와 관련된 값이 포함되어 있고, View에서 state의 변경을 확인하면 적절한 처리를 수행하는 방식으로 side effect를 처리하고 있습니다.
    - 화면 이동  
      Blueprint는 싱글액티비티 구조로 이루어져 있어, navigation을 통해 화면 이동을 수행합니다.
    Composable에 화면 이동을 수행하는 람다를 전달하는 방식입니다. 
    - 토스트나 다이얼로그 표시  
      스낵바를 띄우는 예시 코드입니. 스낵바는 one-off 이벤트이므로 이벤트 수행 후 
    관련된 값을 다시 초기값으로 변경하여 해당 이벤트가 1회만 수행되도록 처리하고 있습니다. 
    
	```kotlin
@Composable
fun AddEditTaskScreen((
		..
    onTaskUpdate: () -> Unit,
    viewModel: AddEditTaskViewModel = hiltViewModel()
) {
    ..

        // Check if the task is saved and call onTaskUpdate event
        LaunchedEffect(uiState.isTaskSaved) {
            if (uiState.isTaskSaved) {
                onTaskUpdate()        // 화면 이동 처리
            }
        }

        // Check for user messages to display on the screen
        uiState.userMessage?.let { userMessage ->
            val snackbarText = stringResource(userMessage)
            // 스낵바 메세지
            LaunchedEffect(scaffoldState, viewModel, userMessage, snackbarText) {
                scaffoldState.snackbarHostState.showSnackbar(snackbarText)
                viewModel.snackbarMessageShown()    // 처리 완료
            }
        }
    }
}
```

    여기서 눈여겨 보아야 할 부분은
	  - 1회만 수행되어야 하는 side effect의 경우 수행 완료 처리를 별도로 해주고 있다는 것입니다.
이는 side effect의 결과가 intent를 발생시켜 새로운 state를 생성하는 것으로 볼 수 있습니다.


  side effect를 이러한 방식으로 처리하는 이유는 안드로이드 권장 사항이기 때문입니다. 
	
	![]({{ 'assets/images/2023-06-18/guideline_event.png' | relative_url }})
- 아키텍쳐 권장 사항에서는 viewModel → UI 이벤트 전송을 권장하지 않습니다.
    - viewModel에서 이벤트를 소비하여 state로 전환 → UI 단에서는 변경된 state에 반응하는 것을 권장합니다.
		
    - 이 방법을 사용했을 때, 처리를 한번 더 해주어야 하는 부분이 번거롭게 느껴졌습니다.



<br>

## 참고 

* [UI events - Android Developers](https://developer.android.com/topic/architecture/ui-layer/events)
* [Android 아키텍처 권장사항 - Android 개발자](https://developer.android.com/topic/architecture/recommendations?hl=ko)
* [상태 지향 아키텍쳐 MVI를 소개합니다](https://jaehochoe.medium.com/%EB%B2%88%EC%97%AD-%EC%83%81%ED%83%9C-%EC%A7%80%ED%96%A5-%EC%95%84%ED%82%A4%ED%85%8D%EC%B2%98-mvi%EB%A5%BC-%EC%86%8C%EA%B0%9C%ED%95%A9%EB%8B%88%EB%8B%A4-mvi-on-android-725cae5b1753)
