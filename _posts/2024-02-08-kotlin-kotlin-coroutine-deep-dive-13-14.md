---
title: "[Kotlin] 스터디 Kotlin Coroutine: Deep Dive 13-14장"
categories:
- Kotlin
tags:
- coroutine
- kotlin
- study
toc: true
---

<div class='notice'>
Kotlin coroutine: Deep Dive 책 스터디 정리 포스트<br>
13-14장(2부) 에 해당하는 내용입니다. <br>
</div>

---

## 13장. 코루틴 스코프 만들기

```kotlin
interface CoroutineScope {
	val coroutineContext: CoroutineContext
}
```

CoroutineScope 는 CoroutineContext를 가진 인터페이스로, 이를 구현하여 코루틴 스코프를 만들 수 있습니다.

CoroutineScope을 사용하고자 하는 클래스에서 아래와 같이 직접 구현할 수도 있지만, 
cancel 등의 함수를 호출하면 더 이상 코루틴을 시작할 수 없다는 문제가 발생하기 때문에, 직접 구현보다는 클래스 내에 CoroutineScope 객체를 가지도록 개발하는 방법을 이용합니다. 

```kotlin
class Test: CoroutineScope {
	override val coroutineContext: CoroutineContext = Job()

	// ...
}
```

### CoroutineScope 팩토리 함수
context를 인자로 받는 아래 **CoroutineScope 팩토리 함수**를 이용하면 쉽게 코루틴 스코프 생성이 가능합니다.

```kotlin
fun CoroutineScope(
   context: CoroutineContext
 ): CoroutineScope =
   ContextScope(
    if(context[Job] != null) context
    else context + Job()
   )

internal class ContextScope(
   context: CoroutineContext
  ): CoroutineScope {
      // ...
}
```


<br>

## 14장. 공유 상태로 인한 문제
### 동기화의 필요성

서로 다른 스레드에서 동작하는 코루틴들이 동시에 접근하여 상태를 조작하는 경우 `공유 상태 문제`가 발생할 수 있습니다. 

간단한 예시인 카운팅을 살펴보면

```kotlin
var count = 0

suspend fun increaseCount() {
   withContext(Dispatchers.Default) {
      repeat(1000) {
         launch {
            repeat(1000) { count ++ }
         }
      }
   }
}
```

1. increaseCount 내에서 1000번 launch
2. launch 로 생성된 코루틴 내에서 1000번 count 증가

위 코드에서 원하는 결과는 count = 10000000 입니다.   
하지만 서로 다른 스레드에서 동시에 count를 증가시키게 되면 
1. 스레드 A 에서 count = 0 접근
2. 스레드 B 에서 count = 0 접근, count = 1로 증가
3. 스레드 A 에서 이미 가져온 count = 0 을 1로 증가시키고, count 값을 반영함

이 경우, 최종 count는 1000000 보다 작은 값이 됩니다. 

때문에, 공유 상태에 대한 동기화가 필요합니다.

<br>

### 동기화 방법

#### syncronized
    
  - 값이 변경되는 부분을 syncronized 블록으로 감싸주는 방법
   - syncronized 의 인자로 들어가는 lock을 가진 객체만 블록 내부에 접근이 가능
    
```kotlin
suspend fun increaseCount() {
   withContext(Dispatchers.Default) {
      repeat(1000) {
         launch {
            repeat(1000) { 
               syncronized(lock) {
                  count ++ 
               }
            }
         }
      }
   }
}
 ```
    
   - 문제점
      - syncronized는 스레드를 블로킹한다는 것 → 자원의 낭비
      - syncronized 블록 내에서 중단 함수 사용 불가

자바의 전통적인 방식인 syncronized를 사용하여 문제를 해결할 수는 있지만, 좀 더 코루틴에 맞는 **블로킹없이 중단하거나 충돌을 피하는 방법**을 사용해야 합니다.

 
<br>

#### Atomic
   - AtomicInteger, AtomicBoolean 등 값이 간단한 경우
   - 스레드를 블로킹하지 않는 방식
   - 멀티 스레딩 환경에서는 성능향상을 위해 CPU 캐시 값을 보게 되는데, Atomic은 **캐시 값과 실제로 메모리에 저장된 값을 동기화**
   - CAS(Compare And Swap) 알고리즘
       - 기존 값(Compared Value)과 변경할 값(Exchanged Value)을 전달하고,
       - 기존 값(Compared Value)이 현재 메모리가 가지고 있는 값(Destination)과 같다면 값 변경
    
```kotlin
var count = AtomicInteger()
    
suspend fun increaseCount() {
   withContext(Dispatchers.Default) {
      repeat(1000) {
         launch {
            repeat(1000) { counter.incrementAndGet() }
         }
      }
   }
}
```
    
   - 문제점
       - 여러 변수를 공유하는 경우에는 적절하지 않음


#### Single Thread Dispatcher
   - limitedParallelism을 이용해 싱글 스레드를 사용하는 디스패처 생성
   - 상태를 변경하는 구문들을 withContext(dispatcher)로 래핑하여 하나의 스레드만 값에 접근하도록 제한


```kotlin
val dispatcher = Dispatcher.IO.limitedParallelism(1)
var count = 0
    
suspend fun increaseCount() {
   withContext(Dispatchers.Default) {
      repeat(1000) {
         launch {
            repeat(1000) { 
               withContext(dispatcher) { count ++ }
            }
         }
      }
   }
}
```
    
#### Mutex
   - lock을 가진 코루틴만 크리티컬 섹션에 접근할 수 있고, 나머지 코루틴은 순차적으로 큐에 들어감
   - syncronized와 달리 스레드를 블로킹하지 않고 **코루틴을 중단 시키는 방식**  
     - lock, unlock을 직접 호출할 수 있지만, 안전하게 withLock을 사용하자
    
```kotlin
val mutex = Mutex()
var count = 0
    
suspend fun increaseCount() {
   withContext(Dispatchers.Default) {
      repeat(1000) {
         launch {
            repeat(1000) { 
               mutex.withLock { count ++ }
            }
         }
      }
   }
}
		
// 구현되어 있는 Mutex
public suspend inline fun <T> Mutex.withLock(owner: Any? = null, action: () -> T): T {
   contract {
      callsInPlace(action, InvocationKind.EXACTLY_ONCE)
   }
    
   lock(owner)
   try {
      return action()
   } finally {
      unlock(owner)
   }
}
```
    
   - 단점
       - lock을 두번 사용하면 withLock { withLock {} } 교착 상태에 진입
       - 코루틴이 중단된 경우 lock을 해제할 수 없음
            
           → lock이 필요한 크리티컬 섹션 내에서는 중단 함수 사용을 피하기 
            
   - Semaphore
       - Mutex가 한번에 하나의 코루틴만 크리티컬 섹션에 접근 가능하다면, Semaphore는 여러 코루틴이 동시 접근 가능
       - **실제로 공유 상태 문제를 해결하는 방법은 아님**
       - Semaphore(1) == Mutex


#### Actor
   - 상태 접근은 단일 스레드로 제한하고, 다른 코루틴에서는 Channel을 통해 상태 변경을 요청
    
```kotlin
val c = actor {
   for (msg in channel) {
      when(msg) {
         Increase -> count ++
      }
   }
}
    
// send messages to the actor
c.send(...)
...
// stop the actor when it is no longer needed
c.close()
```
