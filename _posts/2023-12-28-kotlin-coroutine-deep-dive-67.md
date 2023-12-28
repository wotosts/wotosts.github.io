---
title: "[Kotlin] 스터디 Kotlin coroutine: Deep Dive 6~8장"
tags:
- kotlin
- coroutine
- study
categories:
- Kotlin
toc: true
---

<div class='notice'>
Kotlin coroutine: Deep Dive 책 스터디 정리 포스트<br>
6~8장(2부) 에 해당하는 내용입니다. <br>
</div>


## 6장. 코루틴 빌더

일반 함수에서 중단 함수를 호출이 불가능한 이유?

→ 앞서 4장에서 살펴보았듯, 중단 함수는 다른 중단 함수로 Coninuation을 전달합니다. 중단 함수의 시작도 Continuation이 전달되어야 하기 때문에 **일반 함수에서는 중단 함수를 호출할 수 없습니다.**


suspend 호출 불가|suspend
-|-
![]({{ 'assets/images/2023-12-28/suspend_func1.png' | relative_url }}) | ![]({{ 'assets/images/2023-12-28/suspend_func2.png' | relative_url }})

중단 함수의 최초 시작점은 일반 함수일텐데, 이 때에는 `코루틴 빌더`를 통하여 일반 함수 내에서 중단 함수를 호출할 수 있습니다.

<center>
<img src="/assets/images/2023-12-28/coroutine_builder.png"/>
</center>

### 필수적인 코루틴 빌더

- launch
    - 각각 동작하는 개별 코루틴 생성
    
    ```kotlin
    fun main() = runBlocking {
    	GlobalScope.launch {
    		delay(1000L)
    		println("출력 1")
    	}
    	GlobalScope.launch {
    		delay(1000L)
    		println("출력 2")
    	}
    	println("출력 3")
    	delay(2000L)
    }
    
    // 출력
    출력 3
    (1초 후)
    출력 1
    출력 2
    ```
    
- async
    - 리턴값이 있는 빌더 - Deferred<T>
    - launch와 유사하게 바로 동작이 수행되며 await로 값을 가져올 수 있음
    
    ```kotlin
    fun main() = runBlocking {
    	var res1 = GlobalScope.async {
    		delay(1000L)
    		"출력 1"
    	}
    	var res2 = GlobalScope.launch {
    		delay(3000L)
    		"출력 2"
    	}
    	var res3 = GlobalScope.launch {
    		delay(2000L)
    		"출력 3"
    	}
    	println(res1)
    	println(res2)
    	println(res3)
    }
    
    // 출력
    (1초 후)
    출력 1
    (2초 후 - main 시작 후 3초)
    출력 2
    출력 3
    ```
    
- runBlocking
    - 코루틴을 시작한 스레드를 블로킹하는 빌더
    - 테스트 등 특별한 경우에만 사용
    
    ```kotlin
    fun main() {
    	runBlocking {
    		delay(1000L)
    		println("출력 1")
    	}
    	runBlocking {
    		delay(1000L)
    		println("출력 2")
    	}
    	println("출력 3")
    }
    
    // 출력
    출력 1
    (1초 후)
    출력 2
    (1초 후)
    출력 3
    ```
    

### 구조화된 동시성

코루틴 사이의 부모-자식 관계이며, 아래 특징을 가집니다.

- 자식 코루틴은 부모 코루틴의 컨텍스트를 상속
- 부모 코루틴은 자식 코루틴이 끝날 때까지 기다림
- 부모 코루틴이 최소되면 자식 코루틴도 모두 취소됨
- 자식 코루틴에서 예외 발생 시 부모 코루틴에 전달되어 같이 소멸함

아래 코드에서는 mainCoroutine과 coroutine1 사이에 구조화된 동시성이 성립하지 않습니다. 

때문에 mainCoroutine은 coroutine1이 끝날 때까지 기다리지 않고 종료됩니다. 따라서 ‘출력 1’이 출력되려면 delay(2000L)이 필요합니다.

```kotlin
// mainCoroutine
fun main() = runBlocking {
	// coroutine1
	GlobalScope.launch {
		delay(1000L)
		println("출력 1")
	}
	println("출력 2")
	delay(2000L)
}

// 출력
출력 2
(1초 후)
출력 1
```

아래 코드에서는 runBlocking 의 CoroutineScope를 이용하여 coroutine1을 생성했습니다. 

때문에 mainCoroutine과 coroutine1 사이에 구조화된 동시성이 성립되어 delay가 없이도 mainCoroutine이 coroutine1이 끝날 때까지 기다립니다.

```kotlin
// mainCoroutine
fun main() = runBlocking {
	// coroutine1
	launch {
		delay(1000L)
		println("출력 1")
	}
	println("출력 2")
}

// 출력
출력 2
(1초 후)
출력 1
```

## 7장. 코루틴 컨텍스트

> 코루틴과 관련된 데이터를 저장하고 전달하는 방법입니다. 
코루틴 내에 저장되어 코루틴의 상태, 어떤 스레드에서 동작할지 등의 작동 방식을 정할 수 있습니다.
> 

CoroutineContext는 원소나 원소들의 집합으로 컬렉션과 유사하며, Key를 통해 원소를 구분합니다.

머그컵에 머그컵을 더한다? ☕ + ☕ ? 이 개념을 이해하는 것이 사실 조금.. 어려웠습니다. plus의 구현과 CombinedContext를 살펴보면 이해가 됩니다. 

```kotlin
@SinceKotlin("1.3")
public interface CoroutineContext {

    public operator fun <E : Element> get(key: Key<E>): E?

    public fun <R> fold(initial: R, operation: (R, Element) -> R): R

    public operator fun plus(context: CoroutineContext): CoroutineContext =
        if (context === EmptyCoroutineContext) this else // fast path -- avoid lambda creation
            context.fold(this) { acc, element ->
                val removed = acc.minusKey(element.key)
                if (removed === EmptyCoroutineContext) element else {
                    // make sure interceptor is always last in the context (and thus is fast to get when present)
                    val interceptor = removed[ContinuationInterceptor]
                    if (interceptor == null) CombinedContext(removed, element) else {
                        val left = removed.minusKey(ContinuationInterceptor)
                        if (left === EmptyCoroutineContext) CombinedContext(element, interceptor) else
                            CombinedContext(CombinedContext(left, element), interceptor)
                    }
                }
            }

    public fun minusKey(key: Key<*>): CoroutineContext

    public interface Key<E : Element>

    public interface Element : CoroutineContext {

        public val key: Key<*>

        public override operator fun <E : Element> get(key: Key<E>): E? =
            @Suppress("UNCHECKED_CAST")
            if (this.key == key) this as E else null

        public override fun <R> fold(initial: R, operation: (R, Element) -> R): R =
            operation(initial, this)

        public override fun minusKey(key: Key<*>): CoroutineContext =
            if (this.key == key) EmptyCoroutineContext else this
    }
}
```

CoroutineContext 내부에 `Element`라는, CoroutineContext의 함수들 get, fold, minusKey를 구현한 인터페이스가 있습니다. 

이 Element를 CombinedContext가 확장하고 있는데, CombinedContext의 일부 구현을 보면 **left 노드를 가지는 리스트 형태**임을 알 수 있습니다.

Element 또한 CoroutineContext이기 때문에, CoroutineContext는 그 자체로 컬렉션이 될 수 있습니다.

<center>
<img src='/assets/images/2023-12-28/coroutine_context.png'/>
</center>
<br>

```kotlin
// this class is not exposed, but is hidden inside implementations
// this is a left-biased list, so that `plus` works naturally
@SinceKotlin("1.3")
internal class CombinedContext(
    private val left: CoroutineContext,
    private val element: Element
) : CoroutineContext, Serializable {

    override fun <E : Element> get(key: Key<E>): E? {
        var cur = this
        while (true) {
            cur.element[key]?.let { return it }
            val next = cur.left
            if (next is CombinedContext) {
                cur = next
            } else {
                return next[key]
            }
        }
    }

		...

    private fun containsAll(context: CombinedContext): Boolean {
        var cur = context
        while (true) {
            if (!contains(cur.element)) return false
            val next = cur.left
            if (next is CombinedContext) {
                cur = next
            } else {
                return contains(next as Element)
            }
        }
    }

		...
}
```

## 8장. 잡과 자식 코루틴 기다리기

### Job

주요 CoroutineContext 중 하나입니다.

코루틴 빌더로 코루틴 생성 시 코루틴 자신만의 Job을 반환합니다. 

- launch - Job
- async - Deferred<T>: Job
    
    ```kotlin
    public interface Deferred<out T> : Job
    ```
    

### Job의 부모-자식 관계

빌더에 Job을 전달하여 코루틴을 생성하면, 생성된 코루틴이 반환한 Job은 전달된 Job의 자식이 됩니다. 

```kotlin
runBlocking {
        val job = Job()
	// coroutine1
        launch(job) {
            val childJob = coroutineContext[Job]
            println("Job is paased: " + (childJob == job.children.firstOrNull()))
        }

	// coroutine2
        launch {
            val childJob = coroutineContext[Job]
            println("no Job is paased: " + (childJob == job.children.firstOrNull()))
        }
 }

// 출력
Job is paased: true
no Job is paased: false
```

위 코드에서 coroutine1과 coroutine2는 runBlocking의 자식 코루틴처럼 보이지만,
실제로는 coroutine2만 자식이고, coroutine1은 자식이 아닙니다. 

새로운 Job이 부모 컨텍스트의 Job을 대체하였기 때문입니다.

```kotlin
runBlocking {
        val job = Job()
        val childJob1 = launch(job) { .. }
        val childJob2 = launch { .. }

        println((childJob1 == coroutineContext[Job]?.children?.firstOrNull()))
        println((childJob2 == coroutineContext[Job]?.children?.firstOrNull()))
}

// 출력
false
true
```
<br>

이 외에, Job은 아래와 같은 특징을 가집니다.  
- 자식 Job ↔ 부모 Job 참조 가능  
- join()을 이용하여 현재 Job 완료 대기 가능
    - 따라서, 자식 Job이 완료되는 것을 기다리려면 자식 Job의 join을 호출해야 합니다.

### 수명주기

Job은 상태로 나타낼 수 있는 수명을 가지고 있고, 취소가 가능합니다. 

![]({{ 'assets/images/2023-12-28/coroutine_job.png' | relative_url }})

- `New`: 지연된 코루틴의 시작 상태(CoroutineStart.LAZY)
- `Active`: 일반적인 코루틴의 시작 상태. 코루틴 실행
- `(Completing)`: 코루틴 실행이 완료되어 종료되기 전, 자식 코루틴의 종료를 기다리는 상태
- `Completed`: 실행 완료된 최종 상태
- `Canceling`: 코루틴이 취소되어 자원 반납 등 후처리가 가능한 상태
- `Cancelled`: Cancelling에서 후처리가 완료된 상태

이들 상태는 isActive, isCompleted, isCancelled 의 값을 통해 알 수 있습니다.
