# Referer 옵션 구현 계획

## 개요

현재 하드코딩되어 있는 Referer 헤더를 `cordova.InAppBrowser.open()` 함수의 옵션 파라미터로 받을 수 있도록 변경합니다.

**작성일**: 2025년 11월 (수정: 2025년 12월 18일)
**상태**: 수정된 계획 (검토 반영됨)

---

## 목표

### 현재 상태
```javascript
// 현재: Referer가 자동으로 앱 패키지/번들 ID로 설정됨
cordova.InAppBrowser.open('https://youtube.com/watch?v=VIDEO_ID', '_blank', 'location=no');
// Android: Referer: https://kr.gukeo.app
// iOS: Referer: https://kr.gukeo.app
```

### 목표 상태
```javascript
// 목표: 사용자가 커스텀 Referer를 지정 가능
cordova.InAppBrowser.open(
    'https://youtube.com/watch?v=VIDEO_ID',
    '_blank',
    'location=no,referrer=https://custom.domain.com'
);
// Referer: https://custom.domain.com

// 옵션 미지정 시: 기존 동작 유지 (하위 호환성)
cordova.InAppBrowser.open('https://youtube.com/watch?v=VIDEO_ID', '_blank', 'location=no');
// Android: Referer: https://kr.gukeo.app (기본값)
// iOS: Referer: https://kr.gukeo.app (기본값)
```

---

## 구현 설계

### 1. 옵션 정의

**옵션 이름**: `referrer` (철자에 주의: referer가 아닌 referrer)

**사용 예시**:
```javascript
// 기본값 사용 (옵션 생략)
cordova.InAppBrowser.open(url, '_blank', 'location=no');

// 커스텀 referrer 지정
cordova.InAppBrowser.open(url, '_blank', 'referrer=https://example.com');

// 여러 옵션과 함께 사용
cordova.InAppBrowser.open(
    url,
    '_blank',
    'location=no,toolbar=yes,referrer=https://example.com'
);
```

### 2. Android 구현 계획

#### 2.1. 상수 추가 및 Customizable Options 수정 (중요)
**파일**: `src/android/InAppBrowser.java`

1. 상수 추가:
```java
private static final String LOCATION = "location";
// ...
private static final String REFERRER = "referrer";  // 🆕 추가
```

2. `customizableOptions` 리스트 업데이트 (필수):
   - 이 리스트에 없으면 값이 무조건 "yes"로 변환되는 문제가 있음.
```java
// 🔧 수정: REFERRER 추가
private static final List customizableOptions = Arrays.asList(CLOSE_BUTTON_CAPTION, TOOLBAR_COLOR, NAVIGATION_COLOR, CLOSE_BUTTON_COLOR, FOOTER_COLOR, REFERRER);
```

#### 2.2. 인스턴스 변수 추가
**위치**: 클래스 멤버 변수 선언부

```java
private boolean showLocationBar = true;
// ...
private String customReferrer = null;  // 🆕 추가
```

#### 2.3. 옵션 파싱 및 초기화 수정
**메서드**: `showWebPage(final String url, HashMap<String, String> features)`

```java
// 🆕 변수 초기화 (필수: 이전 호출의 값 잔존 방지)
customReferrer = null;

if (features != null) {
    String show = features.get(LOCATION);
    // ... 기존 옵션 파싱

    // 🆕 Referrer 옵션 파싱 추가
    String referrer = features.get(REFERRER);
    if (referrer != null && !referrer.isEmpty()) {
        customReferrer = referrer;
    }
}
```

#### 2.4. Referer 헤더 설정 로직 수정
**위치**: `inAppWebView.loadUrl()` 호출 부분

```java
// 🔧 수정: 옵션 우선, 없으면 기본값 사용
String referer;
if (customReferrer != null && !customReferrer.isEmpty()) {
    // 사용자 지정 referrer 사용
    referer = customReferrer;
} else {
    // 기본값: 패키지 이름 사용
    String packageName = cordova.getActivity().getPackageName();
    referer = "https://" + packageName;
}

HashMap<String, String> headers = new HashMap<>();
headers.put("Referer", referer);
inAppWebView.loadUrl(url, headers);
```

### 3. iOS 구현 계획

#### 3.1. CDVInAppBrowserOptions 확장
**파일**: `src/ios/CDVInAppBrowserOptions.h`

```objective-c
@interface CDVInAppBrowserOptions : NSObject {}

// ... 기존 프로퍼티들
@property (nonatomic, copy) NSString* referrer;  // 🆕 추가

// 파싱 로직은 .m 파일의 루프에서 자동 처리되므로 별도 수정 불필요
@end
```

#### 3.2. CDVWKInAppBrowserViewController 헤더 수정 (필요 시)
**파일**: `src/ios/CDVWKInAppBrowser.h` (또는 해당 인터페이스 선언부)

재사용 시 옵션 업데이트를 위해 `browserOptions` 프로퍼티 접근이 필요할 수 있음.
(현재 `CDVWKInAppBrowser.m` 내부에 구현이 있다면 해당 부분 확인 필요, 만약 readonly라면 readwrite로 변경하거나 메서드 추가)

#### 3.3. CDVWKInAppBrowser 수정
**파일**: `src/ios/CDVWKInAppBrowser.m`

**메서드**: `openInInAppBrowser:withOptions:`

```objective-c
- (void)openInInAppBrowser:(NSURL*)url withOptions:(NSString*)options
{
    CDVInAppBrowserOptions* browserOptions = [CDVInAppBrowserOptions parseOptions:options];
    
    // ... (기존 코드)

    if (self.inAppBrowserViewController == nil) {
        self.inAppBrowserViewController = [[CDVWKInAppBrowserViewController alloc] initWithBrowserOptions: browserOptions andSettings:self.commandDelegate.settings];
        // ...
    } else {
        // 🆕 뷰 컨트롤러 재사용 시 옵션 업데이트 (필수)
        // 기존 뷰 컨트롤러 인스턴스는 이전 browserOptions를 가지고 있으므로 갱신해줘야 함.
        if ([self.inAppBrowserViewController respondsToSelector:@selector(setBrowserOptions:)]) {
            self.inAppBrowserViewController.browserOptions = browserOptions;
        }
    }

    // ... (기존 코드)
}
```

**메서드**: `navigateTo:`

```objective-c
- (void)navigateTo:(NSURL*)url
{
    if ([url.scheme isEqualToString:@"file"]) {
        [self.webView loadFileURL:url allowingReadAccessToURL:url];
    } else {
        // 🔧 수정: 옵션 우선, 없으면 기본값 사용
        NSString *referrer;
        // _browserOptions 접근 (프로퍼티를 통해 접근하도록 수정 필요할 수 있음)
        if (_browserOptions.referrer != nil && ![_browserOptions.referrer isEqualToString:@""]) {
            referrer = _browserOptions.referrer;
        } else {
            NSString *bundleId = [[NSBundle mainBundle] bundleIdentifier];
            referrer = [[NSString stringWithFormat:@"https://%@", bundleId] lowercaseString];
        }

        NSURL *referrerUrl = [NSURL URLWithString:referrer];

        NSMutableURLRequest *request = [NSMutableURLRequest requestWithURL:url];
        [request addValue:[referrerUrl absoluteString] forHTTPHeaderField:@"Referer"];

        [self.webView loadRequest:request];
    }
}
```

---

## 구현 단계 (수정됨)

### Phase 1: Android 구현
1. ✅ `REFERRER` 상수 추가 및 `customizableOptions` 리스트에 추가 (Critical Fix)
2. ✅ 인스턴스 변수 `customReferrer` 추가
3. ✅ `showWebPage()` 초입에 `customReferrer` 초기화 로직 추가 (Critical Fix)
4. ✅ 옵션 파싱 로직 추가
5. ✅ `loadUrl()` 호출 부분 조건부 로직 수정
6. ✅ 테스트

### Phase 2: iOS 구현
1. ✅ `CDVInAppBrowserOptions.h`에 `referrer` 프로퍼티 추가 (자동 파싱됨)
2. ✅ `CDVWKInAppBrowser.m`의 `CDVWKInAppBrowserViewController` 인터페이스에서 `browserOptions` 프로퍼티 접근성 확인 및 수정
3. ✅ `openInInAppBrowser` 메서드에서 재사용 시 `browserOptions` 갱신 로직 추가 (Critical Fix)
4. ✅ `CDVWKInAppBrowserViewController`의 `navigateTo:` 메서드 수정 (조건부 로직)
5. ✅ 테스트

---

## 하위 호환성 및 테스트 계획 (기존과 동일)

### 검증 시나리오
1. **옵션 없음 (하위 호환)**: `Referer`가 앱 번들 ID로 설정되는지 확인.
2. **커스텀 Referrer**: 지정한 URL로 설정되는지 확인.
3. **재사용 테스트**: 브라우저를 닫지 않고 `open`을 다시 호출하거나, 닫았다가 다른 옵션으로 다시 열 때 `Referer`가 올바르게 갱신되는지 확인.

---

**검토자**: Cline (AI Assistant)
**검토 일자**: 2025-12-18
**주요 수정 사항**:
- Android `customizableOptions` 누락 수정 (옵션값 파싱 오류 방지)
- Android 상태 변수 초기화 추가
- iOS 옵션 파싱 불필요 단계 제거
- iOS 뷰 컨트롤러 재사용 시 옵션 갱신 로직 추가
