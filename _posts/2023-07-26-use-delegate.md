---
title: Delegate 사용하기
categories:
- design-pattern
tags:
- software
- kotlin
---

요 근래 블로그 글 중, 아래의 Delegate 관련 글을 흥미롭게 읽었습니다. <br>
개발하다보면 이것저것 공통 처리한다고 Base 클래스를 만든 후, 이들이 점점 거대해지는 경험을 해본 적이 다들 있으실텐데요. (저도 물론 😇)<br>
위임 패턴으로 Base 클래스의 거대화를 막고, 유연하게 사용하자 라는 것이 매우 유용해 보였습니다.<br><br>
물론 일부 기본 구현이 있는 Interface를 사용할 수도 있지만, 기본 구현이 아닌 특정한 구현을 여러 곳에서 동일하게 사용해야 한다면 위임을 사용하는 것이 더 적절할 것이라는 생각이 듭니다. 


<div class = "notice--success" >
<a href = "https://medium.com/@mobidroid92/hello-delegates-goodby-base-classes-c8aeedc2b855"> "Hello Delegates, Goodbye Base Classes"</a><br>
</div>

대충 번역하고, 정리한 글입니다. 

---

<br>

프로젝트를 생성하고, 코딩을 시작할 때 주로 Base 클래스를 만들고 이를 상속받습니다. 
Base 클래스를 통해 중복되는 코드를 제거하고 공통 기능을 사용할 수 있기 때문입니다.

하지만 이렇게 사용하는 것이 SOLID 원칙을 따른다고 할 수 있을까요? 

- 단일책임 원칙은?
- 베이스가 엄청나게 거대하다면?
- 모든 자식 클래스들이 그들이 사용하지 않는 기능들도 상속받아야 하나?
- 베이스에 변화가 생긴다면 자식들은?
- 베이스 클래스와 다른 방식이 필요하다면?

베이스 클래스는 많은 이점이 있지만, 위와 같은 면에서는 리스크가 있습니다. 

>💡 SOLID 원칙?
>	
> * S(Single Responsibility Principle). `단일 책임 원칙`
>    - 하나의 모듈은 오직 하나의 액터(사용자)에 대한 책임만 가진다.
> * O(Open-Close Principle). `개방-폐쇄 원칙`
>    - 개체는 확장에는 열려 있어야 하고, 변경에는 닫혀 있어야 한다.
> * L(Liskov Substitution Principle). `리스코프 치환 원칙`
>    - 상위 타입 개체의 위치에 하위 타입을 치환할 수 있다.
> * I(Interface Segregation Principle). `인터페이스 분리 원칙`
>    - 특정 객체가 사용하지 않는 메소드에 의존하지 않는다.
> * D(Dependency Inversion Principle). `의존성 역전 원칙`
>   - 소스 코드 의존성이 추상에 의존하며, 구현체에 의존하지 않는다.



## 일반적으로, 코드 재사용에는 여러 relationship option이 있습니다.

### 상속

`is-a` 형태

클래스는 한번에 하나만 상속 가능하며, 가능한 피해야 하는 방법입니다.

### 조합(관계? Association)

특정 기능을 사용하고자 할 때 해당 기능을 가진 객체를 생성하여 사용하는 방법입니다.

Association은 아래 방식으로 이룰 수 있습니다.

1. **Aggregation(집합) - `refer to`**
    
    모든 고용자들은 출입카드를 가지고 있다와 같은 <b>약한 관계</b>. 고용자들은 출입카드 없이도 존재할 수 있고, 출입카드도 고용자 없이 존재할 수 있습니다.
    
2. **Composition(구성) - `has-a`**
    
    집에는 방이 있다 와 같은 <b>강한 관계</b>. 방은 집 없이 존재할 수 없고, 방의 생명주기는 집이 관리합니다. 
		
		
<b>조합은 런타임에서 객체 동작을 바꿀 수 있기 때문에 상속보다 유연합니다.(물론 상속도 제대로 사용하면 좋아요)</b>
    

<br>
<br>
그래서 우리는 조합을 상속과 같이 사용할 수 있을까요? 답은 가능! 요것이 Delegate의 멋있는 부분이에요.

## Delegation pattern

Delegation pattern은 상속과 마찬가지로 코드를 재사용할 수 있는 패턴입니다. 하지만 상속과 달리 다른 객체에 하고자 하는 동작을 위임합니다.


상속은 아시듯, 부모의 것을 자식들이 사용하는 것이 가능해집니다. 
![]({{ 'assets/images/2023-07-24/inheritance.png' | relative_url }})

위임은 아래와 같은 방식입니다.

* 나는 기능 A를 할 수 있어!
* 하지만 내가 기능 A를 직접 수행하지는 않을거야
* 기능 A를 수행하는 B에게 대신 수행하도록 할거야


## Delegate 사용 예시

안드로이드 툴바에 Delegate를 사용하는 예시입니다.

1. Delegate를 정의합니다.
    
    ```kotlin
    interface ToolbarDelegate {
    
        fun addToolbar(toolbar: MaterialToolbar, title: String? = null,
                       enableHomeAsUp: Boolean = false, homeAsUpDrawable: Drawable? = null)
    
        fun setToolbarTitle(newTitle: String?)
    
        fun setActivityForToolbarDelegate(activity: AppCompatActivity?)
    }
    ```
    
2. 1에서 선언한 Delegate interface를 구현합니다.
    
    ```kotlin
    class ToolbarDelegateImpl : ToolbarDelegate {
    
        private var toolbar: MaterialToolbar? = null
        private var activity: AppCompatActivity? = null
    
        override fun setActivityForToolbarDelegate(activity: AppCompatActivity?) {
            this.activity = activity
        }
    
        override fun addToolbar(toolbar: MaterialToolbar, title: String?,
                                enableHomeAsUp: Boolean, homeAsUpDrawable: Drawable?) {
    
            this.toolbar = toolbar
    
            activity?.setSupportActionBar(toolbar)
    
            activity?.supportActionBar?.let { supportActionBar ->
                supportActionBar.setDisplayHomeAsUpEnabled(enableHomeAsUp)
                homeAsUpDrawable?.let {
                    supportActionBar.setHomeAsUpIndicator(it)
                }
            }
    
            setToolbarTitle(title)
        }
    
        override fun setToolbarTitle(newTitle: String?) {
            toolbar?.title = newTitle
        }
    }
    ```
    

1. 사용하고자 하는 클래스에서 1의 delegate를 구현하여, 내부에서 2의 구현체를 사용하도록 합니다.
    
    ```kotlin
    class ManualDemoActivity :
        AppCompatActivity(),
        ToolbarDelegate {
    
        private val toolbarDelegate: ToolbarDelegate = ToolbarDelegateImpl()
    
        override fun onCreate(savedInstanceState: Bundle?, persistentState: PersistableBundle?) {
            super.onCreate(savedInstanceState, persistentState)
    
            addToolbar(
                toolbar = findViewById(R.id.toolbar),
                title = getString(R.string.app_name),
                enableHomeAsUp = true
            )
        }
    
        override fun addToolbar(
            toolbar: MaterialToolbar,
            title: String?,
            enableHomeAsUp: Boolean,
            homeAsUpDrawable: Drawable?
        ) {
            toolbarDelegate.addToolbar(toolbar, title, enableHomeAsUp, homeAsUpDrawable)
        }
    
        override fun setToolbarTitle(newTitle: String?) {
            toolbarDelegate.setToolbarTitle(newTitle)
        }
    
        override fun setActivityForToolbarDelegate(activity: AppCompatActivity?) {
            toolbarDelegate.setActivityForToolbarDelegate(activity)
        }
    
    }
    ```
    

## In Kotlin

예시를 쭉 보다보니 너무 많은 코드들을 작성해야 함을 알 수 있습니다 ..  

kotlin 에서는 by 키워드를 이용하면 이를 간단하게 처리할 수 있습니다. 

```kotlin
class DemoActivity :
    AppCompatActivity(),
    ToolbarDelegate by ToolbarDelegateImpl() { // 간단~

    override fun onCreate(savedInstanceState: Bundle?, persistentState: PersistableBundle?) {
        super.onCreate(savedInstanceState, persistentState)

        addToolbar(
            toolbar = findViewById(R.id.toolbar),
            title = getString(R.string.app_name),
            enableHomeAsUp = true
        )
    }
}
```

- 단, 이러한 방식의 by 키워드사용은 interface의 구현에 대해서만 허용됩니다.
- 프로퍼티에도 Delegate를 사용할 수 있습니다.

## 장점과 단점

글의 처음에 언급된 여러 문제 상황들을 조합과 delegate를 통해 해결할 수 있습니다. 

- 추가로, 
    - Impl 에 대한 관리 → 관리포인트가 줄어듭니다.
   
- 단점
    - 인터페이스 설계가 그렇듯...표준화된 인터페이스를 구성하기 쉽지 않습니다.
