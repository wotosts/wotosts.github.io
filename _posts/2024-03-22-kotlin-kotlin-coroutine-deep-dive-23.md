---
title: "[Kotlin] 스터디 Kotlin Coroutine: Deep Dive 23장"
toc: true
categories:
- Kotlin
tags:
- Flow
- coroutine
- kotlin
- study
---

<div class='notice'>
Kotlin coroutine: Deep Dive 책 스터디 정리 포스트<br>
23장(3부) 에 해당하는 내용입니다. <br>
</div>

---
## 23장. 플로우 처리

플로우 생성과 최종 연산 사이의 연산들을 의미합니다.  
**[FlowMarbles](https://flowmarbles.com/)** 라고 각 연산들이 동작하는 방식을 직접 살펴볼 수 있는 웹페이지가 있으니 참고하면 좋을 것 같습니다.

### map

플로우의 각 값들을 map에 전달된 연산을 적용하여 변환합니다.

```kotlin
flowOf(1,2,3)
	.map { it * it }
	.collect { println("$it") }

// 출력
1
4
9
```

### filter

플로우의 값들 중 주어진 조건에 맞는 값들만 방출합니다.

```kotlin
(1..10).asFlow()
	.filter { it > 3 }
	.filter { it % 4 == 0 }
	.collect { println("$it") }

// 출력
4
8
```

### take, drop

take를 통해 특정 개수 만큼의 값만 받거나, drop을 통해 특정 개수 만큼의 값을 버릴 수 있습니다.

```kotlin
('A' .. 'Z').asFlow()
	.take(3)
	.collect { println("$it")
	
// 출력
A
B
C
```

```kotlin
('A' .. 'Z').asFlow()
	.drop(23)
	.collect { println("$it")
	
// 출력
X
Y
Z
```

### merge, zip, combine

merge, zip, combine 모두 여러 플로우를 하나의 플로우로 합칠 때 사용합니다. (단, zip은 2개)

예시는 모두 아래 두 개의 플로우를 사용합니다.

```kotlin
val f1 = flowOf(1, 2, 3).onEach { delay(400) }
val f2 = flowOf(0.1, 0.2, 0.3).onEach { delay(1000) }
```

- `merge`
    
    단순하게 여러 플로우에서 방출되는 값을 하나의 플로우로 방출합니다.  
    서로 다른 플로우의 원소 방출을 기다리지는 않습니다.
    
    ```kotlin
    merge(f1, f2).collect { println("$it") }
    
    // 출력
    1
    2
    0.1
    3
    0.2
    0.3
    ```
    
- `zip`
    
    두 플로우에서 방출되는 값을 쌍으로 만듭니다.   
		다른 플로우에 쌍이 될 값이 방출될 때까지 기다립니다. 플로우 하나가 끝나면 종료됩니다.
    
    ```kotlin
    f1.zip(f2) { o1, o2 -> 
    	"${o1}_${o2}"
    }
    .collect { println("$it") }
    
    // 출력
    1_0.1 // 1초 뒤
    2_0.2 // 1초 뒤
    3_0.3 // 1초 뒤
    ```
    

- `combine`
    
    zip과 마찬가지로 플로우에서 방출되는 값을 쌍으로 만들지만, 다른 플로우의 값 방출을 기다리지 않고 기존 값을 그대로 이용합니다.  
    어느 플로우에서라도 새 값을 방출하면 새 쌍을 만들 수 있습니다.
    
    ```kotlin
    f1.combine(f2) { o1, o2 -> 
    			"${o1}_${o2}"
    }
    .collect { println("$it") }
    
    // 출력
    2_0.1 // 1초 뒤
    3_0.1 // 0.2
    3_0.2 // 0.8
    3_0.3 // 1초 뒤
    ```
    

### fold, scan

fold와 scan은 중간 연산이 아닌 최종 연산으로, 플로우에서 방출되는 모든 값을 하나로 합칩니다. 

둘의 차이점은, scan은 람다식을 실행한 중간 결과들을 알 수 있다는 점입니다.

```kotlin
// fold
val sum = flowOf(1, 2, 3, 4)
	.fold(0) { acc, i -> acc + i }
println("$sum")

// 출력
10
```

```kotlin
// scan
val sum = flowOf(1, 2, 3, 4)
	.scan(0) { acc, i -> acc + i }
	.collect { println("$it") }

// 출력
0
1
3
6
10
```

### flatMapConcat, flatMapMerge, flatMapLatest

컬렉션에 flatMap을 적용한 것과 같이 평탄화된 플로우를 만듭니다. 플로우의 원소가 방출되는 시간이 다르기 때문에 flatMap이 없는 대신 flatMapConcat, flatMapMerge, flatMapLatest 등이 있습니다.

예시는 모두 아래의 플로우를 사용합니다.

```kotlin
fun flowFrom(e: String) = flowOf(1, 2, 3)
	.onEach { delay(1000) }
	.map { "${it}_${e}")
```

- `flatMapConcat`
    
    생성된 플로우를 순차적으로 처리, 이전 플로우가 완료된 후에 다음 플로우를 처리합니다.
    
    ```kotlin
    flowOf("A", "B", "C")
    	.flatMapConcat { flowFrom(it) }
    	.collect { println(it) }
    	
    // 출력
    1_A // 1초
    2_A // 1초
    3_A // 1초
    1_B // 1초
    2_B // 1초
    3_B // 1초
    1_C // 1초
    2_C // 1초
    3_C // 1초
    ```
    

- `flatMapMerge`
    
    생성된 플로우를 동시에 처리하며, concurrency 인자를 통해 동시 실행 가능한 플로우 수를 관리합니다.
    
    ```kotlin
    flowOf("A", "B", "C")
    	.flatMapMerge { flowFrom(it) } // A, B, C 플로우 3개 모두 실행
    	.collect { println(it) }
    	
    // 출력
    1_A // 1초
    1_B
    1_C
    2_A // 1초
    2_B
    2_C
    3_A // 1초
    3_B
    3_C
    ```
    

- `flatMapLatest`
    
    가장 나중에 생성된 플로우만 동작하고, 이전 플로우는 종료합니다. 
    
    ```kotlin
    flowOf("A", "B", "C")
    	.flatMapLatest { flowFrom(it) }
    	.collect { println(it) }
    	
    // 출력
    3_A // 1초
    3_B // 1초
    3_C // 1초
    ```
    

### retry

에러 발생시 재시도 처리로 retry, retryWhen을 제공합니다.  
플로우 단계 내에서 에러가 발생했을 때 사용하여 플로우 재시작 여부를 결정합니다.

```kotlin
flow {
	emit(1)
	emit(2)
	error("으악")
	emit(3)
}.retry(2) {
	println("에러 $it")
	true
}.collect { println("$it") }

// 출력
1
2
으악
1
2
으악
1
2
으악
// error
```

### distinct

플로우에서 방출된 값이 이전 값과 동일할 경우 제거합니다.  
distinctUntilChanged, distinctUntilChangedBy 를 제공합니다.

```kotlin
flowOf(1,2,2,3,4,4)
	.distinctUntilChanged()
	.collect { println("$it") }
	
// 출력
1
2
3
4
```

### 최종 연산

collect 외에 플로우가 완료되었을 때 값을 반환하는 연산입니다.  
reduce, count, first, firstOrNull, fold 등이 있습니다.
