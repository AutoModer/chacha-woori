# 차차, 우리 — Bye & By, Us

가족의 목소리를 태워 드립니다. 운전하는 동안에도 늘 가족이 곁에 있는 Pleos Connect 차량 앱입니다.

> 현대자동차그룹 차량용 앱 개발 공모전 출품작 (개발 중)

## 핵심 기능
가족의 목소리가 두 가지 방식으로 차에 탑니다. 가족이 먼저 다가오고, 운전자는 듣고 "응"만 하면 됩니다.

**소통 — 가족이 보낸 목소리**
- **언제든 보내는 메시지**: 가족은 언제든 음성·글 메시지를 보내고, 원하면 장소를 붙입니다.
- **운전 중엔 바로, 아니면 시동 켤 때**: 운전 중이면 화면 없이 소리로 바로 전하고(글은 TTS로 읽음), 아닐 때 온 메시지는 시동을 켤 때 인사 영상과 함께 들려줍니다.
- **"응" 한마디로 목적지**: 장소가 있으면 "목적지로 할까요?"를 묻고 [예] [아니요] [전화하기] 중 하나를 말로 고릅니다.
- **차에서만 시작하는 위치 공유**: 목적지가 집이나 가족이 보낸 장소면 출발 때 공유를 묻고, 도착하면 자동으로 끝납니다.

**안전 — 가족이 미리 녹음한 목소리**
- **중요한 길목 안내**: 고속도로 출구·분기점만 골라 "아빠, 이번에 오른쪽으로 나가야 돼"으로 미리 알리고, 깜빡이를 켜면 멈춥니다. 놓치면 "괜찮아, 다음에서 나가면 돼".
- **휴식 권유**: 오래 쉬지 않고 달리면 다음 쉼터 전에 "오래 달렸네, 쉬었다 가자".

자세한 기획은 [docs/차차우리_통합정리.md](docs/차차우리_통합정리.md)를 참고하세요. 문서 목록은 [docs/README.md](docs/README.md)에 있습니다.

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
