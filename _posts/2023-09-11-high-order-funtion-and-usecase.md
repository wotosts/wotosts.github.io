---
title: "[Kotlin] 고차함수와 UseCase"
categories:
- Kotlin
tags:
- Architecture
- kotlin
---

<div class = "notice--success">
Android Weekly 보다가 흥미로운 시리즈가 연재되고 있길래, 일단 발행된 두 편을 묶어서 정리해보았습니다. <br>
재밌더라구요.
</div>

<br>


## Igniting High-Order Thinking: Empowering Code with High-Order Functions
[<kbd><br> 👉 원본 링크<br></kbd>][Link1]

[Link1]:https://proandroiddev.com/igniting-high-order-thinking-empowering-code-with-high-order-functions-493fcc994de3

- High order thinking? High order function?
    - `고차 함수`: 다른 함수를 인자로 받을 수도 있고, 함수 실행 결과로도 반환할 수 있는 타입의 함수
    - `High order thinking`: 문제를 작은 부분으로 나누고, 각 부분들끼리의 연관성을 파악하여 복잡한 문제를 효과적으로 해결할 수 있는 방식

- 둘이 합하면?
    - 복잡한 문제 해결: 문제를 작은 고차 함수로 만들어 해결할 수 있음
    - 분명한 의도: 의도를 분명하게, 이해를 쉽게하여 코어 로직에 집중 가능
    - 반복 감소: 일반적인 작업들을 캡슐화하고 코드를 더 유지보수가 쉽도록 함
    - 코드의 유연함: 여러 상황에 대해 대응 가능
    - 구조화된 코드 작성 가능


아래의 경우, calculator를 인자로 받는 고차 함수 calculatorTotal를 사용하면
sum 외에도 다양한 상황에서 calculatetTotal 를 사용할 수 있게 된다.

예를 들어, 합이 아닌 곱을 구한다고 하면 calculatorTotal에 곱을 구하는 calculator를 전달하면 된다. 

```kotlin
fun calculateTotal(
    prices: List<Double>, 
    calculator: (List<Double>) -> Double
): Double {
    return calculator(prices)
}

val sumCalculator: (List<Double>) -> Double = { prices ->
    prices.sum()
}

val prices = listOf(12.99, 8.75, 24.50, 10.0)
val totalAmount = calculateTotal(prices, sumCalculator)
println("Total amount: \$${totalAmount}")
```



<br>
<br>

## Recreating UseCase: Embracing a Fluent and Fun Approach
[<kbd><br> 👉 원본 링크<br></kbd>][Link2]

[Link2]:https://proandroiddev.com/recreating-usecase-embracing-a-fluent-and-fun-approach-2023-71e9aad43d89

고차함수를 이용해 UseCase를 개선한다.

이를 위한 기준은

1. 함수 우선 개발(일급)
    클래스 개발보다 함수 작성이 먼저다.
    
2. `순수 함수`
    
    **함수형 프로그래밍 자체가, 문제를 작게작게 쪼개어 순수 함수를 만들어 문제를 해결하는 것**
    **순수 함수는 사이드 이펙트가 없기 때문에, 결과를 예측할 수 있고 믿을 수 있다.**
    
3. 고차함수의 유틸화
    
    다양한 상황에 동적으로 유연하게 대처할 수 있다.
    
4. 함수 구성하기
    
    함수를 섞어서 새로운 함수를 만들어낸다. 복잡한 동작을 하는 함수도 이렇게 작은 블록을 조립하듯이 만든다.
    
		
<br>

일반적인 UseCase 를 개선해보자.

```kotlin
fun placeOrderUseCase(orderRepository: OrderRepository) {
    if (orderRepository.isLoggedIn()) {
        val cart = orderRepository.getCart()
        if (cart.isNotEmpty()) {
            if (orderRepository.hasEnoughFunds(getTotalPrice())) {
                orderRepository.updateProductStock()
                orderRepository.clearCart()
            } else {
                throw InsufficientFundsException(
                    "Not enough funds in the wallet."
                )
            }
        } else {
            throw EmptyCartException("The cart is empty.")
        }
    } else {
        throw NotLoggedInException("User is not logged in.")
    }
}
```

코드를 살펴보면 if가 그득그득하다. 

<br>


1. 먼저 isLogin 조건을 캡슐화하여 고차 함수로 분리한다.

```kotlin
fun placeOrderUseCase(orderRepository: OrderRepository) {
    executeWhenUserLoggedIn(orderRepository) {
        val cart = orderRepository.getCart()
        if (cart.isNotEmpty()) {
            if (orderRepository.hasEnoughFunds(getTotalPrice())) {
                orderRepository.updateProductStock()
                orderRepository.clearCart()
            } else {
                throw InsufficientFundsException(
                    "Not enough funds in the wallet."
                )
            }
        } else {
            throw EmptyCartException("The cart is empty.")
        }
    }
}

private fun executeWhenUserLoggedIn(
    orderRepository: OrderRepository,
    executeFunction: () -> Unit
) {
    if (orderRepository.isLoggedIn()) executeFunction()
    else throw NotLoggedInException("User is not logged in.")
}
```

executeWhenUserLoggedIn 에 전달되는 함수에, 실행하고 싶은 코드를 넣어서
이름 그대로 유저가 로그인했을 때 동작을 수행한다는 것을 쉽게 알 수 있다.
<br>
<br>


2. 이번에는 cart 조건 캡슐화

```kotlin
fun placeOrderUseCase(orderRepository: OrderRepository) {
    executeWhenUserLoggedIn(orderRepository) {
        executeWhenCartNotEmpty(orderRepository) {
            if (orderRepository.hasEnoughFunds(getTotalPrice())) {
                orderRepository.updateProductStock()
                orderRepository.clearCart()
            } else {
                throw InsufficientFundsException(
                    "Not enough funds in the wallet."
                )
            }
        }
    }
}

private fun executeWhenCartNotEmpty(
    orderRepository: OrderRepository,
    executeFunction: () -> Unit
) {
    val cart = orderRepository.getCart()
    if (cart.isNotEmpty()) executeFunction()
    else throw EmptyCartException("The cart is empty.")
}
```

<br>

3. 마지막 조건 캡슐화

```kotlin
fun placeOrderUseCase(orderRepository: OrderRepository) {
    executeWhenUserLoggedIn(orderRepository) {
        executeWhenCartNotEmpty(orderRepository) {
            executeWithSufficientFunds(orderRepository) {
                orderRepository.updateProductStock()
                orderRepository.clearCart()
            }
        }
    }
}

private fun executeWithSufficientFunds(
    orderRepository: OrderRepository,
    executeFunction: () -> Unit
) {
    val totalPrice = getTotalPrice()
    if (orderRepository.hasEnoughFunds(totalPrice))
        executeFunction()
    else throw InsufficientFundsException(
        "Not enough funds in the wallet."
    )
}
```

아직도 뭔가 더 고치고 싶다면!  orderRepository 확장함수를 이용한다!

요렇게

```kotlin
OrderUseCase.kt

fun OrderRepository.placeOrderUseCase() {
    executeWhenUserLoggedIn {
        executeWhenCartNotEmpty {
            executeWithSufficientFunds {
                updateProductStock()
                clearCart()
            }
        }
    }
}

private fun OrderRepository.executeWhenUserLoggedIn(
    executeFunction: () -> Unit
) {
    if (isLoggedIn()) executeFunction()
    else throw NotLoggedInException("User is not logged in.")
}

private fun OrderRepository.executeWhenCartNotEmpty(
    executeFunction: () -> Unit
) {
    val cart = getCart()
    if (cart.isNotEmpty()) executeFunction()
    else throw EmptyCartException("The cart is empty.")
}

private fun OrderRepository.executeWithSufficientFunds(
    executeFunction: () -> Unit
) {
    val totalPrice = getTotalPrice()
    if (hasEnoughFunds(totalPrice))
        executeFunction()
    else throw InsufficientFundsException(
        "Not enough funds in the wallet."
    )
}
```



3겹의 if 조건들이 잘게 잘게 쪼개진 작은 함수가 되었다. 





이런 방식의 이점

- 모듈성: 각각의 함수가 하나의 동작만 하기 때문에 코드 관리가 쉬워짐
- 재사용성: 다양한 상황, 앱 내 여러 곳에서 재사용 가능
- 가독성: 함수 이름을 통해 코드 이해도가 높아짐. 
- 에러 핸들링: 오류가 중앙에서 관리되므로, 에러 메세지가 일관적 <- ?..
- 용이한 테스트 


<br>

---




예시가 비교적 간단한 케이스라, 고차 함수로 쪼개는 과정이 어렵지 않습니다. 
깔끔하기도 하고, placeOrderUseCase 내부 로직이 훨씬 잘 읽힙니다. 

그러나, 개인적으로 너무 복잡한 로직의 경우는 고민을 많이 해봐야 할 것 같더라구요. 
고차 함수로 적절하게 쪼개는 조건을 잘 정의해야 할 듯 합니다. 너무 잘게 쪼개면 오히려 따라가기가 번거로울 것 같아요. 
(개인적으로 들여쓰기 단계가 많은 것은 불호..)
