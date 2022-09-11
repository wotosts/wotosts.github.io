---
title: "[Android] MockWebserver 사용하기 - Hilt"
layout: single
date: '2022-09-10 15:53:00'
categories:
- Android
tags:
- test
- hilt
toc: true
toc_label: "MockWebserver 사용하기- Hilt"
toc_sticky: true
---

개발을 하다보면 서버-클라 사이에서 API 통신을 하는 경우가 많습니다. 모든 상황에서 통신이 성공하면 매우 좋겠지만, 그렇지 않은 경우들이 종종 있습니다. 때문에 통신에 실패한 경우를 처리하기 위한 코드를 작성하고, 이 코드가 제대로 동작하는지에 대한 확인이 필요합니다.
이 때 `MockWebserver`를 사용하여 API 통신 결과를 조작하여 원하는 상황을 테스트 할 수 있습니다. 혼자 서버까지 개발한다면 서버에서 실패 값을 주도록 하면 되겠지만, 그렇지 않을 때 사용하면 유용합니다.



## 준비
### 라이브러리 추가
build.gradle에 MockWebserver 의존성을 추가합니다.
``` kotlin
androidTestImplementation("com.squareup.okhttp3:mockwebserver:latestVersion")
androidTestImplementation("com.jakewharton.espresso:okhttp3-idling-resource:latestVersion")
```

### Rule 추가
[이전 글(테스트 환경 구축하기 with Hilt)](https://wotosts.gihub.io/android/android-test-with-hilt1) 에서 구성해둔 환경과 유사하게, 필요한 Rule을 추가합니다.

```kotlin
@get:Rule(order = 0)
var hiltRule = HiltAndroidRule(this)

@get:Rule(order = 1)
var taskRule = InstantTaskExecutorRule()

@get:Rule(order = 2)
var idlingResourceRule = DataBindingIdlingResourceRule()
```
<br>

### host address 변경
개발 중인 프로덕트는 정상적으로 동작하는 서버 주소를 사용하고 있습니다. 테스트 코드에서는 이 서버 주소를 `localHost` 주소로 바꿀 필요가 있습니다.
서버 주소를 Hilt 모듈을 통해 주입하고 있다면, 해당 모듈을 대신하여 localHost 주소를 주입하는 모듈이 필요합니다.

``` kotlin
@UninstallModules(NetworkHostModule::class)  // 1.
class Test {    
   //...

   @Module
   @InstallIn(SingletonComponent::class)
   inner class FakeNetworkHostModule {     // 2.

       @Provides
       @Singleton
       fun provideUrl(): String = "http://localhost:8080/"    // 3.
   }
}
```
1. `@UninstallModules`
 테스트 중, 프로덕트에서 사용하는 모듈을 제거하는 어노테이션입니다.
2. 위에서 제거한 모듈을 대체하는 `FakeNetworkHostModule`을 작성합니다.
3. localHost 주소를 제공합니다.
<br>

### IdlingResource
``` kotlin
//...
@UninstallModules(NetworkHostModule::class)
class Test {    
   //...

   @Module
   @InstallIn(SingletonComponent::class)
   inner class FakeNetworkHostModule {     

       //...

       @Provides
       @Singleton
       fun provideIdlingResource(okHttpClient: OkHttpClient):
           OkHttp3IdlingResource {
               return OkHttp3IdlingResource.create(
                   "okhttp",
                   okHttpClient
               )
           }
   }

   @Inject
   lateinit var okhttpIdlingResource: OkHttp3IdlingResource

   @Before
   fun setUp() {
       hiltRule.inject()
       IdlingRegistry.getInstance().register(okhttpIdlingResource)
       //...
   }
}
```
데이터바인딩과 마찬가지로 Idle 상태일 때 테스트가 가능하도록 하려면 `OkHttp3IdlingResource`를 등록해주어야 합니다. host address 변경과 동일하게 FakeNetworkMoudle을 통해 Test 클래스에 OkHttp3IdlingResource를 주입합니다.



## 적용하기
준비가 끝났습니다!
이제 MockWebserver를 사용하여 API 호출 시 원하는 응답을 보낼 수 있습니다.
``` kotlin
...

lateinit var mockWebserver: MockWebserver  	// 1

@Before
fun setUp() {
 hiltRule.inject()						// 2

   mockWebserver = MockWebserver()
   mockWebserver.start(8080)				// 3

   IdlingRegistry.getInstance().register(okhttpIdlingResource)		// 4
}

@Test
fun test() {
   server.enqueue(
           MockResponse().setResponseCode(200).setBody(	// 5
               "{" +
                   // body!
                   "}"
           )
       )
   //...
}

@After
fun tearDown() {
   mockWebserver.shutdown()				// 6
}

...
```
1. mockWebserver 변수를 선언합니다.
2. Hilt를 통해 변수에 값을 주입하기 위해 `hiltRule.inject()`를 호출합니다.
3. @Before 어노테이션을 붙인 함수에서 테스트 시작 전에 `mockWebserver`를 할당하고, 원하는 포트에서 mockWebserver를 작동시킵니다.
4. FakeNetworkHostModule을 통해 주입된 `okhttpIdlingResource`를 등록합니다.
5. `MockResponse().setResponse().setBody()` 에 원하는 응답을 작성합니다.
6. @After 어노테이션을 붙인 함수에서 테스트가 끝나면 mockWebserver 작동을 중지합니다.
<br>


예를 들어, API 호출에서 정상적인 답을 받지 못했을 때의 UI가 정상 동작하는지 확인하려고 한다면
```kotlin
 @Test
 fun serverError_showText(): Unit = runBlocking {
    server.enqueue(
     MockResponse().setResponseCode(200).setBody(
      "{" +
       "\"code\": \"SERVER_ERROR\"," +
       "\"message\":\"일시적 오류가 발생했습니다.\"" +
      "}"
     )
    )

    launchActivity<XXXActivity>()

    onView(
     withText("일시적 오류가 발생했습니다.")
    ).check(matches(isDisplayed()))
 }
```
위와 같이 테스트 코드를 작성할 수 있습니다.


### 문제?
이전 글과 마찬가지로 Mockk 라이브러리를 사용하고, 코루틴이 필요한 부분을 mock(spy 등) 처리할 경우 문제가 있을 수 있습니다.
* Mockk 라이브러리의 coEvery 등 코루틴 관련 함수는 Api 27 이상부터 정상동작합니다.
* MockWebserver 라이브러리는 Api 27 까지만 정상동작하는 것을 확인했습니다.(Network sequrity 문제라고 하네요.)

그렇기 때문에, 애뮬레이터가 필요한 테스트를 수행하는 경우 Api 27 기기에서만 테스트를 수행할 수 있었습니다.

방법을 알고 계신다면 공유 부탁드려요.



## 참고
* [https://www.raywenderlich.com/33855511-testing-rest-apis-using-mockwebserver](https://www.raywenderlich.com/33855511-testing-rest-apis-using-mockwebserver)
* [https://developer.android.com/training/dependency-injection/hilt-testing?hl=ko](https://developer.android.com/training/dependency-injection/hilt-testing?hl=ko)
