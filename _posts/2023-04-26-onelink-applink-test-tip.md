---
title: 앱링크/원링크 테스트 개발 환경에서 제대로 동작하지 않을 때
categories:
- Android
tags:
- applink
- onelink
- tip
---

제목 그대로...

---

### 문제 상황

앱링크/원링크를 테스트 하는 중, 분명히 맞는 주소를 사용했음에도 불구하고
* 앱이 열리지 경우
* 스토어로 이동하지 않는 경우
* 프로덕션 앱의 설치 링크(플레이 스토어)가 열리는 경우

등의 문제가 발생할 수 있습니다.

<br>
<br>

### 해결 방법
이런 경우 `지원되는 웹 주소` 설정을 변경하면 정상적으로 테스트가 가능합니다.

> 갤럭시 기준
>* 설정 → 애플리케이션 → 기본 앱 선택 → 링크 열기 → 링크를 열 수 있는 앱 -> 테스트할 앱 선택 -> 지원되는 웹 주소
>* 설정 → 애플리케이션 → 테스트할 앱 선택 → 기본으로 설정 -> 지원되는 웹 주소


지원되는 웹 주소 목록 중 테스트하고자 하는 주소가 Off 상태라면 On 으로 변경해주세요.


<br>

#### 유의 사항
프로덕션 앱의 설정을 먼저 Off 로 변경하신 후에 테스트 환경 앱의 설정을 변경하는 것이 좋습니다.
반대로 해봤는데... 설정값 적용이 되지 않는 경우가 있었습니다.

예시로, Galaxy Store와 Galaxy Themes 를 봅시다.

<img src = "/assets/images/2023-04-26/link_tip1.jpeg" width = "30%"/>{: .align-center}


두 앱은 동일한 링크를 지원합니다.
(apps.samsung.com)

Galaxy Store | Galaxy Themes
:---:|:---:
<img src="/assets/images/2023-04-26/link_tip2.jpeg" width = "70%"/>{: .align-center}|<img src="/assets/images/2023-04-26/link_tip4.jpeg" width = "70%"/>{: .align-center}
<img src="/assets/images/2023-04-26/link_tip3.jpeg" width = "70%"/>{: .align-center} |<img src="/assets/images/2023-04-26/link_tip6.jpeg" width = "70%"/>{: .align-center}

<br>
Galaxy Store는 웹 주소에 On/Off 스위치가 없는데,   
Galaxy Themes 앱에서 링크를 지원하고 싶다면
Galaxy Store의 지원되는 링크 열기를 Off 로 설정해주세요.

<img src="/assets/images/2023-04-26/link_tip5.jpeg" width = "30%"/>{: .align-center}

---


이 방법으로 해결되면 베스트,
다른 방법으로 해결하셨다면 저도 궁금하니 댓글에 남겨주세요, 부탁드립니다! 🙏🏻

가장 좋은 방법은 테스트 앱에서 사용하는 주소와 프로덕션 앱에서 사용하는 주소를 분리하는 것이라고 생각되네요.
