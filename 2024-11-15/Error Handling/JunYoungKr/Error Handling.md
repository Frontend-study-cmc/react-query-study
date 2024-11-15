# Error Handling

지금까지 우린 모든 것이 잘 작동하는 예시를 다뤘다.
하지만 항상 잘 작동하지는 않는다. 따라서 이때 발생하는 오류를 적절하게 처리하는 것이 좋다.

### 첫 번째 방법 : `queryFn`에서 에러 던지기

```js
function useRepos() {
  return useQuery({
    queryKey: ["repos"],
    queryFn: async () => {
      const response = await fetch(
        "https://api.github.com/orgs/TanStack/repos"
      );

      // 요청이 실패했을 경우 에러를 던집니다.
      if (!response.ok) {
        throw new Error(`Request failed with status: ${response.status}`);
      }

      // 요청이 성공하면 JSON 데이터를 반환합니다.
      return response.json();
    },
  });
}
```

위 코드에서 볼 수 있듯이 `queryFn`에서 요청 실패 시 **throw new Error()**를 통해 에러를 던질 수 있다.
때로는 fetch 요청의 응답을 디버깅하거나, `queryFn` 내부에서 응답을 처리해야 할 때가 있다.
이 경우 직접 에러를 catch하고 싶을 수 있다.

```js
function useRepos() {
  return useQuery({
    queryKey: ["repos"],
    queryFn: async () => {
      try {
        const response = await fetch(
          "https://api.github.com/orgs/TanStack/repos"
        );

        if (!response.ok) {
          throw new Error(`Request failed with status: ${response.status}`);
        }

        return response.json();
      } catch (e) {
        console.log("Error: ", e);
      }
    },
  });
}
```

위 코드는 정상적으로 작동할 것처럼 보이지만 아주 큰 문제가 있다.
에러를 직접 catch하는 경우, catch 블록 내부에서 다시 에러를 던지지 않는 한, 에러를 삼키게된다.
이는 에러가 React Query까지 전파되지 않도록 막는다.

#### 문제점

1. 쿼리 상태 업데이트 문제

- 에러를 직접 catch하면, React Query는 에러가 발생했는지 인식하지 못한다.
- 따라서 쿼리의 상태를 `error`로 업데이트하지 않는다.

2. 자동 재시도(retry) 문제

- React Query는 기본적으로 요청이 실패할 경우 자동으로 3번 재시도한다.
- 이때 \*\*지수 백오프(exponential backoff) 알고리즘을 사용하여 재시도 간의 대기 시간을 점차 길게 만든다.
- 기본적으로 첫 번째 재시도는 1초후, 최대 대기 시간은 30초라고 한다.

```js
useQuery({
  queryKey: ["repos"],
  queryFn: fetchRepos,
  retry: 5, // 요청이 실패할 경우 5번 재시도
  retryDelay: 5000, // 각 재시도 사이에 5000밀리초 (5초) 대기
});

catch (e) {
        console.log("Error: ", e);
        throw e; // 에러를 다시 던져 React Query가 감지할 수 있도록 함
      }
```

### 더 세밀한 제어 필요 시 `retry`와 `retryDelay` 옵션 사용

더 세밀한 제어가 필요하다면, retry와 retryDelay 옵션에 함수를 전달할 수 있다.
이 함수들은 `failuerCount`(실패 횟수)와 `error`(에러 객체)를 인자로 받으며, 이를 통해 값을 결정할 수 있다.

```js
useQuery({
  queryKey: ["repos"],
  queryFn: fetchRepos,
  retry: (failureCount, error) => {
    // HTTPError 인스턴스이며, 상태 코드가 500 이상인 경우
    if (error instanceof HTTPError && error.status >= 500) {
      // 실패 횟수가 3회 미만일 때만 재시도
      return failureCount < 3;
    }

    // 그렇지 않으면 재시도하지 않음
    return false;
  },
  // 재시도 간의 지연 시간: 실패 횟수에 1000밀리초(1초)를 곱함
  retryDelay: (failureCount) => failureCount * 1000,
});
```

### `retry` 옵션

- `retry`는 요청이 실패할 때마다 호출되며, 재시도할지 여부를 결정한다.
- 예시에서는 error가 500 이상(서버 에러)이면서 3회 미만으로 실패했을 때만 재시도하도록 설정한다.
- 이 경우, 클라이언트 에러(4xx)나 실패 횟수가 3회를 넘으면 재시도하지 않는다.

### `retryDelay` 옵션

- `retryDelay`는 각 재시도 사이의 대기 시간을 결정한다.
- 예시에서는 **실패 횟수(failureCount) × 1000밀리초(1초)**의 대기 시간을 설정했다.
- 예를 들어, 첫 번째 실패 시 1초, 두 번째 실패 시 2초, 세 번째 실패 시 3초의 지연 시간이 적용된다.

### 이점

- `retry` 함수에서는 에러의 상태 코드나 타입을 기반으로 재시도할지 말지를 결정할 수 있다.
- `retryDelay` 함수에서는 기본 지수 백오프 대신, 실패 횟수를 고려한 커스텀 지연 시간을 설정할 수 있다.

```js
import * as React from "react";
import { useQuery } from "@tanstack/react-query";
import { fetchTodos } from "./api";

function useTodos() {
  return useQuery({
    queryKey: ["todos", "list"],
    queryFn: fetchTodos,
    retryDelay: 1000, // 각 재시도 간 대기 시간: 1초
  });
}

export default function TodoList() {
  const { status, data, failureCount } = useTodos();

  // 대기(pending) 상태일 때
  if (status === "pending") {
    return (
      <div>
        {/* 요청이 오래 걸리고 있음을 사용자에게 알림 */}
        {failureCount > 1 ? (
          <span>(This is taking longer than expected. Hang tight.)</span>
        ) : null}
        <i>retry attempts: {failureCount}</i> {/* 재시도 횟수 표시 */}
      </div>
    );
  }

  // 에러 상태일 때
  if (status === "error") {
    return <div>There was an error fetching the todos</div>;
  }

  // 데이터가 성공적으로 로드되었을 때
  return (
    <div>
      <ul>
        {data.map((todo) => (
          <li key={todo.id}>{todo.title}</li>
        ))}
      </ul>
    </div>
  );
}
```

재시도가 진행되는 동안, 쿼리는 `pending` 상태로 유지된다.

사용자에게는 queryFn이 데이터를 가져오기 위해 여러 번 실행되는 것은 크게 문제가 되지 않지만, 필요하다면 React Query는 `useQuery`의 반환 객체에 `failureCount`(실패 횟수)와 `failureReason`(실패 이유) 속성을 포함시킨다.

또 쿼리가 `success`(성공) 상태로 전환되면 이 값들은 초기화된다.

이 값들을 사용하면 데이터 요청이 실패한 경우 UI를 업데이트할 수 있는 유연성을 제공한다.
예를 들어, 요청이 예상보다 오래 걸리고 있음을 사용자에게 알리거나, 시도한 요청 횟수를 표시할 수도 있다.

#### failureCount

- 요청이 실패할 때마다 실패 횟수가 증가한다.
- 이 값을 사용해 UI에서 재시도 횟수를 표시하거나, 요청이 오래 걸리고 있음을 사용자에게 알릴 수 있다.

#### `status` 속성

- pending : 요청이 진행 중일 때
- error : 요청이 실패했을 때
- success : 요청이 성공했을 때

### 글로벌 설정 (전역 설정)

`retry`나 `retryDelay` 설정을 직접 구성하고 싶다면 전역 수준에서 설정하는 것이 좋다.
이렇게 하면 애플리케이션 전반에서 일관성을 유지할 수 있다.

```js
const queryClient = new QueryClient({
  defaultOptions: {
    queries: {
      retry: 5, // 재시도 횟수: 5회
      retryDelay: 5000, // 각 재시도 간 대기 시간: 5초
    },
  },
});
```

**재시도**가 요청의 성공을 보장하지는 않는다.
다만 성공할 수 있는 몇 가지 기회를 더 제공할 뿐이다.
만약 요청이 결국 실패하여 쿼리가 `error` 상태로 전환되면 우아하게(?) 처리해야 한다.

1. 에러 상태 확인 및 처리

가장 간단한 방법은 쿼리의 `status`를 확인해 에러 UI를 표시하는 것이다.

```js
if (status === "error") {
  return <em>Error: {error.message}</em>;
}
```

2. 에러 바운더리 (error boundary) 사용

컴포넌트에서 직접 에러를 처리하는 대신, `Error Boundary`를 사용하면 애플리케이션 전반에 걸쳐 더 높은 수준의 에러 처리를 구현할 수 있다.

```js
<ErrorBoundary fallback={<Error />}>
  <App />
</ErrorBoundary>
```

3. React Query와 Error Boundary 연동

React Query는 데이터 페칭 중 발생한 에러를 렌더링 흐름 외부에서 처리하기 때문에, 기본적으로 Error Boundary에서 잡히지 않는다.
이를 해결하기 위해 React Query는 `throwOnError` 옵션을 제공한다.

```js
useQuery({
  queryKey: ["todos", "list"],
  queryFn: fetchTodos,
  retryDelay: 1000,
  throwOnError: true, // 에러를 다시 던져 Error Boundary에서 잡히도록 설정
});
```

이제 React Query에서 에러가 발생하면, Error Boundary가 이를 감지하고 대체 UI를 표시할 수 있다.

4. Error Boundary 리셋 및 재요청

```js
function Fallback({ error, resetErrorBoundary }) {
  return (
    <>
      <p>Error: {error.message}</p>
      <button onClick={resetErrorBoundary}>Try again</button>
    </>
  );
}

export default function App() {
  return (
    <QueryErrorResetBoundary>
      {({ reset }) => (
        <ErrorBoundary onReset={reset} FallbackComponent={Fallback}>
          <TodoList />
        </ErrorBoundary>
      )}
    </QueryErrorResetBoundary>
  );
}
```

사용자가 다시 시도할 수 있도록, Error Boundary를 리셋하고 React Query의 데이터를 재요청해야 할 수 있다.

5. Toast 알림 사용하기

```js
const queryClient = new QueryClient({
  queryCache: new QueryCache({
    onError: (error) => {
      toast.error(error.message);
    },
  }),
});
```

### 요약

- queryFn에서 에러를 던지고, 재시도(retry, retryDelay) 옵션으로 요청 실패 시 자동 재시도 가능.
- `throwOnError`: true 설정 시, Error Boundary에서 에러를 잡아 대체 UI 표시 가능.
- `QueryErrorResetBoundary`와 `resetErrorBoundary`로 Error Boundary 리셋 및 재요청 지원.
- `QueryCache`의 `onError` 콜백으로 전역 에러 핸들링 가능 (Toast 알림 등).
- `전역 설정(defaultOptions)`을 통해 일관된 에러 처리 및 재시도 정책 구현 가능.
