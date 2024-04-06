---
title: "[Kotlin] 스터디 Kotlin Coroutine: Deep Dive 24장"
toc: true
categories:
- Kotlin
tags:
- study
- Flow
- coroutine
- kotlin
---

<div class='notice'>
Kotlin coroutine: Deep Dive 책 스터디 정리 포스트<br>
24장(3부) 에 해당하는 내용입니다. <br>  

25장은 테스트에 관련된 내용이므로 생략합니다.  
</div>

---

## 24장. 공유플로우와 상태플로우

**SharedFlow와 StateFlow는 여러 구독자가 하나의 데이터 변경을 감지하고자 할 때 사용합니다.** 

콜드 데이터 소스인 Flow는 요청 시에 값을 계산하지만, SharedFlow, StateFlow는 이미 계산된 값을 가지고 있기 때문에 핫 데이터 소스입니다.

### SharedFlow

여러 코루틴이 이벤트나 데이터 변경을 감지하는 경우에 사용하며, `MutableSharedFlow` 를 이용하여 값 전송이 가능한 SharedFlow를 만듭니다.

- emit을 통해 값을 보내면, 대기중인 구독자 모두에게 값이 전달됩니다.
- MutableSharedFlow를 생성할 때, replay 값을 설정하면 replay 값 만큼 이미 전송한 값을 저장합니다. 이후 새로운 구독자가 생기면 새 구독자에게 저장된 값을 전달합니다.
    
    ```kotlin
    val sharedFlow = MutableSharedFlow<String>(replay = 2)
    
    // 구독하는 곳. 종료되지 않는다.
    coroutineScope.launch {
    	sharedFlow.collect {
    		println("1. $it")
    	}
    }
    coroutineScope.launch {
    	sharedFlow.collect {
    		println("2. $it")
    	}
    }
    
    // 값을 보내는 곳
    coroutineScope.launch {
    	sharedFlow.emit("FirstData")
    }
    
    // 출력
    1. FirstData
    2. FirstData
    ```
    
- 주의해야 할 점은, SharedFlow를 구독하고 있는 코루틴은 종료되지 않는다는 점
- 주로 아래와 같이 감지와 변경의 책임을 나누어 작성합니다.
    
    ```kotlin
    private val _sharedFlow = MutableSharedFlow<String>()
    val sharedFlow: SharedFlow<String> = _sharedFlow
    // val sharedFlow = _sharedFlow.asSharedFlow()
    ```
    
<br>

#### shareIn

Flow는 구독자별로 각각 계산된 값을 방출합니다. 하나의 Flow를 여러 곳에서 구독하고 싶다면 shareIn을 통해 Flow를 SharedFlow로 변경할 수 있습니다.

```kotlin
val flow = flowOf(1,2,3).onEach { delay(1000) }

suspend fun main() = coroutineScope {
	// 이 시점에서 flow 가 값 계산을 시작함
	val sharedFlow = flow.shareIn(
		scope = coroutineScope,
		started = SharingStarted.Eagerly,
		// replay = 0
	)
	
	delay(500)
	launch { 
		sharedFlow.collect { println("1. $it") } 
	}
	
	delay(500)
	launch { 
		sharedFlow.collect { println("2. $it") } 
	}
}

// 출력
1. 1 // 1초 뒤
1. 2 // 1초 뒤
2. 2
1. 3 // 1초 뒤
2. 3
```

shareIn을 호출한 시점에서 flow의 값 계산이 시작됩니다. shareIn 함수가 SharedFlow를 생성하고, flow에서 방출된 값들을 SharedFlow로 전달합니다.

```kotlin
public fun <T> Flow<T>.shareIn(
    scope: CoroutineScope,
    started: SharingStarted,
    replay: Int = 0
): SharedFlow<T> {
    val config = configureSharing(replay)
    val shared = MutableSharedFlow<T>(
        replay = replay,
        extraBufferCapacity = config.extraBufferCapacity,
        onBufferOverflow = config.onBufferOverflow
    )
    @Suppress("UNCHECKED_CAST")
    val job = scope.launchSharing(config.context, config.upstream, shared, started, NO_VALUE as T)
    return ReadonlySharedFlow(shared, job)
}
```

shareIn이 받는 started 인자는 리스너의 수에 따라 감지 시작 타이밍을 결정합니다.

- `SharingStarted.Eagerly`: 즉시 감지
- `SharingStarted.Lazily`: 첫 구독자가 나타나면 감지 시작
- `WhileSubscribed()`: 구독자가 있을 때만 감지(첫 구독자 시작 ~ 마지막 구독자 종료), 플로우가 멈췄을 때 새 구독자가 나타나면 다시 시작
- 그 외 SharingStarted 커스텀 구현 가능

### StateFlow

상태를 나타낼 때 사용하며, replay = 1로 설정한 SharedFlow처럼 동작합니다. 

- 값 하나, 즉 현재 상태를 가지고 있으며
- value 프로퍼티를 통해 현재 상태에 접근이 가능합니다.

`MutableStateFlow`를 통해 값 변경이 가능한 StateFlow를 만들 수 있습니다. 

```kotlin
val stateFlow = MutableStateFlow("A") // 초기값 필수

launch {
	stateFlow.collect { println("$it") }
}
delay(1000)
stateFlow.value = "B"
delay(1000)
stateFlow.value = "C"

// 출력
A
B // 1초 뒤
C // 1초 뒤
```

값이 덮어 씌워지는 형태이기 때문에, 관찰이 늦는 경우 중간 상태를 받지 못하는 경우가 발생할 수 있습니다. 

```kotlin
fun main() = runBlocking {
    val state = MutableStateFlow('X')

    launch {
        state.collect { 
            delay(1000)
            println("$it")
        }
    }

    launch {
        for(c in 'A'..'E') {
            delay(500)
            state.value = c
        }
    }

    return@runBlocking
}

// 출력
X
A // 0.5초 후
C // 0.5초 후
E // 0.5초 후
```

<br>

#### stateIn

shareIn과 마찬가지로 Flow를 StateFlow로 변환하며, 2가지 형태를 가집니다.

- scope만 전달하는 경우
    - 중단함수이며, 초기값 계산이 완료될 때까지 중단됩니다.
    
    ```kotlin
    val flow = flowOf(1, 2, 3)
        .onEach { delay(1000) }
        .onEach { println("만들어졌다 $it")}
    
    fun main() = runBlocking {
        val stateFlow = flow.stateIn(
            scope = this
        )
    
        println("stateFlow ${stateFlow.value}")
        stateFlow.collect(FlowCollector {
            print("$it")
        })
    
        return@runBlocking
    }
    
    // 출력
    만들어졌다 1 // 1초 뒤
    stateFlow 1
    1
    만들어졌다 2 // 1초 뒤
    2
    만들어졌다 3 // 1초 뒤
    3
    ```
    
- scope, started, initialValue를 전달하는 경우
    - 중단 함수가 아니기 때문에 초기값을 전달해야 합니다.
    
    ```kotlin
    val flow = flowOf(1, 2, 3)
        .onEach { delay(1000) }
        .onEach { println("만들어졌다 $it")}
    
    fun main() = runBlocking {
        val stateFlow = flow.stateIn(
            scope = this,
            started = SharingStarted.Lazily,
            initialValue = 0
        )
    
        println("stateFlow ${stateFlow.value}")
        stateFlow.collect(FlowCollector {
            print("$it")
        })
    
        return@runBlocking
    }
    
    // 출력
    stateFlow 0
    만들어졌다 1 // 1초 뒤
    1
    만들어졌다 2 // 1초 뒤
    2
    만들어졌다 3 // 1초 뒤
    3
    ```
