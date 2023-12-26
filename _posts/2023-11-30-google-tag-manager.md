---
title: Android 에서 구글 태그 매니저(GTM) 사용하기
categories:
- ETC
tags:
- 기타
- GTM
---

<div class = "notice--success">
비교적 자유롭게 이벤트 수집 설정이 가능한 GTM. 사용 방법을 정리합니다. 
</div>

---


## GTM 기본

[Android용 Google 태그 관리자 / 모바일용 Google 태그 관리자(Android) / Google Developers](https://developers.google.com/tag-platform/tag-manager/android/v5?hl=ko)

공식 문서에서는 아래와 같이 구글 태그 매니저를 설명합니다. 
> 개발자는 애플리케이션 바이너리를 다시 빌드하고 앱 마켓에 다시 제출할 필요 없이 Google 태그 관리자를 사용하여 모바일 애플리케이션에서 측정 태그 및 픽셀을 구현하고 관리할 수 있습니다. Firebase용 Google 애널리틱스 SDK를 사용하는 개발자는 앱이 출시된 후에도 손쉽게 태그 관리자를 추가하여 구현을 관리하고 변경할 수 있습니다.
>
>개발자는 중요한 이벤트를 로깅하고, 실행할 태그 또는 픽셀을 추후에 결정할 수 있습니다.

<br>

즉, 구글 태그 매니저를 이용하면 앱 수정 후 배포 없이 데이터 수집에 관련된 작업을 자유롭게 변경할 수 있습니다.

> **전제 조건**
>1. GA를 사용 중이다.
>2. 수집과 관련된 동작을 변경할 대상 이벤트가 GA로 전송되고 있다.


예를 들어 A 이벤트를 GA로 전송 중이고, A 이벤트가 전송될 때 특정 API를 호출하기를 원합니다. 

이 경우, 
1. A 이벤트를 전송하는 곳에 API 호출 코드를 일일이 추가하거나
2. 이벤트 전송 시, 이벤트가 A인지 확인하는 조건을 추가하여 API를 호출하거나 

등의 방법으로 구현이 가능하지만, API 호출이 더 이상 필요하지 않거나 다른 이벤트에도 같은 동작을 추가하고 싶다면
 다시 한번 코드를 수정해주어야 합니다.

이 때 GTM을 이용하면 코드 수정 없이 GA에 전송되는 이벤트들에 대해서 자유롭게 동작을 추가할 수 있습니다.
(물론 태그매니저 추가는 필요합니다.)


<br>

## 적용하기

GTM을 앱에 적용하려면 Tag와 Trigger, 변수 등을 정의해 둔 컨테이너가 필요하고, 
GTM의 기본 제공 동작 외에 API 호출과 같은 동작을 구현하기 위해서는  CustomTagProvider 가 필요합니다.

#### 컨테이너 설정

트리거 설정
<img src='/assets/images/2023-11-30/gtm_trigger.png'/>

태그 설정
<img src='/assets/images/2023-11-30/gtm_tag.png'/>


<img src='/assets/images/2023-11-30/gtm.png'/>


1. GTM에서 컨테이너를 생성하고,
2. 앱에 GTM을 추가한 뒤, 1에서 생성한 컨테이너 파일을 앱에 추가합니다.
3. 원하는 Tag와 Trigger를 설정하면 최대 12시간 내에 앱에 반영됩니다. 

- Tag - 마케팅 및 분석에 사용되는, 수집하고자 하는 데이터 구분
- Trigger - Tag가 실행되는 시점 정의


<img src='/assets/images/2023-11-30/gtm2.png'/>


#### CustomTagProvider 설정
1. Dependency 를 추가합니다.

``` groovy
dependencies {
  // ...
  compile 'com.google.android.gms:play-services-tagmanager:18.0.3'
}
```

2. CustomTagProvider를 생성합니다. `com.google.android.gms.tagmanager.CustomTagProvider` 를 상속받습니다.

``` kotlin
class MyTagProvider: com.google.android.gms.tagmanager.CustomTagProvider {
  override execute(params: Map<String, Any>) {
    // 구현
  }
```


이제 앱에서 GA로 이벤트를 전송하면, 트리거와 연관된 태그에 정의된 동작을 수행합니다.


<br>

## 컨테이너
### 컨테이너 프리뷰

[Android용 Google 태그 관리자 / 모바일용 Google 태그 관리자(Android) / Google for Developers](https://developers.google.com/tag-platform/tag-manager/android/v5?hl=ko#4_preview_debug_and_publish_your_container)

컨테이너 버전이 적용되기 까지는 꽤 긴 시간이 필요하기 때문에 컨테이너가 정상적으로 동작하는지 바로 확인하기 어렵습니다.

배포 전 프리뷰를 이용해 컨테이너를 미리 사용할 수 있습니다.

1. 초록색 칸 Version 클릭

<img src='/assets/images/2023-11-30/gtm_home.png' />
        
2. 좌측 상단 메뉴 → Preview 클릭
    
<img src='/assets/images/2023-11-30/gtm_preview.png' />
        
3. 프리뷰를 이용할 앱 ID(패키지명)을 작성하고, Generate begin preview link 를 눌러 링크를 만들어줍니다.
    
 <img src='/assets/images/2023-11-30/gtm_preview_start.png' />
        
    
4. 3에서 만든 링크를 애뮬레이터 또는 기기에서 열어줍니다.
5. 링크에 들어가면 Preview ~~ 어쩌구 팝업이 뜨는 경우가 있는데 그냥 Continue 눌러주세요.
6. 로그를 통해 프리뷰를 정상적으로 불러왔는지 확인할 수 있습니다.

 <img src='/assets/images/2023-11-30/gtm_preview_log.png' />
 
7. 프리뷰를 종료하고싶다면 Generate end preview link를 눌러 링크를 생성하고, 애뮬레이터 또는 기기에서 다시 열어줍니다.
    
<img src='/assets/images/2023-11-30/gtm_preview_end.png' />
            

<br>

### 컨테이너 디버깅

터미널에 아래 명령어를 입력하면, GTM 로깅이 가능합니다. 

```json
$ adb shell setprop log.tag.GoogleTagManager VERBOSE

// 종료
$ adb shell setprop log.tag.GoogleTagManager .none.
```

로그캣에 GoogleTagManager로 검색해보면 로그들이 쭉쭉 올라오는데,
Log passthrough event 를 찾아보시면, 커스텀 이벤트가 GA로 잘 전송되었는지를 확인할 수 있습니다.


 <img src='/assets/images/2023-11-30/gtm_log_event.png' />
 
트리거와 태그를 잘 작성했는데 원하는 동작을 하지 않는다면 디버깅을 통해 확인하면 좋을 것 같네요.
