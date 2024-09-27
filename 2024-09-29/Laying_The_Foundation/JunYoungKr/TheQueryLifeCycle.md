### 1. 비동기 로직을 동기적으로 처리

React Query는 비동기 로직을 마치 동기적인 것처럼 느끼게 만들어준다.
예를 들어 브라우저의 `MediaDevices API`를 사용해 사용자의 입력 및 출력 장치를 나열하려면 `useQuery`를 사용할 수 있다.

```js
function MediaDevices() {
  const { data } = useQuery({
    queryKey: ["mediaDevices"],
    queryFn: () => {
      return navigator.mediaDevices.enumerateDevices();
    },
  });

  return (
    <ul>
      {data.map((device) => (
        <li key={device.deviceId}>{device.label}</li>
      ))}
    </ul>
  );
}
```

위 코드는 `MediaDevices API`에서 반환된 장치 목록을 `useQuery`를 통해 가져와 UI에 표시하는 방식이다.
그러나 비동기 작업이 진행 중일 때 `data`가 아직 존재하지 않기 때문에, 그 데이터를 바로 사용할 수 없다는 문제가 발생할 수 있다.

### 2. 비동기 상태 관리

비동기 요청이 진행 중일 때 데이터를 사용할 수 없는 경우, React Query는 쿼리의 상태를 나타내는 다양한 `Query States`를 제공한다.

이 상태는 `Promise`의 상태 (대기 중, 성공, 실패)와 유사하다.

- pending: 쿼리가 아직 완료되지 않았고 데이터가 존재하지 않는 상태
- success: 쿼리가 성공적으로 완료되어 데이터가 존재하는 상태
- error: 쿼리가 실패하여 에러가 발생한 상태

위 상태들을 통해 데이터를 안전하게 처리할 수 있다.

#### 상태처리 예시 코드

```js
function MediaDevices() {
  const { data, status } = useQuery({
    queryKey: ["mediaDevices"],
    queryFn: () => {
      return navigator.mediaDevices.enumerateDevices();
    },
  });

  if (status === "pending") {
    return <div>Loading...</div>;
  }

  if (status === "error") {
    return <div>We were unable to access your media devices.</div>;
  }

  return (
    <ul>
      {data.map((device) => (
        <li key={device.deviceId}>{device.label}</li>
      ))}
    </ul>
  );
}
```

### 3. 상태에 따른 boolean 값

```js
function MediaDevices() {
  const { data, isPending, isError } = useQuery({
    queryKey: ["mediaDevices"],
    queryFn: () => {
      return navigator.mediaDevices.enumerateDevices();
    },
  });

  if (isPending) {
    return <div>Loading...</div>;
  }

  if (isError) {
    return <div>We were unable to access your media devices.</div>;
  }

  return (
    <ul>
      {data.map((device) => (
        <li key={device.deviceId}>{device.label}</li>
      ))}
    </ul>
  );
}
```

React Query는 쿼리의 상태를 좀 더 직관적으로 다룰 수 있도록 `isPending`, `isSuccess`, `isError`와 같은 `boolean` 값도 제공한다.

### 4. Query LifeCycle의 단계

쿼리는 주로 다음 세 가지 상태 중에 하나 있다.

1. Pending : 요청이 진행 중일 때
2. Success : 요청이 성공적으로 완료되었을 때
3. Error : 요청이 실패했을 때

React Query는 이 라이프사이클을 기반으로 상태 관리를 쉽게 할 수 있도록 도와준다.
