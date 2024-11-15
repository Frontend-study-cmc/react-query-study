# Performance Optimizations (성능 최적화)

애플리케이션을 실행할 떄 한 컴포넌트가 생각보다 훨씬 자주 렌더링 되고 있을 수 있다.
이럴땐 보통 컴포넌트의 렌더링을 줄이기보다 **더 빠르게 렌더링되로록** 만드는 것이 좋다.

```js
import * as React from "react";

function Wave() {
  console.count("Rendering Wave");
  return (
    <span role="img" aria-label="hand waving">
      👋
    </span>
  );
}

export default Wave;
```

위 코드는 버튼 클릭 시 `Rendering Wave`라는 콘솔 로그가 계속 출력된다.
이는 `Wave` 컴포넌트가 props에 의존하지 않음에도 불구하고, 부모 컴포넌트의 상태 변화로 인해 다시 렌더링되고 있다는 것을 의미한다.

### React.memo 사용하기

props가 실제로 변경될 때만 다시 렌더링되도록 하기 위해서, React의 `React.memo` 고차 컴포넌트를 사용할 수 있다.

```js
import * as React from "react";

function Wave() {
  console.count("Rendering Wave");
  return (
    <span role="img" aria-label="hand waving">
      👋
    </span>
  );
}

export default React.memo(Wave);
```

이제 `React.memo`로 Wave 컴포넌트를 래핑하면, 부모 컴포넌트의 상태가 변경되더라도 Wave 컴포넌트는 props가 변경되지 않는 한 다시 렌더링되지 않습니다.

`React.memo`는 고차 컴포넌트로 컴포넌트를 **메모이제이션**한다.
메모이제이션은 동일한 입력값이 주어졌을 때, 이전의 결과를 재사용하는 최적화 기법이다.

그렇다면 언제 `React.memo`를 사용해야 할까?

- 비용이 많이 드는 컴포넌트가 있는 경우
- 자식 컴포넌트가 props 변화에 의존하지 않는 경우

이제 버튼을 몇 번 클릭하든 상관없이, Wave 컴포넌트는 초기 렌더링 때만 한 번 렌더링된다.

```js
import * as React from "react";

function Wave({ onClick, options }) {
  console.count("Rendering Wave");
  const { animate, tone } = options;
  return (
    <span
      role="img"
      aria-label="hand waving"
      onClick={onClick}
      className={animate ? "animate-wave" : ""}
    >
      👋{tone}
    </span>
  );
}

export default React.memo(Wave);
```

props를 받지 않는 대신, 이제 options prop을 전달하여 이모지를 설정할 수 있도록 해보자.
구체적으로 Wave 컴포넌트의 소비자가 이모지의 피부 색상과 애니메이션 여부를 설정할 수 있도록 할 것이다.

`React.memo`는 props가 변경될 때만 컴포넌트를 다시 렌더링한다.
근데 React는 props가 변경되었는지 어떻게 판단할까? 간단히 말해 **==(동등 연산자)**를 사용한다.

```js
<Wave options={{ animate: true, tone: 4 }} onClick={handleWaveClick} />
```

우리가 작성한 `Wave` 컴포넌트는 두 개의 props를 받는다. 이들은 모두 **참조 값**이다.
참조 값은 메모리 위치로 비교되기 때문에, 함수는 동일해 보이고 객체의 속성들도 동일하게 유지되더라도, 우리는 매번 새로운 객체와 함수를 생성하고 전달하고 있다.
이렇게 되면 React.memo의 최적화가 무효화된다.

### 그럼 어떻게 해결할까?

참조가 일관되도록 props 값을 유지할 방법이 필요하다.
다행히도 React는 이를 위한 `useMemo`와 `useCallback` 훅을 제공한다.

- useMemo : 계산 결과를 렌더링 사이에 캐싱한다.
- useCallback: 함수 자체를 캐싱하여 참조의 안정성을 유지한다.

```js
// options 객체 메모이제이션
const options = React.useMemo(() => {
  return {
    animate: true,
    tone: waveIndex,
  };
}, [waveIndex]);

// handleWaveClick 함수 메모이제이션
const handleWaveClick = React.useCallback(() => {
  setWaveIndex((i) => (i === 5 ? 0 : i + 1));
}, []);
```

`handleWaveClick` 함수가 React.useCallback으로 메모이제이션되어 있기 때문에 이 함수는 컴포넌트가 처음 렌더링될 때 한 번만 생성되고, 이후의 렌더링에서는 동일한 참조를 유지한다.

#### React.memo와의 연동

만약 Wave 컴포넌트가 `export default React.memo(Wave)`처럼 감싸져 있다면, `handleWaveClick` 함수가 매번 동일한 참조를 유지하기 때문에, `Wave` 컴포넌트는 `handleWaveClick`이 변경되지 않는 한 다시 렌더링 되지 않는다.

따라서 `waveIndex` 상태가 변경되거나, `options` 객체의 참조가 변경될 때 렌더링이 발생한다.

### React Query와의 관련성

`useQuery`를 호출할 때마다 새로운 객체(참조 값)를 반환한다.
만약 React Query가 값을 메모이제이션하지 않고, 컴포넌트를 React.memo로 감싸지 않았다면 쿼리가 실행될 때마다 컴포넌트 트리가 다시 렌더링될 것이다.

이를 해결하기 위해 React Query는 **구조적 공유**와 **Observer** 두 가지 최적화를 사용한다.

### 구조적 공유 (Structural Sharing)

쿼리가 실행되고 `queryFn`이 호출될 때, React Query는 일반적으로 새로운 객체를 반환한다. (예: `res.json()`)
React Query는 이 객체를 캐시에 넣기 전에, 객체의 속성과 값이 실제로 변경되었는지 먼저 확인한다.
**변경이 없는 경우**, React Query는 새로운 객체를 생성하거나 캐시를 재사용하지 않고 기존 객체를 그대로 사용한다.
이렇게 하면 **참조가 동일하게 유지**된다.

### Observer

Observer는 쿼리 캐시와 React 컴포넌트 간의 연결 고리이다. React 컴포넌트 트리 외부에 존재하며, 쿼리가 업데이트될 떄 컴포넌트에 변경 사항을 알릴지 결정한다.

```js
import { useQuery } from "@tanstack/react-query";

export default function App() {
  const { data, refetch } = useQuery({
    queryKey: ["user"],
    queryFn: () => {
      console.log("queryFn runs");
      return Promise.resolve({ name: "Dominik" });
    },
  });

  console.log("render");

  return (
    <div>
      <button onClick={() => refetch()}>refresh</button>
      <p>{data?.name}</p>
    </div>
  );
}
```

`refetch`로 인해 `queryFn`이 실행되더라도, 데이터가 변경되지 않으면 Observer는 컴포넌트를 다시 렌더링하지 않는다.

### `select` 옵션을 통한 최적화

만약 `queryFn`이 반환하는 객체에 매번 변경되는 속성이 포함되어 있지만, 이 속성을 컴포넌트에서 사용하지 않는다면 어떻게 될까?

React Query는 컴포넌트에서 실제로 어떤 데이터를 사용하는지 알지 못하기 때문불필요한 리렌더링이 발생할 수 있다.

```js
useQuery({
  queryKey: ["user"],
  queryFn: () => {
    console.log("queryFn runs");
    return Promise.resolve({ name: "Dominik", updatedAt: Date.now() });
  },
  select: (data) => ({ name: data.name }),
});
```

이제 `updatedAt` 속성이 변경되더라도, `select`를 사용해 필요한 데이터만 필터링했기 때문에 컴포넌트는 **다시 렌더링되지 않습니다.**

### Tracked Properties

React Query는 useQuery에서 반환하는 결과 객체를 만들 때 **커스텀 게터(getter)**를 사용한다.

이게 중요한 이유는, Observer가 어떤 필드가 렌더 함수에서 사용되었는지 추적할 수 있기 때문이라고 한다.
이를 통해 해당 필드가 변경된 경우에만 컴포넌트를 다시 렌더링한다.

```js
const { data, status, error, fetchStatus } = useQuery(["todos"], fetchTodos);
```

React Query는 `useQuery`를 호출할 때, 쿼리 데이터와 함께 여러 가지 메타 정보를 반환한다.
하지만 컴포넌트에서 **`data`**만 사용하고 **`fetchStatus`**는 사용하지 않는다면 어떨까?

```js
function TodoList() {
  const { data, fetchStatus } = useQuery(["todos"], fetchTodos);

  return (
    <ul>
      {data.map((todo) => (
        <li key={todo.id}>{todo.title}</li>
      ))}
    </ul>
  );
}
```

위의 코드에서는 `fetchStatus`가 컴포넌트에서 사용되지 않음에도 불구하고, `fetchStatus` 값이 변경되면 React는 기본적으로 컴포넌트를 다시 렌더링한다. 이렇게 되면 불필요한 리렌더링이 발생할 수 있다.

이때, React Query는 `Tracked Properties`라는 최적화 기법을 사용하여 이 문제를 해결한다.

#### Tracked Properties란?

- 커스텀 getter를 사용하여 컴포넌트가 실제로 접근한 속성만 추적한다.
- React Query는 `useQuery`의 반환 객체를 생성할 때, Proxy 객체나 커스텀 게터를 통해 어떤 속성이 컴포넌트에서 사용되었는지를 기록한다.
- 컴포넌트가 렌더링될 때 접근된 속성만 추적하므로, 접근되지 않은 속성이 변경되더라고 컴포넌트를 다시 렌더링하지 않는다고 한다.

위 코드에서는 `data`만 사용하고 있지만 React Query는 기본적으로 모든 값이 변경될 때 컴포넌트를 다시 렌더링한다고 했다.
하지만 `Tracked Properties`를 적용 후, Proxy 객체나 커스텀 게터를 통해 컴포넌트가 실제로 **data**에만 접근했음을 기록한다.
따라서 fetchStatus가 변경되더라도, React Query는 컴포넌트가 이 값을 사용하지 않는다고 판단하고 리렌더링을 생략한다.

### 커스텀 Getter란?

객체의 속성에 접근할 때, 특정한 로직을 실행할 수 있도록 하는 메서드이다.
JS에서는 객체의 `get` 메서드를 사용해 커스텀 Getter를 정의할 수 있다.

이 메서드는 객체의 속성에 접근할 때 자동으로 호출되며, 기본적인 속성 접근과는 달리, 속성의 값을 가져오는 동시에 추가적인 로직을 수행할 수 있다고 한다.

```js
function createTrackedResult(data) {
  return {
    get data() {
      console.log("data 속성에 접근함");
      return data;
    },
    get status() {
      console.log("status 속성에 접근함");
      return "success";
    },
  };
}

// 사용 예시
const queryResult = createTrackedResult({ todos: [] });

console.log(queryResult.data); // "data 속성에 접근함" 출력 후, { todos: [] } 반환
console.log(queryResult.status); // "status 속성에 접근함" 출력 후, "success" 반환
```

### 요약

1. **useMemo**와 **useCallback**은 참조 값을 안정적으로 유지해 불필요한 리렌더링을 방지한다.
2. 구조적 공유는 동일한 데이터를 반환할 때 캐시 객체를 재사용하여 참조가 변경되지 않도록 한다.
3. Observer는 쿼리 캐시와 컴포넌트 간의 연결 고리로, 데이터가 변경된 경우에만 컴포넌트를 업데이트한다.
4. Tracked Properties는 커스텀 Getter를 사용해 컴포넌트에서 접근된 속성만 추적하여, 실제로 사용된 속성 변경 시에만 렌더링이 발생하게 한다.
