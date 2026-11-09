# 엄마, 오른쪽 (Mom, right!)

혼자 운전하는 부모님 곁에서, 자녀가 직접 녹음한 목소리로 고속도로 출구·분기점 같은 길목을 한 박자 먼저 알려 주는 Pleos Connect 차량용 앱입니다.

> 현대자동차그룹 차량용 앱 개발 공모전 출품작 (개발 중)

## 핵심 기능
- **미리 알림**: 경로 전체 길목(NaviHelper TBT)과 속도로 남은 시간을 계산해, 놓치면 위험한 길목만 미리 알립니다.
- **반응 감지**: 출구 쪽 깜빡이·조향 변화가 있으면 알림을 즉시 멈춥니다.
- **"괜찮아요" 재안내**: 놓치면 재촉하지 않고 다음 출구로 차분하게 안내합니다.
- **자녀 목소리**: "엄마, 다음에 오른쪽이야"를 자녀가 직접 녹음합니다.
- **오늘의 운전**: 정차 후 칭찬 중심의 짧은 요약을 보여 줍니다.

자세한 기획은 [docs/concept.md](docs/concept.md)를 참고하세요.

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
