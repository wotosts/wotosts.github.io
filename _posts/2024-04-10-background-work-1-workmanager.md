---
title: "[Android] 백그라운드 작업과 알림 사용하기1 - WorkManager"
toc: true
categories:
- Android
tags:
- BackgroundWork
- WorkManager
- AlarmManager
---

## Intro
작년 여름 배포 이후 틈틈히 업데이트 중이던 사이드 프로젝트 `사부작` 의 개발은 더 이상 진행하지 않는 것으로 팀원들과 협의했다.
별 기능이 없어보일 수도 있겠지만 `사부작`을 개발하면서 실무에서 사용해보지 않았던 것, 사용해보고 싶었던 것들을 이것저것 해보았는데 아쉽게 되었다. 
특히 알림, 백그라운드 작업에 대한 이해도를 많이 높일 수 있었는데, 이 부분들에 대해 정리해보고자 한다.



주요 요구 사항 2가지
1. 매일 통계 데이터를 갱신해야 한다.
2. 실행 시간에 맞추어 사용자에게 노티 알림을 보내야 한다. 


두 가지 모두 앱이 실행되지 않은 상태에서도 수행되어야 하기 때문에 백그라운드 작업으로 보고, 관련 문서를 다시 열심히 읽었었다. 백그라운드 작업 관련 문서는 생각보다 자주 읽게 되는 것 같다.
최종적으로 1번 요구 사항은 WorkManager, 2번 요구 사항은 AlarmManager를 이용하여 개발하였다. 그 이유는 아래와 같다.

1. **매일 통계 데이터를 갱신해야 한다.**
    
    이 경우 특정 시간에 작업이 수행되도록 해야 하기 때문에 AlarmManager를 사용할 수도 있으나  
   사용자가 앱을 실행했을 때에만 계산된 데이터가 보이면 되기 때문에, 앱이 백그라운드에 있거나 실행되지 않은 상태에서는 굳이 정확한 시간을 지킬 필요가 없다. 그냥 그 시간 즈음에 작업이 수행되기만 하면 된다. 
    
    AlarmManager를 이용하여 inexact repeating 알림을 발생시켜서 그 때 workmanager를 수행하는 것도 괜찮은 방법이었을 것 같다. 이러면 또 부팅 시마다 알림을 등록하는 코드를 추가해야 하지만...
    
2. **실행 시간에 맞추어 사용자에게 노티 알림을 보내야 한다.**
    
    실행 시간에 맞춘다 = 정확한 시간에 동작하는 것이 필요하기 때문에 AlarmManager를 사용했다.
		
이번 글에서는 백그라운드 작업 개요와 WorkManager 사용을 다룬다. 
다만... 오래 전에 정리해 둔 글을 포스팅하는 것이기 때문에 그 사이에 공식 문서가 업데이트 되어 현재는 찾을 수 없는 내용이 있을 수 있다.
    

## Background 실행? Background 작업?
공식 문서 - [Background Work Overview  |  Android Developers](https://developer.android.com/guide/background)  
백그라운드 작업과 앱이 백그라운드에서 실행된다는 것은 차이가 있다. 

### Background 실행
안드로이드에서 아래 2가지 조건을 모두 만족한다면 백그라운드에서 앱이 동작한다고 정의한다. 

- 유저에게 앱의 어떤 활동(Activity, UI) 도 노출되지 않음
- 동작하는 포그라운드 서비스가 없음

### Background 작업
앱이 작업을 수행할 때, 
[공식 문서](https://developer.android.com/develop/background-work/background-tasks?_gl=1*1ogpca0*_up*MQ..*_ga*NTI0OTkwMDY3LjE3MTI3NTY3ODQ.*_ga_6HH9YJMN9M*MTcxMjc1Njc4My4xLjAuMTcxMjc1Njc4My4wLjAuMA..)에서는 적절한 방법 선택을 위하여 2가지 상황에 따른 결정 트리를 제공하고 있다.

1. 작업이 사용자에 의해 시작된 경우
2. 작업이 앱 내외부에서 이벤트 응답에 의해 시작된 경우

2가지 상황에서 결정 트리는 조금 다르지만, 중요하게 고려해야 하는 것은 작업 수행 시간과 지연 가능성이다.  
이에 따라 작업을 `비동기`, `백그라운드`, `Foreground Service` 및 `기타 방식`으로 처리할 수 있다. 

결정 트리에 따르면,   
사부작은 작업이 앱 내외부 이벤트에 의해 시작되었다고 볼 수 있고, 지연 가능하며, Foreground Service를 사용할 필요가 없으므로 **백그라운드 작업**으로 처리한다.

백그라운드 작업 또한 몇가지 유형으로 나눌 수 있다.  
 `즉시`, `지연`, `긴 작업 시간`의 3가지 유형과, 앱이 재시작되거나 기기가 재부팅되어도 유지되어야 하는 작업(`Persistent`)과, 그렇지 않은 작업(`Impersistent`) 유형이다.

<img src="https://developer.android.com/static/images/guide/background/background.svg" />
<br>

모든 Persistent 작업은 `WorkManager`,  즉시 실행하는 Impersistent 작업은 `coroutine` 등을 이용하며, 실행 시간이 길거나, 지연되는 impersistent 작업은 하지 말아야 한다. 이런 경우는 WorkManager를 이용하여 persistent 작업으로 변경하는 것을 권장하고 있다. 

| 작업 유형 | Persistent | Impersistent |
| --- | --- | --- |
| 즉시 | WorkManager | async |
| 긴 실행 시간 | WorkManager | Persistent 작업으로 바꾸어 WorkManager 사용 |
| 지연 | WorkManager | Persistent 작업으로 바꾸어 WorkManager 사용 |


업데이트 전(2024.4.10 이전의 문서) 한국어 문서에는 

- 즉시
- 지연
- 긴 작업 시간

이 세가지 작업 유형 외에 `정시` 라는 유형이 하나 더 있고, 이 유형은 AlarmManager를 이용하라고 가이드하고 있는데, 영문 문서에서는 알림을 백그라운드 작업은 아닌 특수한 케이스라고 가이드한다. 

때문에, AlarmManager는 백그라운드 작업에는 사용해서는 안된다. 

> You should only use AlarmManager **only** for scheduling exact alarms such as alarm clocks or calendar events. When using AlarmManager to schedule background work, it wakes the device from Doze mode and its use can therefore have a negative impact on battery life and overall system health
>


## Doze mode
WorkManager를 사용하기 전, 도즈 모드에 대해 알아둘 필요가 있다.   
[Optimize for Doze and App Standby  |  Android Developers](https://developer.android.com/training/monitoring-device-state/doze-standby)

배터리 성능 향상(사용량 감소)를 위해 Android 6.0 부터 도입된 기능으로, 기기가 충전기에 연결된 상태가 아니라면 일정 시간동안 동작하지 않을 경우 도즈 모드에 진입한다.

도즈 모드에 진입하면

- 네트워크 통신
- wake lock
- alarm manager
- wifi 검색
- JobSchedular

등 배터리가 소모되는 작업을 maintenance window(유지보수 기간)으로 미루게 되고, 중간 중간의 maintenance window 기간에 작업들을 수행한다. 

- ACTIVE
- INACTIVE - 화면 꺼짐, 기기 활성화
- IDLE_PENDING - 도즈모드 직전
- IDLE - 도즈모드
- IDLE_MAINTENANCE - 도즈모드 중 maintenance window


### 도즈 모드 테스트

```xml
// 도즈 모드 강제 진입
$ adb shell dumpsys deviceidle force-idle

// 도즈 모드 종료
$ adb shell dumpsys deviceidle unforce

// 기기 활성화
$ adb shell dumpsys battery reset
```

### 도즈모드 vs 앱 대기 모드

- 도즈 모드는 화면이 꺼지고 일정 시간 지나면 네트워크 미연결, 배터리 절약 등 수행 - 시스템 전체
- 앱 대기 모드는 앱을 오래 사용하지 않을 때 - 단일 앱. 네트워크 연결 등을 막고 배터리 소모 방지

둘 다 전원이 연결된 상태에서는 해당 사항이 없다. 


## WorkManager
앱 실행 중 바로 실행되어야 하는 백그라운드 작업이 아니라면 대부분 `WorkManager`를 사용하는 것이 권장되기 때문에, 사부작에서 WorkManager를 사용하였다.

WorkManager는 백그라운드 작업을 일관성 있게 수행할 수 있게 해주며, OS 버전에 따라 적절한 API를 사용한다.

>WorkManager uses an underlying job dispatching service when available based on the following criteria:
>
>- Uses JobScheduler for API 23+
>- Uses a custom AlarmManager + BroadcastReceiver implementation for API 14-22

WorkManager는 도즈 모즈가 해제되는 시간(maintenance window)에 등록된 작업을 수행하기 때문에 정확한 시간에 실행되는 것이 보장되지 않는다. 만약 정확한 시간에 맞추어 동작할 필요가 있다면 AlarmManager를 사용해야 한다.


WorkManager 사용 시 이점은 아래와 같다.

- 작업 체인
    - 여러 작업을 순차적으로 또는 동시에 수행할 수 있음
    - 이전 작업 -> 이후 작업으로 데이터를 전달할 수 있음
- 작업 제약 조건 설정
- 작업 재시도 가능
- 앱 재부팅시 작업 재등록
    - 확인해보니 기기 DB에 작업이 저장된다.
- 장치의 상태 존중하여, 적절한 타이밍에 동작
    - 기기의 추가적인 배터리 소모를 줄임(도즈 모드 참고)

그리고 작업 반복 유무에 따라 `OnTime`, `Periodic` Work로 구분된다.

- OneTime Work : 1회성 작업
- Periodic Work : 주기적으로 반복해야 하는 작업

### 사용 방법
WorkManager 사용을 위해 필요한 클래스와 역할이다.
- `Worker`
    - 실제로 수행할 작업의 코드를 작성하는 추상 클래스
    - doWork()에서 Result 반환. 반환값에 따라 재시도 여부 결정
- `WorkRequest`
    - 요청할 작업의 제약조건, 데이터를 설정하는 등 정보가 담긴 작업 클래스
- `WorkManager`
    - 작업을 큐에 넣고 처리, 관리하는 클래스

<br>

이제 코드를 작성해보자.

1. `Worker` 를 확장하여 원하는 작업을 하는 클래스 정의
    
    Worker.doWork()는 기본적으로 백그라운드 스레드에서 수행된다.
    코틀린 사용 시 Coroutine의 이점을 최대한 누릴 수 있는 CoroutineWorker를 권장한다.  
    CoroutineWorker는 기본적으로 Dispatcher.Default 로 동작하지만, 필요에 따라 적절한 Dispatcher를 사용하면 된다.
    
    ```kotlin
    class MyWork(appContext: Context, workerParams: WorkerParameters):
           Worker(appContext, workerParams) {
       override fun doWork(): Result {
    
           // Do the work here--in this case, upload the images.
           doSomething()
    
           // Indicate whether the work finished successfully with the Result
           return Result.success()
       }
    }
    ```
    
2. `WorkRequest` 생성
    
    ```kotlin
    val constraints = Constraints.Builder()
       .setRequiredNetworkType(NetworkType.UNMETERED)
       .setRequiresCharging(true)
       .build()
    
    // one time
    val myWorkRequest: WorkRequest =
       OneTimeWorkRequestBuilder<MyWork>()
           .setConstraints(constraints) // 제약 조건 설정
    			 .addTag("work") // tag 설정
    			 .setInitialDelay(10, TimeUnit.MINUTES) // 지연 작업일 경우, 지연 시간 설정
    			 .setInputData(workDataOf(    // 작업에 전달할 데이터 설정
    	       "IMAGE_URI" to "http://..."
    			  ))
           .build()
    
    // periodic
    val saveRequest =
           PeriodicWorkRequestBuilder<MyWorker>(1, TimeUnit.HOURS)
        // Additional configuration
               .build()
    ```
    
    WorkRequest를 생성한다. 이 때 필요에 따라 여러 조건이나 값을 설정할 수 있다.
    
3. 작업 예약
    
    생성한 WorkRequest를 `WorkManager` 큐에 전달한다.
    
    ```kotlin
    WorkManager
        .getInstance(myContext)
        .enqueue(myWorkRequest)
    ```
    
#### Hilt와 함께 사용
DI로 Hilt를 사용할 경우에는 Initializer 설정이 필요하다.  
[맞춤 WorkManager 구성 및 초기화  |  Android Developers](https://developer.android.com/topic/libraries/architecture/workmanager/advanced/custom-configuration?hl=ko#remove-default)


* Manifest.xml 

```xml
<provider
    android:name="androidx.startup.InitializationProvider"
    android:authorities="${applicationId}.androidx-startup"
    tools:node="remove">
 </provider>
```

* Initializer  

```kotlin
class CustomApplication : Application(), Configuration.Provider {
    @Inject
    lateinit var workerFactory: HiltWorkerFactory

	override fun getWorkManagerConfiguration() =
	    Configuration.Builder()
	        .setWorkerFactory(workerFactory)
	        .build()
}
```

* Worker   

```kotlin
@HiltWorker
class CustomWorker @AssistedInject constructor(
    @Assisted appContext: Context,
    @Assisted workerParams: WorkerParameters,
) : Worker(appContext, workerParams) {

    override fun doWork(): Result {
TODO("Not yet implemented")
    }
}
```

#### 작업 등록 확인
아래 커맨드를 이용해 실제로 작업이 등록되었는지 확인할 수 있다.
터미널 로그 중간의 Minimum latency가 다음 작업 수행까지 남은 시간을 의미한다.

```kotlin
adb shell dumpsys jobschedular
```

<img src = "/assets/images/2024-04-10/workmanager.png">



### Usecase 
사부작은 매일 1번씩 통계 데이터를 갱신한다. 즉, 하루에 1번 특정 시간에 주기적으로 백그라운드 작업이 필요하다는 의미이다. 여기에 재부팅 시에도 동작해야 하고 지연되어도 상관없기 때문에 WorkManager가 적절한 선택이다. 

WorkManager 적용 시 PeriodickWork를 사용할 수도 있었으나, 중요한 것은 WorkManager는 기기가 도즈 모드인 경우에 maintenance window 기간에 실행된다는 것이다. 
실행 주기를 24시간으로 지정하여 등록해도, maintenance window가 작업 수행을 원하는 특정 시간에 겹쳐있을 것이라는 보장이 없다. 때문에 정확한 시간에 작업을 수행하는 것이 보장되지 않으며, 작업 수행 후 다음 작업을 예약하는 방식이기 때문에 작업 예약 주기 오차가 커질 수 밖에 없다.

예를 들어, 작업이 오늘은 12시 10분에 수행되었으나, 내일 작업이 12시 15분에 수행되었다면 다음 작업은 12시 15분 기준 24시간 뒤로 예약된다. 

PeriodicWork는 적절하지 않다는 것이다. 따라서 OneTimeWork로 현재 실행된 시간을 확인 후, 주기가 늦춰지지 않게 시간 차를 계산하여 새로운 OneTimeWork를 생성하는 방법을 사용하였다.  
참고 - [WorkManager Periodicity](https://medium.com/androiddevelopers/workmanager-periodicity-ff35185ff006)

```kotlin
fun getTimeUsingInWorkRequest(): Long {
    val currentDate = Calendar.getInstance()
    val dueDate = Calendar.getInstance()

    dueDate.set(Calendar.HOUR_OF_DAY, 0)
    dueDate.set(Calendar.MINUTE, 1)
    dueDate.set(Calendar.SECOND, 0)

    if (dueDate.before(currentDate)) {
        dueDate.add(Calendar.HOUR_OF_DAY, 24)
    }

    return dueDate.timeInMillis - currentDate.timeInMillis
}

fun WorkManager.enqueueTimeChecker() {
    val request =
        OneTimeWorkRequestBuilder<CheckerWorker>()
            .setInitialDelay(getTimeUsingInWorkRequest(), TimeUnit.MILLISECONDS)
            .build()

    enqueueUniqueWork(
        "StateChecker",
        ExistingWorkPolicy.REPLACE,
        request,
    )
}
```
