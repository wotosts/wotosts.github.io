---
title: "[Kotlin] 스터디 Kotlin Coroutine: Deep Dive 12장"
toc: true
tags:
- coroutine
- kotlin
- study
categories:
- Kotlin
---

<div class='notice'>
Kotlin coroutine: Deep Dive 책 스터디 정리 포스트<br>
12장(2부) 에 해당하는 내용입니다. <br>
</div>

---

## 12장. 디스패처

코루틴이 실행할 스레드(스레드풀)를 결정합니다.

Dispatchers의 코드를 따라가보면, 디스패처 또한 CoroutineContext 임을 확인할 수 있습니다. 

```kotlin
public abstract class CoroutineDispatcher :
    AbstractCoroutineContextElement(ContinuationInterceptor), ContinuationInterceptor {
```

디스패처 설정은 withContext를 이용합니다.  
withContext를 호출하면 코루틴이 중단되고 큐에서 대기하다가 재개되기 때문에 비용이 듭니다.

``` kotlin
viewModelScope.launch {
	withContext(Dispatchers.IO) { }
}
```

<br>

### 디스패처의 종류

- `Main`
    - UI 와 상호작용하는 스레드를 사용하는 디스패처
    - `Main.immediate`
        - 코드가 이미 Main 디스패처에서 호출된 경우 스레드 배정 없이 코드 즉시 실행
- `Unconfined`
    - **스레드 스위칭이 일어나지 않는 디스패처**
    - 동작할 스레드를 별도로 지정해두지 않은 디스패처이기 때문에, 코루틴이 실행, 재개된 스레드에서 계속 작업을 이어감
    - 모든 작업을 Unconfined에서 실행하는 경우, 연산이 모두 같은 스레드에서 실행됨
    - 블로킹이 일어나는 코드에서 사용 주의!

실제로 Unconfined가 스레드를 변경하지 않는지 확인해봅시다.
디스패처는 다르지만 같은 main 스레드에서 동작하는 것을 확인할 수 있습니다. 

```kotlin
viewModelScope.launch(Dispatchers.Main) {
	val a = coroutineContext[ContinuationInterceptor] as CoroutineDispatcher
	Log.d("coroutine", a.toString())
	Log.d("coroutine", Thread.currentThread().name)
	
	withContext(Dispatchers.Unconfined) {
		val b = coroutineContext[ContinuationInterceptor] as CoroutineDispatcher
		Log.d("coroutine", b.toString())
		Log.d("coroutine", Thread.currentThread().name)
	}
}

// 출력
Dispatchers.Main
main
Dispatchers.Unconfined
main
```


- `Default`
    - Kotlin 기본 디스패처
    - CPU 집약적 연산 수행
    - 스레드 풀 내 스레드 수는 일반적으로 **CPU 수**
- `IO`
    - 파일을 읽고 쓰거나 스레드를 블로킹할 때 사용하는 디스패처
    - 스레드 풀 내 스레드 수는 일반적으로 **코어 수**

**Default, IO는 (스레드가 무한한) 같은 스레드 풀을 공유합니다.** 

<center>
<img src= '/assets/images/2024-01-25/coroutine_dispatcher1.drawio.png'/>
</center>
<br>

실제로 아래 코드를 실행해보면, 둘 다 같은 스레드에서 실행됨을 확인할 수 있습니다. (대부분의 경우)

```kotlin
fun main() = runBlocking(CoroutineName("parent")) {
    withContext(Dispatchers.Default) {
        println(Thread.currentThread().name)
        withContext(Dispatchers.IO) {
            println(Thread.currentThread().name)
        }
    }
}

// 출력
DefaultDispatcher-worker-1
DefaultDispatcher-worker-1
```

Default와 IO 디스패처는 같은 스레드 풀을 공유하지만, 스레드 수 제한은 각각 적용되므로 다른 디스패처의 스레드를 고갈시키지는 않습니다. 

### limitedParallelism

`limitedParallelism` 을 통해 같은 시간에 사용되는 스레드 수를 제한할 수 있습니다. 

```kotlin
private val dispatcher = Dispatchers.Default.limitedParallelism(5)
```

Default와 IO에 사용할 때의 결과가 다릅니다. 

- Dispatchers.Default 에 사용
    - 디스패처에 스레드 수 제한 추가
- Dispatchers.IO 에 사용
    - 새로운 스레드 풀을 가진 디스패처 생성
    - 스레드를 자주 블로킹하는 클래스에서 활용

<center>
<img src = '/assets/images/2024-01-25/coroutine_dispatcher2.drawio.png' />
</center>
<br>

limitedParallelism(1)을 이용하여 스레드 수를 1로 제한하여, 공유 상태 문제를 해결할 수도 있습니다. 

### 성능 정리

- 블로킹 작업의 경우, 스레드 수가 많을수록 모든 코루틴이 종료되는 시간이 빨라집니다.
	- 블로킹 작업이 100개라고 가정하면, 
    - 스레드가 60일 때, 60개 먼저 작업 후 나머지 40개가 동작
    - 스레드가 100개일 때, 100개 모두 같이 동작
- CPU 집약적 연산의 경우 Dispatchers.Default가 가장 좋습니다.
    - 스레드가 많을수록 스레드 스위칭 시간이 늘어나기 때문에 더 많은 시간이 필요함
    - Dispatchers.IO는 사용하지 않는 것이 좋음
        - 블로킹 용도이기 때문에, 전체 스레드 블록 가능성이 있음
- 메모리 집약적 연산(대부분 시간이 메모리 접근, 할당 등에 사용되는 작업)은 더 많은 스레드를 사용하는 것이 좋습니다.
