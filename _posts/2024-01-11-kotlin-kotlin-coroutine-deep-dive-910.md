---
title: "[Kotlin] 스터디 Kotlin coroutine: Deep Dive 9~10장"
tags:
- kotlin
- study
- coroutine
toc: true
categories:
- Kotlin
---

<div class='notice'>
Kotlin coroutine: Deep Dive 책 스터디 정리 포스트<br>
9~10장(2부) 에 해당하는 내용입니다. <br>
</div>

## 9장. 취소

### cancel()과 join()

Job에 포함된 `cancel()` 함수로 코루틴 취소가 가능합니다.

cancel() 호출 시, 

- 취소 명령 이후 첫 중단점에서 `CancellationException`을 던지며 취소됩니다.
    - **중단점이 없는 경우에는 취소되지 않습니다.**
        - 중단점을 추가하거나, Job의 상태를 관찰하여 inActive 상태일 때에만 취소 가능
- 자식 코루틴이 있는 경우 같이 취소합니다. 부모 취소 X
- 취소된 코루틴은 내부에서 새로운 코루틴을 실행할 수 없습니다.

일반적으로 cancel() 호출 후 join()을 호출하여 취소 과정의 완료를 기다립니다.

cancel()과 join()을 개별 호출하여 사용해도 되지만, `cancelAndJoin()` 확장함수를 사용해도 좋습니다.

- join()을 호출하지 않은 경우
    
    ```kotlin
    suspend fun test() = coroutineScope {
    	val job = Job()
    	launch(job) {
    		repeat(1000) { 
    			delay(200)
    			println("$it)
    			// 아주 오래 걸리는 작업
    		}
    	}
    
    	delay(2000)
    	job.cancel()
    	println("cancel")
    }
    
    // 출력
    0
    1
    2
    cancel  // 코루틴 동작 종료를 기다리지 않는다!
    3
    ```
    

- join()을 호출한 경우
    
    ```kotlin
    suspend fun test() = coroutineScope {
    	val job = Job()
    	launch(job) {
    		repeat(1000) { 
    			delay(200)
    			println("$it)
    			// 아주 오래 걸리는 작업
    		}
    	}
    
    	delay(2000)
    	job.cancel()
			job.join()
    	println("cancel")
    }
    
    // 출력
    0
    1
    2
    3
    cancel  // 동작 종료까지 join에서 대기 후 출력
    ```
    

### 자원 정리

코루틴을 취소한 후 자원 정리가 필요할 때에는 아래의 방법들을 이용할 수 있습니다.

- try-catch-finally의 finally 블록
    - 코루틴이 취소될 때 예외를 던지기 때문에 가능한 방법
    - 자원 정리가 끝날 때까지 코루틴은 계속 실행될 수 있습니다.
    
    ```kotlin
    suspend fun test() = coroutineScope {
    	val job = Job()
    	launch(job) {
    		try {
    			// do
    		} catch(e: CancellationException) {
    			// 취소시 exception을 그냥 던지는게 좋다.
    		} finally {
    			// 자원 정리하기
    		}
    	}
    }
    ```
    
- Job.invokeOnCompletion
    - Job의 상태가 마지막에 도달했을 때 호출하는 핸들러
    - 코루틴이 종료될 때 발생한 예외를 전달합니다.
    
    ```kotlin
    suspend fun test() = coroutineScope {
    	val job = launch {
    		// do
    	}.invokeOnCompletion { e ->
    		// do
    	}
    
    	delay(2000)
    	job.cancelAndJoin()
    }
    ```
    
- suspendCancellableCoroutine
    - suspendCoroutine과 비슷하게 코루틴을 중단하고, 취소가 가능한 Continuation을 사용
    - 아래와 같이 invokeOnCompletion을 이용
    
    ```kotlin
    suspend fun test() = suspendCancellableCoroutine { cont ->
    	cont.invokeOnCompletion { e ->
    		// do
    	}
    	
    	// 작업~
    }
    ```
    

## 10장. 예외 처리

코루틴에서 예외 발생 시

1. 예외는 부모 코루틴에 전파되어 부모 코루틴이 취소되고
2. 부모 코루틴은 다른 자식 코루틴을 취소합니다.

자식 코루틴에서 전파되는 예외로 전체 그룹의 코루틴이 취소되는 것입니다.

<div align="center">
<img src= '/assets/images/2024-01-11/coroutine_exception.png' />
</div>

**단, 예외가 CancellationException 인 경우에는 부모에게 예외가 전파되지 않기 때문에 자기 자신 및 자식 코루틴만 취소됩니다.**

### 예외 무시하기

자식 코루틴에서 발생한 예외를 무시하게 되면, 예외가 발생한 자식 코루틴만 종료됩니다. 
SupervisorJob, supervisorScope 을 이용하여 자식에서 발생한 예외를 무시할 수 있습니다.

<div align="center">
<img src='/assets/images/2024-01-11/coroutine_supervisor.drawio.png'/>
</div>

- `SupervisorJob`
    
    ```kotlin
    fun main() = runBlocking {
    	val job = SupervisorJob()
    
    	launch(job) {
    		delay(500)
    		throw Exception("Error~")
    	}
    
    	launch(job) {
    		delay(1000)
    		println("Done")
    	}
    
    	job.join()
    }
    
    // 출력
    Error~
    Done	
    ```
    
- `supervisorScope`
    - SupervisorJob으로 CoroutineScope 생성
    
    ```kotlin
    fun main() = runBlocking {
    	supervisorScope {
    		launch {
    			delay(500)
    			throw Exception("Error~")
    		}
    
    		launch {
    			delay(1000)
    			println("Done")
    		}
    	}
    	job.join()
    }
    
    // 출력
    Error~
    Done	
    ```
    

<div class="notice">
💡 안드로이드 개발 시 많이 사용되는 viewModelScope 또한 SupervisorJob을 가집니다.  
<br>
</div>

```kotlin
public val ViewModel.viewModelScope: CoroutineScope
    get() {
        val scope: CoroutineScope? = this.getTag(JOB_KEY)
        if (scope != null) {
            return scope
        }
        return setTagIfAbsent(
            JOB_KEY,
            CloseableCoroutineScope(SupervisorJob() + Dispatchers.Main.immediate)
        )
    }
```

### 예외 핸들러

코루틴 예외 발생 시, 기본 동작을 정의하고자 할 때에는 `CoroutineExceptionHandler` 를 사용할 수 있습니다.

```kotlin
val handler = CoroutineExceptionHandler { context, exception ->
	Log.d("CoroutineExceptionHandler", exception.toString())
}
        
// handler를 일일이 넣어줘야 하니, 확장함수를 만들어서 사용해도 좋겠다.
viewModelScope.launch(handler) {
	delay(1000L)
	throw Exception("1초")
}

viewModelScope.launch(handler) {
	delay(2000L)
	throw Exception("2초")
}

viewModelScope.launch(handler) {
	delay(3000L)
	throw Exception("3초")
}

// 출력
java.lang.Exception: 1초
java.lang.Exception: 2초
java.lang.Exception: 3초
```
