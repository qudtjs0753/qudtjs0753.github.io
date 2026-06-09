---
title: "CSR, SSR, Hydration, 그리고 Next.js의 Suspense까지"
date: 2026-06-09 00:00:00 +0900
categories: [Programming]
tags: [Next.js, SSR, Suspense]
---

> 이 글은 AI가 작성하고 사람이 검토했습니다.

## 들어가며

Next.js로 개발하다 보면 `useSearchParams()`를 쓸 때 Suspense로 감싸라는 경고를 마주친다. 왜 그래야 하는지 이해하려면 CSR, SSR, Hydration 개념부터 짚어야 한다.
카카오 로그인을 예시로 들었다.

---

## CSR (Client Side Rendering)

CSR에서 서버는 이것만 줍니다.

```html
<!DOCTYPE html>
<html>
  <body>
    <div id="root"></div>
    <script src="/bundle.js"></script>
  </body>
</html>
```

`<div id="root">`는 React가 HTML을 집어넣을 자리만 잡아둔 것입니다. 실제 UI는 없습니다.

```
브라우저: 빈 div + bundle.js 받음
    ↓
JS 다운로드
    ↓
React 실행 → <div id="root"> 안에 HTML 생성 → 화면 표시
```

**HTML을 서버가 만드는 게 아니라 브라우저에서 JS가 직접 만드는 구조**입니다.

### CSR의 단점

- JS 실행 전까지 빈 화면 → 사용자 체감 느림
- 검색엔진 크롤러는 JS를 실행하지 않고 HTML만 읽음 → `<div id="root">` 하나뿐이라 페이지 내용을 파악 못함 → SEO 불리

---

## SSR (Server Side Rendering)

SSR에서 서버는 React를 실행해서 **완성된 HTML**을 미리 만들어 줍니다.

```
서버: React 컴포넌트 → HTML 문자열 생성 → 브라우저에 전송
브라우저: HTML 수신 즉시 화면 표시 → JS 다운로드 → hydration
```

크롤러가 읽으면:

```html
<h1>카드 혜택 지도</h1>
<p>내 주변 카드 혜택을 확인하세요</p>
```

내용이 다 있어서 SEO에 유리하고, 사용자도 JS 로드 전에 화면을 먼저 볼 수 있습니다.

### CSR vs SSR 비교

| | CSR | SSR |
|---|---|---|
| 서버가 주는 것 | 빈 HTML + JS | 완성된 HTML + JS |
| 화면이 보이는 시점 | JS 실행 후 | HTML 수신 즉시 |
| JS 역할 | HTML 생성 + 인터랙션 | 인터랙션만 (hydration) |
| SEO | 불리 | 유리 |

---

## Hydration

서버가 만든 HTML은 **정적인 화면**입니다. 버튼이 보여도 클릭해도 아무 반응이 없습니다.

JS가 로드된 후 React가 그 HTML을 보고 "이게 어떤 컴포넌트인지" 파악해서 onClick, useState, useEffect 등을 연결합니다. 이 과정이 **Hydration**입니다.

```
서버 HTML: <button>카카오로 시작하기</button>  ← 클릭 안 됨
JS 로드 후: React가 onClick 핸들러 연결        ← 클릭 됨
```

마네킹에 신경계를 연결하는 것과 같습니다.

CSR은 이 과정이 없습니다. JS가 처음부터 HTML을 만들고 인터랙션까지 한 번에 처리하기 때문입니다.

---

## Next.js의 렌더링 방식

Next.js는 SSR 기반인데, HTML을 만드는 **시점**이 두 가지입니다.

### Static Rendering (기본값)

```
빌드 타임에 HTML 미리 생성
→ 요청마다 캐시된 HTML 전송
→ 빠름, 모든 사용자에게 동일한 HTML
```

### Dynamic Rendering

```
요청이 올 때마다 HTML 새로 생성
→ URL 파라미터, 쿠키 등 요청별 정보를 HTML에 반영 가능
→ 느림
```

Next.js는 성능을 위해 **최대한 Static Rendering을 유지**하려 합니다.

---

## useSearchParams() 문제

`/login?code=XXXX`에서 `?code=XXXX`는 **요청마다 다른 정보**입니다.

Static Rendering은 빌드 타임에 HTML을 만드니까 `?code=` 값을 알 수 없습니다.

그래서 Next.js는 `useSearchParams()`를 감지하면 이렇게 판단합니다.

```
"이 컴포넌트가 URL 파라미터를 읽네.
 Static Rendering으로는 처리 못하겠다.
 페이지 전체를 Dynamic Rendering으로 바꿔야겠다."
```

→ **페이지 전체가 Dynamic Rendering으로 강제 전환**됩니다.

---

## Suspense가 해결하는 것

Suspense로 감싸면 **경계선** 역할을 합니다.

```tsx
// Suspense 없이
<KakaoCallbackHandler />  // useSearchParams() 사용
// → 페이지 전체 Dynamic Rendering 강제

// Suspense로 감싸면
<Suspense>
  <KakaoCallbackHandler />
</Suspense>
// → 경계 안쪽만 클라이언트에서 처리
```

```
LoginPage        ← Static Rendering 유지
├── LoginHero    ← 서버에서 HTML 생성
├── Suspense
│   └── KakaoCallbackHandler  ← 클라이언트 hydration 후 URL 읽기
└── KakaoLoginButton          ← 서버에서 HTML 생성
```

Next.js에게 "Suspense 안쪽은 클라이언트에서 알아서 처리해라, 바깥 페이지는 Static Rendering 유지해라"고 알려주는 것입니다.

---

## 최종 흐름 정리

```
1. 빌드 타임
   LoginPage HTML 생성
   (KakaoCallbackHandler 자리는 빈 칸)

2. 사용자가 /login 접속
   캐시된 HTML 즉시 전송 → 화면 바로 보임

3. JS 다운로드 → hydration
   버튼 클릭 등 인터랙션 연결

4. 사용자가 카카오 로그인 후 /login?code=XXXX 리다이렉트
   동일한 캐시 HTML 전송 → 화면 바로 보임

5. hydration 후 KakaoCallbackHandler 실행
   useSearchParams()로 ?code=XXXX 읽기
   → BFF 호출 → 로그인 처리
```

Suspense가 없었다면 4번에서 매 요청마다 서버가 HTML을 새로 만드는 Dynamic Rendering이 됐을 것입니다.

---

## 정리

| 개념 | 핵심 |
|---|---|
| CSR | 브라우저가 JS로 HTML을 직접 생성 |
| SSR | 서버가 완성된 HTML을 미리 생성 |
| Hydration | 서버 HTML에 JS 인터랙션을 연결하는 과정 |
| Static Rendering | 빌드 타임에 HTML 생성, 캐시 활용 |
| Dynamic Rendering | 요청마다 HTML 새로 생성 |
| Suspense | useSearchParams() 등 동적 정보를 쓰는 컴포넌트를 격리해 페이지 나머지를 Static으로 유지 |
