### 1. useQuery와 캐시 사용

React Query에서 `useQuery`는 데이터를 가져오기 위해 `queryFn`을 실행하고 그 결과를 `queryKey`를 기준으로 캐시에 저장한다.

이후 동일한 `queryKey`를 가진 또 다른 `useQuery` 호출이 있을 경우, 캐시에 저장된 데이터를 바로 반환하여 중복된 API 호출을 방지한다.

이를 통해 성능을 최적화하고 불필요한 네트워크 요청을 줄일 수 있다.

### 2. 중복 제거 예시

```js
import {
  QueryClient,
  QueryClientProvider,
  useQuery,
} from "@tanstack/react-query";

const queryClient = new QueryClient();

function LuckyNumber() {
  const { data } = useQuery({
    queryKey: ["luckyNumber"],
    queryFn: () => Promise.resolve(Math.random()),
  });

  return <div>Lucky Number is: {data}</div>;
}

export default function App() {
  return (
    <QueryClientProvider client={queryClient}>
      <LuckyNumber />
      <LuckyNumber />
    </QueryClientProvider>
  );
}
```

위 코드에서 두 개의 `LuckyNumber` 컴포넌트가 존재하지만, 동일한 `queryKey`를 가지고 있기 때문에 첫 번째 `useQuery` 호출에서 반환된 결과가 캐시에 저장된다. 그 후 두 번째 `LuckyNumber` 호출을 `queryFn`을 다시 실행하지 않고, 캐시에 저장된 값을 바로 반환한다.

잠깐 `Promise.resolve(value)`에 대해 알아보고 가자.

위 메소드는 JS에서 Promise 객체를 생성하고 즉시 해결(fulfilled)된 상태로 반환하는 메소드이다.
이 메소드를 사용하면 이미 성공한 값을 가진 Promise를 만들 수 있다.

### 3. 글로벌 캐시

React Query의 캐시는 `Global`하게 작동하므로 동일한 `QueryClientProvider` 아래에 있는 컴포넌트들은 모두 같은 캐시를 공유한다.

즉, 컴포넌트 트리 어디서든 동일한 `queryKey`를 사용하면 캐시된 데이터를 재사용할 수 있다.

### 4. 옵저버 패턴과 성능

React Query는 `옵저버 패턴(Observer Pattern)`을 사용하여 캐시-컴포넌트 간의 데이터를 동기화한다.
각 `useQuery` 호출마다 옵저버가 생성되어 특정 `useQuery`의 데이터를 감시하고, 캐시가 업데이트르되면 해당 데이터를 사용하는 컴포넌트가 다시 렌더링된다.

이를 통해 UI가 항상 최신 상태를 반영하며, 성능도 최적화된다.

옵저버 패턴은 객체 간 `일대다 관계`를 정의하여 한 객체의 상태 변화가 있을 때 관련된 다른 객체들이 자동으로 알림받고 업데이트될 수 있도록 하는 디자인 패턴이다.

이 패턴은 주로 **이벤트 기반 시스템이나 데이터 변경 감지를 구현**할 때 사용된다.
