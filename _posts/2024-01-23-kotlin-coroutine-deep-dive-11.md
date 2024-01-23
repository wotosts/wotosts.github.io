---
title: "[Kotlin] 스터디 Kotlin coroutine: Deep Dive 11장"
tags:
- study
- kotlin
- coroutine
categories:
- Kotlin
toc: true
---

<div class='notice'>
Kotlin coroutine: Deep Dive 책 스터디 정리 포스트<br>
11장(2부) 에 해당하는 내용입니다. <br>
</div>

---
## 11장. 코루틴 스코프 함수

코루틴 스코프가 필요한 상황을 살펴봅시다. 

```kotlin
suspend fun getList1() {
	// do. 약 1초 소요된다고 가정
}

suspend fun getList2() {
	// do. 약 2초 소요된다고 가정
}

suspend fun getMergedList() {
	val list1 = getList1()
	val list2 = getList2()

	return list1 + list2
}
```

위 코드는 list1의 동작 후 list2가 동작하여 총 3초가 필요합니다.
list1과 list2가 동시에 실행되도록 async를 사용하려면, 코루틴 스코프가 필요합니다.

좋지 않은 방법들

- GlobalScope.async 사용
    - GlobalScope을 사용하면 함수를 호출한 코루틴과의 관계가 사라져 취소 불가능
    - 부모 컨텍스트 상속 X
    - 단위 테스트가 힘듦
- getMergedList 함수의 인자로 scope 넘기기
    - 예상치 못한 부작용 발생 가능성


<br>

### coroutineScope

```kotlin
suspend fun <R> coroutineScope(
	block: suspend CoroutineScope.() -> R
): R
```

- 스코프를 시작하는 중단함수


- 새로운 코루틴을 생성하는 대신, **새로운 코루틴이 동작을 완료할 때까지 호출한 코루틴 중단**
	- 아래 코드에서, coroutineScope로 생성한 두 코루틴이 순차적으로 실행됩니다.
    a가 실행될 때 test를 호출한 코루틴이 중단되고, a의 실행이 완료되어야 b가 실행됩니다.
    
    ```kotlin
    suspend fun test() {
    	val a = coroutineScope {
    		delay(1000)
    		println("coroutine a")
    	}
    	val b = coroutineScope {
    		delay(1000)
    		println("coroutine b")
    	}
    }
    
    // 출력
    (1초 뒤)
    a
    (1초 뒤)
    b
    ```
    
	<br>
		
- coroutineScope 는 부모의 책임을 이어받음  
   - 아래 코드에서, coroutineScope는 내부의 자식 코루틴이 모두 완료될 때까지 종료되지 않습니다.
    
    ```kotlin
    suspend fun test() = coroutineScope {
    	launch {
    		delay(1000)
    		println("coroutine a")
    	}
    	launch {
    		delay(2000)
    		println("coroutine b")
    	}
    }
    
    // 출력
    (1초 뒤)
    a
    (1초 뒤)
    b
    ```
    
<br>

이제 가장 처음에 보았던 코드에서  
suspend 함수의 getList1(), getList2() 를 coroutineScope, async를 사용하여 병렬 호출로 변경할 수 있습니다.

```kotlin
suspend fun getMergedList() {
	val list1 = getList1()
	val list2 = getList2()

	return list1 + list2
}

suspend fun getMergedList() = coroutineScope {
	val list1 = async { getList1() }
	val list2 = async { getList2() }

	list1.await() + list2.await()
}
```

<br>


### 코루틴 빌더 vs 코루틴 스코프

|  | 코루틴 빌더 | 코루틴 스코프 |
| --- | --- | --- |
| 종류 | launch, async 등 | coroutineScope, supervisorScope 등 |
| 형태 | CoroutineScope 확장 함수 | 중단 함수  |
| 사용하는 Context | CoroutineScope 리시버에 전달된 Context | Continuation 객체의 Context |
| 예외 전파 | Job을 통해 부모로 예외 전파 <br>(예외 핸들러 별도로 사용) | 일반적인 예외 throw<br>(try-catch 로 예외 처리 가능) |
| 시작 위치 | 호출 시 비동기 코루틴 시작 | 코루틴 빌더가 호출된 곳에서 코루틴을 시작하며, <br> 호출한 코루틴 중단 |

### 코루틴 스코프 함수의 종류

- `coroutineScope`
- `withContext`
    - 코루틴 스코프의 Context를 변경할 수 있음
    - 기존 스코프의 Context와 다른 Context를 사용하고자 할 때 (주로 Dispatcher와 함께)
- `supervisorScope`
    - SupervisorJob과 마찬가지로 한 자식에서 발생한 에러가 다른 자식의 동작에 영향을 미치지 않음
- `withTimeout`
    - 인자로 들어온 람다 수행에 시간 제약을 걸어줌
    - 시간이 오래 걸리면 TimeoutCancellationException

또한, 코루틴 스코프 함수는 원하는 동작을 위해 아래와 같이 연결하여 사용할 수 있습니다.

```kotlin
withContext(Dispatchers.IO) {
	withTimeoutOrNull(10000) {
		// IO 스레드에서 동작하고, 타임아웃 10초가 필요한 경우
	}
}
```
