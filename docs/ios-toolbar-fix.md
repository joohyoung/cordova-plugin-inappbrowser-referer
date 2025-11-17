# iOS Toolbar Options Fix

## 목적

iOS에서 InAppBrowser의 다음 옵션들이 작동하지 않는 문제를 해결:
- `toolbartranslucent`: Toolbar 반투명 설정 옵션
- `toolbarcolor`: Toolbar 색상 설정 옵션

## 문제 원인

### 초기 시도 (실패)
Toolbar 자체의 속성들을 수정:
- `toolbar.barStyle` 변경
- `toolbar.opaque` 변경
- `toolbar.translucent` 변경

→ **결과: 문제 해결되지 않음**

### 실제 원인
```objective-c
self.view.backgroundColor = [UIColor clearColor];  // 투명한 배경
```

View의 배경이 투명하여, toolbar 뒤로 webView의 콘텐츠가 비쳐 보이는 것이 근본 원인.

## 해결 방법

### 핵심 수정
```objective-c
// View 배경을 toolbar 색상과 동일하게 설정
if (_browserOptions.toolbarcolor != nil) {
    self.view.backgroundColor = [self colorFromHexString:_browserOptions.toolbarcolor];
} else {
    self.view.backgroundColor = [UIColor whiteColor];
}
```

### View 계층 구조
```
[WebView - 맨 뒤]
    ↓
[View.backgroundColor - 불투명!] ← 핵심 해결책
    ↓
[Toolbar - 앞]
```

View의 배경을 불투명하게 만들어서 webView가 비치지 않도록 차단.

## 기술적 세부사항

### toolbar 속성의 실제 효과

| 속성 | 실제 효과 |
|------|-----------|
| `toolbar.opaque` | 렌더링 최적화 힌트일 뿐, 시각적 효과 없음 |
| `toolbar.translucent` | 블러 효과 제어, 하지만 view 배경이 더 중요 |
| `toolbar.barStyle` | Deprecated API 제거용, 시각적 차이 미미 |

### 핵심 인사이트

**toolbar 속성만으로는 부족:**
- `toolbar.opaque = YES`로 설정해도, 부모 view의 투명한 배경은 해결되지 않음
- iOS 렌더링은 각 레이어를 독립적으로 합성
- 부모 view가 투명하면 아래 레이어(webView)가 비쳐 보임

**실제 해결책:**
- `view.backgroundColor`를 불투명하게 설정
- Toolbar 색상과 동일한 색으로 설정하여 seamless 연결
- 결과: webView가 완전히 가려져서 문제 해결

## 관련 Commit

1. Commit #1: Toolbar 속성 정규화 (코드 품질 개선)
2. Commit #2: View 배경색 설정 (실제 문제 해결)

## 참고사항

- iOS UIView의 `opaque` 속성은 투명도를 제어하는 것이 아니라 렌더링 최적화 힌트
- 실제 배경색은 `backgroundColor` 속성으로 제어해야 함
- View 계층 합성 시 각 레이어의 `backgroundColor`가 실제 픽셀을 결정
