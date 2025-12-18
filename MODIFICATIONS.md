# 커스텀 수정 사항

이 fork는 원본 [cordova-plugin-inappbrowser](https://github.com/apache/cordova-plugin-inappbrowser)에 다음과 같은 커스텀 수정사항을 포함하고 있습니다.

## 버전 정보

**기본 버전**: 6.0.0 (Apache Cordova InAppBrowser)
**Fork 날짜**: 2025년 11월
**현재 버전**: 6.0.0-custom

---

## 🔧 적용된 수정사항 (Master 브랜치)

### 1. Referer 헤더 지원
**커밋**: `bab78c3`, `d877f29`
**날짜**: 2025년 11월 12일
**플랫폼**: Android, iOS

#### 요약
InAppBrowser를 통해 이루어지는 모든 요청에 HTTP Referer 헤더를 자동으로 추가합니다.

#### 변경사항
- **Android**: `src/android/InAppBrowser.java` 수정
  - WebView 요청에 Referer 헤더 추가
  - 16줄 수정 (12줄 추가, 4줄 삭제)

- **iOS**: `src/ios/CDVWKInAppBrowser.m` 수정
  - WKWebView 네비게이션에 Referer 헤더 추가
  - 242줄 수정 (124줄 추가, 118줄 삭제)

#### 문서
상세 구현 내용은 [docs/referer-header.md](docs/referer-header.md)를 참조하세요.

---

### 2. iOS Toolbar 반투명 효과 버그 수정
**커밋**: `cd73853`, `184d125`, `b9a4186`
**날짜**: 2025년 11월 17-22일
**플랫폼**: iOS

#### 요약
`toolbartranslucent=yes` 옵션(기본값) 사용 시 반투명 효과 대신 불투명한 흰색 툴바가 표시되는 버그를 수정했습니다.

#### 문제점
- view 배경색을 무조건 설정하여 반투명 툴바가 깨짐
- 반투명 효과가 활성화되어 있어도 불투명한 흰색 툴바가 표시됨

#### 해결방법
- 배경색 설정 전에 `toolbartranslucent` 옵션을 먼저 확인
- `toolbartranslucent=yes` (기본값): 투명 배경을 사용하여 반투명 블러 효과 구현
- `toolbartranslucent=no`: 불투명 배경 사용 (toolbarcolor 또는 흰색)

#### 변경사항
- `src/ios/CDVWKInAppBrowser.m` 수정
- `docs/ios-toolbar-fix.md` 상세 분석 문서 생성

#### 문서
기술 상세 내용은 [docs/ios-toolbar-fix.md](docs/ios-toolbar-fix.md)를 참조하세요.

---

## 📦 보류 중인 수정사항 (브랜치: reserved/ios-safe-area-fix)

### 3. iOS Safe Area 지원
**커밋**: `f6cedc0`, `a03651b`
**날짜**: 2025년 11월 17일
**플랫폼**: iOS
**상태**: ⚠️ 아직 master에 병합되지 않음

#### 요약
노치/Dynamic Island가 있는 iPhone X 이상 기기를 위한 완전한 Safe Area Insets 지원.

#### 문제점
- 노치가 있는 iPhone에서 InAppBrowser 하단에 빈 공간 발생
- 34pt 빈 공간 (Home Indicator 영역)을 통해 하위 앱 콘텐츠가 비쳐 보임
- Safe Area를 무시하고 툴바가 잘못 배치됨

#### 해결방법
- Safe Area Insets를 고려한 툴바 위치 조정
- `createViews`, `showToolBar`, `showLocationBar`, `rePositionViews` 메서드 수정
- 기기 회전 시 자동 재배치를 위한 `viewSafeAreaInsetsDidChange` 추가
- iOS 11 이하 호환성 유지

#### 변경사항
- `src/ios/CDVWKInAppBrowser.m` 수정 (159줄 추가, 20줄 삭제)
- 포괄적인 문서 생성:
  - `docs/ios-safe-area-fix.md` (480줄)
  - `docs/ios-safe-area-reposition-fix.md` (433줄)

#### 접근 방법
```bash
git checkout reserved/ios-safe-area-fix
```

---

## 🎯 사용 사례

이러한 수정사항은 다음 용도로 설계되었습니다:

1. **Referer 헤더**:
   - 네비게이션 소스 추적
   - 서버 측 분석
   - Referrer 기반 접근 제어

2. **Toolbar 반투명 수정**:
   - 적절한 반투명 툴바 외관
   - iOS 디자인 가이드라인과 일관된 UI

3. **Safe Area 지원** (보류 중):
   - 전체 화면 YouTube 동영상 재생
   - 라이선스 페이지, 후원 페이지
   - 최신 iPhone에서 전체 화면 InAppBrowser 사용

---

## 📋 호환성

| 플랫폼 | 기본 플러그인 | 이 Fork |
|--------|--------------|---------|
| iOS 11-14 | ✅ | ✅ |
| iOS 15+ | ✅ | ✅ 향상됨 (반투명 수정) |
| iOS 11+ (노치) | ⚠️ 하단 빈 공간 | ✅ 수정됨 (reserved 브랜치) |
| Android | ✅ | ✅ 향상됨 (Referer 헤더) |

---

## 🔄 Upstream 동기화

upstream Apache Cordova InAppBrowser와 동기화할 때:

1. 다음 파일에서 충돌 확인:
   - `src/android/InAppBrowser.java`
   - `src/ios/CDVWKInAppBrowser.m`

2. 커스텀 수정사항 보존:
   - Referer 헤더 구현
   - Toolbar 반투명 로직
   - Safe Area 계산 (병합된 경우)

3. 병합 후 모든 사용 사례 테스트

---

## 📚 문서 목록

- [Referer 헤더 구현](docs/referer-header.md)
- [iOS Toolbar 반투명 수정](docs/ios-toolbar-fix.md)
- [iOS Safe Area 수정](docs/ios-safe-area-fix.md) (reserved 브랜치)
- [iOS Safe Area Reposition 수정](docs/ios-safe-area-reposition-fix.md) (reserved 브랜치)

---

## 👤 작성자

**박지환 (jhpark)**
이메일: akalpa@naver.com
날짜: 2025년 11월

---

## 📄 라이선스

이 fork는 기본 플러그인의 원본 Apache License 2.0을 유지합니다.
자세한 내용은 [LICENSE](LICENSE)를 참조하세요.
