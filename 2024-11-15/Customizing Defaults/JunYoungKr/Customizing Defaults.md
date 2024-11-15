# Managing Query Keys

우리는 지금까지 `staleTime`, `refetchInterval`, `refetchOnMount`, `refetchOnWindowFocus`, `refetchOnReconnect`, `gcTime`, `enabled` 등 여러 옵션들을 사용했다.

그리고 쿼리나 뮤테이션을 커스터마이징할 필요가 있을 때마다, `useQuery`나 `useMutation`에 옵션 객체를 직접 전달하여 처리해왔다.

위 방식들은 잘 작동했지만 프로젝트 규모가 커질수록 동일한 옵션을 반복해서 사용할 될 수 있다.
예를 들어, 쿼리마다 별도로 설정하지 않는 한 기본 `staleTime`은 10초로 설정하고 싶을 때가 있을 것이다.
이를 해결하기 위해서는 **공통 옵션 객체**를 만들어 필요한 곳에서 가져와 사용하는 것이다.

### defaultQueryOptions

```js
// queryOptions.js
export const defaultQueryOptions = {
  staleTime: 10000, // 10초 동안 데이터가 최신으로 간주됨
  refetchOnWindowFocus: false, // 윈도우 포커스 시 리패치 비활성화
  retry: 2, // 실패 시 최대 2번 재시도
};

// MyComponent.js
import { useQuery } from "react-query";
import { defaultQueryOptions } from "./queryOptions";

function MyComponent() {
  const { data, error } = useQuery("myData", fetchMyData, {
    ...defaultQueryOptions,
    // 추가로 개별 설정을 덮어쓸 수도 있습니다.
    enabled: true,
  });

  return <div>{data ? "데이터 로드 성공" : "데이터 없음"}</div>;
}
```

위 코드처럼 반복적으로 사용되는 옵션들을 `defaultQueryOptions`를 따로 컴포넌트화 후 `MyComponent`에 불러와서 사용하는 것이다.
이 방법은 정상적으로 작동하지만, **확장성이 부족하고 유지보수가 어려운 솔루션이다**.

**다행히도** React Query는 이를 더 간단히 해결할 수 있는 **Query Defaults** 기능을 제공한다.

React Query에서는 useQuery에 전달할 수 있는 대부분의 옵션 (단, queryKey는 제외)은
`queryClient`를 생성할 때 `defaultOptions` 객체를 통해 기본값을 설정할 수 있다.

### queryClient 생성 시 defaultOptions 객체를 통해 기본값 설정

```js
const queryClient = new QueryClient({
  defaultOptions: {
    queries: {
      staleTime: 10 * 1000,
    },
  },
});

function MyApp({ Component, pageProps }) {
  return (
    <QueryClientProvider client={queryClient}>
      <Component {...pageProps} />
    </QueryClientProvider>
  );
}
```

위 `queryClient`는 전역적으로 설정하는 것이다. App.tsx에 위치시킨 queryClient에 적용하면 된다.

### Fuzzy Matching (유사 일치)

Fuzzy Matching은 특정 쿼리 키 집합에 대해서만 `defaultOptions`을 정의할 수 있는 유연성을 제공한다.

```js
["todos", "list", { sort: "id" }][("todos", "list", { sort: "title" })][
  ("todos", "detail", "1")
][("todos", "detail", "2")][("posts", "list", { sort: "date" })][
  ("posts", "detail", "23")
];
```

위 같은 쿼리 키들이 존재한다고 가정해보자. 그리고 `todos/detail/n`에 대해서만 `staleTime`을 10초로 설정하고 싶다고 가정해보자.
이를 위해 `queryClient.setQueryDefaults` 메서드를 사용할 수 있다.
`queryKey`와 적용할 옵션을 전달하면, 해당 쿼리 키와 일치하는 모든 쿼리에 기본 옵션이 적용된다.

```js
queryClient.setQueryDefaults(["todos", "detail"], {
  staleTime: 10 * 1000, // 'todos/detail' 쿼리 키에 대해서만 staleTime을 10초로 설정
});
```

`Fuzzy Matching` 덕분에 ['todos', 'detail']과 일치하는 모든 쿼리는 기본 `staleTime`으로 10초를 상속받게 된다.

### 옵션 설정 우선순위

React Query는 다음과 같은 우선순위로 옵션을 적용한다.

1. 전역 기본값 (`queryClient` 생성 시 설정)
2. 부분 쿼리 기본값 (`setQueryDefaults`로 설정)
3. 개별 쿼리 옵션 (`useQuery` 또는 `useMutation`에서 설정)

이렇게 하면 좀 더 유연하고 세밀한 옵션 설정을 통해 애플리케이션의 다양한 요구사항을 쉽게 만족시킬 수 있다.

앞서 언급했듯이 `queryKey`를 제외한 `useQuery`에 전달할 수 있는 모든 옵션은 기본값으로 설정할 수 있는데
심지어는 `QueryFn`도 기본값으로 지정할 수 있다.

#### 왜 유용할까?

모든 요청이 동일한 API로 향하는 경우, 기본 `queryFn`을 설정하면 중복된 코드 작성을 줄이고, 코드의 일관성을 유지할 수 있다.

```js
function usePostList() {
  return useQuery({
    queryKey: ["posts"],
    queryFn: async () => {
      const response = await fetch("/api/posts");

      if (!response.ok) {
        throw new Error("Failed to fetch posts");
      }

      return response.json();
    },
  });
}

function usePost(path) {
  return useQuery({
    queryKey: ["posts", path],
    queryFn: async () => {
      const response = await fetch(`/api/posts${path}`);

      if (!response.ok) {
        throw new Error(`Failed to fetch post: ${path}`);
      }

      return response.json();
    },
  });
}
```

위 2개의 `usePostList`와 `usePost`는 각각 다른 `queryKey`를 사용하고 있지만 코드의 중복이 발생한다.
만약 우리가 **D.R.Y(Don't Repeat Yourself)** 원칙을 따른다고 한다면, 중복된 `queryFn`을 하나의 공통 함수로 추출하려고 할 것이다.

```js
async function fetchPosts(path = "") {
  const baseUrl = "/api/posts";
  const response = await fetch(baseUrl + path);

  if (!response.ok) {
    throw new Error("Failed to fetch");
  }

  return response.json();
}
```

`fetchPosts` 함수를 통해 공통의 fetch 로직을 추출했다.
하지만 여전히 각 쿼리에서 `queryFn`을 정의해야 하고, 올바른 `path`를 전달해야 한다는 문제가 있다.

```js
function usePostList() {
  return useQuery({
    queryKey: ["posts"],
    queryFn: () => fetchPosts(),
  });
}

function usePost(path) {
  return useQuery({
    queryKey: ["posts", path],
    queryFn: () => fetchPosts(path),
  });
}
```

### setQueryDefaults

`setQueryDefaults`를 사용해 `['posts']` 키와 일치하는 모든 쿼리에 대해 기본 `queryFn`을 설정하면 어떨까?
설정하게 되면 `queryFn`을 정의할 필요가 없어져 아래와 같이 간단하게 쿼리를 사용할 수 있다.

```js
function usePostList() {
  return useQuery({
    queryKey: ["posts"],
  });
}

function usePost(path) {
  return useQuery({
    queryKey: ["posts", path],
  });
}
```

이 방법의 핵심은 `queryKey`에서 요청 URL을 유도할 수 있어야 한다는 것이다.
React Query에서는 `queryFn` 내부에서 `QueryFunctionContext` 객체를 통해 `queryKey`에 접근할 수 있다.

### setQueryDefaults란?

setQueryDefaults는 `queryClient`에서 특정 `queryKey`에 대해 기본 설정값을 지정할 수 있게 해주는 메서드이다.
여기서 기본 `queryFn`을 설정하면, 해당 `queryKey`를 사용하는 모든 쿼리에 대해 자동으로 이 `queryFn`이 사용된다.

```js
queryClient.setQueryDefaults(["posts"], {
  queryFn: async ({ queryKey }) => {
    const baseUrl = "/api/";
    const slug = queryKey.join("/"); // queryKey를 '/'로 합쳐서 경로 생성
    const response = await fetch(baseUrl + slug);

    if (!response.ok) {
      throw new Error("fetch failed"); // 요청 실패 시 에러 처리
    }

    return response.json(); // JSON 형식의 데이터 반환
  },
  staleTime: 5 * 1000, // 데이터의 유효 기간 설정 (5초)
});
```

### QueryFunctionContext란?

React Query는 `queryFn`에 `QueryFunctionContext`라는 객체를 전달한다.
이 객체에는 중요한 정보들이 포함되어 있다.

- queryKey : 쿼리의 고유한 키
- meta : 추가적인 메타 데이터

예를 들어

1. queryKey : ['posts']라면 `/api/posts` 요청
2. queryKey : ['posts', '123']라면 `/api/posts/123` 요청

### 요약

React Query의 `setQueryDefaults` 기능은 특정 `queryKey`에 대한 기본 옵션을 설정해 반복되는 queryFn 코드 작성을 줄이고 유지보수를 쉽게 해줍니다. `QueryFunctionContext` 객체에서 `queryKey`를 사용해 요청 URL을 동적으로 생성할 수 있으며,
이를 통해 API 요청을 간결하게 관리할 수 있습니다. 옵션 적용 우선순위는 `전역 기본값 → setQueryDefaults 설정값 → 개별 쿼리 옵션 순`으로 이루어집니다.
