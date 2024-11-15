# Managing Query Keys

React Query를 사용하는 애플리케이션이 커질수록 **query Key** 관리의 복잡성도 함께 증가한다.
특히 쿼리 키를 정의한 위치와 이를 무효환(invalidate)하거나 변경(mutate)해야 할 위치가 다를 경우 문제가 발생할 수 있다.

```js
// 쿼리에서 queryKey 사용
export default function useTodos(sort) {
  return useQuery({
    queryKey: ["todos", "list", { sort }],
    queryFn: () => fetchTodos(sort),
  });
}

// 뮤테이션에서 queryKey 사용
useMutation({
  mutationFn,
  onSuccess: () => {
    queryClient.invalidateQueries({
      queryKey: ["todos", "list"],
    });
  },
});
```

위 코드에서 볼 수 있듯이 `queryKey: ['todos', 'list']`를 반복해서 작성해야 한다.
이때 오타가 발생하면 큰 문제가 될 수 있다.

### Query Key Factory 패턴

이 문제를 해결하기 위해 Query Key Factory 패턴을 사용한다. 쿼리 키를 중앙에서 관리하는 방법이다.

```js
// keys.js
export const todoKeys = {
  allLists: () => ["todos", "list"],
  list: (sort) => ["todos", "list", { sort }],
};
```

`todoKeys`를 설정 후 쿼리 키가 필요할 때마다 `todoKeys` 객체에서 가져다 사용한다.

```js
// 쿼리에서 사용
import { todoKeys } from "./keys";

export default function useTodos(sort) {
  return useQuery({
    queryKey: todoKeys.list(sort),
    queryFn: () => fetchTodos(sort),
  });
}
```

#### 장점

1. 오타 방지 : 쿼리 키를 재작성할 필요가 없으므로 오타 문제를 줄일 수 있다.
2. 코드 일관성 : 쿼리 키의 일관성을 유지할 수 있다.

#### 단점

1. 분리 문제 : `queryKey`와 `queryFn`이 분리되어 있어, 쿼리 키가 필요한 곳에서 `queryFn`을 참조하기 어렵다

### Query Factory 패턴

`Query Key Factory` 패턴의 문제를 해결하기 위해 Query Factory 패턴을 도입할 수 있다.
여기서는 쿼리 키와 쿼리 옵션을 한 곳에 정의한다.

```js
// todoQueries 객체: "todos" 기능과 관련된 모든 queryKey와 queryFn을 관리하는 Query Factory입니다.
const todoQueries = {
  // 1. "all" 메서드:
  // 가장 기본적인 queryKey입니다. 'todos'라는 단일 배열을 반환합니다.
  all: () => ["todos"],

  // 2. "allLists" 메서드:
  // "all" 메서드의 queryKey를 기반으로 'list'를 추가하여 새로운 queryKey를 생성합니다.
  // 이 메서드는 모든 목록 데이터를 가져올 때 사용됩니다.
  allLists: () => [...todoQueries.all(), "list"],

  // 3. "list" 메서드:
  // 특정 정렬 기준(sort)을 사용해 쿼리 객체를 반환합니다.
  // - queryKey: 'todos', 'list', 그리고 정렬 기준(sort)이 포함된 배열입니다.
  // - queryFn: fetchTodos(sort) 함수로 데이터를 가져옵니다.
  // - staleTime: 데이터가 최신으로 간주되는 시간(5초)입니다.
  list: (sort) => ({
    queryKey: [...todoQueries.allLists(), sort],
    queryFn: () => fetchTodos(sort),
    staleTime: 5 * 1000,
  }),

  // 4. "detail" 메서드:
  // 특정 ID의 todo 상세 데이터를 가져올 때 사용되는 쿼리 객체를 반환합니다.
  // - queryKey: 'todos', 'detail', 그리고 ID가 포함된 배열입니다.
  // - queryFn: fetchTodo(id) 함수로 데이터를 가져옵니다.
  // - staleTime: 데이터가 최신으로 간주되는 시간(5초)입니다.
  detail: (id) => ({
    queryKey: [...todoQueries.all(), "detail", id],
    queryFn: () => fetchTodo(id),
    staleTime: 5 * 1000,
  }),
};

const { data } = useQuery(todoQueries.list(sort));
```

### TypeScript를 사용

TypeScript를 사용하면 이런 문제들을 줄일 수 있다.
TypeScript는 반환 타입을 검사해 주기 때문에 잘못된 반환 값을 쉽게 잡아낼 수 있다.

```js
queryClient.invalidateQueries(todoQueries.allLists()); // 오류 발생 가능
queryClient.invalidateQueries({ queryKey: todoQueries.allLists() }); // 올바른 사용
```

### invalidateQueries란 무엇인지 다시 알고 가자

`invalidateQueries`는 React Query에서 데이터를 무효화하는 메서드이다.
이 메서드는 특정 쿼리의 데이터를 오래된 것으로 표시하고 다음 번에 사용될 때 데이터를 다시 가져오도록 트리거한다.

**주로 언제 사용할까?**

- 데이터 변경 이후 : 예를 들어, 데이터를 수정/삭제한 후 화면에 보여지는 목록이 업데이트되어야 할 때 사용한다.
- 데이터 동기화 : 백엔드에서 데이터가 변경되었을 가능성이 있는 경우, 데이터를 다시 가져와 최신 상태로 유지한다.

## 요약

- Query Key Factory 패턴은 쿼리 키 관리의 복잡성을 줄이고 코드의 일관성을 높인다.
- Query Factory 패턴은 쿼리 키와 `queryFn`을 결합해 코드 가독성을 높이며, 옵션 재사용성을 제공한다.
- TypeScript를 사용하면 반환 값의 일관성 문제를 더 쉽게 관리할 수 있다.
