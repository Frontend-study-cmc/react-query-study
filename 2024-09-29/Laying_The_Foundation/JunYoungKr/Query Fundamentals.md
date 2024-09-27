### 1. React Query 세팅

```js
import { QueryClient } from "@tanstack/react-query";

const queryClient = new QueryClient();

export default function App() {
  return (
    <QueryClientProvider client={queryClient}>
      <App />
    </QueryClientProvider>
  );
}
```

### 2. QueryClient와 QueryCache

React Query의 핵심은 `QueryClient`이다.

`QueryCache`는 모든 데이터를 저장하는 위치이며, 이 캐시 덕분에 React Query의 주요 기능들이 동작한다.
`QueryClient`는 위 코드에서 볼 수 있듯이 React 컴포넌트 외부에서 생성되어야 하며, 이는 애플리케이션이 다시 렌더링될 때도 캐시가 안정적으로 유지되도록 하기 위함이다.

### 3. QueryClientProvider

`QueryClient`를 애플리케이션 전체에서 사용할 수 있게 하려면 `QueryClientProvider`로 애플리케이션의 최상위 컴포넌트, 즉 `App.tsx`를 감싸야한다.

이 과정은 React Context를 사용하여 캐시를 전체 컴포넌트 트리에서 사용할 수 있게 한다.

### 4. useQuery hook

React Query의 주요 기능은 `useQuery` 훅을 통해 이뤄진다.
이 훅은 `QueryCache`에 구독하고 있으며, 캐시에 있는 데이터가 변경될 때마다 다시 렌더링을 트리거한다.

#### useQuery의 주요 요소

- queryKey : 데이터를 캐시에 저장할 때 사용할 고유 키
- quertFn : 데이터를 가져오는 **비동기 함수**로, 이 함수는 데이터를 가져와 캐시에 저장한다.

```js
const { data } = useQuery({
  queryKey: ["pokemon", id],
  queryFn: () =>
    fetch(`https://pokeapi.co/api/v2/pokemon/${id}`).then((res) => res.json()),
});
```

위 코드는 `useQuery` 훅을 사용하여 데이터를 가져오는 코드이다.

#### 설명

1. `useQuery` 훅을 통해 서버에서 데이터를 가져오고, 데이터를 캐시하여 여러 컴포넌트에서 재사용할 수 있게 해준다. 이 훅은 비동기 서버 상태를 관리하는 데 사용된다.

2. `queryKey` 안의 ['pokemon', id]라는 배열이 키로 사용되며, `id`는 포켓몬의 고유한 ID 값이다.
   이 키는 캐시에 데이터의 유무를 확인 후, 없으면 새로 데이터를 가져오는 기준으로 사용된다.

3. `queryFn`는 실제로 데이터를 가져오는 함수이다.
   `fetch` API를 사용하여 `PokeAPI`에서 포켓몬 데이터를 요청한다. API 요청이 완료된 후 응답을 JSON 형식으로 변환하여 반환한다.

4. data는 캐시되어 동인한 `queryKey`를 가진 다른 컴포넌트에서도 재사용될 수 있다.

정리하자면 `React Query`는 데이터가 캐시에 없으면 `queryFn`에서 정의한 비동기 함수를 실행해 데이터를 가져오는데,
만약 데이터가 **캐시에 있다면** API 호출 없이 즉시 캐시된 데이터를 반환,
데이터가 **캐시에 없다면** API 호출을 통해 서버에서 데이터를 가져온다.
