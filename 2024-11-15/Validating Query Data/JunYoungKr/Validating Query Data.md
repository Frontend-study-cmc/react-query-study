# Validating Query Data

```js
const data = [
  {
    id: 1,
    name: "Dominik",
  },
  {
    id: 2,
    name: "Tyler",
  },
];

function getGreetings(input) {
  return input.map(({ name }) => `Hello, ${name.toUpperCase()}`);
}
```

객체 배열을 정의한 다음, 각 사람에게 인사를 반환하는 함수를 작성한다고 가정해보자.

1. 첫 번째 검증 방법 : 테스트 작성
   함수의 첫 번째 검증 단계는 테스트 코드를 작성하는 것이다.

```js
describe("getGreetings", () => {
  it("should return greetings", () => {
    const input = [
      { id: 1, name: "Dominik" },
      { id: 2, name: "Tyler" },
    ];
    expect(getGreetings(input)).toEqual(["Hello, DOMINIK", "Hello, TYLER"]);
  });
});
```

위 테스트는 `getGreetings` 함수가 예상대로 동작하는지 검증한다.

2. 두 번째 검증 방법 : TypeScript 타입 정의
   TypeScript를 사용한다면, 함수에 타입 정의를 추가하는 것도 검증의 한 단계이다.

```js
function getGreetings(input: ReadonlyArray<{ name: string }>) {
  return input.map(({ name }) => `Hello, ${name.toUpperCase()}`);
}
```

`ReadonlyArray`는 배열의 요소를 수정하지 못하도록 보호한다.
`name: string` 타입 정의는 `name` 속성이 문자열임을 보장한다.

### 제 3자 API 데이터의 문제점

- 일회성 요청을 통해 응답 구조를 검사할 수는 있지만, 모든 응답이 항상 동일한 구조를 가질 것이라고 보장할 수 없다.
- 웹 애플리케이션에서 가장 흔한 버그는 개발자가 예상하는 데이터 구조와 실제로 받은 데이터 구조 간의 불일치에서 발생한다.
- 이러한 버그는 주로 다음과 같은 에러 메시지를 유발한다:

```js
Uncaught TypeError: Cannot read properties of undefined (reading 'name')
```

### 해결 방법 : 데이터 검증 (Validation)

`Zod` 라이브러리를 활용하여 스키마 검증을 한다.
Zod는 응답의 예상 구조를 정의하고, 실제 응답을 해당 스키마와 비교해 검증한다.
스키마와 일치하지 않는 데이터는 통과할 수 없다. 따라서 React Query의 `queryFn`과 통합하기에 적합하다.

### 트레이드오프 (Trade-off)

- 런타임 검증 비용 : 검증 작업은 네트워크 응답을 분석하고 타입을 확인하는 추가 작업을 요구하므로, 큰 응답 데이터를 자주 검증할 경우 성능에 영향을 줄 수 있다.

- API 신뢰 여부 : 만약 제어할 수 있는 API라면, 검증 없이도 안정적인 데이터를 받을 수 있다.
  그러나 제3자 API처럼 신뢰할 수 없는 경우에는 데이터 검증을 고려해야 한다.
