---
title: "React Query useQuery로 서버 상태 관리하기: 캐싱이 실제로 하는 일"
date: 2025-06-02
categories: [프론트 화면 관련 글, React]
tags: [react, frontend, react-query]
---

`useState`로 서버 데이터를 관리해본 적이 있다면 로딩 상태, 에러 처리, 캐싱, 중복 요청
방지 같은 걸 전부 손으로 짜야 했던 경험이 있을 것이다. React Query(TanStack Query)의
`useQuery`는 이걸 대신해주는데, 여기서 중요한 건 "편하다"가 아니라 **서버에서 온 데이터는
클라이언트 state와 근본적으로 다른 종류의 상태**라는 인식이다.

## 서버 상태는 "내 것"이 아니다

`useState`로 관리하는 값(입력 필드, 모달 열림 여부 등)은 클라이언트가 진실의 원천(source
of truth)이다. 반면 서버에서 가져온 데이터는 진실의 원천이 서버에 있고, 클라이언트가
들고 있는 건 그 시점의 **스냅샷**일 뿐이다. 이 스냅샷은 시간이 지나면 낡는다(stale) —
다른 사용자가 데이터를 바꿨을 수도 있고, 내가 다른 탭에서 수정했을 수도 있다. React
Query가 관리하는 건 정확히 이 "낡음"의 문제다.

## 설정

```jsx
const queryClient = new QueryClient();

root.render(
  <QueryClientProvider client={queryClient}>
    <App />
  </QueryClientProvider>
);
```

## 기본 사용

```jsx
const fetchPosts = async () => {
  const response = await fetch('https://api.example.com/posts');
  if (!response.ok) throw new Error('네트워크 응답이 좋지 않습니다.');
  return response.json();
};

function PostsList() {
  const { isLoading, isError, data, error, refetch } = useQuery({
    queryKey: ['posts'],
    queryFn: fetchPosts,
  });

  if (isLoading) return <div>불러오는 중...</div>;
  if (isError) return <div>오류: {error.message}</div>;

  return (
    <ul>
      {data.map((post) => (
        <li key={post.id}>{post.title}</li>
      ))}
    </ul>
  );
}
```

## queryKey는 단순 식별자가 아니라 캐시의 좌표다

`queryKey: ['posts']`는 이 쿼리 결과를 캐시에 저장할 위치이자, 나중에 "이 데이터를
다시 불러와라" 또는 "이 데이터는 이제 낡았다"고 지시할 때 가리키는 좌표다. 같은 키로
다시 요청하면 캐시에 있는 값을 즉시 보여주면서(로딩 스피너 없이) 백그라운드에서
재검증한다 — 이게 **stale-while-revalidate** 패턴이다.

키 설계가 실제로 중요해지는 지점은 파라미터가 붙을 때다. `['posts', { page, filter }]`처럼
파라미터를 키에 포함시키면, 페이지나 필터가 바뀔 때마다 자동으로 별개의 캐시 엔트리로
취급된다. 반대로 파라미터를 키에서 빼먹으면, 필터를 바꿔도 이전 필터의 캐시된 결과를
그대로 보여주는 버그가 생긴다 — React Query를 쓰면서 가장 흔하게 겪는 실수다.

## "새로고침 버튼"이 있다는 건 대개 설계가 잘못됐다는 신호

`refetch()`를 수동 새로고침 버튼에 연결하는 패턴은 편해 보이지만, 실무에서는 이게
필요하다는 사실 자체가 캐시 무효화 전략을 제대로 안 짰다는 신호인 경우가 많다.
데이터를 변경하는 뮤테이션(글 작성, 삭제 등) 이후에는 사용자가 버튼을 누르게 하는 대신
`queryClient.invalidateQueries(['posts'])`로 관련 쿼리를 자동으로 무효화하는 게
정석이다. 수동 새로고침 버튼은 "언제 데이터가 낡는지 모르겠으니 사용자에게 맡긴다"는
설계 포기에 가깝다.

## staleTime을 안 정하면 생기는 일

기본값으로는 데이터가 즉시 stale로 간주되어, 컴포넌트가 다시 마운트되거나 윈도우가
포커스를 얻을 때마다 재요청이 나간다. 자주 안 바뀌는 데이터(예: 카테고리 목록)에
기본값을 그대로 쓰면 불필요한 네트워크 요청이 계속 발생한다. `staleTime`을 명시적으로
설정해서 "이 데이터는 5분 동안은 신선하다고 간주해도 된다"고 알려주는 것이, 캐싱
라이브러리를 쓰는 진짜 이유에 가깝다 — 기본값만 쓰면 캐싱의 이점을 절반만 얻는 셈이다.

## 언제 안 쓰는 게 나은가

폼 입력값, 모달 열림 여부, 탭 선택 상태처럼 **서버와 무관한 순수 클라이언트 상태**에는
React Query가 맞지 않는다. 이런 값은 애초에 "낡을" 대상이 아니라서 캐싱/재검증
메커니즘이 오버엔지니어링이 된다 — `useState`나 `useReducer`로 충분하다. React Query는
"서버에 원본이 있고, 그 원본과 내 화면을 동기화해야 하는" 데이터에만 쓰는 도구라고
구분해두면 남용을 피할 수 있다.

## `useQuery`가 반환하는 값들

| 값 | 의미 |
|---|---|
| `data` | 성공 시 실제 데이터. 로딩/에러 중엔 `undefined`거나 이전 캐시값 |
| `isLoading` | 최초 로딩 중인지 |
| `isError` / `error` | 실패 여부와 에러 상세 |
| `refetch` | 수동 재요청(단, 위에서 말했듯 남용하지 않는 게 좋다) |
