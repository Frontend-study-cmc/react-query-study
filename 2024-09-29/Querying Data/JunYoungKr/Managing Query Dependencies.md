## 쿼리 의존성 관리 (Managing Query Dependencies)

지금까지 우리는 고정된 endpoint에서 비동기 상태를 관리하기 위해 `useQuery`를 사용하는 데 익숙해졌을 것이다.
하지만 사실 고정되어있지 않다. 사실, 고정된 endpoint는 예외일 때가 많다.

예시 코드를 통해 알아보자.
만약 `useRepos` 훅을 업데이트하여 `sort` 매개변수를 받아 URL에 전달하려면 어떻게 해야 할까?

```js
fetch("https://api.github.com/orgs/TanStack/repos?sort=created");

function useRepos(sort) {
  return useQuery({
    queryKey: ["repos"],
    queryFn: async () => {
      const response = await fetch(
        `https://api.github.com/orgs/TanStack/repos?sort=${sort}`
      );
      if (!response.ok) {
        throw new Error(`Request failed with status: ${response.status}`);
      }
      return response.json();
    },
  });
}
```

위 코드는 작동하지 않는다.
우리는 무의식적으로 React Query가 컴포넌트가 다시 렌더링될 때마다 `queryFn`을 다시 실행할 것이라고 가정할 것이지만
사실 React Query는 그렇게 작동하지 않는다.

컴포넌트가 다시 렌더링되는 이유는 여러 가지가 있지만, 그때마다 데이터를 다시 가져오는 건 우리가 원하는 동작이 아니다.
이 문제를 해결하기 위해서, 정렬 매개변수가 변경될 때마다 React Query에게 `refetch()`함수를 통해 데이터를 다시 가져오라고 명령할 수 있다.
아래가 **명령적 해결 방법**이다.

```js
const { data, status, refetch } = useRepos(selection);

onChange={(event) => {
  const sort = event.target.value;
  setSelection(sort);
  refetch();
}};
```

하지만 React의 세계에서는 **선언적 해결 방법**이 더 우위에 있다.
해결 방법은 간단하다. `QueryKey`를 사용하는 것이다.

`queryKey` 배열의 값이 변경될 때마다 React Query는 queryFn을 다시 실행한다.
즉, queryFn에서 사용되는 모든 값은 `queryKey` 배열에 포함되어야 한다는 뜻이다.

```js
function useRepos(sort) {
  return useQuery({
    queryKey: ["repos", sort],
    queryFn: async () => {
      const response = await fetch(
        `https://api.github.com/orgs/TanStack/repos?sort=${sort}`
      );
      if (!response.ok) {
        throw new Error(`Request failed with status: ${response.status}`);
      }
      return response.json();
    },
  });
}
```

`queryKey`는 캐시의 항목과 직접적으로 연결된다.

`queryKey` 배열의 값이 변경되면, 새로운 관찰 대상이 설정된다. 이는 새로운 키를 구독하게 되는 것이며, 해당 키에 대한 데이터를 캐시에서 읽어온다.

해당 키에 대해 **처음** 접근하는 것이라면, 캐시에 해당 데이터가 없을 확률이 높다.
이 경우, 새로운 캐시 항목이 생성되고, 그 상태는 `pending`이 된다. 이후 `queryFn`이 호출되어 데이터를 가져온다.
이를 통해 **자동으로 캐싱된 데이터를 빠르게 제공**하면서도 자동으로 갱신(fetching)할 수 있다.

### 캐시에 있는 `queryKey`로 다시 변경하는 무슨 일이 일어날까?

```js
-["repos", { sort: "updated" }] + ["repos", { sort: "created" }];
```

만약 sort를 `updated`에서 `created`로 변경하여 정렬을 다시 선택하는 경우를 가정하자.
이번에는 캐시가 이미 데이터로 채워져 있기 때문에, React Query는 즉시 해당 데이터를 제공할 수 있으며, 쿼리의 상태는 **성공**으로 바로 전환된다.

이것이 `queryKey`의 강력한 기능이다.

`React Query`는 데이터를 **의존성에 따라 저장**하여 매개변수가 다른 요청이 서로 덮어쓰는 일이 없도록 한다.
대신, 각 매개변수에 맞는 데이터가 독립적으로 캐시에 저장되며, 이를 통해 매번 데이터를 다시 가져올 필요 없이 빠르게 조회할 수 있다.

또한, `React Query`는 경합 조건(race condition)을 방지한다.

### race condition이란?

여러 비동기 작업이 동시에 실행되면서, 그 작업들이 서로의 결과에 영향을 미치거나, 예상치 못한 순서로 완료되어 잘못된 동작이 발생하는 문제를 의미한다고 한다.

예시를 통해 알아보자. A, B 두 개의 네트워크 요청이 거의 동시에 시작되었지만, A 요청이 B 요청보다 먼저 완료될 것이라고 가정하자. 실제로는 B 요청이 먼저 완료되는 상황이 발생할 수 있는데, 이렇게 되면 의도와는 다르게 다른 데이터가 화면에 표시될 수 있다.

하지만 React Query는 `queryKey`를 사용하여 이런 경합 조건을 방지한다.
`queryKey`가 변경될 때마다 해당 요청에 대한 구독을 업데이트하고, 새로운 데이터가 들어오면 적절한 쿼리 키에 해당하는 데이터만을 반영한다.

1. 사용자가 A와 B 데이터를 각각 요청했다고 할 때, 쿼리 키를 사용하여 A와 B 데이터를 각각 고유한 키로 관리한다.
2. A 데이터를 먼저 요청했지만, B 데이터가 먼저 도착하더라도 React Query는 쿼리 키를 기반으로 올바른 데이터만을 반영하게 되어 경합 조건을 방지한다.

따라서 우리는 `queryKey` 배열에 `queryFn` 내부에서 사용하는 모든 값을 포함시키기만 하면 된다.
