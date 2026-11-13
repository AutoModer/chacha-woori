# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## 프로젝트 개요

**차차, 우리** (영문 부제 Bye & By, Us. 코드·저장소 이름은 `&`를 못 써서 `chacha-woori`) — Pleos Connect(현대자동차그룹 AAOS 기반) 차량용 앱. 대표 문구 "가족의 목소리를 태워 드립니다". **가족이 먼저 다가오고 운전자는 듣고 "응"만 한다**(음성 비서가 아니다). 두 축: **소통** — 가족이 언제든 음성·글 메시지(장소 선택)를 보내면 운전 중엔 바로 소리로, 아니면 시동 켤 때 전하고, 장소가 있으면 [예][아니요][전화하기]를 음성으로 묻는다. **안전** — 고속도로 출구·분기점과 휴식 권유 때만 가족이 미리 녹음한 목소리로 알린다. 위치 공유는 차에서만 시작한다. 현대차그룹 차량용 앱 공모전 출품작이며 **접수 마감은 2026-11-15 23:59**.

- 현재 코드는 뼈대 단계다(`MainActivity`가 앱 이름만 표시). 기획은 `docs/차차우리_통합정리.md`에 정리되어 있다
- 구성: 차량 앱(`app/`, 이 저장소) + 가족 앱(Android, Kotlin, 예정) + Firebase(Auth, Firestore, Storage, Cloud Functions, 예정)
- 2026-11-13에 앱 이름을 "퇴근길"에서 "차차, 우리"로 바꿨다. 이전 컨셉은 고령 운전자 길목 안내 "아빠, 오른쪽"(`papa-right`, 그 전엔 `mom-right`)이었다. 그 길목 안내는 2026-11-13 다시 핵심 기능(D)으로 들어왔다

## 빌드·실행

```
./gradlew installDebug        # 실행 중인 Pleos Connect AVD에 설치
adb logcat                    # 로그 확인
```

- 환경: JDK 17, Gradle 8.11.1, AGP 8.8.2, Kotlin 2.0.0, compileSdk/targetSdk 35, minSdk 28, `useLibrary("android.car")`
- 에뮬레이터: Android Studio SDK Manager → SDK Update Sites에 `https://nexus-playground.pleos.ai/repository/raw-releases/connect-avd/2.x/crp-sys-img.xml` 추가 → API 34 Pleos Connect system image 설치 → Automotive `Pleos Connect v2.1.0` AVD (Cold boot, RAM 4GB)
- 의존성: `ai.pleos.playground:Vehicle:2.3.2`, `ai.pleos.playground:NaviHelper:3.0.5` (공개 Maven `nexus-playground.pleos.ai`, 인증 불필요)
- 테스트 코드는 아직 없다
- 제출 APK: `apk/팀명_앱이름_release.apk`

### 차량 상태 조작 (Virtual Console)

에뮬레이터는 주행 시뮬레이션이 안 되므로(속도·GPS를 바꿔도 내비 위치가 안 움직임) 차량 신호는 Virtual Console로 넣는다.

```
adb forward tcp:8088 tcp:8088
POST http://127.0.0.1:8088/browser/session?client=<임의 ID>   # 먼저 안 하면 503
PUT  /vehicle/gear?gear=Drive                               # 속도 전에 기어 D
PUT  /vehicle/controls?speed=80
PUT  /vehicle/turn-signal?mode=BlinkRight|BlinkLeft|Off
PUT  /vehicle/controls?steering=35                          # SDK에는 -35로 들어옴 (부호 반대)
PUT  /vehicle/body?ambient_brightness=40
GET  /vehicle/state, /vehicle/catalog
```

## 핵심 설계

기준 문서는 `docs/차차우리_통합정리.md`(**최신본**)이다. 다른 문서와 다르면 이 문서를 따른다. 구현할 때 알아야 할 요지:

**구조**: 무거운 처리(녹음 보정, 보조 장소 제안)는 Cloud Functions에서 하고 차량은 재생과 SDK 호출만 한다 (주행 중 CPU 5% 미만, RAM 400MB 이하 기준). 차량 수신은 Firestore 실시간 리스너가 기본이고, FCM은 Play 서비스 유무 확인 후 결정한다. 차량 앱은 포그라운드 서비스로 백그라운드에서 이벤트를 받는다.

**SDK 실측 결과**
- 쓸 수 있음: NaviHelper 경로 이벤트(`onRouteStarted`, `onDestinationArrived`, `onRouteCancelled`), `requestRoute`·`requestRouteWithWaypoints`, 속도 `PERF_VEHICLE_SPEED`(m/s), `TurnSignal`, `SteeringWheel`, `GEAR_SELECTION`, `CarUxRestrictions`(D → requiresDO=true), 앰비언트 조명(차 전체 한 구역), 오디오 포커스
- 쓸 수 있음(2026-11-13): Pleos TextToSpeech·SpeechToText `ON_DEVICE` 모드 — 신청·registerApp 불필요. TTS는 `initialize` onSuccess 뒤에 리스너·speak. STT는 `pleos.car.permission.STT_SERVICE` + `RECORD_AUDIO`(에뮬레이터는 `pm grant --user current`)
- 안 됨: 안드로이드 기본 `SpeechRecognizer`(기본 서비스 Gleo가 결과를 주지 않음), `DRIVER_ATTENTION`·`PRIVATE_CAR_SAFETY`(실차 특수 권한, 쓰지 않음), 차로 정보, 주행 시뮬레이션
- 확인 필요: 차량 위치 API, `IGNITION_STATE`(시동 감지), 전화 걸기(`tel:` 인텐트), Firestore 수신

**동작 규칙**
- 주행 중에는 화면 없이 소리만. 영상·긴 메시지는 기어 P에서만, D로 바꾸면 즉시 정지
- 경로 변경은 항상 운전자가 "응" 해야 실행(STT + 큰 버튼 셋: 예 / 아니요 / 전화하기). 장소는 가족 앱에서 가족이 고른다
- 운전 중이 아닐 때 온 메시지는 모아 두었다가 시동 직후(기어 P) 인사 영상과 함께 전한다
- 위치 공유는 차에서만 시작한다(목적지가 집·가족이 보낸 장소면 출발 때 묻기). 도착(`onDestinationArrived` + 기어 P)하면 자동 종료. 가족 앱에 위치 요청 기능은 없다
- 길목 안내는 고속도로 출구·진입·분기점만, 남은 시간 기준 미리 알림 → 한 번 더(다급하지 않게, 출구 이름은 TTS·방향은 가족 목소리), 깜빡이·조향이 나오면 멈추고 너무 가까우면 "괜찮아, 다음에서 나가면 돼". 안내 중에는 메시지를 미룬다
- 휴식 권유는 졸음 감지가 아니다. 연속 주행 시간·시간대 기준값 + 안전 하한선으로 판단하고, 다음 쉼터 전에 권한다. 개인 습관 학습·조향 패턴은 쓰지 않는다(X1, 나중에 검토). 판단 결과는 차 안에서만 쓰고 가족에게 보내지 않는다
- 인사 영상은 정차(기어 P) 중에만 재생한다
- 도착 인사는 시동을 끄기 직전이 아니라 도착 후 기어 P일 때 재생한다 (시동을 끄면 화면도 꺼짐)

## 표현 규칙 (문서·UI 문구)

- "위치 추적·감시·모니터링" ✗ → "공유·안심"
- "졸음 감지" ✗ → "휴식 권유". "조는 거 아니지?", "일어나!" 같은 의심·다그치는 말 ✗
- "메신저 앱" ✗ → "가족의 목소리를 태워 주는 차량 앱"
- 길목 안내는 "내비를 대신하지 않는 주의 알림" ("길 안내 앱" ✗, 단일 경로 정책)
- "AI가 차를 조작" ✗ → 실행은 항상 운전자 "응"

## 문서

- `docs/차차우리_통합정리.md` — 최신 기준. 기능 ID(A 가족 메시지, B 위치 공유, C 인사 영상, D 길목 안내·휴식 권유, E 가족 앱, F 시스템, X 발전 계획). 11장은 결정 기록
- `docs/기획서_작성가이드.md` — 공모전 제출 양식 초안, PPT 구성, 예상 질문
- `docs/근거_수집목록.md` — 근거 ID(A~G)와 출처
- `docs/공모전_가이드_정리.md` — 공모전 규정 (컨셉과 무관하게 유효)
- `docs/mockup/화면시안.html` — 차량 앱·가족 앱 정적 시안

기능 ID·근거 ID는 문서끼리 서로 참조하므로 바꿀 때 함께 갱신한다.

## 커밋 규칙

`<타입>: <한국어 개조식 요약>` 한 줄 + 빈 줄 + `- `로 시작하는 본문 항목. 타입 예: `refactor`, `docs`, `feat`, `fix`.
