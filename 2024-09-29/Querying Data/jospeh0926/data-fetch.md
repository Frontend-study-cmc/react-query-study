## 02 - 데이터 패칭

웹 애플리케이션을 빌드할 때, Promise를 다루는 가장 일반적인 상황은, API에서 데이터를 가져오는 것입니다.
앞서 언급했듯이 이러한 Promise를 다루는데 특화되어있는 라이브러리가 React Query입니다.

단, 주의해야할 점은 React Query는 Promise가 어디서 오는지 신경 쓰지 않고, React Query가 비동기, Promise 기반의 상태 관리자라는 것을 계속 인지하고있어야합니다.

위의 주의점을 인지하고 있으면 아래의 질문같이 Promise가 어디서 오는지와 관련된 질문들에서 React Query는 자유롭다는 점입니다.

- 응답 헤더를 어떻게 읽을 수 있나요?
- 인증 토큰을 요청에 어떻게 추가할 수 있나요?
  위의 질문들에 대한 답변은 동일합니다 ⇒ React Query 없이 하던 대로 하시면 됩니다.

사실, React Query는 요청을 직접 실행하지 않기 때문에 이러한 것들에 대해 전혀 인식하지 못합니다.
**React Query가 신경 쓰는 것은 Promise의 상태와 그것의 데이터뿐입니다**.

물론, React Query에게 제공할 Promise를 생성할 방법은 필요하고, 이러한 방법으로 가장 많이 사용되는 것이 fetch와 axios 입니다.

```jsx
const promise = fetch(url, options);
// 또는
const promise = axios(url, config);
```

해당 예시처럼 fetch / axios가 호출되면, 브라우저는 즉시 요청을 시작하고 Promise를 반환합니다.

그 이후로, 응답을 받는 것은 일반적으로 두 단계로 이루어집니다.

1. 서버가 응답하는 즉시 Fetch에서 반환된 프로미스는 Response 객체로 해결됩니다. 이 객체에는 응답에 대한 정보(헤더, HTTP 상태 코드 등)가 포함되지만 실제 데이터는 포함되지 않습니다.
   - 주의할점은, fetch는 요청이 실패해도 Promise를 거부하지 않는다는 것입니다. 즉, 응답의 상태 코드가 4xx 또는 5xx 범위에 있더라도 Promise는 정상적으로 해결됩니다.
     - axios는 반대입니다. axios는 요청이 실패하면 Promise가 거부되고 try/catch 블럭으로 감싸있다면 catch 블럭에 잡힙니다.
   - 따라서 fetch를 통한 요청이 성공인지 실패인지 여부는 일반적으로 response.ok를 활용합니다

```jsx
const fetchRepos = async () => {
  try {
    const response = await fetch("https://api.github.com/orgs/TanStack/repos");

    if (response.ok) {
      // ...
    } else {
      throw new Error(`요청이 실패했습니다. 상태: ${response.status}`);
    }
  } catch (error) {
    // 네트워크 오류 처리
  }
};
```

반면 axios는 아래와 같습니다

```jsx
const fetchRepos = async () => {
  try {
    const response = await axios.get(
      "https://api.github.com/orgs/TanStack/repos"
    );
    return response.data;
  } catch (error) {
    if (error.response) {
      // 서버가 2xx 범위를 벗어나는 상태 코드로 응답한 경우
      throw new Error(`요청이 실패했습니다. 상태: ${error.response.status}`);
    } else if (error.request) {
      // 요청이 전송되었지만 응답을 받지 못한 경우
      throw new Error("서버로부터 응답을 받지 못했습니다.");
    } else {
      // 요청 설정 중에 오류가 발생한 경우
      throw new Error("요청 설정 중 오류가 발생했습니다: " + error.message);
    }
  }
};
```

2. 이제 실제 데이터를 가져오는 파트입니다. 응답 데이터가 json 형태일 경우, response.json을 호출하여 구문 분석된 json 데이터로 해결되는 또 다른 Promise를 반환할 수 있습니다.

```jsx
const fetchRepos = async () => {
  try {
    const response = await fetch("https://api.github.com/orgs/TanStack/repos");

    if (response.ok) {
      const data = await response.json();
      return data;
    } else {
      throw new Error(`요청이 실패했습니다. 상태: ${response.status}`);
    }
  } catch (error) {
    // 네트워크 오류 처리
  }
};
```

- 이 파트도 axios는 다릅니다, axios는 기본적으로 응답을 JSON으로 파싱합니다
  - 따라서 response.json 단계를 거칠 필요없이 response.data로 바로 접근가능합니다.

위에서 작업한 내용은 fetch / axios를 통해 외부 API에 요청을 보내고 응답을 가져오는 상황이였습니다.

이제, 이것을 useQuery를 활용하면 다음과 같은 결과를 얻을 수 있습니다.

```jsx
// fetch 기준

function useRepos() {
  return useQuery({
    queryKey: ["repos"],
    queryFn: async () => {
      const response = await fetch(
        "https://api.github.com/orgs/TanStack/repos"
      );

      if (!response.ok) {
        throw new Error(`요청이 실패했습니다. 상태: ${response.status}`);
      }

      return response.json();
    },
  });
}

// axios 기준

function useRepos() {
  return useQuery({
    queryKey: ["repos"],
    queryFn: async () => {
      const { data } = await axios.get(
        "https://api.github.com/orgs/TanStack/repos"
      );
      return data;
    },
  });
}
```

몇가지 차이점이 보입니다.

1. try/catch 코드를 제거할 수 있었습니다. queryFn에서 오류를 throw하여 React Query에 오류가 발생했음을 알리고, 이를 받은 쿼리는 쿼리의 상태를 error로(또는 isError) 설정할 수 있습니다.
2. response.json()을 직접 반환할 수 있었습니다(await 사용하지 않고).
   - 즉, 쿼리 함수는 최종적으로 Promise를 반환해야 합니다.
     - 만약 promise가 해결된것을 반환하면?
       - axios가 이경우에 해당합니다. axios는 내부적으로 Promise 해결한 data를 직접 반환하기때문에 data는 Promise가 아닌 js 객체, 배열등입니다.
       1. React Query는 이를 자동으로 Promise로 감싸서 처리합니다.
       2. 쿼리 함수가 즉시 완료된 것으로 간주되어 데이터가 즉시 사용 가능한 상태가 됩니다.
       3. 비동기 작업이 없기 때문에 로딩 상태가 매우 짧거나 거의 없을 수 있습니다.
       4. 캐시와 관련된 React Query의 기능들은 정상적으로 작동합니다.
          - 즉, 문제는 없습니다
          - 동기적인 데이터 소스나 이미 해결된 Promise를 반환하는 경우에도 문제없이 작동하도록 설계되어 있습니다
       - 하지만 일반적으로 쿼리 함수는 비동기 작업을 수행하므로, Promise를 반환하는 것이 가장 일반적이고 권장되는 패턴입니다.

그리고 useRepos 훅을 실제 앱에 넣으면 예상대로 작동합니다.

```jsx
import * as React from "react";
import {
  QueryClient,
  QueryClientProvider,
  useQuery,
} from "@tanstack/react-query";

const queryClient = new QueryClient();

function useRepos() {
  return useQuery({
    queryKey: ["repos"],
    queryFn: async () => {
      const response = await fetch(
        "https://api.github.com/orgs/TanStack/repos"
      );

      if (!response.ok) {
        throw new Error(`요청이 실패했습니다. 상태: ${response.status}`);
      }

      return response.json();
    },
  });
}

function Repos() {
  const { data, status } = useRepos();

  if (status === "pending") {
    return <div>...</div>;
  }

  if (status === "error") {
    return <div>데이터를 가져오는 중 오류가 발생했습니다</div>;
  }

  return (
    <ul>
      {data.map((repo) => (
        <li key={repo.id}>{repo.full_name}</li>
      ))}
    </ul>
  );
}

export default function App() {
  return (
    <QueryClientProvider client={queryClient}>
      <Repos />
    </QueryClientProvider>
  );
}
```

위에서 다룬것들은 매우 합리적이지만, 다루지 않은 시나리오가 하나 있습니다.  
지금까지의 예제는 모두 api가 정적이였습니다.
하지만, 실제 앱은 정적이지 않은 api들을 많이 다룹니다.

예시로 사용되는 Github API를 구체적으로 보면, 결과를 정렬하는 쿼리파라미터를 지정할 수 있습니다.
(sort=created, updated, pushed, full_name)

```jsx
fetch("https://api.github.com/orgs/TanStack/repos?sort=created");
```

따라서 sort에 대한 값을 url에 전달할 수 있도록 useRepos 훅을 변경할 수 있습니다

```jsx
function useRepos(sort) {
  return useQuery({
    queryKey: ["repos"],
    queryFn: async () => {
      const response = await fetch(
        `https://api.github.com/orgs/TanStack/repos?sort=${sort}`
      );

      if (!response.ok) {
        throw new Error(`요청이 실패했습니다. 상태: ${response.status}`);
      }

      return response.json();
    },
  });
}
```

이렇게 변경해볼수있지만, 이것은 실제로 동작하지 않습니다.
아마 이렇게 구성한 사람들의 생각은, React Query가 컴포넌트가 다시 렌더링될 때마다 queryFn을 다시 실행할 것이라고 가정하고 있을 것입니다.
하지만 그렇지 않습니다.

- queryFn이 재렌더링에 영향받지 않는 이유?
  - useEffect와 비슷합니다.
  - useEffect도 종속성 배열의 상태가 변경되지 않으면 재렌더링 여부와 상관없이 내부 로직은 실행되지 않습니다
  - useQuery도 마찬가지입니다. 쿼리 키의 값이 변경되지 않으면 queryFn을 다시 트리거하지 않습니다.

그러면 또 다른 방법으로 아래를 생각해볼수있습니다.
정렬이 바뀔때마다 강제로 refetch를 하는 방법입니다.

```jsx
    const { data, status, refetch } = useRepos(selection)

    onChange={(event) => {
      const sort = event.target.value
      setSelection(sort)
      refetch()
    }}

```

동작하긴합니다,,,
하지만 이것은 명령형입니다,,, 리액트는 선언형으로 작성하는 것을 권장하고 있습니다.

그렇다면 이를 선언적으로 해결하는 방법은 무엇일까요?
앞서 언급했듯이 queryKey를 활용하면됩니다.

useQuery는 queryKey 배열의 값이 변경될 때마다, React Query는 queryFn을 다시 실행합니다.
즉, 마치 useEffect의 종속성배열처럼 queryFn도 내부에서 사용하는 모든 값을 queryKey 배열에 포함시켜야 한다는 의미입니다.

- 하지만 useEffect와의 중요한 차이점도 존재합니다.
  - 참조적 안정성 문제 해결
    - useEffect의 경우, 의존성 배열에 객체나 배열을 넣으면 매 렌더링마다 새로운 참조가 생성되어 불필요한 실행이 발생할 수 있습니다.
      - 따라서 의존성 배열에 객체나 배열등을 넣는 것이 권장되지 않았습니다.
    - 반면, useQuery의 queryKey는 이런 문제가 없습니다.
  - 결정적 해싱
    - 결정적 해싱이란?
      - 동일한 입력에 대해 항상 동일한 해시 값을 생성하는 방식입니다.
    - React Query는 queryKey의 내용을 깊게 비교하고 결정적으로 해싱합니다. 즉, 객체나 배열의 내용이 같다면 다른 참조여도 같은 것으로 취급합니다.
    - 내부 작동방식
      - 참고: [@tanstack/query 깃허브 1](https://github.com/TanStack/query/blob/941ba88d20e4f5d3a027a4c9dd1320d5187169c1/packages/query-core/src/utils.ts#L221) (키가 객체일 경우 두 키(객체)를 비교하는 함수: b라는 키가 a라는 키의 부분집합인지 확인 ⇒ 쿼리 필터링, 캐시 조회 등에서 사용) / [@tanstack/query 깃허브 2](https://github.com/TanStack/query/blob/941ba88d20e4f5d3a027a4c9dd1320d5187169c1/packages/query-core/src/utils.ts#L205) (키를 고유한 문자열 해시로 변환하는 함수: 만약 키가 객체인 경우 정렬하여 새로운 객체를 만듬, 이후 들어오는 객체 키를 정렬했을때 같다면 동일한 해시를 생성하도록 하는 함수 ⇒ 캐시 키 생성, 캐시 비교 등에서 사용)
      - React Query는 queryKey의 모든 요소를 재귀적으로 순회합니다.
      - 객체의 키와 값, 배열의 요소 등 모든 데이터를 고려합니다.
      - 이 데이터를 기반으로 유일한 문자열 해시를 생성합니다.
      ```jsx
      // 이 두 queryKey는 다른 객체지만 같은 것으로 취급됩니다
      ["user", { id: 1, name: "John" }][("user", { name: "John", id: 1 })];
      ```
    - 반면 useEffect의 의존성배열은 얕은 비교를 하기 때문에 의도한 것과 다른 결과를 나타낼수도있습니다

이제 queryKey를 sort 매개변수를 포함하도록 업데이트해보겠습니다.

```jsx
function useRepos(sort) {
  return useQuery({
    queryKey: ["repos", { sort }],
    queryFn: async () => {
      const response = await fetch(
        `https://api.github.com/orgs/TanStack/repos?sort=${sort}`
      );

      if (!response.ok) {
        throw new Error(`요청이 실패했습니다. 상태: ${response.status}`);
      }

      return response.json();
    },
  });
}
```

이제 완벽하게 작동합니다.

- 위의 내용에 대한 비하인드씬

  - queryKey 배열의 값이 변경되면, 옵저버가 관찰하는 대상을 변경합니다.
  - 즉, 구독하는 키가 다른 키로 전환됩니다

  ```
  ['repos', { sort: 'created' }]
  ['repos', { sort: 'updated' }]
  ```

  - 구독하는 키가 변경되면, 해당 키에 대한 데이터를 캐시에서 읽으려고 시도합니다.
    - 만약 해당 캐시 항목에 대한 데이터가 없다면, 새로운 항목이 생성됩니다.
    - 새 항목은 pending 상태에서 시작하며, queryFn이 데이터를 가져오도록 호출됩니다.
  - 반면 이미 캐시에 있는 queryKey로 다시 전환하면, 옵저버가 관찰하는 대상을 변경하는 것은 동일하지만, 이미 캐시가 채워져 있으므로, 즉시 데이터를 제공하고 상태는 success입니다.

- 팁
  ```bash
  @tanstack/eslint-plugin-query
  ```
  - 해당 린트는, queryKey의 일부가 아닌 것을 queryFn 내에서 사용하려고 하면 오류 메시지와 해결책을 제시해줍니다
  ```
  ESLint: 다음 의존성이 queryKey에 누락되었습니다: sort(@tanstack/query/exhaustive-deps)
  ```

---

컴퓨터 과학 분야에서 유명한 말이 있다고 합니다.
**컴퓨터 과학에서 어려운 것은 딱 두가지 뿐이다. 캐시 무효화와 이름 짓기.**

여기서 핵심은 캐싱하는 것 자체가 어렵다기 보다는, 캐싱된 것을 무효화하는 것이 어렵다는 것입니다.
그렇다면 React Query는 어떻게 캐시 무효화를 처리하는지 알아봐야합니다.
즉, 캐시를 무효화 시켜서 서버 상태와 다시 동기화되는 처리 방법에 대해서 알아봐야합니다.

대부분의 캐시는 일정 시간 후에 캐시를 무효화하는 로직을 포함합니다.
예를들어 이전에 봤던 Github API를 사용할 때 응답의 cache-control 헤더를 보면 이를 확인할 수 있습니다.

```
cache-control: public, max-age=60
```

이 헤더는 브라우저에게 동일한 URL에 대해 다음 60초 동안 추가 요청을 하지 말고 브라우저 캐시에서 리소스를 제공하라고 지시합니다.

이렇게 헤더에 설정된 캐시옵션이 있지만, React Query의 경우 직접 네트워크 요청을 실행하지 않기 때문에 cache-control 헤더에 대해 알지 못합니다.

대신, React Query에는 staleTime이라는 비슷한 개념이 있습니다.
stale은 fresh의 반대입니다. (stale: 오래된)

즉, 설정한 statleTime이 지나기 전까지는 캐시가 fresh 하기 때문에 캐시를 반환합니다.
위와 같은 말을 다르게 표현하면 staleTime이 지나기 전까지는 queryFn을 실행시키지 않습니다

그렇다면 staleTime의 기본값은 몇 ms일까요?
![Inline-image-2024-07-10 07.15.49.247.png](/wikis/3421892279754781394/files/3844158044610177350)

0ms 입니다.
즉, 모든 쿼리가 즉시 stale로 간주된다는 것을 의미합니다.
이유는 위의 사진에도 나와있듯이 데이터가 오래된것보단, 약간의 낭비를 감수하더라도 항상 최신인 상태인 것이 더 합리적이기 때문입니다.

사실 리액트의 리렌더링과 같은 맥락입니다.
리렌더링을 최소화시켜 앱을 최적화해야하는 것은 맞지만, 리렌더링이 적어서 화면이 업데이트 되지 않는 것보단 약간 과한 리렌더링이 나은 것과 같습니다

React Query는 기본적으로 우리가 가져온 데이터를 즉시 stale로 간주함으로써 가능한 한 최신 상태로 유지하려고 합니다.
물론 당연히 개발자가 직접 staleTime을 설정할 수 있습니다

```jsx
useQuery({
  queryKey: ["repos", { sort }],
  queryFn: () => fetchRepos(sort),
  staleTime: 5 * 1000, // (5초)
});
```

그럼 쿼리가 stale이되면 React Query는 어떻게 동작할까요? ⇒ 특별한 동작을 하지는 않습니다
그저 특정 트리거가 걸리면 다시 데이터를 가져오고 동기화하고 캐시에 저장합니다

그렇다면 여기서 말하는 특정 트리거란 무엇일까요?

1. queryKey가 변경될 때
2. 새로운 옵저버가 마운트될 때
   - 즉 컴포넌트가 마운트될 때
3. React Query refetch 조건에 부합할때 (포커스, 온라인 재연결)

   ```jsx
   useQuery({
     queryKey: ["repos", { sort }],
     queryFn: () => fetchRepos(sort),
     refetchOnMount: false,
     refetchOnWindowFocus: false,
     refetchOnReconnect: false,
   });
   ```

만약 데이터가 절대 변경되지 않을 데이터라면, staleTime을 Infinity로 설정하여 캐시된 데이터를 영원히 fresh하게 만들 수도 있습니다

```jsx
useQuery({
  queryKey: ["repos", { sort }],
  queryFn: () => fetchRepos(sort),
  staleTime: Infinity,
});
```

## 05 - 데이터 패칭 상태 관리하기

앞선 예제들은 모두 컴포넌트가 마운트될 때 즉시 데이터를 가져오는 예제들이였습니다.

보통 이것이 일반적이지만, 어떠한 경우에는 특정 조건까지 데이터 가져오기를 멈추고 싶을 때도 존재합니다.

예를 들어, 사용자의 입력을 먼저 받아와야 하는 경우가 될 수 있습니다.

예제의 Github Issues 앱에 검색 창을 추가하는 것을 가정하고 아래처럼 수행할 수 있습니다.

```jsx
const [search, setSearch] = React.useState("");

if (search) {
  return useQuery({
    queryKey: ["issues", search],
    queryFn: () => fetchIssues(search),
  });
}
```

이 코드는 합리적으로 보입니다.

하지만, React는 훅 규칙을 위반하기 때문에 이를 허용하지 않습니다. (조건부로 훅을 호출할 수 없다는 규칙)

대신, React Query는 `enabled`라는 다른 설정 옵션을 제공합니다.

`enabled`는 `useQuery`에 boolean 값을 전달하여 쿼리 함수가 실행될지 여부를 결정합니다.

예제의 경우, `enabled`는 검색어가 있을 때만 useQuery가실행하도록 지시할 수 있습니다.

```jsx
function useIssues(search) {
  return useQuery({
    queryKey: ["issues", search],
    queryFn: () => fetchIssues(search),
    enabled: search !== "",
  });
}
```

하지만 이것도 완벽하지는 않습니다.
위처럼 코드를 작성하면 사용자가 입력하지 전부터 로딩창이 돌아갈 것입니다. (status===”pending” 일때 로딩 컴포넌트를 표시했을때를 가정)

이유는 아래와 같습니다.
쿼리는 항상 세 가지 상태 중 하나에 있을 수 있습니다 ⇒ `pending`, `success`, `error`

- `success`는 캐시에 데이터가 있다는 의미
- `error`는 데이터를 가져오는 데 실패했다는 의미
- `pending`은 그 외의 모든 경우를 의미

현재, 로딩 표시를 보여주기 위해 status가 `pending` 상태인지 확인하고 있습니다.

```jsx
if (status === "pending") {
  return <div>...</div>;
}
```

하지만, 앞서 언급했듯이 `pending` 상태는 캐시에 데이터가 없고 데이터를 가져오는 데 실패하지 않았음을 의미할 뿐입니다.

즉, 현재 쿼리가 실행 중인지 여부는 알려주지 않습니다.

쿼리 함수가 현재 실행 중인지 확인할 방법이 필요합니다.

이것이 로딩 표시를 보여줄지 여부를 결정하는 데 도움이 됩니다.

React Query는 쿼리 객체의 `fetchStatus` 속성을 통해 이를 노출합니다.

```jsx
const { data, status, fetchStatus } = useIssues(search);
```

`fetchStatus`가 `fetching`일 때 쿼리 함수가 실행 중입니다.
이를 사용하여 쿼리 상태와 함께 로딩 표시를 더 정확하게 유도할 수 있습니다.

```jsx
const { data, status, fetchStatus } = useIssues(search);

if (status === "pending") {
  if (fetchStatus === "fetching") {
    return <div>...</div>;
  }
}
```

`status`가 `pending`이면 캐시에 데이터가 없다는 의미이고, `fetchStatus`가 `fetching`이면 쿼리 함수가 실행 중이라는 의미입니다.

즉, 캐시에 데이터가 없고 쿼리 함수가 실행 중이라면 로딩을 표시하는 것이 합리적입니다.

이 패턴은 매우 흔하기 때문에 React Query는 이를 축약한 `isLoading` 값을 제공합니다.

```jsx
const { data, status, isLoading } = useIssues(search);

if (isLoading) {
  return <div>...</div>;
}
```

그렇다면 isLoading을 통해 데이터를 가져오는 동안 로딩 컴포넌트를 띄워주고, 해당 로딩이 완료되면 데이터 렌더링을 하는 것에 문제가 없을까요?

```
    Cannot read properties of undefined (reading 'items')
```

결과적으로 에러가 발생하고, 아직 데이터가 제대로 가지고 있다는 조건이 부족합니다.

이 문제가 발생하는 이유는 `isLoading`이 `false`이고 `status`가 `error`가 아니면 데이터를 가지고 있다고 가정하기 때문입니다.

`isLoading`은 상태가 `pending`이고 `fetchStatus`가 `fetching`인지 여부만 알려줍니다.

즉, `pending` 상태는 캐시에 데이터가 없다는 의미이고, `fetchStatus`가 `fetching`이면 쿼리 함수가 현재 실행 중이라는 의미입니다.

따라서 캐시에 데이터가 없고, 쿼리 함수가 실행 중이지 않은 경우에는 어떻게 될까요? 이 경우 `isLoading`은 `false`가 됩니다.

```jsx
const data = undefined; // 캐시에 데이터가 없습니다.
const status = "pending"; // 캐시에 데이터가 없습니다.
const fetchStatus = "idle"; // 쿼리 함수가 현재 실행 중이 아닙니다.
const isLoading = status === "pending" && fetchStatus === "fetching"; // false

if (isLoading) {
  return <div>...</div>;
}

if (status === "error") {
  return <div>There was an error fetching the issues</div>;
}

return (
  <p>
    <ul>
      {data.items.map((issue) => (
        <li key={issue.id}>{issue.title}</li>
      ))}
    </ul>
  </p>
);
```

예제로 자세히 살펴보면, 처리하지 않은 두 가지 시나리오가 있습니다.

1. 검색어가 없어서 쿼리 함수가 활성화되지 않은 경우
2. API 요청이 데이터를 반환하지 않은 경우입니다.

이 두 가지 문제를 해결하려면 쿼리 상태가 `success`인지 명시적으로 확인해야 합니다.

즉, `success` 상태는 캐시에 데이터가 있음을 의미합니다.

```jsx
function useIssues(search) {
  return useQuery({
    queryKey: ["issues", search],
    queryFn: () => fetchIssues(search),
    enabled: search !== "",
  });
}

function IssueList({ search }) {
  const { data, status, isLoading } = useIssues(search);

  if (isLoading) {
    // 캐시에 데이터가 없고
    // 쿼리 함수가 현재 실행 중입니다.
    return <div>...</div>;
  }

  if (status === "error") {
    // 데이터를 캐시에 넣는 도중 오류가 발생했습니다.
    return <div>There was an error fetching the issues</div>;
  }

  if (status === "success") {
    // 캐시에 데이터가 있습니다.
    return (
      <p>
        <ul>
          {data.items.map((issue) => (
            <li key={issue.id}>{issue.title}</li>
          ))}
        </ul>
      </p>
    );
  }

  // 그 외의 경우
  return <div>Please enter a search term</div>;
}
```

- Tip
  - react-query에는 `skipToken` 이라는 것이 존재합니다
    - `skipToken` 는 enabled: false를 내부적으로 설정해줍니다
  - 아래와 같이 사용할 수 있습니다

```jsx
import { skipToken, useQuery } from "@tanstack/react-query";

declare function fetchIssue(id: number): Promise;

function useIssue(id: number | undefined) {
  return useQuery({
    queryKey: ["issues", id],
    queryFn: id === undefined ? skipToken : () => fetchIssue(id),
  });
}
```
