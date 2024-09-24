[참고: tanstack query]
[참고: query.gg]

보통 라이브러리나 프레임워크를 사용할때에는 여러가지 이유가 존재하지만, 가장 큰 이유는 본인이 직접 하기 어렵거나 까다로운 작업을 이미 사용하기 편하게 추상화된 라이브러리(또는 프레임워크)를 이용하여 조금 더 편하게 작업하기 위한 이유가 가장 큽니다.

**The library for web and native user interfaces**

당장 리액트만 해도 UI를 편하게 구현하기 위한 자바스크립트 라이브러리입니다.

라이브러리등을 이용하는 이유가 위와 같다면, 좋은 라이브러리란 해결하기 까다로운 부분을 사용하기 쉽게 추상화 해놓은 라이브러리일것입니다.

이번 글에서 다룰 리액트 쿼리 또한 어떠한 문제에 대해서 이해하고 사용하기 쉽게 추상화 해놓은 라이브러리 중 하나압니다.

그렇다면 리액트 쿼리는 어떤 부분을 추상화하였을까요?

리액트를 사용하다보면 자연스럽게 리액트 훅을 사용하게 됩니다.

리액트 훅은 리액트가 클래스형에서 함수형으로 넘어오면서 여러 로직들이 미리 정의 해놓고 사용자가 쉽게 컨트롤할 수 있게 제공하는 기능들입니다.

좀 더 자세히 살펴보면 앞서 언급한 대로 리액트는 UI 렌더링을 도와주는 라이브러리입니다.

여기서 도와준다는 것은 사용자(개발자)가 상태만 컨트롤하면 그거에 따른 UI는 리액트에서 알아서 그려줍니다.

또한 이렇게 상태와 로직 그리고 UI를 캡슐화해서 관리하는 것이 `Component`입니다.

리액트 팀에서는 시각적인 요소를 캡슐화해서 컴포넌트로 관리하듯이, 비시각저인 요소도 캡슐화를 하여 재사용성등을 높여주기 위해 `React Hooks`를 제공해주었습니다.

예를들어 state를 생성해주고, 해당 state가 업데이트되면 리렌더링을 트리거하는 `useState`훅이 있습니다.

이렇게 훅들을 쭉 살펴보다보면 무언가 빠진느낌이 들수있습니다.

사실 현대 웹/앱에서 가장 자주 사용되는 비시각적 요소 중 하나는 데이터 가져오기입니다.

하지만 리액트 훅들중 직접적으로 데이터 가져오기를 관리해주는 훅은 존재하지 않습니다.

물론 `useEffect`에 데이터 패칭 로직을 넣어서 가져온 후 `useState`로 그것을 상태화하여 관리하는 것은 가능하지만, 이것을 직접적인 데이터 패칭 기능이라고 할 수 없을거같습니다.

또한 이렇게 관리하면 개발자가 고려해야할 것들이 너무 많습니다.

예시는 아래와 같습니다

```jsx
export default function App() {
  const [id, setId] = React.useState(1);
  const [data, setData] = React.useState(null);

  React.useEffect(() => {
    const handleFetchData = async () => {
      setData(null);

      const res = await fetch(`https://example.api.com/api/v1/${id}`);
      const json = await res.json();
      setData(json);
    };

    handleFetchData();
  }, [id]);

  return <>{/_ 데이터 표시 _/}</>;
}
```

앞서 언급한대로 이렇게 데이터 패칭 로직을 작성할 수 있습니다.

하지만 이게 좋은 코드일까요?

여기에는 로딩, 에러 처리가 빠져있습니다... 아래는 로딩, 에러 처리가 추가된 코드입니다.

```jsx
export default function App() {
  const [id, setId] = React.useState(1);
  const [data, setData] = React.useState(null);
  const [isLoading, setIsLoading] = React.useState(true);
  const [error, setError] = React.useState(null);

  React.useEffect(() => {
    const handleFetchData = async () => {
      setData(null);
      setIsLoading(true);
      setError(null);

      try {
        const res = await fetch(`https://example.api.com/api/v1/${id}`);
        if (!res.ok) {
          throw new Error(`${id}의 데이터를 가져오는데 실패하였습니다.`);
        }
        const json = await res.json();
        setData(json);
        setIsLoading(false);
      } catch (err) {
        setError(e.message);
        setIsLoading(false);
      }
    };

    handleFetchData();
  }, [id]);

  return <>{/_ 데이터 표시 _/}</>;
}
```

이제 완벽한 데이터 패칭 코드일까요?

아닙니다. 여전히 문제가 될만한 요소가 존재하고 그 부분에 대한 처리가 없습니다.

만약 어떠한 이유로 요청이 동시에 두번 날라갔다고 가정해보면 => `id: 1`에 대한 요청이 날라간 직후 바로 `id: 2`에 대한 요청이 날라간 경우

클라이언트에서는 현재 id는 3이지만 최종 도착한 데이터는 2에 대한 데이터라면 뷰는 2에 대한 뷰일것이고, 이거는 상태와 뷰의 불일치를 만들 수 있습니다.

이거를 처리하는 코드는 아래와 같습니다.

```jsx
export default function App() {
  const [id, setId] = React.useState(1);
  const [data, setData] = React.useState(null);
  const [isLoading, setIsLoading] = React.useState(true);
  const [error, setError] = React.useState(null);

  React.useEffect(() => {
    let ignore = false;

    const handleFetchData = async () => {
      setData(null);
      setIsLoading(true);
      setError(null);

      try {
        const res = await fetch(`https://example.api.com/api/v1/${id}`);

        if (ignore) {
          return;
        }

        if (!res.ok) {
          throw new Error(`${id}의 데이터를 가져오는데 실패하였습니다.`);
        }
        const json = await res.json();
        setData(json);
        setIsLoading(false);
      } catch (err) {
        setError(e.message);
        setIsLoading(false);
      }
    };

    handleFetchData();

    return () => {
      ignore = true;
    };
  }, [id]);

  return <>{/_ 데이터 표시 _/}</>;
}
```

단 하나의 데이터 패칭을 위해 로직을 이렇게 길게 작성해야합니다. (물론 커스텀 훅으로 분리 가능합니다)

하지만 이렇게 해도 아직 문제가 남아있습니다

만약에 위의 데이터를 여러 컴포넌트에서 써야해서 위의 로직을 커스텀 훅으로 분리한 후 A 컴포넌트와 B 컴포넌트에서 데이터를 각각 가져올때, 데이터 중복 / 데이터 불일치 문제가 일어날 수 있습니다.

첫번째 데이터 중복은 분명 같은 데이터를 가져오는것이지만, 각 컴포넌트는 고유한 인스턴스이므로 두 컴포넌트 모두에서 데이터를 가져오는 동안 로딩 표시를 해주어야할것입니다.

두번째 데이터 불일치 문제는 더 문제가됩니다.

만약에 A 컴포넌트에서는 데이터 가져오는데 성공하였지만, B에서는 실패한다면 이는 리액트의 근본적인 아이디어인 예측성을 무너트릴것입니다.

아니면 A 컴포넌트를 가져올때는 데이터 결과가 a였지만 몇초후 B 컴포넌트에서 가져올때에는 a-1일수도있는 문제가 있습니다.

이 모든 문제를 해결하기 위해서 Context API를 도입하여 전역적으로 데이터를 관리할 수는 있습니다.

하지만 그 순간 이제 `useState`, `useEffect`, `Context`를 모두 관리해야하는 어려움이 생기며, Context API의 최적화 문제도 고려해야합니다

만약 다른 전역 상태 관리 라이브러리를 사용해도 여전히 서버 상태를 클라이언트 상태로 관리하는 문제,, 즉 비동기 상태를 동기 상태로 관리하는 문제가 존재합니다

이러한 문제를 효과적으로 처리하는데 도움을 줄 수 있는 것이 React Query입니다.
