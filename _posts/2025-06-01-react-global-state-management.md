---
title: "React 전역 상태 관리: Context가 언제 부족해지는가"
date: 2025-06-01
categories: [프론트엔드]
---

전역 상태를 다루는 방법은 Context API 하나가 아니다. 문제는 "Context를 쓸 것이냐 상태
관리 라이브러리를 쓸 것이냐"가 아니라, **각 방식이 정확히 어떤 비용을 지불하는지**를
아는 것이다.

## Context API: props drilling을 없애는 대가

컴포넌트 트리가 깊어지면 데이터를 여러 단계 아래로 넘기기 위해 중간 컴포넌트들까지
props를 계속 전달해야 하는 문제(props drilling)가 생긴다. Context는 이걸 우회하는
장치다.

```jsx
// Context 생성
export const MyContext = createContext('default value');

// Provider로 트리를 감싸고 값을 주입
function App() {
  const value = 'Hello from Context';
  return (
    <MyContext.Provider value={value}>
      <Child />
    </MyContext.Provider>
  );
}

// 하위 컴포넌트에서 소비
function Child() {
  const contextValue = useContext(MyContext);
  return <div>{contextValue}</div>;
}
```

문제는 여기서 발생한다. **Provider의 `value`가 바뀌면, 그 값을 실제로 쓰지 않는
하위 컴포넌트까지 포함해서 Provider 아래 모든 컴포넌트가 리렌더링 대상이 된다.**
Context는 "구독"이 컴포넌트 단위가 아니라 Provider 트리 단위로 이뤄지기 때문이다.

## useMemo가 필요한 진짜 이유

```jsx
const contextValue = useMemo(() => ({
    message, count, theme, incrementCount, toggleTheme,
  }), [message, count, theme, incrementCount, toggleTheme]);
```

자바스크립트에서 객체는 참조로 비교된다. `useMemo` 없이 매 렌더링마다 새 객체
리터럴을 `value`에 넘기면, 내부 값이 이전과 동일해도 참조가 달라져서 React는
"값이 바뀌었다"고 판단한다. 결과적으로 Context를 구독하는 모든 컴포넌트가
불필요하게 다시 렌더링된다 — 이게 Context 기반 전역 상태 관리에서 성능 저하가
생기는 가장 흔한 원인이다. `useMemo`는 최적화 옵션이 아니라, Context에 객체를
넘기는 순간 사실상 필수에 가깝다.

## 그래도 남는 근본 문제

`useMemo`로 불필요한 객체 재생성은 막을 수 있지만, **값 자체가 실제로 바뀌었을
때의 광범위한 리렌더링**은 막지 못한다. `count`만 바뀌었는데 `theme`만 구독하는
컴포넌트까지 다시 렌더링되는 건 Context의 구조적 한계다 — Context는 "이 값을 쓰는
컴포넌트만" 골라서 구독하는 선택적 구독을 지원하지 않는다.

이 지점이 Context와 상태 관리 라이브러리(Zustand, Redux 등)의 실질적인 차이다.

## Zustand: 선택적 구독으로 이 문제를 해결

```bash
npm install zustand
```

```js
// 스토어 정의
export const useStore = create((set, get) => ({
    user: { name: '', email: '' },
    cartItems: [],
    count: 0,
    actions: {
        setUserInfo: (param) => set(state => ({
            user: { ...state.user, name: param.name, email: param.email }
        })),
        incrementCount: () => set(state => ({ count: state.count + 1 }))
    }
}));
```

```jsx
function UserProfile() {
    // 필요한 상태만 선택자 함수로 구독 — 다른 상태가 바뀌어도 리렌더링 안 됨
    const userName = useStore(state => state.user.name);
    const currentCount = useStore(state => state.count);
    ...
}
```

핵심은 `useStore(state => state.count)`처럼 **선택자(selector) 함수**로 상태의
일부만 구독한다는 것이다. `count`만 구독한 컴포넌트는 `cartItems`가 바뀌어도
리렌더링되지 않는다 — Context가 구조적으로 못 하는 것을 Zustand는 기본으로
제공한다. 게다가 Provider로 트리를 감쌀 필요가 없어서 설정도 더 단순하다.

## 그럼 Context는 언제 쓰는가

Zustand가 항상 더 나은 건 아니다. 판단 기준은 **값이 얼마나 자주 바뀌는가**다.

- 테마, 로그인 여부, 언어 설정처럼 **한번 설정되면 거의 바뀌지 않는 값** →
  리렌더링 비용이 애초에 낮으므로 Context로 충분하다. 별도 라이브러리를 추가할
  이유가 없다.
- 카운터, 장바구니, 폼 입력값처럼 **자주 바뀌고, 그 변화를 일부 컴포넌트만 구독해야
  하는 값** → Context는 리렌더링 범위를 제어할 수 없어서 성능 문제가 실제로
  체감된다. 이런 경우엔 Zustand 같은 선택적 구독 라이브러리가 낫다.

"전역 상태니까 무조건 Context" 또는 "전역 상태니까 무조건 라이브러리"가 아니라,
**갱신 빈도와 구독 범위**로 판단하는 게 맞는 접근이다.
