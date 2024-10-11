## Pagination (페이지 네이션)

API들은 다양한 형식과 크기의 데이터를 반환할 수 있다.
소규모의 단일 객체에서부터 수천 개의 레코드를 포함한다.

예를 들어 facebook의 경우 10,000개 이상의 이슈가 있을 수 있고, Github API의 단일 이슈 레코드는 3KB이다.
이를 한 번에 요청하는 것은 매우 많은 데이터를 반환한다.

이처럼 대규모 데이터셋을 효율적으로 처리하기 위해 API는 종종 `pagination` 기법을 사용한다.
이는 데이터를 여러 페이지로 나눠 반환하는 방식이다.

### 페이지네이션을 사용하는 API 작업 방법

API는 요청 매개변수에 페이지 번호와 페이지 당 레코드 수를 지정할 수 있는 방법을 제공한다.

API는 요청을 처리하고, 지정된 페이지 번호와 레코드 수에 따라 **전체 데이터의 하위 집합을 반환**한다.

예를 들면, 페이지네이션된 데이터와 함께 **response**에는 총 레코드 수, 현재 페이지 번호, 전체 페이지 수와 같은 추가 메타 데이터가 포함된다.

다음 페이지의 결과를 가져오려면 페이지 번호(page number)를 증가시켜 동일한 페이지 당 레코드 수로 요청을 계속한다.

### 페이지네이션의 장점

페이지네이션된 API는 대규모 데이터를 작은 단위로 나누어 전송하는 데이터를 줄이고 성능을 개선하며 애플리케이션에서 데이터를 효율적으로 검색하고 표시할 수 있게 해준다.

### React-Query의 Pagination 예시

```js
export async function fetchRepos(sort, page) {
  const response = await fetch(
    `https://api.github.com/orgs/TanStack/repos
      ?sort=${sort}
      &per_page=4
      &page=${page}`
  );

  if (!response.ok) {
    throw new Error(`Request failed with status: ${response.status}`);
  }

  return response.json();
}
```

위 코드에서는 매개변수로 `page`와 `per_page`를 보내준다.

`page`는 어떤 페이지를 가져와야 하는지 정의하고, `per_page`는 각 페이지에 몇 개의 항목이 있어야 하는지 정의한다.

여기서는 동적일 필요가 없으므로 `per_page`를 4로 하드코딩 해줬다.

### `Repos` 컴포넌트

```js
export default function Repos() {
  const [selection, setSelection] = React.useState("created");
  const [page, setPage] = React.useState(1);

  const handleSort = (sort) => {
    setSelection(sort);
    setPage(1);
  };

  return (
    <div>
      <Sort value={selection} onSort={handleSort} />
      <RepoList sort={selection} page={page} setPage={setPage} />
    </div>
  );
}
```

`Repos` 컴포넌트에서는 페이지 번호를 `Repos` 컴포넌트 내부에서 상태를 관리하여, `Sort`와 `RepoList` 자식 컴포넌트에서 해당 상태를 props를 통해 접근 및 수정할 수 있게 한다.

### `Sort` 컴포넌트

```js
export default function Sort({ value, onSort }) {
  return (
    <label>
      Sort by:
      <select value={value} onChange={(event) => onSort(event.target.value)}>
        <option value="created">Created</option>
        <option value="updated">Updated</option>
        <option value="pushed">Pushed</option>
        <option value="full_name">Name</option>
      </select>
    </label>
  );
}
```

### `useRepos` 컴포넌트

```js
import * as React from "react";
import { useQuery } from "@tanstack/react-query";
import { fetchRepos } from "./api";
import Sort from "./Sort";

function useRepos(sort, page) {
  return useQuery({
    queryKey: ["repos", { sort, page }],
    queryFn: () => fetchRepos(sort, page),
    staleTime: 10 * 1000,
  });
}

function RepoList({ sort, page, setPage }) {
  const { data, status } = useRepos(sort, page);

  if (status === "pending") {
    return <div>...</div>;
  }

  if (status === "error") {
    return <div>레포지토리를 불러오는 중 오류가 발생했습니다.</div>;
  }

  return (
    <div>
      <ul>
        {data.map((repo) => (
          <li key={repo.id}>{repo.full_name}</li>
        ))}
      </ul>
      <div>
        <button onClick={() => setPage((p) => p - 1)} disabled={page === 1}>
          이전
        </button>
        <span>페이지 {page}</span>
        <button onClick={() => setPage((p) => p + 1)}>다음</button>
      </div>
    </div>
  );
}
```

`RepoList` 내부에서는 버튼을 추가하고 `useRepos`에서 페이지 prop을 인자로 전달하여,
이를 `queryKey`에 추가하고 `fetchRepos` 함수에 넘겨줄 수 있다.

### Loading Indicator 최소화하기

페이징을 할 때 기존의 목록을 유지한 채 새로운 목록이 준비될 때까지 보여주면 레이아웃 변경을 최소화할 수 있다.

이를 위해 `placeholderData`를 사용해, 이전 데이터를 새로운 데이터가 준비될 때까지 표시할 수 있다.

```js
function useRepos(sort, page) {
  return useQuery({
    queryKey: ["repos", { sort, page }],
    queryFn: () => fetchRepos(sort, page),
    staleTime: 10 * 1000,
    placeholderData: (previousData) => previousData,
  });
}
```

`useQuery`의 `isPlaceHolderData` 속성을 이용해, 이전 데이터를 보여줄 때 리스트의 불투명도(opacity)를 조절해 사용자에게 피드백을 줄 수 있다.

```js
<ul style={{ opacity: isPlaceholderData ? 0.5 : 1 }}>
  {data.map((repo) => (
    <li key={repo.id}>{repo.full_name}</li>
  ))}
</ul>
```

`opacity` 속성을 이용함으로써 페이지 간의 탐색이 꽤 부드럽게 느껴진다.

흥미로운 것은 단순히 페이징에만 해당되는 것이 아닌 정렬 기준을 변경할 때도 동일한 동작을 볼 수 있다는 것이다.

왜냐하면 React Query 입장에서는 `queryKey`가 변경되기만 하면 상관없기 때문이다.
페이지 변경이든, 정렬 변경이든 관계없이 `queryKey`만 바뀌면 된다.

배포하기 전에 처리해야 할 마지막 문제가 하나 더 있다.
버튼을 적절할 때 비활성화하는 작업이 잘 처리되지 않고 있다.

구체적으로 말하자면, 애플리케이션이 새로운 데이터를 가져오는 동안과 목록의 끝에 도달했을 때 버튼을 비활성화하고 싶다.

데이터를 가져오는 중일 때 비활성화하기 위해서는 `isPlaceholderData`에 이미 접근할 수 있으며, 이것이 우리가 필요한 그 기능이다.

```js
function RepoList({ sort, page, setPage }) {
  // useRepos 훅을 통해 API 호출을 하고, 반환값으로 데이터, 상태, 그리고 placeholder 상태를 받아옴
  const { data, status, isPlaceholderData } = useRepos(sort, page);

  return (
    <div>
      {/* 데이터가 로드 중일 경우 리스트의 불투명도를 낮춰 사용자에게 시각적 피드백 제공 */}
      <ul style={{ opacity: isPlaceholderData ? 0.5 : 1 }}>
        {data.map((repo) => ({}))}
      </ul>
      <div>
        {/* 페이지 감소 버튼: 첫 페이지에서는 비활성화하고, 데이터가 로드 중일 때도 비활성화 */}
        <button
          onClick={() => setPage((p) => p - 1)}
          disabled={isPlaceholderData || page === 1}
        >
          이전
        </button>
        {/* 페이지 증가 버튼: 데이터가 로드 중일 때는 비활성화 */}
        <button
          disabled={isPlaceholderData || data?.length < PAGE_SIZE}
          onClick={() => setPage((p) => p + 1)}
        >
          다음
        </button>
      </div>
    </div>
  );
}
```

위 코드에서 볼 수 있듯이 `button`의 속성에 `disabled`를 봐보자.

`disabled = {isPlaceholderData}`과 각각에 조건을 달아준다.
페이지 감소 버튼의 경우 첫 페이지에서 비활성화되고,
페이지 증가 버튼의 경우 우리가 받아온 데이터의 길이가 `per_page`보다 작으면 마지막 페이지로 간주할 수 있다.

이제 우린 완전한 페이지네이션을 구현했다.
여기에 추가로 `prefetching`을 추가하면 어떨까?

```js
function getReposQueryOptions(sort, page) {
  return {
    queryKey: ["repos", { sort, page }],
    queryFn: () => fetchRepos(sort, page),
    staleTime: 10 * 1000,
  };
}

function useRepos(sort, page) {
  const queryClient = useQueryClient();

  React.useEffect(() => {
    queryClient.prefetchQuery(getReposQueryOptions(sort, page + 1));
  }, [sort, page, queryClient]);

  return useQuery({
    ...getReposQueryOptions(sort, page),
    placeholderData: (previousData) => previousData,
  });
}
```

`onMouseEnter` 이벤트를 기다리는 대신, 항상 다음 페이지를 백그라운드에서 미리 불러오는 것이다.
이렇게 하면 사용자가 **다음** 버튼을 클릭할 때 이미 캐시에 데이터가 저장되어 있어 UI를 즉시 보여줄 수 있다.

### 요약

- React Query를 사용해 `queryKey`가 변경될 때 데이터를 캐싱한다.
- 페이지 전환 시 기존 데이터를 유지해 UI 변화 없이 부드럽게 처리할 수 있다.
- `isPlaceholderData`를 사용해 로딩 상태를 처리하고, 데이터를 백그라운드에서 미리 불러오는 `prefetching` 기능을 추가해 성능을 향상시킬 수 있다.
