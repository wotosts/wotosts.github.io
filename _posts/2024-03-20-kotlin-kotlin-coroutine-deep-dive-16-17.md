---
title: "[Kotlin] 스터디 Kotlin Coroutine: Deep Dive 16-17장"
toc: true
categories:
- Kotlin
tags:
- coroutine
- kotlin
- Flow
- study
---

<div class='notice'>
Kotlin coroutine: Deep Dive 책 스터디 정리 포스트<br>
16-17장(3부) 에 해당하는 내용입니다. <br>
드디어 3부!! 15장은 테스트에 관련된 내용이고, 나머지 부분을 마무리한 후 다시 공부할 예정입니다.
</div>

---

## 16장. Channel

- 코루틴끼리의 통신을 위한 API
- **공유 상태 문제가 발생하지 않음**
- 값을 기다리는 코루틴을 큐로 가지고 있기 때문에 값 분배가 공평
- 송신자, 수신자 수 제한이 없지만, 채널을 통해 전송된 각 값은 단 한번만, 하나의 수신자만 받을 수 있음
- 주로 한쪽에서는 데이터를 생성하고, 한쪽에서는 처리만 하는 경우에 사용

예를 들어, Channel이 아래와 같이 3개의 송신자와 2개의 수신자를 가지고 있을 때, 아래와 같이 동작합니다. 

<div align = 'center'>
<img src = '/assets/images/2024-03-20/channel.png'/>
</div>


- 송신자가 여럿, 수신자가 하나일 경우 Fan-in
- 송신자가 하나, 수신자가 여럿일 경우 Fan-out

<br>
Channel는 인터페이스로, SendChannel, ReceiveChannel 두 개의 인터페이스를 확장합니다.  

- `SendChannel`
    - send를 통해 값 전송
        - Channel 용량이 다 찼다면 중단
    - 채널을 닫을 수 있음

- `ReceiveChannel`
    - receive를 통해 값 수신
        - Channel에 들어온 값이 없다면 값이 들어올 때까지 중단

```kotlin
interface Channel<E>: SendChannel<E>, ReceiveChannel<E>

interface SendChannel<in E> {
	suspend fun send(element: E)
	fun trySend(element: E): ChannelResult<Unit>
	...
}

interface ReceiveChannel<out E> {
	suspend fun receive(): E
	fun tryReceive(): ChannelResult<E>
	...
}
```

포인트는 값을 주고 받는 send, receive 함수 모두 중단 함수라는 점입니다. 

trySend, tryReceive라는 중단 함수가 아닌 함수들도 지원하지만 버퍼가 없는 Channel에서는(아래와 같이 생성하는 채널) 동작하지 않습니다. 

```kotlin
val channel = Channel<Int>()
```
<br>

### Channel 생성 및 사용하기

Channel은 `Channel<E>()` 또는 `produce`로 생성할 수 있습니다.   
Channel() 함수를 통해 Channel을 생성할 때 Channel 타입과, 버퍼 오버플로우 시 동작, 값이 처리되지 않을 경우 핸들러를 전달하여 커스텀이 가능합니다.


#### Channel 타입
Channel의 타입은 용량에 따라 구분됩니다.

- `Channel.UNLIMITED`
    - 용량 제한 없음
- `Channel.BUFFERED`
    - 특정한 용량 크기가 설정됨(기본 64)
    - 버퍼가 가득 찰 때까지 값을 전송할 수 있고, 버퍼가 가득 차면 버퍼에 공간이 생길 때까지 값 전송 중단
- `Channel.RENDEVOUS`
    - 용량이 0
    - 송신자와 수신자가 만날 때에만 값 교환이 가능
- `Channel.CONFLATED`
    - 용량이 1
    - 기존 원소가 있다면 새 원소가 덮어씀 → 최근 원소만 전달


#### 버퍼 오버플로우
Channel<E>() 함수를 통해 Channel 생성 시, 버퍼가 가득 찼을 때의 동작을 다음 옵션들로 설정할 수 있습니다.

- `BufferOverflow.SUSPEND`
    - 버퍼가 가득 찼을 때 send 중단
- `BufferOverflow.DROP_OLDEST`
    - 버퍼가 가득 찼을 때, 가장 오래된 값 제거 → Channel.CONFLATED
- `BufferOverflow.DROP_LATEST`
    - 버퍼가 가득 찼을 때, 최신 값 제거

	
#### onUndeliveredElement
채널의 값이 처리되지 않은 경우를 처리하는 핸들러입니다.
채널이 취소되거나 닫히는 경우, 예외가 발생했을 때 처리를 위해 사용할 수 있습니다.

위 속성들을 모두 적용한 예시입니다.

```kotlin
val channel = Channel<Int>(
                capacity = Channel.BUFFERED,
                onBufferOverflow = BufferOverflow.DROP_OLDEST,
                onUndeliveredElement = {
                    // 자원 정리
                    println("undelivered $it")
                }
            )
```

<br>
Channel은 양 끝에 송신자 하나와 수신자 하나로 이루어지는 것이 일반적이고, 아래와 같이 사용합니다. 

- 값 전송
    
    ```kotlin
    val channel = Channel<Int>()
    launch {
    	repeat(5) { index -> 
    		delay(1000)
    		channel.send(index * 2)
    	}
    }
    ```
    
    - Channel을 닫아주는 것을 잊을 수 있기 때문에 `produce`를 이용하여 Channel을 생성해도 좋습니다.
    - produce는 이유에 상관없이 코루틴이 종료되면 채널을 닫습니다. (CoroutineScope.produce 확장함수)
    
    ```kotlin
    produce {
    	repeat(5) { x -> send(x) }
    }
    ```
 
	
- 값 수신
    - Channel에 전달되는 값의 개수를 아는 경우
    
    ```kotlin
    launch {
    	repeat(5) { val received = channel.receive() }
    }
    ```
    
    - 보통의 경우에는 Channel에 전달되는 값의 개수를 모르기 때문에, for나 consumeEach 를 이용합니다.
    
    ```kotlin
    launch {
    	for(e in channel) { println(e) }
    }
    
    // 또는
    
    launch {
    	channel.consumeEach { e -> println(e) }
    }
    ```
    
    - fan-out 형태일 때는 consumeEach보다 for문 사용하는 것이 안전
        - consumeEach는 내부적으로 consume을 이용하고, consume은 에러 발생 시 Channel을 종료하기 때문에, 다른 수신자 코루틴들이 값을 더 이상 받을 수 없습니다.
    - consumeEach와 receiveAsFlow, consumeAsFlow
        
        
|  | consumeEach | consumeAsFlow | receiveAsFlow |
| --- | --- | --- | --- |
| 값 수신 | 하나의 수신자만 값 수신 가능 | 하나의 수신자만 값 수신 가능 | 여러 수신자가 같은 값 수신 가능 (Fan-out) |
| 사용 위치 | suspend 함수 또는 코루틴 내 | 일반 함수 내에서 사용 가능 | 일반 함수 내에서 사용 가능 |
| 블록 내에서 에러 발생 시 | Channel close | Channel close | Channel에 영향 없음 |
| 값 가공 |  | map, filter 등 수신된 값의 가공이 필요할 때 | map, filter 등 수신된 값의 가공이 필요할 때 |

<br>

## 17장. Select

다음의 경우 select 함수를 사용할 수 있습니다. 

- 가장 먼저 완료되는 코루틴의 결과를 얻을 때(async 경합)
- 여러 channel을 이용할 때, 남은 공간이 있는 채널에 데이터를 전송할 때(전송 가능한 채널 확인)
- 이용 가능한 값이 있는 채널에서 데이터를 받고 싶을 때(수신 가능한 채널 확인)
- 여러 데이터 소스로부터 결과값을 합칠 때

```kotlin
suspend fun test1(): String {
	delay(10000)
	return "test1"
}

suspend fun test2(): String {
	delay(1000)
	return "test2"
}

val scope = CoroutineScope()
suspend fun requestFirstReturn(): String {
	val res1 = scope.async { test1() }
	val res2 = scope.async { test2() }

	return select { res1.onAwait { it }, res2.onAwait { it } }
			.also { coroutineContext.cancelChildren() }
}

suspend fun main = coroutineScope {
	println(requestFirstReturn())
}
```

코드에서 select를 사용하여 먼저 얻을 수 있는 값인 test2() 의 반환값 “test2”이 requestFirstReturn의 반환값이 됩니다. 

이 때, test1 함수는 계속 동작하기 때문에, 취소를 위해 coroutineContext.cancelChildren() 을 호출하였습니다.

채널에서도 유사하게 select를 사용합니다.

```kotlin
fun main() = runBlocking {
	val ch1 = Channel<Char>(capacity = 2)
	val ch2 = Channel<Char>(capacity = 2)

	launch {
		for(c in 'A'..'Z') {
			delay(400)
			select {
				ch1.onSend(c) { println("sent ch1 $it") }
				ch2.onSend(c) { println("sent ch2 $it") }
			}
		}
	}

	launch {
		while(true) {
			delay(1000)
			select {
				ch1.onReceive { println("ch1 $it") },
				ch2.onReceive { println("ch2 $it") }
			}
		}
	}

	coroutineContext.cacelChildren()
}

// 출력
sent ch1 A
sent ch1 B
ch1 A
sent ch1 C
sent ch2 D
ch1 B
sent ch1 E
sent ch2 F
ch1 C
sent ch1 G
ch1 E
sent ch1 H
...
```

`onSend`를 사용하여, 빈 용량이 있는 채널을 선택하여 값을 전송합니다. 

`onReceive`를 사용하여 채널이 값을 가지고 있을 때, 해당 채널을 선택하여 값을 수신할 수 있습니다. onReceive대신 onReceiveCatching을 사용할 수 있습니다.

- onReceiveCatching
    - 채널이 값을 가지고 있거나 닫혔을 때 선택
    - 채널 닫힘 유무와 수신된 값을 알려주는 ChannelResult를 람다로 전달하고, 람다 결과값을 반환
        
        ```kotlin
        channel.onReceiveCatching { it.onFailure { it.getOrThrow() } }
        ```
