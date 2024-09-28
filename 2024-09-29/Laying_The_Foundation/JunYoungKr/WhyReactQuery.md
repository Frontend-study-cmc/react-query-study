### 왜 React-Query인가?

React 훅이 출시되면서 새로운 시대가 열렸다.
우리는 이것을 **데이터를 어떻게 `fetching` 할 것인가**의 시대라고 부를 수 있다.

React의 모든 내장 훅 중에 실무에서 `data fetching`에 대해 전용으로 제공되는 훅은 하나도 없다.
가장 가까운 것은 `useEffect` 안에서 데이터를 `fetching` 후, 그 응답을 `useState`로 유지하는 것이다.

예시를 통해 알아보자

```js
import * as React from "react";
import PokemonCard from "./PokemonCard";
import ButtonGroup from "./ButtonGroup";

export default function App() {
  const [id, setId] = React.useState(1);
  const [pokemon, setPokemon] = React.useState(null);

  React.useEffect(() => {
    const handleFetchPokemon = async () => {
      setPokemon(null);

      const res = await fetch(`https://pokeapi.co/api/v2/pokemon/${id}`);
      const json = await res.json();
      setPokemon(json);
    };

    handleFetchPokemon();
  }, [id]);

  return (
    <>
      <PokemonCard data={pokemon} />
      <ButtonGroup handleSetId={setId} />
    </>
  );
}
```

위 코드에서는 **두 가지 문제**가 발생한다.

1. 로딩 상태를 처리하지 않는다는 것

- 이는 사용자 경험에서 가장 큰 두 가지 UX 문제 중 하나이 **누적 레이아웃 이동(Cumulative Layout Shift)**를 초래한다.
- 해결하는 방법 중 하나는 요청이 진행 중일 때 빈 카드를 표시하는 것이다.
- 하지만 그러기 위해서는 더 많은 상태를 추가해야한다.
- 기본값을 true로 설정한 다음 요청이 완료되면 false로 설정해야한다.

2. 무한 로딩 화면이 발생할 수 있다.

- PokeAPI 요청이 실패했을 떄의 경우를 처리하지 않고 있다.
- 이를 해결하기 위해 에러 상태를 추가하면 된다.

#### 아래 코드는 에러 상태를 추가한 코드이다.

```js
export default function App () {
  const [id, setId] = React.useState(1)
  const [pokemon, setPokemon] = React.useState(null)
  const [isLoading, setIsLoading] = React.useState(true)
  const [error, setError] = React.useState(null)

  React.useEffect(() => {
    const handleFetchPokemon = async () => {
      setPokemon(null)
      setIsLoading(true)
      setError(null)

      try {
        const res = await fetch(`https://pokeapi.co/api/v2/pokemon/${id}`)

        if (res.ok === false) {
          throw new Error(`Error fetching pokemon #${id}`)
        }

        const json = await res.json()

        setPokemon(json)
        setIsLoading(false)
      } catch (e) {
        setError(e.message)
        setIsLoading(false)
      }
    }

    handleFetchPokemon()
  }, [id])
```

하지만 아직도 끝난 게 아니다. 눈에 띄지 않으면서도 낭비가 가장 심한 버그라고 한다.
힌트는 **레이스 컨디션**이라고 한다.

#### 레이스 컨디션이란?

여러 `process`나 `thread`가 동시에 실행되면서 예상치 못한 순서로 실행되거나 자원을 경쟁하여 발생하는 문제라고 한다.
레이스 컨디션이 발생하면 프로그램의 결과가 실행 순서나 타이밍에 따라 달라질 수 있어 이를 예측하기 어렵기 때문에 버그가 발생하기 쉽다고 한다.

React에서 레이스 컨디션이 자주 발생하는 상황 중 하나는 `useEffect` 안에서 비동기 요청을 처리할 때이다.
비동기 요청이 완료되기 전에 다른 요청이 발생하거나 `component`가 다시 렌더링 될 수 있다.
이럴 때 React에서는 `useEffect`의 **cleanUp** 기능을 사용하여 방지할 수 있다.

아무튼 본문으로 돌아가서 `fetch`를 호출할 때 해당 요청이 얼마나 오래 걸릴지 알 수 없다.
그 사이에 User가 버튼을 클릭해 다시 렌더링을 유발하면, 새로운 ID로 또 다른 요청이 발생한다.

위 시나리오에서는 두 개의 요청이 진행 중인데 어떤 요청이 먼저 완료될지 모른다.
따라서 UI는 나중에 완료된 요청의 결과로 표시될 수 있는데 이것을 **레이스 컨디션**이라고 한다.

#### 해결 방안

`useEffect`의 `cleanUp` 기능을 사용한다.
`useEffect`에서 함수를 반환하면, React는 해당 함수를 다음 `useEffect` 호출 전에 호출하고, 마지막으로 컴포넌트가 DOM에서 제거될 때 호출한다.

이를 활용해 상태 업데이트를 무시할 수 있다.

아래는 `cleanup` 함수 사용 예시이다.

```js
useEffect(() => {
  let ignore = false;

  async function fetchData() {
    const res = await fetch(`https://pokeapi.co/api/v2/pokemon/${id}`);
    if (!ignore) {
      setPokemon(res);
    }
  }

  fetchData();

  return () => {
    ignore = true;
  };
}, [id]);
```

위 코드를 사용하면 `id`가 변경될 때 이전 요청의 응답을 무시할 수 있다.
`ignore`는 비동기 요청을 안전하게 처리하기 위한 보호 장치로, 컴포넌트가 언마운트된 후에도 상태 업데이트를 시도하는 것을 방지하는 역할을 한다.

### 커스텀 훅과 데이터 중복 문제

React에서 데이터를 `fetching`하는 로직을 커스텀 훅으로 추상화할 수 있다.
하지만 기본적으로 React에서 가져온 데이터는 `local` 상태에만 존재하며, 여러 컴포넌트에서 재사용하려면 매번 데이터를 재요청해야한다.
이로 인해 **데이터 중복 문제**가 발생한다.

해결 방안으로 **Lift State** 또는 `Context API`를 사용하는 것인데 이 방식은 코드가 복잡해지고 유지보수가 어려워질 수 있다.
`Context API`를 사용할 때 최적화도 고려해줘야 하기 때문이다.

### React Query의 등장

**React Query**는 단순한 데이터 페칭 라이브러리가 아닌 **비동기 상태 관리 도구**이다.
데이터를 직접 가져오지 않고, 사용자가 제공한 `Promise`를 통해 데이터를 처리한다.
이를 통해 여러 컴포넌트에서 데이터 중복 없이 데이터를 공유하고, 로딩 상태, 에러 상태, 캐싱, 자동 재페칭 등을 간단하게 처리할 수 있다.

#### React Query의 주요 기능

1. 캐시 관리 : 데이터를 `cashing`하여 중복 요청을 방지
2. 캐시 무효화 : 특정 이벤트나 조건에 따라 캐시를 무효화하고 새로운 데이터 요청
3. 자동 재페칭 : 네트워크 상태가 변경되거나 포커스가 변경되면 데이터를 자동으로 다시 요청
4. 오프라인 지원 : 네트워크가 연결되지 않았을 때 데이터를 캐싱하여 오프라인 모드 지원
5. 스크롤 복구, 무한 스크롤, Pagination 등 다양한 데이터 처리 기능을 제공
