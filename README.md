# 퇴근길 (Way Home)

운전 중인 아빠와 집에 있는 가족을, 휴대폰을 보지 않고도 이어 주는 Pleos Connect 차량용 앱입니다.

> 현대자동차그룹 차량용 앱 개발 공모전 출품작 (개발 중)

## 핵심 기능
가족이 먼저 다가오고, 운전자는 듣고 "응"만 하면 됩니다.

- **퇴근길 위치 공유**: 퇴근을 시작하면 가족에게 "아빠 출발했어요"를 알리고, 가족 앱 지도에 위치와 도착 예정 시간을 보여 줍니다. 도착하면 자동으로 끝납니다.
- **장소 붙인 가족 메시지**: 가족이 영상·음성 메시지에 장소를 붙여 보냅니다("오늘 치킨 먹자" + OO치킨).
- **소리로 전하고 묻기**: 주행 중에는 화면 없이 소리로 전하고 "목적지를 OO치킨으로 바꿀까요?"라고 묻습니다.
- **경로에 바로 반영**: 운전자가 "응" 하면 내비 목적지를 바꾸고, 가족 앱에 결과를 알립니다.
- **시동·도착 인사 영상**: 시동 직후와 도착 후(기어 P)에 가족 영상 인사를 재생합니다.
- **가족 목소리**: 졸음이 의심되거나 오래 운전하면 가족이 미리 녹음한 말("아빠 일어나!!")을 들려줍니다.

자세한 기획은 [docs/퇴근길_통합정리.md](docs/퇴근길_통합정리.md)를 참고하세요. 문서 목록은 [docs/README.md](docs/README.md)에 있습니다.

## 구성
- `app/` — 차량 앱 (Pleos Connect, Kotlin)
- 가족 앱 (Android, Kotlin)과 서버(Firebase)는 개발 예정입니다.

## 실행 방법
1. Android Studio 최신 버전을 설치합니다.
2. SDK Manager → SDK Update Sites에 `https://nexus-playground.pleos.ai/repository/raw-releases/connect-avd/2.x/crp-sys-img.xml`(Windows/Linux)을 추가하고, API 34의 Pleos Connect system Image를 설치합니다.
3. Device Manager에서 Automotive → `Pleos Connect v2.1.0` AVD를 만들고 실행합니다 (Cold boot, RAM 4GB).
4. 프로젝트 루트에서 빌드·설치합니다.
   ```
   ./gradlew installDebug
   ```

## 테스트 환경
- Pleos Connect Emulator v2.1.0 (Android 14, API 34)
- Vehicle SDK 2.3.2, NaviHelper SDK 3.0.5
- JDK 17, Gradle 8.11.1, Android Gradle Plugin 8.8.2, Kotlin 2.0.0
- 차량 상태 조작: Pleos Connect Virtual Console (`adb forward tcp:8088 tcp:8088` 후 `http://127.0.0.1:8088/`)

## APK
- 위치: `apk/` 폴더 (파일명 `팀명_앱이름_release.apk`)
