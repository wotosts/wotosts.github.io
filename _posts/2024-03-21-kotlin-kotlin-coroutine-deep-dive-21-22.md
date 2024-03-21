---
title: "[Kotlin] 스터디 Kotlin Coroutine: Deep Dive 21-22장"
toc: true
categories:
- Kotlin
tags:
- Flow
- coroutine
- study
- kotlin
---

<div class='notice'>
Kotlin coroutine: Deep Dive 책 스터디 정리 포스트<br>
21-22장(3부) 에 해당하는 내용입니다. <br>
</div>

---

## 21장. 플로우 만들기

필요한 경우에 따라서 플로우를 만드는 여러 방법이 있습니다.

**flowOf**
- 플로우를 만드는 가장 간단한 방법으로, 플로우가 가져야 할 값을 나열
- 값이 없는 플로우를 만들 경우 emptyFlow() 를 사용

```kotlin
// 1,2,3,4,5 를 가지는 플로우
flowOf(1, 2, 3, 4, 5).collect { /* consume */ }

// 값이 없는 플로우
emptyFlow<Int>().collect { /* consume */ }
```

<br>
**asFlow**

확장 함수로 정의된 asFlow를 통해 아래 타입들을 플로우로 바꿀 수 있습니다. 

- Iterable, Iterator, Sequence, Array
    
    ```kotlin
    // 확장함수. Iterator, Sequence도 동일
    public fun <T> Iterable<T>.asFlow(): Flow<T> = flow {
        forEach { value ->
            emit(value)
        }
    }
    
    listOf(1, 2, 3, 4)
    	// .setOf(...) 가능
    	// .sequenceOf(...) 가능
    	.asFlow()
    	.collect { /* consume */ }
    ```
    
- 함수
    - 함수를 플로우로 변환하면, 해당 플로우는 함수의 결과값 하나만 가짐
    - 일반적인 함수는 변환이 안되고, 람다식만 변환 가능
    
    ```kotlin
    // 확장함수
    public fun <T> (suspend () -> T).asFlow(): Flow<T> = flow {
        emit(invoke())
    }
    public fun <T> (() -> T).asFlow(): Flow<T> = flow {
        emit(invoke())
    }
    
    // 사용
    val function1 = suspend { /* something */ }
    val function2 = { /* something */ }
    
    function1.asFlow().collect { /* consume */ }
    function2.asFlow().collect { /* consume */ }
    ```
    
- Rx
    - 리액티브 스트림을 사용하고 있는 경우, Flux, Flowable, Observable 은 asFlow를 사용할 수 있도록 구현되어 있기 때문에 바로 변환하여 사용가능
    
    ```kotlin
    Observable.range(1, 5).asFlow().collect { /* consume */ }
    ```
    
<br>
**flow 빌더**

- 플로우를 만들 때 가장 많이 사용되는, 기본적인 방법
- flow 함수를 호출하고, 람다식 내에서 emit으로 값을 방출

```kotlin
val numbers = flow {
	repeat(10) {
		delay(1000)
		emit(it)
	}
}
```

### ChannelFlow

채널과 플로우를 합친 형태의 플로우입니다. 

플로우는 Cold 스트림이기 때문에 필요할 때에만 값을 생성하고, 방출된 값의 처리는 동기적으로 진행됩니다. 즉, 값이 방출된 이후 collect에서 처리가 완료되기 전까지 다음 값을 방출하지 않습니다.
채널플로우는 값의 방출과 처리를 동시에, 별개로 처리하고 싶을 때 사용합니다.

다음 코드는 Item 리스트를 페이징하고, 특정 아이템을 찾는 코드입니다.

```kotlin
// Item 클래스가 이미 정의되었다고 가정
const val PAGE_SIZE = 3

class ItemApi {
	val items = List(100) { Item(it) }
	
	suspend fun getItemsBy(page: Int): List<Item> {
		delay(1000)
		return items.drop(page * PAGE_SIZE).take(PAGE_SIZE)
	}
}

fun createItemFlow(itemApi: ItemApi) = flow {
	var page = 0
	do {
		println("Page $page")
		val items = itemApi.getItemsBy(page)
		emitAll(items.asFlow())
	} while(items.size >= PAGE_SIZE)
}

suspend fun main() {
	val items = createItemFlow(ItemApi())
	val item = items
			.flow {
				println("Check Item $it")
				delay(1000)
				it.`val` == 4
			}
	println("$item")
}
```
<br>
flow를 사용한 코드이기 때문에 createItemFlow에서 아이템을 생성하는 동작과, main에서 아이템을 찾는 동작이 동기적으로 진행됩니다.

```kotlin
// 출력
Page 0
// 1초 뒤
Check Item Item(0)
// 1초 뒤
Check Item Item(1)
// 1초 뒤
Check Item Item(2)
// 1초 뒤
Page 1
// 1초 뒤
Check Item Item(3)
// 1초 뒤
Check Item Item(4)
// 1초 뒤
Item(5)
```

<br>
createItemFlow의 반환값을 flow 대신 channelFlow로 변경하면, 페이지를 가져오는 동시에 값을 처리하는 것을 확인할 수 있습니다. 

```kotlin
fun createItemFlow(itemApi: ItemApi) = channelFlow {
	var page = 0
	do {
		println("Page $page")
		val items = itemApi.getItemsBy(page)
		items.forEach { send(it) }
	} while(items.size >= PAGE_SIZE)
}

// 출력
Page 0
// 1초 뒤
Check Item Item(0)
Page 1
// 1초 뒤
Check Item Item(1)
Page 2
// 1초 뒤
Check Item Item(2)
Page 3
// 1초 뒤
Check Item Item(3)
Page 4
// 1초 뒤
Check Item Item(4)
Page 5
// 1초 뒤
Item(5)
```
<br>
채널플로우의 특징입니다. 

- Flow 인터페이스를 구현하기 때문에 collect와 같은 최종 연산이 호출되면 시작됨
- 값의 처리완료를 기다리지 않고 다음 값을 생성
- ProducerScope가 SendChannel을 구현하고 있기 때문에 값의 방출은 emit 대신 `send`
- ProducerScope가 CoroutineScope를 구현하고 있기 때문에 launch 등의 코루틴 빌더 사용 가능

```kotlin
// Flow 인터페이스를 구현하고 있는 모습
public interface FusibleFlow<T> : Flow<T> { ... }

@InternalCoroutinesApi
public abstract class ChannelFlow<T>(
    ...
) : FusibleFlow<T> {
}

// ProducerScope를 사용하는 channelFlow
public fun <T> channelFlow(@BuilderInference block: suspend ProducerScope<T>.() -> Unit): Flow<T> =
    ChannelFlowBuilder(block)

public interface ProducerScope<in E> : CoroutineScope, SendChannel<E> {
    public val channel: SendChannel<E>
}
```

### CallbackFlow

클릭 등의 활동 변화를 감지하기 위한 이벤트 플로우가 필요할 때 사용합니다. 

채널 플로우와 거의 유사하지만, **콜백을 래핑하는 형태로 사용**하며 몇가지 유용한 함수들을 주로 사용합니다.

- awaitClose
    - 채널이 닫힐때까지 중단되며, 채널이 닫힌 후 인자로 전달된 함수 실행
    - callbackFlow 빌더에서 awaitClose 를 호출하지 않은 경우 빌더가 바로 종료됨
- trySendBlocking
    - send와 유사하지만 중단 대신 블로킹을 사용하기 때문에 일반 함수에서도 사용 가능
- close
    - 채널 종료
- cancel
    - 채널 종료하고 플로우에 취소 예외를 던짐

```kotlin
fun eventFlow(item: Item): Flow<T> = callbackFlow {
	val callback = Object: Callback {
		...
	}

	item.registerCallback(callback)
	awaitClose { item.unregisterCallback(callback) }
}
```

## 22장. 플로우 생명주기 함수

```kotlin
flow()
	.onStart { /* 플로우 시작 */ }
	.onCompletion { /* 플로우 종료 */ }
	.flowOn(Dispatcher.IO)
	.onEach { /* collect 연산을 여기에 작성하기도 함 */ }
	.catch { /* 예외 잡기 */ }
	.launchIn(viewModelScope)
```

**onEach**

- 플로우의 값을 하나씩 받을 때 사용
- 중단 함수이기 때문에 값은 하나씩, 순차적으로 처리

**onStart**

- 플로우가 시작될 때(첫번째 값 요청 시) 호출
- 값 방출 가능

**onCompletion**

- 플로우의 마지막 값이 전달되어 플로우가 완료되었을 때 호출

**onEmpty**

- 예외 상황 등으로 인해 값을 내보내기 전에 플로우가 완료되는 경우 호출

**catch**

- 플로우 중간에 예외가 발생한 경우, 예외를 잡고 플로우를 종료할 때 사용
- collect 에서 발생하는 예외는 잡을 수 없기 때문에, collect 연산을 onEach로 옮기고 catch를 사용하는 방법을 많이 사용
- 윗 부분에 있는 연산에서 발생한 예외만을 잡음

**flowOn**

- 윗 부분에 있는 연산들이 동작하는 코루틴 컨텍스트를 변경할 때 사용

**launchIn**

- 인자로 받은 코루틴 스코프에서 플로우를 수행할 때 사용
