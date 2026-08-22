---
title: "React 자식 컴포넌트 제어: forwardRef와 useImperativeHandle"
date: 2025-05-29
categories: [프론트엔드]
---

React의 기본 원칙은 단방향 데이터 흐름이다 — 부모가 props로 자식에게 값을 내려주고,
자식은 그걸 받아 렌더링만 한다. 부모가 자식의 내부 메서드를 직접 호출한다는 발상 자체가
이 원칙에 어긋난다. 그런데도 `forwardRef`와 `useImperativeHandle`이 존재하는 이유는,
현실에는 선언적으로 표현하기 애매한 상황이 실제로 있기 때문이다 — 토스트 팝업을 열고
닫거나, 특정 input에 포커스를 주거나, 비디오를 재생/정지시키는 것처럼 "명령형" 동작이
자연스러운 경우.

## 왜 굳이 forwardRef가 필요한가

함수형 컴포넌트는 기본적으로 `ref`를 prop처럼 받지 못한다. `ref`는 React 내부에서
특별하게 처리되는 예약어라서, 그냥 `props.ref`로 접근할 수 없다. `forwardRef`는
부모가 넘긴 `ref`를 컴포넌트 함수의 두 번째 인자로 받을 수 있게 열어주는 어댑터다.

```jsx
function BottomToast(props, propRef) { // propRef가 forwardRef를 통해 전달됨
  const toastPopupRef = useRef(null);
  const overlayRef = useRef(null);
  const [texts, setTexts] = useState([]);

  const openToastPopup = useCallback((textList) => {
    setTexts(Array.isArray(textList) ? textList : [textList]);
    toastPopupRef.current.style.display = 'block';
    overlayRef.current.style.display = 'block';
  }, []);

  const closeToastPopup = useCallback(() => {
    toastPopupRef.current.style.display = 'none';
    overlayRef.current.style.display = 'none';
  }, []);

  useImperativeHandle(propRef, () => ({
    openToastPopup,
    closeToastPopup,
  }));

  return (
    <>
      <div ref={overlayRef} />
      <div ref={toastPopupRef}>
        {texts.map((text, i) => <p key={i}>{text}</p>)}
      </div>
    </>
  );
}

export default forwardRef(BottomToast);
```

## useImperativeHandle: 노출 범위를 직접 통제한다

`forwardRef`만 쓰면 부모는 자식의 최상위 DOM 노드에 직접 접근하게 된다 — 이건
캡슐화가 전혀 없는 상태다. `useImperativeHandle`은 여기에 경계를 하나 세운다:
부모에게 실제로 노출할 인터페이스를 직접 정의하는 것이다. 위 예시에서 부모가 받는
`ref.current`는 실제 DOM이 아니라 `{ openToastPopup, closeToastPopup }` 객체다.
부모는 이 두 메서드 외에는 `toastPopupRef`나 `overlayRef` 같은 내부 상태에 전혀
접근할 수 없다 — 클래스의 `public` 메서드와 `private` 필드를 나누는 것과 같은 역할을
ref 레벨에서 하는 셈이다.

## useCallback이 여기서 왜 필수인가

`openToastPopup`, `closeToastPopup`을 `useCallback([])`으로 감싼 이유는 단순
최적화가 아니다. 이 함수들은 `useImperativeHandle`을 통해 **부모에게 넘어가는
안정적인 참조**가 돼야 한다. 만약 `useCallback` 없이 매 렌더마다 새 함수를 만들면,
`useImperativeHandle`의 두 번째 인자(의존성 배열)를 제대로 관리하지 않는 이상
부모가 들고 있는 `ref.current.openToastPopup`이 오래된(stale) 클로저를 참조하게 될
위험이 생긴다. 즉 이 패턴에서 `useCallback`은 "노출되는 API의 참조 안정성을 보장하는
장치"로 봐야 정확하다.

## 언제 써야 하고, 언제 쓰면 안 되는가

이 패턴은 강력한 만큼 남용하기도 쉽다. 판단 기준은 하나다 — **부모가 자식의 "언제
실행할지"만 결정하고, "무엇을 어떻게 렌더링할지"는 자식이 계속 소유하는가**. 토스트
팝업, 모달, 포커스 제어처럼 자식이 자기 내부 애니메이션/타이밍을 스스로 관리해야 하는
경우가 여기 해당한다.

반대로 부모가 자식의 *데이터*를 제어하고 싶은 거라면 (예: 자식이 보여줄 목록을
바꾸고 싶다) 이건 imperative handle이 아니라 그냥 props로 풀어야 하는 문제다.
"props로 표현하기 귀찮으니 ref로 우회한다"는 선택은 나중에 그 컴포넌트를 테스트하거나
서버사이드 렌더링할 때 반드시 대가를 치른다 — imperative API는 스냅샷 테스트나
선언적 상태 추적이 안 되기 때문이다.

## 참고: React 19 이후

React 19부터는 함수형 컴포넌트가 `ref`를 일반 prop처럼 직접 받을 수 있게 되어
`forwardRef`로 감싸는 절차 자체가 필수는 아니게 됐다. 다만 `useImperativeHandle`로
노출 범위를 좁히는 패턴 자체는 그대로 유효하다 — 바뀐 건 ref를 "전달받는 방법"이지,
"무엇을 노출할지 스스로 통제해야 한다"는 원칙이 아니다.
