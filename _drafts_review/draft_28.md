---
title: "React useEffect의 모든 것"
date: 2025-06-16
categories: [프론트 화면 관련 글, React]
---

`useEffect`가 헷갈리는 이유는 API 자체가 어려워서가 아니라, "언제 다시 실행되는가"를
결정하는 의존성 배열의 동작 원리를 정확히 모르고 쓰기 때문이다. 기본 문법부터
실전에서 자주 틀리는 지점까지 정리한다.

## 왜 useEffect가 필요한가

React는 렌더링될 때마다 컴포넌트 함수를 처음부터 다시 실행한다. 그런데 데이터
가져오기, 구독 설정, DOM 직접 조작, 타이머 설정 같은 작업은 "렌더링 결과"에
영향을 주는 게 아니라 컴포넌트 바깥 세계에 영향을 주거나 거기서 데이터를
가져오는 행위다. 이런 걸 렌더링 도중에 직접 실행하면 렌더링이 순수하지 않게
되고 예측 불가능해진다. `useEffect`는 이런 사이드 이펙트를 "렌더링이 끝난 후"로
분리해서 안전하게 실행시켜주는 장치다.

```js
useEffect(callbackFunction, dependencyArray);
```

## 의존성 배열의 세 가지 패턴과 각각의 실제 의미

**1. 빈 배열 `[]`** — 마운트 시 딱 한 번만 실행.

```js
useEffect(() => {
  console.log('컴포넌트가 마운트될 때 한 번만 실행됩니다.');
  return () => {
    console.log('언마운트될 때 실행되는 클린업 함수');
  };
}, []);
```

초기 데이터 로딩, 한 번만 실행되어야 하는 설정에 쓴다.

**2. 배열 자체를 생략** — 렌더링마다 매번 실행.

```js
useEffect(() => {
  console.log('렌더링될 때마다 항상 실행됩니다.');
});
```

거의 쓸 일이 없다. 이 안에서 `setState`를 호출하면 렌더링 → 이펙트 실행 →
상태 변경 → 재렌더링 → 이펙트 재실행으로 이어지는 무한 루프에 빠지기 쉽다.

**3. 값이 있는 배열 `[value1, value2]`** — 마운트 시 1회 + 값이 바뀔 때마다 재실행.

```js
useEffect(() => {
  console.log(`count 값이 변경되었습니다: ${count}`);
}, [count]);
```

가장 흔히 쓰는 패턴. 특정 상태/props 변화에 반응하는 로직에 쓴다.

## 클린업 함수: 언제, 왜 필요한가

콜백이 함수를 반환하면 그게 클린업 함수가 된다. 두 시점에 실행된다 — 컴포넌트가
언마운트될 때, 그리고 **의존성이 바뀌어 이펙트가 다시 실행되기 직전**. 후자를
놓치기 쉬운데, 이게 없으면 이전 이펙트가 만든 부작용(타이머, 구독, 이벤트
리스너)이 정리되지 않은 채로 새 이펙트가 또 하나 쌓인다 — 메모리 누수와 중복
실행의 흔한 원인이다.

```js
useEffect(() => {
  const timer = setTimeout(() => {
    console.log('2초 후 실행');
  }, 2000);

  return () => {
    clearTimeout(timer);
  };
}, []);
```

## 실전: OAuth 콜백 처리에서 의존성 배열이 실제로 하는 일

Next.js App Router 환경에서 `useSearchParams`와 `useRouter`를 `useEffect`와
결합해 소셜 로그인 콜백을 처리하는 예시.

```jsx
'use client';

function SignupDetailsContent() {
  const router = useRouter();
  const searchParams = useSearchParams();

  useEffect(() => {
    const authorizationCode = searchParams.get('code');
    const storedFormData = localStorage.getItem('signupFormData');
    const storedProvider = localStorage.getItem('loginProvider');

    if (authorizationCode && storedFormData && storedProvider) {
      const formData = JSON.parse(storedFormData);

      const completeSignup = async () => {
        const response = await fetch(`/api/auth/login/${storedProvider}`, {
          method: 'POST',
          headers: { 'Content-Type': 'application/json' },
          body: JSON.stringify({ authorizationCode, ...formData }),
        });
        const result = await response.json();
        localStorage.removeItem('signupFormData');
        localStorage.removeItem('loginProvider');
        router.push('/');
      };

      completeSignup();
    }
  }, [searchParams, router]);
}
```

**`searchParams`가 의존성에 있는 이유(핵심 트리거)**: 카카오 인증을 마치고
`?code=...`가 붙은 URL로 돌아오면, `useSearchParams()`는 쿼리가 바뀔 때마다
**새로운 객체 인스턴스**를 반환한다. JS의 의존성 비교는 참조 비교이므로, 새
인스턴스라는 것 자체가 "값이 달라졌다"는 신호가 되어 콜백이 다시 실행된다.
이 참조 비교 특성을 모르면 "값은 안 바뀐 것 같은데 왜 다시 실행되지?" 또는
반대로 "URL이 바뀌었는데 왜 반응이 없지?" 같은 착각에 빠지기 쉽다.

**`router`가 의존성에 있는 이유(관례)**: 콜백 안에서 `router.push()`를 쓰고
있으므로 ESLint의 `exhaustive-deps` 규칙이 요구한다. `router` 객체 자체의
참조는 앱 생명주기 동안 거의 안 바뀌기 때문에 실질적으로 재실행을 유발하진
않지만, 코드의 견고성(나중에 router 구현이 바뀌어도 안전)을 위한 표준 관례다.

**순서는 상관없다**: `[searchParams, router]`든 `[router, searchParams]`든
동작은 같다 — `useEffect`는 배열의 각 요소를 개별적으로 비교할 뿐 순서를 보지
않는다.

## 결론

`useEffect`를 안전하게 쓰는 핵심은 콜백의 로직이 아니라 **의존성 배열이 무엇을
"변화"로 인식하는가**를 정확히 아는 것이다. 원시값은 값 비교, 객체/배열/함수는
참조 비교라는 걸 놓치면 "의존성을 다 넣었는데 왜 무한 루프가 도는지" 또는
"의존성을 넣었는데 왜 반응하지 않는지" 둘 다 설명할 수 없게 된다.
