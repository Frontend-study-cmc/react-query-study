## Managing Mutations

상태 관리자는 데이터를 제공하고 그 데이터를 업데이트할 수 있게 해준다.

React의 `useState`는 첫 번째 요소로 상태를, 두 번째 요소로 그 상태를 업데이트하는 함수를 반환하는 `tuple`을 제공한다.

```js
const [number, setNumber] = useState(0);
```

우린 이 상태의 소유자이기 때문에 `setNumber`를 언제든 안전하게 호출할 수 있고, 이 업데이트르 동기적인 것처럼 취급할 수 있다.

**비동기 상태 업데이트**의 경우는 어떨까?

이 상황에서 우리는 상태의 소유자가 아니므로, 직접 캐시에 기록하더라도 다음 `refetch`가 발생하면 덮어쓰여질 수 있다.

```js
// 사용자의 정보를 업데이트하는 함수이다.
// id와 newName을 받아서 해당 사용자의 이름을 업데이트하기 위해 PATCH 요청을 보낸다.
function updateUser({ id, newName }) {
  // fetch를 사용하여 서버에 PATCH 요청을 보낸다.
  return fetch(`/user/${id}`, {
    method: "PATCH", // PATCH 메서드를 사용하여 부분적으로 데이터를 업데이트한다.
    headers: {
      "content-type": "application/json", // JSON 형식으로 데이터를 보낸다는 것을 명시한다.
    },
    body: JSON.stringify({
      name: newName, // 새 이름을 요청 본문에 포함시킨다.
    }),
  }).then((res) => res.json()); // 요청이 성공하면 JSON 형식으로 응답을 받는다.
}
```

예를 들어 DB에 있는 사용자 엔티티를 업데이트하고 FE에서 `PATCH` 요청을 보내 이를 수행하고자 한다고 가정해보자.
이 작업은 즉시 일어나지 않는다. 만약 그랬다면 비동기 상태가 아닐 것이기 때문이다.

```js
// 서버에 업데이트 요청을 보내는 useQuery 훅이다.
// 주어진 사용자 id와 새로운 이름(newName)을 받아서 업데이트 작업을 진행한다.
function useUpdateUser(id, newName) {
  return useQuery({
    queryKey: ["user", id, newName], // 쿼리 키는 업데이트할 사용자 id와 새로운 이름으로 구성된다.
    queryFn: () => updateUser({ id, newName }), // updateUser 함수를 쿼리 함수로 호출하여 서버에 요청을 보낸다.
  });
}
```

우린 이런 업데이트를 수행할 때 쿼리와 유사한 라이프사이클을 거친다. `pending` -> `error` 또는 `success`
그렇다면 단순히 `useQuery`를 사용하여 서버에 업데이트 요청을 보내는 방법을 생각해 볼 수 있다.

이것은 흥미로운 아이디어이긴 하지만 여러 가지 이유로 작동하지 않을 것이다.

1. 쿼리는 컴포넌트가 마운트될 때 즉시 실행된다.
   우린 보통 이벤트 발생 후에 실행되길 원할 것이다. 이를 해결하기 위해 `enabled` 옵션을 사용할 수 있겠지만 더 큰 문제는 쿼리는 여러 번 그리고 종종 자동으로 실행된다는 점이다.

   이런 부작용은 자동으로 또는 여러 번 발생하는 것을 원하지 않는다. 대신 **특정 이벤트**가 발생할 때 명령적으로 일어나길 원한다.

이를 위해 React Query는 `useMutation`이라는 또 다른 훅을 제공한다.
`useMutation`은 직접 변이를 수행하는 것이 아니라 `useMutation`의 라이프사이클을 관리한다.

### 작동 방식

`useMutation`을 호출할 때 `mutationFn` 메서드가 있는 객체를 제공한다.
이 hook은 `mutate` 메서드를 가진 객체를 반환한다.

### `mutate`

`mutate`는 변이를 실행하는 역할을 한다. 우리가 정의한 `mutationFn`을 호출하고, 서버에 데이터를 보낼 때 사용된다.
`mutate`를 통해 서버에 변경 사항을 전달하고 **성공** 또는 **실패** 여부에 따라 후속 작업을 처리할 수 있다.

따라서 위 `useUpdateUser` 예제를 React Query에 맞춰 수정하면 다음과 같이 된다.
먼저 `useMutation`을 커스텀 훅으로 캡슐화하고 `updateUser`를 `mutationFn`으로 전달한다.

```js
// updateUser 함수를 mutationFn으로 사용하는 커스텀 훅
function useUpdateUser() {
  return useMutation({
    mutationFn: updateUser, // 업데이트 로직을 포함한 함수
  });
}
```

이렇게 하면 `useUpdateUser` 훅을 통해 `mutate` 메서드를 호출하여 원하는 시점에 데이터를 업데이트할 수 있다.
우리가 `mutate`에 전달하는 객체는 `mutationFn`에 인수로 전달된다.

```js
// ChangeName 컴포넌트는 폼 제출 시 사용자 이름을 업데이트
function ChangeName({ id }) {
  const { mutate } = useUpdateUser(); // useUpdateUser 훅에서 mutate 메서드를 가져온다.

  return (
    <form
      onSubmit={(event) => {
        event.preventDefault(); // 폼의 기본 동작을 방지합니다.
        const newName = new FormData(event.currentTarget).get("name"); // 폼에서 입력된 'name' 값을 가져온다.
        mutate({ id, newName }); // mutate를 호출하여 id와 newName을 전달하여 업데이트 요청을 보낸다.
      }}
    >
      <input name="name" /> {/* 사용자가 새로운 이름을 입력할 수 있는 필드 */}
      <button type="submit">Update</button> {/* 제출 버튼 */}
    </form>
  );
}
```

따라서 위 코드에서 `form`이 제출될 때 `mutate`가 호출되어 `id`와 `newName`이 `updateUser` 함수로 전달된다.
`useMutation` 훅은 변이 과정을 관리하고 React Query의 상태 관리 기능을 활용해 요청의 **라이프 사이클**을 처리한다.

꼭 기억해야 할 점은 `useMutation`의 핵심은 **변이의 라이프사이클을 관리**하는 것이다.
그 자체로 어떤 변이를 수행하거나 캐시를 변이시키는 것이 아니다.

아래 코드에서 `useMutation`을 호출하면 `mutate` 함수 외에도 현재 변이의 상태를 알려주는 `status` 속성도 함께 반환된다.
`status` 속성은 `pending`, `error`, `success`, `idle` 중 하나이다.

```js
// ChangeName 컴포넌트는 사용자가 이름을 변경할 수 있는 폼
function ChangeName({ id }) {
  const { mutate, status } = useUpdateUser(); // mutate 함수와 변이 상태(status)를 가져옴

  return (
    <form
      onSubmit={(event) => {
        event.preventDefault(); // 폼의 기본 제출 동작을 방지
        const newName = new FormData(event.currentTarget).get("name"); // 폼에서 입력된 'name' 값을 가져옴
        mutate({ id, newName }); // mutate 호출 시 id와 newName을 전달하여 업데이트 수행
      }}
    >
      <input name="name" /> {/* 사용자가 새로운 이름을 입력할 수 있는 필드 */}
      <button type="submit" disabled={status === "pending"}>
        {/* 변이가 진행 중이면 버튼을 비활성화 */}
        {status === "pending" ? "..." : "Update"} {/* 상태에 따라 버튼 텍스트 변경 */}
      </button>
    </form>
  );
}
```

또 상태를 관찰하는 것 이상으로 `mutation`의 라이프사이클에서 특정 순간에 반응할 수 있따.
`onSuccess`, `onError`, `onSettled` 콜백을 추가하여 변이의 결과에 따라 특정 동작을 실행할 수 있다.

이런 콜백들은 `mutate`의 두 번째 인수로 전달하거나 `useMutation`에 전달되는 객체에 속성으로 추가할 수 있다.

```js
// ChangeName 컴포넌트는 사용자가 이름을 변경할 수 있는 폼입니다.
function ChangeName({ id }) {
  const { mutate, status } = useUpdateUser(); // mutate 함수와 변이 상태를 가져옵니다.

  return (
    <form
      onSubmit={(event) => {
        event.preventDefault(); // 폼의 기본 제출 동작을 방지합니다.
        const newName = new FormData(event.currentTarget).get("name"); // 폼에서 입력된 'name' 값을 가져옵니다.
        mutate(
          { id, newName },
          {
            onSuccess: () => event.currentTarget.reset(), // 변이가 성공하면 폼을 리셋합니다.
          }
        );
      }}
    >
      <input name="name" /> {/* 사용자가 새로운 이름을 입력할 수 있는 필드 */}
      <button type="submit" disabled={status === "pending"}>
        {" "}
        {/* 변이가 진행 중이면 버튼을 비활성화 */}
        {status === "pending" ? "..." : "Update"}{" "}
        {/* 상태에 따라 버튼 텍스트 변경 */}
      </button>
    </form>
  );
}
```

예를 들어 변이가 성공했을 때 폼을 `reset`하고 싶다면 `mutate`의 두 번째 인수로 `onSuccess` 콜백을 전달하여 처리한다.

물론 `useUpdateUser` 안에도 `onSuccess` 콜백을 사용할 수 있다.

```js
// updateUser 함수를 mutationFn으로 사용하는 커스텀 훅
function useUpdateUser() {
  return useMutation({
    mutationFn: updateUser, // 업데이트 로직을 수행하는 함수
    onSuccess: () => {
      alert("Name updated successfully"); // 변이가 성공하면 경고 창을 표시합니다.
    },
  });
}
```

`useMutation` 내부에서 변이가 성공했을 때 경고(alert)를 표시한 예제이다.

`useMutation`는 쿼리를 함께 사용할 때 더 두드러진다.
새로운 사용자 정보를 캐시 업데이트에 활용하고 싶다면 어떻게 해야 할까?

- 가장 간단한 방법은 `onSuccess` 콜백에서 `queryClient.setQueryData`를 호출하여 캐시를 직접 업데이트하는 것이다.

`setQueryData`는 첫 번째 인수로 `queryKey`, 두 번째 인수로 새로운 데이터를 받는다.

```js
// updateUser 함수를 mutationFn으로 사용하는 커스텀 훅
function useUpdateUser() {
  const queryClient = useQueryClient(); // useQueryClient를 사용하여 queryClient에 접근

  return useMutation({
    mutationFn: updateUser, // 서버에 사용자 업데이트 요청을 수행하는 함수
    onSuccess: (newUser) => {
      // 변이가 성공하면 캐시를 업데이트합니다.
      queryClient.setQueryData(["user", newUser.id], newUser); // 사용자 정보를 캐시 업데이트
    },
  });
}
```

여기서 `onSuccess` 콜백이 실행될 때 `newUser` 데이터를 받아 `queryClient.setQueryData`를 호출하여 특정 쿼리키에 해당하는 캐시 데이터를 새로운 사용자 정보로 업데이트 한다.

### 근데 왜 캐시를 업데이트 할까?

이 방법은 새 사용자 정보를 서버에서 다시 받아오지 않고 이미 서버에서 받아온 `newUser` 정보를 캐시에 반영한다.
이를 통해 불필요한 `refetching`을 방지하고 UX를 향상시킬 수 있다.

중요한 점은 React Query는 데이터의 출처를 구분하지 않는다는 것이다. `refetch`, `prefetch`를 통해 캐시에 데이터가 들어는 방식 모두 동일하게 취급된다.
캐시가 업데이트된 후 해당 데이터는 새로운 데이터로 간주된다.

따라서 `staleTime`이 설정된 시간 동안 이 데이터는 신선한(fresh) 데이터로 간주되고, 그 동안에는 자동으로 다시 가져오는 작업이 발생하지 않는다.
이 말은 수동으로 캐시를 업데이트하더라도 React Query가 동일한 캐시 관리 적용한다는 뜻이다.

이 방식은 서버 요청을 줄이고 사용자에게 최신 데이터를 빠르게 제공할 수 있는 좋은 방법이다.

### `updateUser`가 업데이트된 사용자를 반환하지 않는 경우에도 캐시를 업데이트할 몇 가지 방법이 있다.

`onSuccess` 콜백의 첫 번째 인수는 `mutationFn`이 반환하는 값이지만 더 유용한 정보는 두 번째 인수로 전달되는 **변이를 호출할 때 전달한 객체**이다.

즉 우리가 `mutate`를 호출할 때 넘긴 `{id, newName}`가 해당된다.
따라서 `onSuccess` 콜백에서 두 번째 인수를 사용해 캐시를 업데이트 할 수 있다.

```js
// updateUser 함수를 mutationFn으로 사용하는 커스텀 훅
function useUpdateUser() {
  const queryClient = useQueryClient(); // queryClient에 접근

  return useMutation({
    mutationFn: updateUser, // 서버에 업데이트 요청을 보내는 함수
    onSuccess: (data, { id, newName }) => {
      // updateUser가 데이터를 반환하지 않더라도, mutate에 전달한 인수로 새 데이터를 추론할 수 있습니다.
      queryClient.setQueryData(["user", id], (oldData) => {
        // 캐시에 있는 기존 사용자 데이터를 업데이트
        return {
          ...oldData, // 기존 사용자 데이터
          name: newName, // 업데이트된 이름
        };
      });
    },
  });
}
```

`queryClient.setQueryData`에서 현재 캐시에 저장된 사용자 데이터를 `oldData`로 가져와, 새로운 `newName`을 덮어씌우는 방식으로 캐시를 업데이트한다.

정리하자면 `mutate` 함수에 전달된 데이터를 활용하여 필요한 정보를 추론하고 캐시를 정확하게 업데이트할 수 있도록 해준다.
따라서 서버가 반환하는 값이 없거나 필요한 정보를 제공하지 않더라도 캐시를 안전하게 업데이트할 수 있다.

또 `queryClient.setQueryData`에 두 번째 인수로 함수를 전달하면 그 함수는 캐시에 저장된 이전 데이터를 인수로 받을 수 있다. 이를 이용해 캐시에서 기존 사용자 데이터를 가져와 새 이름을 적용할 수 있다.

이렇게 하면 `previosUser` 데이터가 있을 때만 새로운 이름으로 업데이트하고 **없을 경우**에는 캐시 데이터를 그대로 유지하게 된다.

```js
// updateUser 함수를 mutationFn으로 사용하는 커스텀 훅
function useUpdateUser() {
  const queryClient = useQueryClient(); // queryClient에 접근

  return useMutation({
    mutationFn: updateUser, // 서버에 사용자 정보를 업데이트하는 함수
    onSuccess: (data, { id, newName }) => {
      // queryClient.setQueryData에 함수 전달, 이전 데이터를 기반으로 캐시 업데이트
      queryClient.setQueryData(
        ["user", id], // 쿼리 키
        (previousUser) =>
          previousUser
            ? { ...previousUser, name: newName } // 이전 데이터가 있을 경우 이름 업데이트
            : previousUser // 이전 데이터가 없을 경우 그대로 유지
      );
    },
  });
}
```

### 코드의 흐름

1. `queryClient.setQueryData`는 `['user', id]` 쿼리 키에 해당하는 데이터를 캐시에서 찾아 그것을 `previousUser`로 전달한다.

2. `previousUser`가 존재하는 경우 `newName`을 사용해 `previousUser` 객체를 업데이트하고 그 외의 데이터는 그대로 유지한다.

3. 만약 `previousUser`가 존재하지 않는다면 (캐시에 데이터가 없을 경우), 이전 상태를 그대로 유지한다.

### 불변성 (immutability)

React Query와 같은 상태관리자는 불변성을 유지해야한다고 한다.
이는 캐시를 업데이트할 때 항상 새로운 객체를 반환해야 한다는 것을 의미한다.

객체를 직접 수정하는 방식은 React Query의 상태 업데이트 방식을 손상시킬 수 있다.

#### 캐시 업데이트에서 `previousUser`를 직접 수정하는 방식

```js
queryClient.setQueryData(["user", id], (previousUser) => {
  if (previousUser) {
    previousUser.name = newName; // 객체를 직접 수정하는 방식 (이 방식은 권장되지 않음)
  }
  return previousUser; // 기존 객체를 반환
});
```

`previousUser`를 직접 수정하는 것은 객체의 불변성을 깨뜨리기 때문에 권장되지 않는다.

**불변성**을 깨뜨리면 어떻게 될까?

React와 React Query는 상태가 변경되었음을 감지하지 못할 수 있다.
React 상태나 props가 변경될 때 컴포넌트가 다시 렌더링하는데, 상태가 불변하지 않으면 변화가 발생해도 React가 이를 인식하지 못할 가능성이 크다.

#### 올바른 방법 : 새로운 객체 반환

```js
queryClient.setQueryData(["user", id], (previousUser) => {
  if (previousUser) {
    return { ...previousUser, name: newName }; // 이전 객체를 복사하고 새로운 값으로 덮어씌움
  }
  return previousUser; // 이전 데이터가 없으면 그대로 반환
});
```

React Query는 참조 기반으로 상태 변화를 감지하므로 항상 새로운 객체가 반환되어야한다.

지금까지 우리가 본 것은 변이가 성공한 후에 캐시를 명령형으로 업데이트하는 단순한 예제였다.
하지만 필터, 정렬 기능이 있는 리스트를 다룰 때는 동일한 데이터가 여러 캐시 항목에 저장되는 경우가 발생할 수 있다.
이럴 땐 변이가 발생하면 여러 개의 캐시 항목을 동시에 업데이트해야 하는 상황이 온다.

```js
import * as React from "react";
import { useQuery, useMutation, useQueryClient } from "@tanstack/react-query";
import { fetchTodos, addTodo } from "./api";

// 새로운 할 일을 추가하는 커스텀 훅
function useAddTodo() {
  return useMutation({
    mutationFn: addTodo, // 할 일 추가 함수
    onSuccess: (data) => {
      console.log(JSON.stringify(data)); // 성공 시 반환 데이터 출력
    },
  });
}

// 할 일 목록을 가져오는 커스텀 훅
function useTodos(sort) {
  const queryClient = useQueryClient();

  return useQuery({
    queryKey: ["todos", "list", { sort }], // 쿼리 키로 정렬 기준을 사용
    queryFn: () => fetchTodos(sort), // 할 일 목록을 가져오는 함수
    placeholderData: () =>
      queryClient.getQueryData(["todos", "list", { sort }]), // 캐시된 데이터를 먼저 사용
    staleTime: 10 * 1000, // 10초 동안 데이터 신선하게 유지
  });
}

// TodoList 컴포넌트
export default function TodoList() {
  const [sort, setSort] = React.useState("id"); // 정렬 기준 상태
  const { status, data, isPlaceholderData, refetch } = useTodos(sort); // 할 일 목록 가져오기
  const addTodo = useAddTodo(); // 새로운 할 일 추가 훅

  // 할 일 추가 이벤트 처리
  const handleAddTodo = (event) => {
    event.preventDefault();
    const title = new FormData(event.currentTarget).get("add"); // 폼 데이터에서 새로운 할 일 제목 가져오기
    addTodo.mutate(title, {
      onSuccess: () => event.target.reset(), // 성공 시 폼 리셋
    });
  };

  // 로딩 상태 표시
  if (status === "pending") {
    return <div>...</div>;
  }

  // 에러 발생 시 표시
  if (status === "error") {
    return <div>Error fetching todos</div>;
  }

  return (
    <div style={{ opacity: isPlaceholderData ? 0.8 : 1 }}>
      <label>
        Sort by:
        <select
          value={sort}
          onChange={(event) => {
            setSort(event.target.value); // 정렬 기준 변경
          }}
        >
          <option value="id">id</option>
          <option value="title">title</option>
          <option value="done">completed</option>
        </select>
      </label>
      <ul>
        {data.map((todo) => (
          <li key={todo.id}>
            {todo.done ? "✅ " : "🗒 "}
            {todo.title}
          </li>
        ))}
      </ul>
      <form
        onSubmit={handleAddTodo}
        style={{ opacity: addTodo.isPending ? 0.8 : 1 }} // 새로운 할 일 추가 중일 때 투명도 적용
      >
        <label>
          Add:
          <input type="text" name="add" placeholder="new todo" />
        </label>
        <button
          type="submit"
          disabled={addTodo.isPending} // 추가 중일 때 버튼 비활성화
        >
          Submit
        </button>
        <button
          type="button"
          onClick={refetch} // 강제로 할 일 목록 재요청
        >
          Refetch
        </button>
      </form>
    </div>
  );
}
```

정렬 기준에 따라 여러 개의 캐시 항목이 존재할 때 변이가 발생하면 모든 관련 캐시 항목을 업데이트해야 하는데 이 작업은 금방 복잡해질 수 있다고 한다.
특히 정렬 옵션이 많아질수록 매번 수동으로 캐시를 업데이트하는 것은 번거롭고 오류가 발생할 가능성이 높다.

### 문제 상황

정렬 옵션이 여러 개 있는 Todo 리스트에서 새로운 항목을 추가하거나 기존 항목을 수정할 때 모든 캐시 항목을 업데이트해야 한다.

1. `['todos', 'list', { sort: 'id' }]`
2. `['todos', 'list', { sort: 'title' }]`
3. `['todos', 'list', { sort: 'done' }]`

위 캐시 항목들을 모두 업데이트하게 되면 복잡성이 증가한다.

따라서 **동일한 로직**을 반복적으로 사용해 모든 캐시 항목을 업데이트하고 이를 함수화해 조금 더 깔끔하게 처리할 수 있다.

### 1. 캐시 업데이트 함수 작성 (GPT한테 더 간결한 코드를 요청)

```js
function updateAllTodoCaches(queryClient, updatedTodo) {
  const sortOptions = ["id", "title", "done"]; // 모든 정렬 기준

  sortOptions.forEach((sort) => {
    queryClient.setQueryData(["todos", "list", { sort }], (previousTodos) => {
      if (!previousTodos) return;

      return previousTodos.map((todo) =>
        todo.id === updatedTodo.id ? { ...todo, ...updatedTodo } : todo
      );
    });
  });
}
```

모든 정렬 옵션을 반복 처리할 수 있는 함수를 만들면 코드의 중복을 줄이고 관리하기 쉬워진다.
`updateAllTodoCaches`는 `queryClient`와 `updateTodo`를 받아서 모든 정렬 기준에 맞춰 캐시된 할 일 목록을 업데이트한다.

### 2. 변이 후 캐시 업데이트 로직

변이 성공 후, `updateAllTodoCaches` 함수를 호출해 정렬 기준에 관계없이 모든 캐시를 업데이트 할 수 있다.

```js
function useUpdateTodo() {
  const queryClient = useQueryClient(); // queryClient 인스턴스 가져오기

  return useMutation({
    mutationFn: updateTodo, // 서버에 할 일을 업데이트하는 함수
    onSuccess: (updatedTodo) => {
      updateAllTodoCaches(queryClient, updatedTodo); // 모든 캐시 업데이트
    },
  });
}
```

이렇게 하면 변이가 성공할 때마다 `updateAllTodoCaches` 함수를 호출해 정렬 기준에 관계없이 모든 캐시를 업데이트할 수 있다.

### 3. TodoList 컴포넌트

이제 `TodoList` 컴포넌트는 `useUpdateTodo`를 사용해 변이 후 캐시를 자동으로 업데이트한다.

```js
const updateTodo = useUpdateTodo(); // 할 일 업데이트 훅

// 할 일 업데이트 로직
const handleUpdateTodo = (todoId, newTitle) => {
  updateTodo.mutate({ id: todoId, title: newTitle });
};

{
  data.map((todo) => (
    <li key={todo.id}>
      {todo.done ? "✅" : "🗒"} {todo.title}
      <button onClick={() => handleUpdateTodo(todo.id, "Updated Title")}>
        Update
      </button>
    </li>
  ));
}
```

### 저자의 코드와 다른 점

1. 저자의 코드에서는 `queryClient.setQueryData`를 통해 정렬 기준별로 개별적으로 캐시를 업데이트 했다.
   하지만 새로운 코드에서는 중복 로직을 함수화하여 반복을 줄이고 코드의 중복을 최소화할 수 있다.

2. 저자의 코드에서는 각 정렬 기준에 맞게 개별적으로 정렬하는 로직을 직접 작성해 필요한 경우 각 캐시마다 다른 정렬 방식을 적용할 수 있다.

새로운 코드에서는 정렬 로직을 한 곳에서 정의하지 않고 각 캐시 항목을 업데이트 하는 방식만 통합했다.
아래는 정렬 기준이 추가된 코드이다.

```js
// 모든 정렬 기준에 따라 할 일 목록을 업데이트하는 함수
function updateAllTodoCaches(queryClient, newTodo) {
  // 정렬 기준 배열에 새로운 정렬 기준을 추가할 수 있습니다.
  const sortOptions = ["id", "title", "done", "priority"]; // 새로운 정렬 기준 'priority' 추가

  sortOptions.forEach((sort) => {
    queryClient.setQueryData(["todos", "list", { sort }], (previousTodos) => {
      if (!previousTodos) return;

      let updatedTodos = [...previousTodos, newTodo]; // 새 할 일을 추가

      // 각 정렬 기준에 따른 정렬 로직
      if (sort === "title") {
        // title을 기준으로 사전 순 정렬
        updatedTodos.sort((a, b) =>
          String(a.title)
            .toLowerCase()
            .localeCompare(String(b.title).toLowerCase())
        );
      } else if (sort === "done") {
        // 완료 여부(done)를 기준으로 정렬 (완료된 항목이 뒤로)
        updatedTodos.sort((a, b) => (a.done ? 1 : -1));
      } else if (sort === "priority") {
        // 새로운 정렬 기준 'priority'로 우선순위에 따라 정렬
        updatedTodos.sort((a, b) => a.priority - b.priority);
      }

      // 정렬된 목록을 반환하여 캐시를 업데이트
      return updatedTodos;
    });
  });
}

// 새로운 할 일을 추가하는 커스텀 훅
...
```

### 3. 중복 코드

저자의 코드에서는 `queryClient.setQueryData`를 세 번 호출해 각 정렬 기준에 맞게 별도의 로직을 작성하고 있다.
이로 인해 중복 코드가 발생하게 되고 나중에 정렬 기준이 추가되거나 수정될 때마다 이 코드를 추가로 수정해야 한다.

새로운 코드에서는 여러 정렬 기준을 반복 처리하여 중복된 코드 작성을 줄인다. 새로운 정렬 기준이 추가되더라도 동일한 함수 내에서 로직을 재사용 할 수 있다.

### 쿼리 무효화

업데이트해야 할 캐시 항목이 여러 개일 경우 모든 항목을 수동으로 업데이트하는 대신 **모든 항목을 무효화하는 것**이 더 나은 접근 방식인데 아래 2가지 이유 때문이다.

1. 모든 활성 쿼리가 다시 요청된다.
2. 나머지 쿼리는 stale(오래된) 상태로 표시된다.

쿼리를 **무효화**하면 해당 쿼리에 관찰자가 있는 경우 React Query는 즉시 다시 요청을 보내 캐시를 업데이트한다.
그렇지 않으면 해당 쿼리는 오래된 상태로 표시되며 React Query는 다음에 트리거가 발생할 때 이를 다시 요청하게 된다.

### 어떻게 쿼리를 무효화하는가?

매우 간단하다. `queryClient.invalidateQueries`를 호출하고 `queryKey`를 전달하기만 하면 된다.
예시를 통해 알아보자

```js
function useAddTodo() {
  const queryClient = useQueryClient();

  return useMutation({
    mutationFn: addTodo, // 할 일 추가 함수
    onSuccess: () => {
      // 모든 'todos' 관련 쿼리를 무효화하여 다시 요청함
      queryClient.invalidateQueries(["todos", "list"]);
    },
  });
}
```

변이가 성공한 후 모든 `todos` 관련 쿼리가 무효화되며 React Query가 자동으로 백엔드에서 새로운 데이터를 가져온다.
이렇게 하면 수동으로 캐시를 업데이트하지 않아도 항상 최신 데이터를 보장할 수 있다.

또, **무효화된 쿼리**는 다시 데이터를 가져오므로 데이터를 자동으로 최신 상태로 유지할 수 있다는 장점이 있다.

`onSuccess`에서 **Promise**를 반환하면 React Query는 Promise가 해결될 때까지 기다렸다가 변이가 완료된 것으로 처리한다. 이렇게 하면 UI가 깜박이는 현상을 피할 수 있다. 이 말은 즉 변이가 완료되기 전에 다시 `refetch`가 발생하는 것을 방지한다.

### 정리

`queryClient.invalidateQueries`는 Promise를 반환한다.
이 Promise는 해당 쿼리의 데이터가 다시 가져와질 떄 완료된다. `onSuccess`에서 이 Promise를 반환하면 React Query는 Promise가 완료될 때까지 변이가 완료되지 않은 것으로 처리하고 기다린다.

변이가 성공한 이후 서버로부터 새로 가져온 데이터를 사용한다.

이 접근 방식에는 몇 가지 장, 단점이 존재한다.

#### 장점

1. 서버 로직을 클라이언트에서 다시 구현할 필요가 없다.
   변이 후 서버 데이터를 다시 가져오는 방식으로 서버에서 최신 상태를 가져오기 때문에 데이터의 일관성이 보장된다.

2. 최신 데이터를 보장할 수 있다.
   역시 변이 후 서버로부터 데이터를 다시 가져오기 떄문에 항상 최신 상태임을 보장할 수 있다.

#### 단점

1. 서버로의 추가 요청
   변이 후 데이터를 가져오기 위해 서버로 추가 요청을 보내야 한다. 이는 네트워크 비용이 추가되며 성능 상 약간의 지연이 발생할 수 있다.

2. 비활성 쿼리는 즉시 다시 가져오지 않는다.
   무효화된 쿼리는 비활성 상태일 경우 즉시 다시 요청되지 않는다. 대신 stale(오래된) 상태로 표시되고 쿼리가 다시 활성화될 때 `refetching`된다.

### 모든 쿼리를 즉시 다시 가져오려면 어떻게 해야할까?

```js
function useAddTodo() {
  const queryClient = useQueryClient();

  return useMutation({
    mutationFn: addTodo,
    onSuccess: () => {
      // 모든 쿼리를 즉시 다시 요청하도록 설정
      return queryClient.invalidateQueries(["todos", "list"], {
        refetchType: "all",
      });
    },
  });
}
```

코드에서 살펴볼 수 있듯이 `refetchType: 'all'` 옵션을 사용하면 비활성 쿼리도 즉시 다시 가져올 수 있다.

### Fuzzy Query Key Matching (흐린 쿼리 키 매칭)

이 개념은 쿼리 키가 정확히 일치하지 않더라도 일부분만 일치하는 쿼리 키에 대해 모든 쿼리를 무효화할 수 있게 해준다.
예를 들어 우리가 `invalidateQueries`를 호출할 때, `['todos', 'list']`라는 쿼리 키를 전달했다고 가정하자.
그렇다면 이 키는 모든 `['todos', 'list']`로 시작하는 쿼리 키를 무효화한다.

이는 정확히 일치하지 않더라도 시작 부분이 동일한 쿼리들을 모두 무효화한다.

#### 그렇다면 어떻게 이 방식이 동작할까?

React Query는 쿼리 키가 배열 형태로 제공되기 때문에 배열의 앞부분이 일치하면 해당 쿼리 키로 시작하는 모든 쿼리를 찾을 수 있다. 또 우리가 `queryKey`를 계층적으로 구조화했기 때문이다. `queryKey`가 배열로 되어 있는 이유는 배열이 엄격한 계층 구조를 내포하고 있기 때문이다.

`queryKey`는 일반적인 것에서 구체적인 것으로 순서대로 정렬해야 한다는 것이다.

```js
["todos", "list", { sort: "id" }][("todos", "list", { sort: "title" })][
  ("todos", "detail", "1")
][("todos", "detail", "2")];
```

위 예시에서 보면 `todos`는 가장 일반적인 것으로 우리 엔티티를 가리킨다.
그 다음엔 문자열 list가 있는데 이건 다른 종류의 `todo` 캐시를 구분하기 위해 추가된 것이다.
마지막으로 가장 구체적인 `sort`를 볼 수 있다.

### 무효화하는 과정

```js
queryClient.invalidateQueries({
  queryKey: ["todos", "list"],
});
```

쿼리 키 `["todos", "list"]`와 일치한 항목들을 필터링 할 수 있다.

```js
["todos", "list", { sort: "id" }][("todos", "list", { sort: "title" })][
  ("todos", "detail", "1")
][("todos", "detail", "2")][("posts", "list", { sort: "date" })][
  ("posts", "detail", "23")
];
```

위에서 1, 2번째 항목만 남는다.

### stale, 활성 상태인 쿼리만 무효화 가능

```js
// stale 쿼리만 무효화
queryClient.invalidateQueries({
  queryKey: ["todos", "list"],
  stale: true,
});

// 활성 상태(관찰자가 있는 쿼리)인 쿼리만 무효화
queryClient.invalidateQueries({
  queryKey: ["todos", "list"],
  tyoe: "active",
});
```

### `invalidateQueries`에 조건(predicate) 함수를 전달할 수도 있다.

조건 함수는 전체 쿼리를 전달받아 필터링에 사용할 수 있고 함수가 true를 반환하면 해당 쿼리가 일치하고 무효화된다.
`false`를 반환하면 제외된다.

이 기능은 매우 강력한데 `queryKey` 구조가 모든 쿼리를 한 번에 타겟팅할 수 없을 때 유용하다.
예를 들어 어떤 엔티티이든 상관없이 모든 상세(detail) 쿼리를 타겟팅 할 수 있다.

```js
queryClient.invalidateQueries({
  predicate: (query) => query.queryKey[1] === "detail",
});
```

결론적으로 적절하게 `queryKey`를 구조화하고 Fuzzy Matching에 의존하면 `invalidateQueries`를 한 번 호출해 쿼리의 하위 집합 전체를 무효화할 수 있다.

## 전체 요약

1. `useMutation`은 서버 데이터 업데이트 시 라이프사이클을 관리해준다.
2. 캐시를 수동으로 업데이트 할 수 있지만 `queryClient.invalidateQueries`로 간단히 무효화하고 최신 데이터를 가져올 수 있다.
3. **Fuzzy Query Key Matching**을 사용하면 쿼리 키 일부만 일치해도 관련된 모든 쿼리를 무효화할 수 있다.
4. 쿼리 상태에 따라 활성/비활성, stale 쿼리만 무효화할 수 있다.
