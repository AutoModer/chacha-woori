# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## 프로젝트 개요

**퇴근길 (Way Home)** — Pleos Connect(현대자동차그룹 AAOS 기반) 차량용 앱. 운전하는 40~50대 가장과 가족을 이어 준다. **가족이 먼저 다가오고 아빠는 듣고 "응"만 한다**(음성 비서가 아니다). 퇴근 출발하면 위치가 공유되고 가족에게 알림이 가며, 가족은 영상·음성 메시지에 장소를 붙여 보낸다. 차는 소리로 전하고 "목적지를 바꿀까요?"를 묻고, "응" 하면 내비 목적지(내부적으로는 경유지)를 바꾼다. 시동·도착 때 가족 영상 인사, 들른 장소에서 가족 한마디, 오래 달리면 가족 목소리 휴식 권유를 재생한다. LLM 의도 파악은 장소를 안 붙인 메시지를 돕는 보조 기능이다. 현대차그룹 차량용 앱 공모전 출품작이며 **접수 마감은 2026-11-15 23:59**.

- 현재 코드는 뼈대 단계다(`MainActivity`가 앱 이름만 표시). 기획은 `docs/퇴근길_통합정리.md`에 정리되어 있다
- 구성: 차량 앱(`app/`, 이 저장소) + 가족 앱(Android, Kotlin, 예정) + Firebase(Auth, Firestore, Storage, Cloud Functions, 예정)
- 이전 컨셉은 고령 운전자 길목 안내 "아빠, 오른쪽"(`papa-right`, 그 전엔 `mom-right`)이었다. 그 판단 엔진은 확장 기능 X1로만 남아 있다

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

기준 문서는 `docs/퇴근길_통합정리.md`(**최신본**)이다. 다른 문서와 다르면 이 문서를 따른다. 구현할 때 알아야 할 요지:

**구조**: 무거운 처리(LLM 의도 파악, TTS 음성 파일 생성, 녹음 보정)는 Cloud Functions에서 하고 차량은 재생과 SDK 호출만 한다 (주행 중 CPU 5% 미만, RAM 400MB 이하 기준). 차량 수신은 Firestore 실시간 리스너가 기본이고, FCM은 Play 서비스 유무 확인 후 결정한다. 차량 앱은 포그라운드 서비스로 백그라운드에서 이벤트를 받는다.

**SDK 실측 결과**
- 쓸 수 있음: NaviHelper 경로 이벤트(`onRouteStarted`, `onDestinationArrived`, `onRouteCancelled`), `requestRoute`·`requestRouteWithWaypoints`, 속도 `PERF_VEHICLE_SPEED`(m/s), `TurnSignal`, `SteeringWheel`, `GEAR_SELECTION`, `CarUxRestrictions`(D → requiresDO=true), 앰비언트 조명(차 전체 한 구역), 오디오 포커스
- 안 됨: Pleos TTS SDK(에뮬레이터 내부 오류 → 서버 TTS 음성 파일로 대체), `DRIVER_ATTENTION`·`PRIVATE_CAR_SAFETY`(실차 특수 권한, 쓰지 않음), 차로 정보, 주행 시뮬레이션
- 확인 필요: `SpeechRecognizer`, 차량 위치 API, 도착 예정 시간, `IGNITION_STATE`, Play 서비스

**동작 규칙**
- 주행 중에는 화면 없이 소리만. 영상·긴 메시지는 기어 P에서만, D로 바꾸면 즉시 정지
- 위치 공유·경로 변경은 항상 운전자가 수락해야 실행(음성 "응" + 큰 버튼 병행). 장소는 가족 앱에서 가족이 고르고, LLM이 임의로 정하지 않는다
- LLM 실패 시 메시지만 읽고 질문은 생략한다
- 위치 공유는 도착(`onDestinationArrived` + 기어 P)하거나 정해진 시간이 지나면 자동 종료한다
- 휴식 권유는 졸음 감지가 아니다. 연속 주행 시간·시간대 기준값 + 안전 하한선으로 판단하고, 다음 쉼터 전에 권한다. 개인 습관 학습·조향 패턴은 쓰지 않는다(X6, 나중에 검토). 판단 결과는 차 안에서만 쓰고 가족에게 보내지 않는다
- 장소 한마디·인사 영상은 정차(기어 P) 순간에만 재생한다
- 도착 인사는 시동을 끄기 직전이 아니라 도착 후 기어 P일 때 재생한다 (시동을 끄면 화면도 꺼짐)

## 표현 규칙 (문서·UI 문구)

- "위치 추적·감시·모니터링" ✗ → "공유·안심"
- "졸음 감지" ✗ → "휴식 권유". "조는 거 아니지?", "일어나!" 같은 의심·다그치는 말 ✗
- "메신저 앱" ✗ → "가족과 이어 주는 차량 앱" (주행 중 알림 정책 대응)
- "AI가 차를 조작" ✗ → AI는 이해하고 묻기까지, 실행은 운전자 수락

## 문서

- `docs/퇴근길_통합정리.md` — 최신 기준. 기능 ID(A 위치 공유, B 장소 붙인 메시지·목적지 변경, C 답·정차 화면, D 출발·도착 인사와 장소 한마디, E 휴식 권유, F 가족 앱, G 시스템, X 확장)
- `docs/기획서_작성가이드.md` — 공모전 제출 양식 초안, PPT 구성, 예상 질문
- `docs/근거_수집목록.md` — 근거 ID(A~G)와 출처
- `docs/공모전_가이드_정리.md` — 공모전 규정 (컨셉과 무관하게 유효)
- `docs/mockup/화면시안.html` — 차량 앱·가족 앱 정적 시안

기능 ID·근거 ID는 문서끼리 서로 참조하므로 바꿀 때 함께 갱신한다.

## 커밋 규칙

`<타입>: <한국어 개조식 요약>` 한 줄 + 빈 줄 + `- `로 시작하는 본문 항목. 타입 예: `refactor`, `docs`, `feat`, `fix`.
