# Referer 헤더 구현

## 개요

이 문서는 Android와 iOS 플랫폼 모두에서 InAppBrowser의 모든 요청에 HTTP Referer 헤더를 자동으로 추가하는 커스텀 구현에 대해 설명합니다.

## 동기

### 문제
Referer 헤더 없이 InAppBrowser에서 특정 웹 콘텐츠(특히 YouTube 동영상)를 로드할 때, 서버가 다음과 같이 동작할 수 있습니다:
- "Error 153" (YouTube 특정 오류)로 요청 거부
- 보안 정책으로 인한 콘텐츠 차단
- 네비게이션 소스 추적 실패

### 해결책
InAppBrowser를 통한 모든 HTTP/HTTPS 요청에 대해 앱의 패키지/번들 식별자를 기반으로 Referer 헤더를 자동으로 주입합니다.

## 구현 상세

### Android 구현

**파일**: `src/android/InAppBrowser.java`
**커밋**: `bab78c3`
**날짜**: 2025년 11월 12일

#### 변경사항

커스텀 헤더를 포함하도록 `loadUrl()` 호출을 수정했습니다:

```java
// 이전 (1023줄)
inAppWebView.loadUrl(url);

// 이후 (1023-1032줄)
// Add Referer header for YouTube playback (prevents "Error 153")
String packageName = cordova.getActivity().getPackageName();
String referer = "https://" + packageName;

HashMap<String, String> headers = new HashMap<>();
headers.put("Referer", referer);
// headers.put("Referrer-Policy", "strict-origin-when-cross-origin");
inAppWebView.loadUrl(url, headers);
```

#### 동작 방식

1. **패키지 이름 가져오기**: 앱의 패키지 이름 가져오기 (예: `com.example.myapp`)
2. **Referer 생성**: HTTPS URL 형식 생성: `https://com.example.myapp`
3. **헤더 추가**: 모든 WebView 요청에 Referer 포함
4. **선택적 정책**: Referrer-Policy 헤더는 주석 처리됨 (필요 시 활성화 가능)

#### 예시

패키지 이름이 `kr.gukeo.app`인 앱의 경우:
```
Referer: https://kr.gukeo.app
```

### iOS 구현

**파일**: `src/ios/CDVWKInAppBrowser.m`
**커밋**: `d877f29`
**날짜**: 2025년 11월 12일

#### 변경사항

Referer 헤더를 추가하도록 `navigateTo:` 메서드를 수정했습니다:

```objective-c
- (void)navigateTo:(NSURL*)url
{
    if ([url.scheme isEqualToString:@"file"]) {
        [self.webView loadFileURL:url allowingReadAccessToURL:url];
    } else {
        // 이전
        // NSURLRequest* request = [NSURLRequest requestWithURL:url];

        // 이후
        NSString *bundleId = [[NSBundle mainBundle] bundleIdentifier];
        NSString *referrer = [[NSString stringWithFormat:@"https://%@", bundleId] lowercaseString];
        NSURL *referrerUrl = [NSURL URLWithString:referrer];

        NSMutableURLRequest *request = [NSMutableURLRequest requestWithURL:url];
        [request addValue:[referrerUrl absoluteString] forHTTPHeaderField:@"Referer"];

        [self.webView loadRequest:request];
    }
}
```

#### 동작 방식

1. **번들 ID 가져오기**: 앱의 번들 식별자 가져오기 (예: `com.example.MyApp`)
2. **Referer 생성**: 소문자 HTTPS URL 형식 생성: `https://com.example.myapp`
3. **Mutable Request 생성**: `NSURLRequest`에서 `NSMutableURLRequest`로 변경
4. **헤더 추가**: HTTP 헤더에 Referer 포함
5. **Request 로드**: Referer 헤더가 포함된 수정된 요청 로드

#### 예시

번들 ID가 `kr.gukeo.app`인 앱의 경우:
```
Referer: https://kr.gukeo.app
```

#### Android와의 주요 차이점

- 번들 식별자가 **소문자**로 변환됨 (iOS 관례)
- immutable `NSURLRequest` 대신 `NSMutableURLRequest` 사용
- 파일 URL (`file://`)은 Referer 헤더 없이 별도로 처리

## 사용 사례

### YouTube 동영상 재생

**문제**: 적절한 Referer 없이 모바일 앱에서 YouTube에 접근하면 "Error 153"을 반환할 수 있습니다.

**해결**: 이 수정사항으로 YouTube가 요청을 앱에서 온 것으로 인식하고 재생을 허용합니다.

```javascript
// 예시: InAppBrowser에서 YouTube 열기
const ref = cordova.InAppBrowser.open(
    'https://www.youtube.com/watch?v=VIDEO_ID',
    '_blank',
    'location=no,toolbar=yes'
);
```

### 서버 측 분석

많은 웹 분석 도구가 Referer 헤더를 추적하여 트래픽 소스를 파악합니다. 이 수정사항은 다음을 보장합니다:
- 서버 로그에 앱에서 온 요청으로 표시
- 분석 도구가 모바일 앱 트래픽을 올바르게 귀속
- A/B 테스트 도구가 앱 기반 세션 식별 가능

### 접근 제어

일부 웹 서비스는 Referer 기반 접근 제어를 사용합니다:
- 특정 앱에서만 요청 허용
- 직접 URL 접근 차단
- 앱별 기능 구현

## 보안 고려사항

### HTTP vs HTTPS

두 구현 모두 Referer URL에 **HTTPS** 스킴을 사용합니다:
```
✅ Referer: https://com.example.app
❌ Referer: http://com.example.app
```

이것이 중요한 이유:
- 최신 브라우저는 보안상 HTTPS에서 HTTP로 Referer를 보내지 않음
- HTTPS 사용으로 "mixed content" 경고 방지
- 웹 보안 모범 사례와 일치

### 프라이버시

Referer 헤더에는 다음이 포함됩니다:
- ✅ 앱 패키지/번들 식별자 (공개 정보)
- ❌ 사용자 데이터 없음
- ❌ 기기 식별자 없음
- ❌ 추적 정보 없음

이는 최소한의 프라이버시 친화적 구현입니다.

### 파일 URL

두 구현 모두 `file://` URL에 대한 Referer를 제외합니다:
- Android: 헤더는 원격 URL에만 적용
- iOS: 파일 URL을 명시적으로 확인하고 건너뜀

이는 로컬 콘텐츠 로드 시 잠재적인 정보 유출을 방지합니다.

## 테스트

### 테스트 케이스

1. **YouTube 재생**
   ```javascript
   cordova.InAppBrowser.open('https://youtube.com/watch?v=TEST', '_blank');
   // "Error 153" 없이 재생되어야 함
   ```

2. **HTTPS 웹사이트**
   ```javascript
   cordova.InAppBrowser.open('https://example.com', '_blank');
   // 서버 로그에 다음이 표시되어야 함: Referer: https://your.app.id
   ```

3. **로컬 파일**
   ```javascript
   cordova.InAppBrowser.open('file:///path/to/local.html', '_blank');
   // Referer 헤더 없음 (예상된 동작)
   ```

### 검증

Referer 헤더가 전송되고 있는지 확인하려면:

1. **헤더를 로깅하는 테스트 서버 사용**:
   ```bash
   # 간단한 Node.js 테스트 서버
   const express = require('express');
   const app = express();
   app.get('/', (req, res) => {
       console.log('Referer:', req.headers.referer);
       res.send('OK');
   });
   app.listen(3000);
   ```

2. **브라우저 DevTools 사용** (WebView에서 접근 가능한 경우):
   - Network 탭 열기
   - 요청 헤더 확인
   - `Referer` 필드 확인

3. **온라인 헤더 체커 사용**:
   - InAppBrowser에서 https://httpbin.org/headers 열기
   - 응답에서 `Referer` 필드 확인

## 호환성

| 플랫폼 | 버전 | 상태 |
|--------|------|------|
| Android 5.0+ | 전체 | ✅ 지원됨 |
| iOS 11+ | 전체 | ✅ 지원됨 |
| iOS 15+ | WKWebView | ✅ 지원됨 |

## 향후 개선사항

향후 버전을 위한 잠재적 개선사항:

### 1. 설정 가능한 Referer

사용자가 커스텀 Referer 값을 지정할 수 있도록 허용:
```javascript
cordova.InAppBrowser.open(
    url,
    '_blank',
    'referer=https://custom.domain.com'
);
```

### 2. Referrer-Policy 지원

주석 처리된 Referrer-Policy 헤더 활성화:
```java
headers.put("Referrer-Policy", "strict-origin-when-cross-origin");
```

옵션:
- `no-referrer`: Referer를 절대 보내지 않음
- `origin`: 전체 URL이 아닌 origin만 전송
- `strict-origin-when-cross-origin`: 기본 보안 동작

### 3. 요청별 헤더

요청당 임의의 헤더 추가 지원:
```javascript
cordova.InAppBrowser.open(url, '_blank', {
    headers: {
        'Referer': 'https://custom.com',
        'X-Custom-Header': 'value'
    }
});
```

## 관련 이슈

### Upstream

이 기능은 공식 Apache Cordova InAppBrowser 플러그인에 없습니다. 다음을 고려하세요:
- 기능 요청으로 제출
- 옵션 플래그가 있는 Pull Request 생성 (기본적으로 off)

### 유사 플러그인

헤더 지원이 있는 다른 플러그인:
- `cordova-plugin-advanced-http`: 헤더 제어가 있는 전체 HTTP 클라이언트
- `cordova-plugin-wkwebview-engine`: 구성이 가능한 커스텀 WebView

## 참고자료

- [MDN: Referer 헤더](https://developer.mozilla.org/ko/docs/Web/HTTP/Headers/Referer)
- [MDN: Referrer-Policy](https://developer.mozilla.org/ko/docs/Web/HTTP/Headers/Referrer-Policy)
- [Apache Cordova InAppBrowser](https://github.com/apache/cordova-plugin-inappbrowser)

---

**작성자**: 박지환 (jhpark)
**이메일**: akalpa@naver.com
**날짜**: 2025년 11월 12일
**커밋**: `bab78c3` (Android), `d877f29` (iOS)
