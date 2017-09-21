---
layout: post
title: 안드로이드 백그라운드 서비스 개발에서 고려할 점
subdir: android-background-service-checklist
thumbnail: android-background.png
comments: true
---

지난 시간엔 사운들리 백엔드에 대해 설명을 드렸었죠. 이번 시간엔 사운들리 서비스중 클라이언트에 해당하는 안드로이드 SDK, 그 중에서도 **백그라운드 서비스**에 초점을 맞추어 설명을 해 볼까 합니다.

안드로이드의 특징 중 하나로 **Service** 를 들 수 있습니다. 이 서비스란 녀석은 백그라운드에서 실행 될 수 있다는 점이 가장 큰 특징인데요. 물론 iOS 에서도 일부 지원은 합니다만 매우 제한적인 경우(음악 재생 등)에만 사용 가능합니다.

![img]({{ site.url }}/assets/images/{{ page.subdir }}/android-background.png)

제가 생각하는 백그라운드 서비스 개발 시 유의 사항은 아래와 같습니다.

- **동작 기간** - 상시 동작 해야 하는가, 특정 조건에서 특정 작업을 할때만 동작 해야 하는가
- **글로벌 프로세스 사용 유무** - 서로 다른 어플리케이션에서 접근이 가능 해야 하는가
- **동작 조건** - 특정 시간 혹은 기간마다 동작 해야 하는가, 특정 이벤트 발생시 동작 해야 하는가

그 외에도 많은 부분들이 있지만 일단 저 정도만 고려해도 개인적인 생각으로는 충분히 개발 가능하다고 생각 합니다.

그러면 각각에 대해서 좀 더 자세하게 알아 볼까요?

## 1. 동작 기간

동작 기간에 대해서 이야기 하기 전에 먼저 유저 레벨에서 가장 많이 사용하는 Service 와 IntentService 의 차이점에 대하여 짚고 넘어가보겠습니다.

### Service를 상속

  - [`Context#startService`](http://developer.android.com/intl/ko/reference/android/content/Context.html#startService(android.content.Intent)), [`Context#stopService`](http://developer.android.com/intl/ko/reference/android/content/Context.html#stopService(android.content.Intent)) 를 통해 수명 조절
  - [`Context#bindService`](http://developer.android.com/intl/ko/reference/android/content/Context.html#bindService(android.content.Intent, android.content.ServiceConnection, int))(+[`BIND_AUTO_CREATE`](http://developer.android.com/intl/ko/reference/android/content/Context.html#BIND_AUTO_CREATE)),[`Context#unbindService`](http://developer.android.com/intl/ko/reference/android/content/Context.html#unbindService(android.content.ServiceConnection)) 를 통해 수명 조절 (Service 내에서 [`Service#stopSelf`](http://developer.android.com/intl/ko/reference/android/app/Service.html#stopSelf()) 를 호출하는 방법도 있습니다.)
  - Service 시작된 이후에 커뮤니케이션 가능
  - 수명 종료 API(stopService or unbindService) 를 호출 하기 전에는 프로세스가 사라지지 않음 (물론 LMK에 의해서 종료 된다던지 등등이 있지만 여기서는 논외로 하겠습니다.)

### IntentService 를 상속

  - startService 를 통해 서비스를 시작함
  - 사용자가 따로 수명 관리를 할 필요가 없음

상기 특징을 보면 Service 는 상시 동작하는 서비스에, IntentService 는 특정 조건에서 동작하는 서비스에 더 특화된 것을 볼 수 있습니다.

사운들리 서비스는 백그라운드에서 상시 신호를 감지해야 하므로 Service 를 상속해서 쓰고 있습니다.

## 2. 글로벌 프로세스 사용 유무

안드로이드 컴포넌트 속성 중 [android:process](http://developer.android.com/intl/ko/guide/topics/manifest/application-element.html#proc)  속성을 소문자로 시작하는 이름을 쓰면 글로벌 프로세스로 사용 할 수 있습니다. 글로벌 프로세스니까 당연히 다른 어플리케이션에서도 접근이 가능하답니다.

아래와 같은 경우에는 글로벌 프로세스를 사용 할 때 더 이점이 있습니다.

1. 불필요한 리소스 사용 자제 - 서버와 통신하는 모듈의 경우, 여러 앱에서 동일한 모듈이 사용 될 때 하나의 통로만 사용 하는 것이 네트워크 리소스를 적게 먹습니다.
2. 공유 불가능한 자원 사용 - 사운들리 SDK 가 이 경우에 해당합니다. 비가청 대역 음파를 사용하는 특성상 마이크를 사용 해야 하나 안드로이드에서는 서로 다른 어플리케이션 간의 마이크 공유가 불가능합니다.

하지만 일반적인 어플리케이션 서비스는 굳이 글로벌 프로세스를 쓸 필요가 없습니다. 모듈 버전에 따른 실행, 데이터 공유 등 골치 아픈게 이것저것이 아니에요... ㅠ

## 3. 동작 조건

동작 조건은 크게 time base 와 event base 로 나눌 수 있는데요. 각각의 경우에 서비스를 동작 시킬 트리거를 다르게 쓰는 것이 좋습니다.

동작 조건에 따라 안드로이드에서 사용 가능한 트리거는 아래와 같습니다.

- 시간 기반 (time base)
  - [AlarmManager](http://developer.android.com/intl/ko/reference/android/app/AlarmManager.html) 의 alarm API (set, setExact, setExactAndAllowWhileIdle 등)
  - Android System Broadcast ([ACTION_TIME_TICK](http://developer.android.com/intl/ko/reference/android/content/Intent.html#ACTION_TIME_TICK) 등)
  - GCM Message
- 이벤트 기반 (event base)
  - Android System Broadcast ([ACTION_SCREEN_ON](http://developer.android.com/intl/ko/reference/android/content/Intent.html#ACTION_SCREEN_ON), [ACTION_POWER_CONNECTED](http://developer.android.com/intl/ko/reference/android/content/Intent.html#ACTION_POWER_CONNECTED) 등)
  - GCM Message
  - 그 외 각종 어플리케이션 사용시 발생되는 이벤트

위에서 이야기한 것을 표로 정리하면 아래와 같습니다.

| 동작 기간     | 동작 조건                                    | 사용해야할 서비스                                | 동작 트리거                                   | 그 외                                      |
| --------- | ---------------------------------------- | ---------------------------------------- | ---------------------------------------- | ---------------------------------------- |
| 상시동작      | 시간기반 동작                                  | Service 를 상속 받아 startService 서비스 시작bindService 를 통해 서비스와 연결하여 커뮤니케이션해당 Service 는 [START_STICKY](http://developer.android.com/intl/ko/reference/android/app/Service.html#START_STICKY) 로 실행 | AlarmManager 혹은 서버에서 주기적으로 동작하는 GCM Message 사용 | 글로벌 프로세스를 사용 해야 한다면 android:process 속성을 사용 |
| 이벤트 기반 동작 | System Broadcast 혹은 GCM Message, [JobService](http://developer.android.com/intl/ko/reference/android/app/job/JobService.html) 등을 사용 |                                          |                                          |                                          |
| 작업 할때만 동작 | 시간 기반 동작                                 | IntentService 를 상속받아 startservice 로 실행Intent 에 작업 관련된 파라매터를 전달 | AlarmManager 혹은 서버에서 주기적으로 동작하는 GCM Message 사용 |                                          |
| 이벤트 기반 동작 | System Broadcast 혹은 GCM Message, JobService 등을 사용 |                                          |                                          |                                          |

## Etc. 유의해야 할 부분

추가로 백그라운드 서비스 개발 시 유의해야 할 점들을 기술 해 보겠습니다.

### i) 배터리 절전 기술

안드로이드 버전이 올라갈수록, 그리고 벤더들의 기술력이 높아질수록 배터리 절전 기술 역시 발전합니다. 이러한 기술들은 사용자 입장에서는 반가운 기술이지만 개발자들에게는 종종 절망을 선사합니다 ㅜㅜ

사운들리 서비스도 개발 과정에서 각종 절전 기술 때문에 고생을 했는데요, 크게 고생한 기술 및 특징은 아래와 같습니다.

#### Doze

- Android 6.0 이후 버전에 적용
- 아래의 상태에서 일정 시간 이후 Doze 모드 진입
  - 충전 중이 아님
  - 스크린 꺼져 있음
  - 일정 수치 이상 움직이지 않음
- 제한되는 사항
  - AlarmManager 의 Alarm
  - JobService
  - WakeLock 무시
  - 네트워크 접근 제한
- 회피 방법
  - AlarmManager#setExactAndAllowWhileIdle() - Doze 에서도 동작하지만 최대 15분에 한 번씩만 동작 가능
  - GCM high priority message

#### App Standby

- Android 6.0 이후 버전에 적용
- 일정 기간 동안 아래 상황 중 하나도 발생하지 않은 경우 시스템에서 해당 앱을 standby state 로 간주
  - 명시적 앱 실행
  - 액티비티나 서비스가 포그라운드(전경)에서 실행 중, 혹은 포그라운드에서 실행 중인 앱이 해당 앱의 컴포넌트 사용중
  - 알림을 생성하고 유저가 잠금 화면이나 알림 트레이에서 확인한 경우
- 제한되는 사항
  - 네트워크 사용 및 동기화 기능 사용 불가
- 회피방법
  - 유저와 상호 작용
  - 유저가 디바이스 충전

#### 삼성 스마트 매니저

- 삼성에서 킷캣 (안드로이드 4.4) 이후의 모델 (일부 제외)에 적용
- 일정 시간 이상 유저가 사용하지 않은 앱은 알림 생성 불가

#### 관련 글 : [구글 개발자 블로그 - Android M 관련 변경점](https://developers-kr.googleblog.com/2015/08/testyourapponandroid60.html)

### ii) LMK (Low Memory Killer)

- 안드로이드의 각각의 프로세스는 특성에 따라 상태가 부여됨
- 각 상태는 제한되는 메모리 사이즈가 정해져 있고, 디바이스의 가용 메모리가 해당 사이즈 이하로 떨어질 시 시스템에서 프로세스를 종료
- START_STICKY 로 실행한 서비스의 경우 일정 시간 이후에 null Intent 를 가진채로 재시작
- 킷캣 이상에서 PID가 0이 된 채로 남아있는 버그가 있음
  - [ActivityManager#getRunningServices](http://developer.android.com/intl/ko/reference/android/app/ActivityManager.html#getRunningServices(int)) 에서 서비스 리스트를 가져 왔을때 찌꺼기가 존재

## 마치며

보기엔 복잡해 보이지만 사실 서비스 기획에 맞게 기능들을 골라서 쓰기만 하면 되니까 생각보단 복잡하진 않습니다. **'사용자 중심의 나이스한 서비스 기획'** 만 있으면 위의 표에서 기능을 골라서 조립만 하면 됩니다.

물론 실제 개발 시에는 훨씬 더 고민 해야 될 부분이 많을 겁니다. 네트워크 트래픽도 최소화 해야 하고, WakeLock 도 적절히 써야 하고, 글로벌 프로세스 사용시는 DB 동기화도 시켜야 하고 GCM 은 downstream 이냐  group 이냐 topic 이냐 등등...

개인적인 전망으론 장기적으로 Google 에서도 iOS 처럼 백그라운드 서비스 사용에 점점 제한을 둘 것 같습니다. 하지만 완전히 없애진 않을 것 같네요. 나름 특색 이니까요. 그러니 없애지만 않으면 방법을 찾아 낼 수 있을 겁니다.

너무 두서없이 주저리주저리 쓴 글 같지만 조금이라도 도움이 되었으면 좋겠습니다.
