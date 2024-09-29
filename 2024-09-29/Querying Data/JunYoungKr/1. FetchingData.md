`React Query`는 Promise의 출처에 상관없이 작동한다.

그 이유는 간단하다. React Query는 비동기, Promise 기반 상태 관리 도구이기 때문이다.

- React Query로 응답 헤더를 읽는 방법은?
- React Query에서 GraphQL을 사용하는 방법은?
- 요청에 인증 토큰을 추가하는 방법은?

위 질문들은 모두 같은 답을 가진다. React Query 없이도 평소 하던대로 하면 된다.
왜냐면 React Query는 **요청 자체를 실행하지 않기 때문에** 이런 것들을 인식하지 못한다.
React Query가 가장 신경 쓰는 것은 Promise의 상태와 그것이 해결된 데이터이다.

물론, React Query에 넘길 **Promise**를 생성할 방법은 필요하며, 데이터를 가져올 때 가장 일반적인 방법은 브라우저의 내장된 `Fetch API`를 사용하는 것이다.

```js
const promise = fetch(url, options);
```

### promise란?

잠깐 `promise` 개념에 대해 알아보고 가자.

`promise`란 JS에서 비동기 작업을 처리하기 위해 사용되는 객체라고 한다.
여기서 비동기 작업이란 시간이 걸릴 수 있는 작업(예: 서버에서 데이터를 가져오는 HTTP 요청이나 파일 읽기/쓰기 작업 등)을 의미한다.
Promise는 이런 비동기 작업의 성공 또는 실패 결과를 표현하는데 매우 유용하다.

#### Promise의 상태

1. 대기(pending) : 비동기 작업이 아직 완료되지 않았고, 결과를 기다리는 상태이다.
2. 성공(fullfilled) : 비동기 작업이 성공적으로 완료되어 결과값을 반환한 상태이다.
3. 실패(rejected) : 비동기 작업이 실패하여 오류가 발생한 상태이다.

### Fetch API와 Promise

```js
fetch("https://api.github.com/users/github")
  .then((response) => {
    if (!response.ok) {
      throw new Error("네트워크 오류 발생");
    }
    return response.json(); // JSON 데이터로 변환된 Promise 반환
  })
  .then((data) => {
    console.log(data); // 성공적으로 가져온 데이터를 출력
  })
  .catch((error) => {
    console.error("데이터를 가져오는 중 오류:", error);
  });
```

위 코드에서 `fetch()`는 Promise를 반환하고, 서버의 응답을 받으면 `then()`을 통해 결과를 처리한다.
응답을 정상적으로 받으면 `JSON` 데이터를 반환하고, 오류가 있으면 `catch()`에서 오류를 처리한다.

`fetch`는 가져오려는 리소스의 URL과 옵션 객체(선택 사항)를 인수로 받는다.
호출되면 브라우저는 즉시 요청을 시작, Promise를 반환한다.
이후 응답을 받는 과정은 보통 두 단계로 이뤄진다.

1. 서버가 header를 포함한 응답을 보낼 때, fetch로 반환된 Promise는 Response 객체와 함께 해결된다.

`Fetch API`에서 가장 놀라운 점 중 하나는 요청이 실패하더라도 Promise가 거부되지 않는다는 것이다.
응답 상태 코드가 4xx, 5xx 범위에 있더라도 Promise는 정상적으로 해결된다.

#### 이유

`Fetch API`에서 4xx 또는 5xx 범위의 상태 코드가 발생했을 때도 Promise가 거부되지 않는 이유는 Fetch API의 설계 철학 때문이다.
`Fetch API`는 네트워크 요청 자체에 대한 성공 여부만을 신경 쓰며, 서버가 어떤 응답을 보내는지(정상적인 응답이든, 에러 응답이든)는 Promise의 성공 또는 실패 기준이 아니기 때문이라고 한다.

따라서, `Fetch API` 사용 시 실패한 응답은 수동으로 오류를 처리해야하는데 다음과 같다.

```js
fetch("https://api.example.com/data")
  .then((response) => {
    if (!response.ok) {
      // 응답 상태 코드가 2xx가 아니면
      throw new Error(`Request failed with status: ${response.status}`);
    }
    return response.json();
  })
  .then((data) => {
    console.log(data);
  })
  .catch((error) => {
    console.error("Error:", error); // 네트워크 오류나 서버 응답 오류 처리
  });
```

`response.ok` 속성을 사용하여 응답 상태가 성공적인지 확인해야한다. 이것은 상태 코드가 2xx일 떄만 `true`를 반환한다.

2. 다음으로는 응답 본문에서 실제 데이터를 가져와야 한다. 아래 코드에서는 JSON을 가져오고 있기 때문에 Response 객체에서 .json을 호출해, JSON으로 구문 분석된 데이터를 반환하는 또 다른 Promise를 얻을 수 있다.

```js
const fetchRepos = async () => {
  try {
    const response = await fetch("https://api.github.com/orgs/TanStack/repos");

    if (response.ok) {
      const data = await response.json();
      return data;
    } else {
      throw new Error(`Request failed with status: ${response.status}`);
    }
  } catch (error) {
    // 네트워크 오류 처리
  }
};
```

### useQuery를 결합한 코드

```js
function useRepos() {
  return useQuery({
    queryKey: ["repos"],
    queryFn: async () => {
      const response = await fetch(
        "https://api.github.com/orgs/TanStack/repos"
      );

      if (!response.ok) {
        throw new Error(`Request failed with status: ${response.status}`);
      }

      return response.json();
    },
  });
}
```

여기서 위 코드와 다르게 차이점들이 있다.

우선, `try/catch` 코드가 사라졌는데 이는 `React Query`가 자체적으로 Promise의 상태(성공, 실패)를 관리하기 때문이다.
React Query는 비동기 함수(queryFn) 내부에서 발생하는 오류를 자동으로 처리하여, `성공`, `로딩`, `오류`로 자동으로 전환해준다.
이 덕분에 개발자는 `try/catch` 블록을 작성할 필요가 없어진다.

두 번째로, `response.json()`을 바로 반환할 수 있다.
기존 `fetch`나 `axios`는 `response.data` 이런 식으로 반환했지만 React Query 사용 이후에는 바로 반환할 수 있다.
