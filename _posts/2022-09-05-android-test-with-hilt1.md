---
title: "[Android] UI 테스트 환경 구축하기 - Hilt"
categories:
- Android
tags:
- test
- hilt
toc: true
toc_label: "UI 테스트 환경 구축하기 - Hilt"
toc_sticky: true
---

요즘에는 거의 필수인 것 같지만 작은 스타트업에서 테스트 코드를 짜면서 개발하는 것이 쉽지는 않은 것 같습니다. 하지만 테스트 코드의 필요성을 느낄 때가 있더라구요.
이번에도 프로젝트를 진행하면서 테스트 코드(일부)를 작성했는데, 이를 수행할 환경을 만드는 것에 생각보다 많은 시간을 소비했습니다.

우리의 시간은 소중하니까요😎, 삽질을 줄이고자 Hilt를 이용한 프로젝트에서 UI 테스트 코드를 작성(**실행!**)하기 위한 기본적인 것들을 정리했습니다.

---

## 준비

필요한 라이브러리를 gradle에 추가해줍니다.

``` kotlin
// espresso
androidTestImplementation("androidx.test.espresso:espresso-core:version")

// hilt test
androidTestImplementation("com.google.dagger:hilt-android-testing:version")
kaptAndroidTest("com.google.dagger:hilt-android-compiler:version")

// test runner, rules
androidTestImplementation("androidx.test:runner:version")
androidTestImplementation("androidx.test:rules:version")

// mockk
androidTestImplementation("io.mockk:mockk-android")

// junit ext
androidTestImplementation("androidx.test.ext:junit")
```
*  UI 테스트 작성을 위한 espresso
* Hilt를 사용했으므로, hilt-android-testing
* Test runner, rules
* Mockk
* Junit4

Mock 라이브러리는 편한 것을 사용하시면 될 것 같아요.
<br>

### TestApplication
라이브러리를 추가했으니, 이제 테스트용 Application을 정의해야 합니다.
위에서 추가한 android-hilt-testing에서 Hilt용 Application(`HiltTestApplication`)을 기본적으로 제공합니다. 물론, 직접 커스텀 Application을 만들어서 사용해도 됩니다.

`HiltTestApplication`을 확장하여 `CustomTestRunner`를 만들어주고
``` kotlin
class CustomTestRunner : AndroidJUnitRunner() {
    override fun newApplication(
       cl: ClassLoader?,
       name: String?,
       context: Context?
    ): Application {
        return super.newApplication(cl, HiltTestApplication::class.java.name, context)
    }
}
```
테스트에서 CustomTestRunner를 사용하도록 build.gradle에 적용합니다.
``` kotlin
...
android {
    defaultConfig {
        testInstrumentationRunner = "com.package.name.CustomTestRunner"
    }
}
...
```

준비는 끝났어요!


이제 Activity, Fragment에서 테스트 코드를 작성해봅시다.

## Activity 테스트 코드 작성

아래와 같이 구성된, 문의사항을 작성할 수 있는 Activity가 있습니다.
<p align="center">
	<img src="/assets/images/android-test-with-hilt-image1.png" width="50%" />
</p>
이 Activity는 로그인 상태 값 `isLogin: LiveData<Boolean>`을 가지고, 이 값에 따라서 이메일 작성란의 표시 유무가 달라진다고 가정합니다.

이를 확인하는 테스트코드를 작성해보도록 합시다.

<div class="notice">
1. 로그인 상태 - 이메일 작성란 안보임<br>
2. 로그아웃 상태 - 이메일 작성란 보임
</div>


먼저, 테스트를 실행할 수 있도록 준비 작업을 합니다.
``` kotlin
@HiltAndroidTest  // 1
@RunWith(AndroidJUnit4::class)  // 2
internal class SampleActivityTest {

    @get:Rule(order = 0)
    var hiltRule = HiltAndroidRule(this)  // 3

    @BindValue
    val viewModel = mockk(SampleViewModel(...))  // 4
}
```
1. `@HiltAndroidTest` hilt를 사용하는 클래스를 테스트
2. `@RunWith(AndroidJUnit4::class)` 해당 클래스는 AndroidJUnit4 러너로 동작
3. `HiltAndroidRule` hilt를 사용하는 테스트 클래스 룰 적용.
해당 코드 아래의 @BindValue 가 적용되지 않을 수 있어, order = 0 으로 설정하여 가장 먼저 적용
4. mock ViewModel
`@BindValue` 어노테이션을 사용하여 Activity가 참조하는 viewModel을 대체
(Mock이 아닌 Fake 형태로 만들어도 괜찮습니다.)

<br>

위에서 살펴본 사항을 토대로 하여 두 개의 테스트 함수를 선언합니다.
``` kotlin
 fun login_hideEmailInput() {}
 fun logout_showEmailInput() {}
```


이메일 입력란의 표시 여부는 `isLogin: LiveData<Boolean>` 값에 의해 바뀐다고 하였으니, `isLogin` 값을 지정해 줄 필요가 있습니다.
(Fake를 사용하지 않는다면) viewModel은 현재 mock 형태이기 때문에 아무런 동작을 하지 않으니, viewModel.isLogin 값 또한 바뀌지 않습니다.

그러니 viewModel.isLogin 대신 다른 조작 가능한 LiveData를 사용해야합니다.

```kotlin
 // ...		

 @get:Rule(order = 1)
 var taskRule = InstantTaskExecutorRule()  // 1

 @get:Rule(order = 2)
 var idlingResourceRule = DataBindingIdlingResourceRule()  // 2

 private val isLoginTest = MutableLiveData<Boolean>()  // 3

 @BindValue
  val viewModel = mockk(SampleViewModel(..)) {  // 4
	  every { isLogin } returns isLoginTest
		// ...
	}

 //...
```
1. `InstantTaskExecutorRule()`

	LiveData 등의 아키텍쳐 컴포넌트가 동일한 스레드에서 동작하도록 하는 테스트룰 적용
2. `DataBindingIdlingResourceRule()`

	DataBinding을 사용할 경우, 업데이트할 데이터가 더 이상 없을 때 테스트를 진행하도록 하는 테스트룰 적용  
    [DataBindingIdlingResourceRule code](https://github.com/android/architecture-components-samples/blob/7f861fd45d158e6277a3c35163c7f663e135b2cf/GithubBrowserSample/app/src/androidTest/java/com/android/example/github/util/DataBindingIdlingResourceRule.kt)  
    [DataBindingIdlingResource code](https://github.com/android/architecture-components-samples/blob/7f861fd45d158e6277a3c35163c7f663e135b2cf/GithubBrowserSample/app/src/androidTest/java/com/android/example/github/util/DataBindingIdlingResource.kt)
3. 테스트 클래스에서 직접 값을 조작할 LiveData 선언
4. viewModel의 LiveData(isLogin) stub


<br>

여기까지 작성 후, 테스트 함수를 실행해봅니다.
테스트가 실행되어도 아무런 화면이 뜨지 않습니다.
Activity을 띄우는 함수를 작성합니다.

``` kotlin
private fun launchActivity(): ActivityScenario<SampleActivity> {
    val scenario = launch(SampleActivity::class.java)   // 1
    idlingResourceRule.idlingResource.monitorActivity(scenario)   // 2

    return scenario
}
```
1. `ActivityScenario`를 이용하여 SampleActivity를 실행하고
2. dataBindingRule이 ActivityScenario를 모니터하도록 설정

<br>

이제 테스트 함수 내부를 구현하고, 테스트를 실행합니다!
``` kotlin
 @Test
 fun login_hideEmailInput() {
  isLoginText.value = true
  launchActivity()

  // visibility = gone
  onView(withId(R.id.email)).check(matches(not(isDisplayed())))
 }

 @Test
 fun logout_showEmailInput() {
  isLoginTest.value = false
  launchActivity()

  // visibility = visible
  onView(withId(R.id.email)).check(matches(isDisplayed()))
 }
```

화면이 반짝하고 열리면서 초록색을 확인하면 성공입니다~✅
<br>

## Fragment 테스트 코드 작성

Fragment의 테스트도 Activity 테스트와 유사하게 작성하면 됩니다.
그러나 Hilt를 사용하고 있어서 `FragmentScenario`를 사용할 수 없습니다.
대신! `@AndroidEntryPoint`를 붙인 Activity에 원하는 Fragment를 붙여 이를 테스트할 수 있습니다.

아래 라이브러리를 gradle에 추가하고,

``` kotlin
androidTestImplementation("androidx.fragment:fragment-testing:version")
```

테스트용 Activity를 작성합니다.
작성한 `HiltTestActivity`는 Fragment를 띄우는 용도로만 사용할 것입니다.
``` kotlin
@AndroidEntryPoint
class HiltTestActivity: AppCompatActivity()
```


Fragment를 띄울 때 사용했던 launchFragmentInContainer 대신 `launchFragmentInHiltContainer` 함수를 작성합니다.
`launchFragmentInHiltContainer` 함수는 HiltTestActivity를 이용하여 Fragment를 화면에 띄워줍니다.

``` kotlin
inline fun <reified T : Fragment> launchFragmentInHiltContainer(
    fragmentArgs: Bundle? = null,
    @StyleRes themeResId: Int = androidx.fragment.testing.R.style.FragmentScenarioEmptyFragmentActivityTheme,
    crossinline action: Fragment.() -> Unit = {}
): ActivityScenario<HiltTestActivity> {
    val startActivityIntent = Intent.makeMainActivity(
        ComponentName(
            ApplicationProvider.getApplicationContext(),
            HiltTestActivity::class.java
        )
    ).putExtra(
        "androidx.fragment.app.testing.FragmentScenario.EmptyFragmentActivity.THEME_EXTRAS_BUNDLE_KEY",
        themeResId
    )

    return ActivityScenario.launch<HiltTestActivity>(startActivityIntent).onActivity { activity ->
        val fragment: Fragment = activity.supportFragmentManager.fragmentFactory.instantiate(
            Preconditions.checkNotNull(T::class.java.classLoader),
            T::class.java.name
        )
        fragment.arguments = fragmentArgs
        activity.supportFragmentManager
            .beginTransaction()
            .add(android.R.id.content, fragment, "")
            .commitNow()

        fragment.action()
    }
}
```

이제 원하는 곳에서 `launchFragmentInHiltContainer` 함수를 호출하여 테스트를 수행할 수 있습니다!
``` kotlin
	@get:Rule(order = 0)
	var hiltRule = HiltAndroidRule(this)

	// ...

	@Before
	fun prepare() {
		launchFragmentInHiltContainer<SampleFragment>()
	}

	// ... test ...
```

---
### 참고
* [https://developer.android.com/training/testing/ui-testing/espresso-testing?hl=ko](https://developer.android.com/training/testing/ui-testing/espresso-testing?hl=ko)
* [https://www.raywenderlich.com/22152158-testing-with-hilt-tutorial-ui-and-instrumentation-tests](https://www.raywenderlich.com/22152158-testing-with-hilt-tutorial-ui-and-instrumentation-tests)
* [https://github.com/android/architecture-samples](https://github.com/android/architecture-samples)
* [https://github.com/android/architecture-components-samples](https://github.com/android/architecture-components-samples)

---
** 잘못된 사항이 있으면 언제든 코멘트 남겨주세요. **
