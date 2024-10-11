## Infinite Queries

React Query는 무한 스크롤을 쉽게 구현할 수 있게 해준다.

무한 리스트(Infinite List)에서는 `useQuery`가 현재 `queryKey`에 대한 데이터만 표시할 수 있다는 점이 오히려 불리하게 작용한다.

우리가 정말 원하는 것은 매번 새로운 데이터를 가져올 때마다 데이터를 추가할 수 있는 **단일 캐시 항목을 갖는 것**이다.

### `useInfiniteQuery` 와 `useQuery`의 차이점

`useInfiniteQuery` 훅은 `useQuery`와 거의 동일하게 작동하지만 몇 가지 근본적인 차이점이 있다.

무한 리스트와 페이지네이션 리스트에서 데이터를 가져올 때, 데이터를 청크 단위로 시간에 따라 가져온다.
이를 위해선 이미 가져온 데이터와 다음에 가져올 데이터를 구분할 수 있는 방법이 필요하다.

보통 이 과정은 다음과 같이 페이지 번호(Page number)를 통해 수행된다.

```js
const [page, setPage] = React.useState(1)

...

const { data, status } = useRepos(sort, page)
```

- 페이지네이션 예제에서는 React 상태로 페이지를 생성하고, 사용자가 UI를 통해 이를 증감시킬 수 있게 해준다.
- 그런 다음 페이지 번호를 사용자 정의 훅에 전달하여 `queryKey`와 `queryFn` 내부에서 사용할 수 있도록 했다.

여기서 React 상태로 관리한다는 개념에 대해 잠깐 살펴보자.

무한 스크롤을 구현할 때 원래는 페이지 번호를 직접 관리해야 했다.
예를 들어, 페이지 1에서 시작해서 다음 페이지로 갈 때마다 직접 페이지 번호를 업데이트하고 그 번호로 데이터를 요청했다.

하지만 `useInfiniteQuery`훅을 사용하면 알아서 페이지 번호를 관리해주기 때문에 페이지 번호를 직접 관리할 필요가 없다.

### `useInfiniteQuery`을 사용한 구현 방식

`useInfiniteQuery`를 사용하게 되면 React 상태로 페이지를 직접 관리할 필요없이 `useInfiniteQuery`가 대신 관리해준다.

기존에 **React 상태로 페이지를 관리하는 방법**에 대해 알아보자.

```js
export async function fetchPosts(page) {
  const url = `https://dev.to/api/articles?per_page=6&page=${page}`;
  const response = await fetch(url);

  if (!response.ok) {
    throw new Error(`Failed to fetch posts for page #${page}`);
  }

  return response.json();
}
```

기존엔 `page`라는 값을 받아서 그 페이지의 post 목록을 가져온다.

하지만 `useInfiniteQuery` 를 사용하게 되면 `initialPageParam`이라는 설정을 통해 `useInfiniteQuery`에 전달할 수 있다.

```js
function usePosts() {
  return useInfiniteQuery({
    queryKey: ["posts"],
    queryFn: ({ pageParam }) => fetchPosts(pageParam),
    initialPageParam: 1,
  });
}
```

위 코드처럼 `initialPageParam`을 사용하면 필요한 경우 시작한 페이지 번호만 설정해주면 된다.
예시에서 `initialPageParam`는 1이므로 `useInfiniteQuery`는 처음에 페이지 1부터 데이터를 가져오고, 스크롤이 진행될수록 자동으로 페이지를 증가시켜서 다음 데이터를 가져오게 된다.

### `QueryFunctionContext`

이전에 사용한 적은 없지만 React Query는 항상 `queryFn` 함수에 `QueryFunctionContext`라는 객체를 전달한다.

`QueryFunctionContext`는 해당 쿼리에 대한 정보를 포함하고 있다.

`QueryFunctionContext`를 통해 우리는 초기 `pageParam`(페이지 번호)에 접근할 수 있다.
쉽게 말해 객체가 전달해주는 정보를 통해 현재 페이지 번호를 가져오는 것이다.

이제부터 React Query에 **다음 페이지를 어떻게 가져올지**에 대해 알아보자.

### `getNextPageParam`

`getNextPageParam` 메서드를 `useInfiniteQuery` 훅의 옵션 객체에 추가함으로써 다음 페이지를 어떻게 가져올지 React Query에 알려줄 수 있다.

```js
function usePosts() {
  return useInfiniteQuery({
    queryKey: ["posts"],
    queryFn: ({ pageParam }) => fetchPosts(pageParam), // 페이지 데이터를 가져오는 함수
    initialPageParam: 1, // 첫 페이지는 1부터 시작
    getNextPageParam: (lastPage, allPages, lastPageParam) => {
      if (lastPage.length === 0) {
        return undefined; // 마지막 페이지가 비어있으면 더 이상 가져올 페이지가 없다는 의미
      }

      return lastPageParam + 1; // 다음 페이지로 넘어감
    },
  });
}
```

`getNextPageParam`는 총 3가지 인자를 받는다.

1. lastPage : 마지막으로 가져온 페이지의 데이터
2. allPages : 지금까지 가져온 모든 페이지의 배열
3. lastPageParam : 마지막 페이지를 가져올 때 사용한 페이지 번호

위 코드의 경우 마지막 페이지 번호에 1을 더해 다음 페이지를 반환한다.
더해서 더 이상 가져올 페이지가 없다고 React Query에 알리고 싶다면, `undefined`를 반환하면 된다.

### `useInfiniteQuery`에서 데이터를 꺼내는 방법

지금까지는 `useInfiniteQuery`를 사용해서 데이터를 캐시에 넣는 방법에 대해 알아봤다.
이젠 데이터를 어떻게 꺼낼 수 있는지 알아보자.

#### `useQuery`와 `useInfiniteQuery`의 차이점

`useQuery`에서는 단순히 `queryKey`에 저장된 데이터를 반환한다.
`useInfiniteQuery`에서는 데이터와 **데이터가 속한 페이지 정보**가 함께 반환된다.

이를 위해 `useInfiniteQuery`는 데이터를 페이지 별로 나눠 다차원 배열 형태로 제공된다.
즉 배열의 각 요소는 특정 페이지에 대한 모든 데이터를 나타낸다.

데이터 구조는 다음과 같다.

```js
{
  "data": {
    "pages": [
      [{}, {}, {}], // 첫 번째 페이지 데이터
      [{}, {}, {}], // 두 번째 페이지 데이터
      [{}, {}, {}] // 세 번째 페이지 데이터
    ],
    "pageParams": [1, 2, 3] // 각 페이지의 페이지 번호
  }
}

// 평평한 배열 형태로 변환하고 싶은 경우
// JS의 Array.flat 메서드를 사용하면 된다.
// 페이지별로 나눠진 데이터를 하나의 배열로 병합해준다.

const { data } = usePosts()

const posts = data?.pages.flat() // [ {}, {}, {} ]
```

### 무한 스크롤을 제대로 구현하는 법

#### 페이지네이션에서 페이지를 관리하는 법

```js
<button onClick={() => setPage((p) => p + 1)}>Next</button>
```

페이지네이션 예제에서는 React 상태를 사용해 페이지를 관리했기 때문에 다음 페이지로 이동하려면 버튼 클릭 시 상태를 증가시키면 됐다.

하지만 `useInfiniteQuery`가 페이지를 관리해주고 있기 때문에 페이지 상태를 직접 관리할 필요가 없다고 앞에서 언급했다.

`useInfiniteQuery`에는 `fetchNextPage`라는 함수를 제공하는데 이 함수는 호출될 때 자동으로 `getNextPageQuery`을 통해 다음 페이지를 가져와서 `queryFn`을 호출한다.

```js
const { status, data, fetchNextPage } = usePosts()

// fetchNextPage를 사용한 예제
<button onClick={() => fetchNextPage()}>
  Load More
</button>
```

이렇게 하면 버튼을 클릭할 때마다 다음 페이지의 데이터를 가져와 리스트에 추가하게 된다.

추가적으로 쿼리 상태에 대한 메타 정보를 제공할 수 있다.

- `isFetchingNextPage` : 다음 페이지 데이터를 요청 중일 때 `true`로 설정된다.
- `hasNextPage` : 더 가져올 페이지가 있을 때 `true`이다. 이는 `getNextPageParam`에서 `undefined`가 반환되지 않은 경우에 결정된다.

두 값을 활용해 `More` 버튼을 조건부로 비활성화한다.
그리고 React Query가 다음 페이지를 가져오는 동안 Loading Indicator를 표시할 수 있다.

```js
<button
  onClick={() => fetchNextPage()}
  disabled={!hasNextPage || isFetchingNextPage} // 더 가져올 페이지가 없거나 요청 중이면 버튼 비활성화
>
  {isFetchingNextPage ? "..." : "More"} // 데이터를 가져오는 동안 로딩 표시
</button>
```

- `disabled` 속성 : `hasNextPage`가 `false` 또는 `isFetchingNextPage`가 `true` 일 때 버튼이 비활성화 된다. 더 이상 가져올 데이터가 없거나, 데이터를 요청 중일 때는 버튼을 클릭할 수 없도록 한다.

- 버튼 텍스트 변경 : `isFetchingNextPage`가 `true`인 경우 로딩 인디케이터가 표시되며, 그렇지 않으면 `More`로 표시된다.

### Infinite Query를 양방향으로 사용하는 방법

뒤로 데이터를 가져오는 것은 앞쪽으로 데이터를 가져오는 것과 패턴이 유사하다.

```js
useInfiniteQuery({
  queryKey, // 쿼리 키
  queryFn, // 데이터를 가져오는 함수
  initialPageParam, // 처음 시작할 페이지 번호
  getNextPageParam: (lastPage, allPages, lastPageParam) => {
    // 다음 페이지를 가져오는 함수
    if (lastPage.length === 0) {
      return undefined; // 더 이상 데이터가 없으면 undefined 반환
    }
    return lastPageParam + 1; // 다음 페이지로 이동
  },
  getPreviousPageParam: (firstPage, allPages, firstPageParam) => {
    // 이전 페이지를 가져오는 함수
    if (firstPageParam <= 1) {
      return undefined; // 첫 페이지에 도달하면 undefined 반환
    }
    return firstPageParam - 1; // 이전 페이지로 이동
  },
});
```

### 끊임없이 스크롤하도록 만드는 경험을 추가하는 방법

사용자가 끊임없이 스크롤하도록 만드는 경험을 추가하고 싶다면, 가장 기본적인 아이디어는 사용자가 리스트의 하단에 도달할 때 자동으로 `fetchNextPage`를 호출하는 것이다.

조금 더 쉽게 작업을 하기 위해 `useHooks` 라이브러리의 `useIntersectionObserver` 훅을 활용할 수 있다.

`useIntersectionObserver` 훅은 특정 DOM 요소가 뷰포트에 들어왔는지 감지하는 기능을 제공한다.

### `useIntersectionObserver` 훅의 동작 방식

- `ref` : 특정 DOM 요소에 연결되는 참조이다. 이 참조를 통해 요소가 화면에 들어오는지 추적할 수 있다.

- `entry` : 이 객체는 해당 요소가 화면에 얼마나 보이는지를 알려준다.

리스트이 마지막 요소 근처에 `ref`를 추가해, 사용자가 그 요소를 스크롤할 때 자동으로 다음 페이지 데이터를 가져오는 방식으로 무한 스크롤을 구현할 수 있다.

```js
import * as React from "react";
import { useInfiniteQuery } from "@tanstack/react-query";
import { fetchPosts } from "./api";
import { useIntersectionObserver } from "@uidotdev/usehooks";

function usePosts() {
  return useInfiniteQuery({
    queryKey: ["posts"],
    queryFn: ({ pageParam }) => fetchPosts(pageParam),
    staleTime: 5000,
    initialPageParam: 1,
    getNextPageParam: (lastPage, allPages, lastPageParam) => {
      if (lastPage.length === 0) {
        return undefined;
      }

      return lastPageParam + 1;
    },
  });
}

export default function Blog() {
  const { status, data, fetchNextPage, hasNextPage, isFetchingNextPage } =
    usePosts();

  const [ref, entry] = useIntersectionObserver(); // 뷰포트에 요소가 들어왔는지 감지하는 훅 사용

  // 리스트의 마지막 요소가 뷰포트에 들어오면 fetchNextPage 호출
  React.useEffect(() => {
    if (entry?.isIntersecting && hasNextPage && !isFetchingNextPage) {
      fetchNextPage(); // 다음 페이지를 가져오는 함수 호출
    }
  }, [entry?.isIntersecting, hasNextPage, isFetchingNextPage]);

  // 상태 취소 (pending, error...)

  return (
    <div>
      {data.pages.flat().map((post, index, pages) => (
        <p key={post.id}>
          <b>{post.title}</b>
          <br />
          {post.description}
          {index === pages.length - 3 ? ( // 리스트의 마지막 근처 요소에 ref 연결
            <div ref={ref} /> // 이 요소가 화면에 들어오면 다음 페이지 데이터를 가져옴
          ) : null}
        </p>
      ))}
    </div>
  );
}
```

#### 동작 방식

1. `useIntersectionObserver` : 이 훅은 `ref`와 `entry`를 반환한다. `ref`는 감시할 DOM 요소에 연결하고, `entry`는 그 요소가 화면에 보이는지 여부를 제공한다.

2. `React.useEffect` : `entry?.isIntersceting`이 `true`가 되고, 아직 `isFetchingNextPage`가 `false`일 때, `fetchNextPage`를 호출하여 다음 페이지 데이터를 불러온다.

3. 자동 무한 스크롤 : 사용자가 리스트의 마지막에 도달할 때마다 다음 페이지가 자동으로 로드된다.

### Infinite Query에서 `refetching`하는 방법

React Query의 가장 큰 장점 중 하나는 **자동 리패칭**을 통해 데이터를 백그라운드에서 항상 최신 상태로 유지한다는 점이다.

이를 통해 사용자가 보는 데이터는 항상 신선한 상태를 유지하게 된다.

### Infinite Query에서 `refetching`의 작동 방식

React Query는 캐시에 있는 첫 번째 페이지부터 `refetching`을 시작한다.
그리고 `getNextPageParam`을 호출해 다음 페이지를 가져온 후, 계속해서 페이지를 `refetching`한다.

위 과정은 모든 페이지가 `refetching`되거나 `getNextPageParam`에서 `undefined`가 반환될 때까지 계속된다.

이 방식이 중요한 이유는 데이터의 일관성 때문인데, 무한 쿼리는 **하나의 캐시 항목**으로 저장되기 때문에 각 페이지는 개별적인 요청이지만 결국 UI에서 하나의 긴 리스트를 형성한다.

만약 일부 페이지만 `refetching`한다면, React Query는 데이터의 일관성을 보장할 수 없게된다.

예를 들어 캐시에 2개의 페이지가 있고 각 페이지에는 4개의 항목이 있다고 가정해보자.
첫 번째 페이지에는 ID가 1~4까지, 두 번째 페이지에는 ID가 5~8까지 있다고 하자.

만약 `BackEnd`에서 ID 3이 삭제되었는데 우리가 첫 번째 페이지만 `refetching` 한다면 어떻게 될까?

첫 번째 페이지에서는 ID 3이 삭제되지만 두 번쨰 페이지는 `refetching`되지 않아 여전히 ID 5~8까지 보여준다.

그렇게 되면 두 페이지에서 ID 5가 중복으로 표시되는 문제가 발생하게 된다.

따라서 React Query는 전체 페이지를 다시 가져와 데이터의 일관성을 보장하는 것이다.

또 다른 예시로 첫 번째 페이지에 ID가 0이라는 새 항목이 생겼다고 가정하자. 근데 만약 우리가 첫 번째 페이지만 refetching되었다면, 캐시에서 ID 4에 해당하는 항목이 누락될 수 있다.

이런 상황 때문에 React Query는 Infinite Query를 refetching 할 때 절대 단축 경로를 사용하지 않는다.

항상 모든 페이지를 다시 가져와서 데이터 일관성을 보장해야 한다.

### `maxPages` 옵션

캐시에 많은 페이지가 있는 경우, **네트워크 비용**과 **메모리 사용** 면에서 문제가 발생할 수 있다.
모든 페이지를 계속해서 캐시에 유지하면 성능 저하로 이어질 수 있다.

이를 방지하기 위해 `useInfiniteQuery`의 `maxPages` 옵션을 사용할 수 있다.
이 옵션은 React Query가 캐시에 유지할 페이지 수를 제한한다.

예를 들어 `maxPages`를 3으로 설정하면, 양방향 무한 쿼리를 사용하더라도 최대 3개의 페이지만 캐시에 남게 된다.

이를 통해 메모리 사용을 최적화 할 수 있다.

```js
useInfiniteQuery({
  queryKey: ["posts"],
  queryFn: ({ pageParam }) => fetchPosts(pageParam),
  staleTime: 5000,
  initialPageParam: 1,
  getNextPageParam: (lastPage, allPages, lastPageParam) => {
    if (lastPage.length === 0) {
      return undefined;
    }
    return lastPageParam + 1;
  },
  getPreviousPageParam: (firstPage, allPages, firstPageParam) => {
    if (firstPageParam <= 1) {
      return undefined;
    }
    return firstPageParam - 1;
  },
  maxPages: 3, // 캐시에 최대 3개의 페이지만 유지
});
```

### 마무리

`useInfiniteQuery`는 `useQuery`보다 조금 복잡할 수 있지만, 이를 통해 제공되는 UX는 다른 방법으로 구현하기 어려울 정도로 강력하다.

`useInfiniteQuery`는 React Query의 다른 기능들과 마찬가지로 약간의 설정만으로도 캐시 관리의 복잡함을 자동으로 처리해준다.

### 정리

1. `useInfiniteQuery`는 무한 스크롤을 쉽게 구현할 수 있으며, 페이지 번호를 자동으로 관리해 데이터 일관성을 유지한다.
2. `getNextPageParam`과 `getPreviousPageParam`을 사용해 다음 또는 이전 페이지를 가져올 수 있다.
3. 캐시 관리의 복잡성은 자동으로 처리되며, `maxPages` 옵션을 사용해 메모리 사용을 최적화할 수 있다.
4. React Query는 이 과정을 통해 사용자가 더 나은 사용자 경험을 누릴 수 있도록 도와준다.
