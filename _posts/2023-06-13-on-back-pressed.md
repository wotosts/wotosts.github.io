---
title: "[Android] OnBackPressed 정리하기"
categories:
- ''
- Android
tags:
- 기타
---

Acitivty.onBackPressed() 가 deprecated 되었습니다.

Android 13(API 33, 티라미수) 이상의 버전에서는 `OnBackPressedDispatcher` 또는 `OnBackInvokedDispatcher` 를 사용하는 것을 권장합니다.

<br>


## OnBackPressedDispatcher
ComponentActivity에서는 `OnBackPressedDispatcher`를 이용하여 뒤로가기 동작을 처리합니다. 

일반적으로 onBackPressed() 오버라이딩 하지 않고 화면 종료로만 사용하는 경우에는,
onBackPressed() 대신 onBackPressedDispatcher.onBackPressed()를 호출하여 동작을 대체할 수 있습니다.

* 콜백을 직접 달아주지 않는 경우에는 <b>화면 종료</b>라는 기본 동작을 수행을 수행합니다. 
ComponentActivity에 이미 fallbackOnBackPressed 콜백 이 구현이 되어있기 때문입니다. 

* 활성화된 콜백이 없는 경우에 동작합니다. 
```java
private final OnBackPressedDispatcher mOnBackPressedDispatcher =
            new OnBackPressedDispatcher(new Runnable() {
                @Override
                public void run() {
                    // Calling onBackPressed() on an Activity with its state saved can cause an
                    // error on devices on API levels before 26. We catch that specific error and
                    // throw all others.
                    try {
                        ComponentActivity.super.onBackPressed();
                    } catch (IllegalStateException e) {
                        if (!TextUtils.equals(e.getMessage(),
                                "Can not perform this action after onSaveInstanceState")) {
                            throw e;
                        }
                    }
                }
            });
```


ComponenetActivity.onBackPressed()도 내부적으로 onBackPressedDispatcher를 호출하고 있습니다.
(그래도 OnBackPressed() 직접 사용하면 줄 그어져요.)


```java
/**
     * Called when the activity has detected the user's press of the back
     * key. The {@link #getOnBackPressedDispatcher() OnBackPressedDispatcher} will be given a
     * chance to handle the back button before the default behavior of
     * {@link android.app.Activity#onBackPressed()} is invoked.
     *
     * @see #getOnBackPressedDispatcher()
     */
    @SuppressWarnings("deprecation")
    @Override
    @MainThread
    public void onBackPressed() {
        mOnBackPressedDispatcher.onBackPressed();
    }
```


<br>

### 커스텀 뒤로가기 동작
기존에는 뒤로가기 동작 커스텀을 위해서 onBackPressed()를 오버라이딩 하는 방법을 사용했습니다. 
OnBackPressedDispatcher를 이용하면 복잡한 뒤로가기 동작을 조금 더 쉽게 사용할 수 있을 것 같아요.

커스텀 뒤로가기 동작을 처리하기 위해 아래와 같이 OnBackPressedCallback 을 등록합니다.
lifecycleOwner를 이용하므로, 화면이 보여지는 경우에만 동작이 적용됩니다.

```kotlin

// Fragment
class FormEntryFragment : Fragment() {
  var flag = false
  override fun onAttach(context: Context) {
    super.onAttach(context)
    val callback = object : OnBackPressedCallback(
      true // default to enabled
    ) {
      override fun handleOnBackPressed() {
        showAreYouSureDialog()
      }
    }

		// here!!
    requireActivity().onBackPressedDispatcher.addCallback(
      this, // LifecycleOwner
      callback,
      enabled = flag
    )
  }
}

// Activity
class FormActivity: AppCompatActivity() {
   override fun onCreate() {
      super.onCreate()
      onBackPressedDispatcher.addCallback(this) {
         // to do onBackPressed
      }
   }
}
```
<br>

Composable 함수 내에서는 BackHandler Composable을 사용합니다.

```kotlin
BackHandler(
            enabled = modalBottomSheetState.isVisible,
            onBack = onClickBack
)
```

BackHandler는 내부적으로 현재 화면의 OnBackPressedDispatcher를 구하여 OnBackPressedCallback을 등록합니다.

```java
@SuppressWarnings("MissingJvmstatic")
@Composable
public fun BackHandler(enabled: Boolean = true, onBack: () -> Unit) {
    // Safely update the current `onBack` lambda when a new one is provided
    val currentOnBack by rememberUpdatedState(onBack)
    // Remember in Composition a back callback that calls the `onBack` lambda
    val backCallback = remember {
        object : OnBackPressedCallback(enabled) {
            override fun handleOnBackPressed() {
                currentOnBack()
            }
        }
    }
    // On every successful composition, update the callback with the `enabled` value
    SideEffect {
        backCallback.isEnabled = enabled
    }
    val backDispatcher = checkNotNull(LocalOnBackPressedDispatcherOwner.current) {
        "No OnBackPressedDispatcherOwner was provided via LocalOnBackPressedDispatcherOwner"
    }.onBackPressedDispatcher
    val lifecycleOwner = LocalLifecycleOwner.current
    DisposableEffect(lifecycleOwner, backDispatcher) {
        // Add callback to the backDispatcher
        backDispatcher.addCallback(lifecycleOwner, backCallback)
        // When the effect leaves the Composition, remove the callback
        onDispose {
            backCallback.remove()
        }
    }
}
```

만약 콜백을 여러개 등록했다면, 등록된 콜백들 중 활성화(enabled = true) 되어있는 콜백 중
<b>가장 나중에 추가된 콜백이 호출</b>됩니다.

enabled 설정에 유의해야 합니다.

```java
   @MainThread
    public void onBackPressed() {
        Iterator<OnBackPressedCallback> iterator =
                mOnBackPressedCallbacks.descendingIterator();
        while (iterator.hasNext()) {
            OnBackPressedCallback callback = iterator.next();
            if (callback.isEnabled()) {
                callback.handleOnBackPressed();
                return;
            }
        }
        if (mFallbackOnBackPressed != null) {
            mFallbackOnBackPressed.run();
        }
    }
```


<br>
<br>
<br>


## Predictive Back gesture
Android 13부터 `Predictive back gesture`가 추가되었습니다.
좌우 스와이프 등을 수행했을 때, 유저가 정말로 뒤로가기 동작을 원하는지를 판단하기 위함입니다. 

![](https://developer.android.com/static/images/about/versions/13/predictive-back-nav-home.gif){: .align-center}

[구글 IO 영상](https://www.youtube.com/watch?v=Elpqr5xpLxQ) 초반을 보면 아주 쉽게 이해할 수 있습니다.

![]({{ 'assets/images/2023-06-13/predictive_back_google_io.png' | relative_url }})

>User research shows that people are most frustrated when System Back unexpectedly returns them to Launcher, forcing them to reopen the app.

>유저 리서치에 따르면, 시스템 뒤로가기가 유저가 런처로 이동되고(예상치 못한 동작) 앱을 다시 열게 되는 상황에서 유저들이 가장 실망감을 느낍니다. 


Android 13에서 Predictive back gesture 는 개발자 옵션에서 사용 설정을 할 수 있습니다.
앱에서는 메니페스트에 `enableOnBackInvokedCallback = true` 설정이 필요합니다.
```xml
<application
    ...
    android:enableOnBackInvokedCallback="true"
    ... >
...
</application>
```


Predictive back gesture를 핸들링 하기 위에서는 뒤로가기 네비게이션 이벤트를 인터셉트가 필요한데, 
onBackPressedDispatcher를 사용하거나 OnBackInvokedDispatcher를 사용해야 합니다.



`OnBackPressedDispatcher`
* AndroidX 사용 시 
* AndroidX Activity 1.6.0-alpha05 업데이트 필수

`OnBackInvokedDispatcher`
* AndroidX를 사용하지 않는 경우
* 플랫폼 api

<br>

####  주의사항
![]({{ 'assets/images/2023-06-13/on_back_pressed_caution.png' | relative_url }})

OnBackPressedDispatcher 또는 OnBackInvokedDispatcher 로 업데이트를 하지 않고 개발한 앱을 그대로 사용하면 이슈가 발생할 수 있다는 것으로 해석됩니다.



<br>
<br>
<br>
<br>


## OnBackInvokedDispatcher
Api 33 부터 사용 가능합니다. 때문에, 사용 시 버전 분기가 필요합니다. 
OnBackPressedDispatcher를 사용할 수 없는 경우에 사용합니다. 

OnBackInvokedDispatcher는 `android:enableOnBackInvokedCallback="true"` 로 설정해야 정상적으로 동작합니다. OnBackPressedDisptacher는 이 값과 상관없이 항상 동작합니다. 


```kotlin
@Override
fun onCreate() {
    if (BuildCompat.isAtLeastT()) {
        onBackInvokedDispatcher.registerOnBackInvokedCallback(
            OnBackInvokedDispatcher.PRIORITY_DEFAULT
        ) {
            /**
             * onBackPressed logic goes here. For instance:
             * Prevents closing the app to go home screen when in the
             * middle of entering data to a form
             * or from accidentally leaving a fragment with a WebView in it
             *
             * Unregistering the callback to stop intercepting the back gesture:
             * When the user transitions to the topmost screen (activity, fragment)
             * in the BackStack, unregister the callback by using
             * OnBackInvokeDispatcher.unregisterOnBackInvokedCallback
             * (https://developer.android.com/reference/kotlin/android/window/OnBackInvokedDispatcher#unregisteronbackinvokedcallback)
             */
        }
    }
}
```



<br>




## 참고
* [https://proandroiddev.com/handling-back-press-in-android-13-the-correct-way-be43e0ad877a](https://proandroiddev.com/handling-back-press-in-android-13-the-correct-way-be43e0ad877a)
* [https://oguzhanaslann.medium.com/onbackpressed-deprecated-so-what-to-use-92ddd55fc21d](https://oguzhanaslann.medium.com/onbackpressed-deprecated-so-what-to-use-92ddd55fc21d)
* [https://onlyfor-me-blog.tistory.com/522](https://onlyfor-me-blog.tistory.com/522)

* Add support for the predictive back gesture  |  Android Developers
[https://developer.android.com/guide/navigation/predictive-back-gesture](https://developer.android.com/guide/navigation/predictive-back-gesture)

* Provide custom back navigation  |  Android Developers
[https://developer.android.com/guide/navigation/custom-back](https://developer.android.com/guide/navigation/custom-back)
