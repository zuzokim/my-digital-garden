---
{"dg-publish":true,"permalink":"/posts/dev/typescript-infer/","tags":["Typescript"],"created":"2025-06-01","updated":"2025-06-01T22:49:00"}
---

TypeScript는 정적 타입 언어로서 코드의 안정성과 예측 가능성을 높이는 데 큰 장점이 있다. 그중에서도 제네릭(Generic)과 조건부 타입, 그리고 `infer` 키워드는 타입 시스템을 훨씬 더 강력하게 만들어주는 도구다.

이번 글에서는 제네릭의 기본 개념부터 시작해, `infer` 키워드가 어떤 맥락에서 등장하고, 어떻게 활용되는지를 예제 중심으로 정리해본다. 제네릭과 infer 키워드를 대충 생각만 하고 코딩하다가 막상 정확히 설명하려니 잘 설명을 못하겠더라.. 이번 기회에 정리!

---

## 제네릭(Generic)이란?

제네릭은 **타입을 파라미터화**하는 문법이다. 즉, 값이 아니라 타입을 함수나 클래스, 인터페이스 등에서 동적으로 받을 수 있도록 해준다. Java나 C++과 같은 언어에서도 존재하는 개념이며, TypeScript에서도 핵심적인 타입 설계 도구다.

```ts

function identity<T>(value: T): T {
  return value;
}

const a = identity<string>("hello"); // string
const b = identity<number>(123);     // number

```

위의 `identity` 함수는 인자로 받은 값을 그대로 반환한다. 중요한 점은 `T`라는 타입 변수(type variable)를 사용해 입력값의 타입을 그대로 반환값에도 사용한다는 것이다. 이 덕분에 `identity` 함수는 어떤 타입이 오더라도 타입 안정성을 유지할 수 있다.

### 제네릭의 장점

- **재사용성**: 같은 로직을 다양한 타입에 대해 적용할 수 있다.
- **타입 안정성**: 타입 추론 덕분에 타입 오류를 미리 방지할 수 있다.
- **가독성 및 유지보수**: 추상화 수준이 높아진다.

## 제네릭과 조건부 타입의 만남

제네릭은 다른 고급 타입과 조합되면 훨씬 더 유용하다. 예를 들어 조건부 타입(Conditional Types)을 통해 입력 타입에 따라 결과 타입을 달리할 수 있다.

```ts
type IsString<T> = T extends string ? true : false;

type A = IsString<"hello">; // true
type B = IsString<123>;     // false

```

이런 조건부 타입이 강력해지는 순간은 타입의 일부를 추론해야 할 때이다. 이때 등장하는 것이 바로 `infer`이다.

## `infer`란 무엇인가?

`infer`는 조건부 타입 안에서 **타입을 추론해 새로운 타입 변수로 사용하는 키워드**이다. 특정 구조의 타입에서 내부 타입을 꺼내 사용하고자 할 때 매우 유용하다.

### 기본 문법

```ts
type Unwrap<T> = T extends Promise<infer U> ? U : T;
```

이 예제는 `Promise<string>`과 같은 타입에서 `string`을 추출한다. 만약 `Promise`가 아니라면 원래 타입을 그대로 반환한다.

```ts
type A = Unwrap<Promise<number>>; // number
type B = Unwrap<string>;          // string
```

## 자주 쓰이는 `infer` 예제

### 1. 배열 요소 타입 추출

```ts
type ElementType<T> = T extends (infer U)[] ? U : never;

type A = ElementType<string[]>; // string
type B = ElementType<boolean>;  // never
```

`infer U`를 통해 배열 요소 타입을 추론하고 있다. 배열이 아닌 경우에는 `never`을 반환한다.

### 2. 함수 반환 타입 추출

```ts
type MyReturnType<T> = T extends (...args: any[]) => infer R ? R : never;

type A = MyReturnType<() => number>; // number
type B = MyReturnType<(x: string) => void>; // void
```

함수의 반환 타입을 추론할 수 있다. 실제로 TypeScript의 내장 유틸리티 타입 `ReturnType<T>`도 이런 방식으로 구현되어 있다.

### 3. 튜플 첫 요소 추출

```ts
type Head<T> = T extends [infer H, ...unknown[]] ? H : never;

type A = Head<[1, 2, 3]>; // 1
type B = Head<[]>;        // never
```

튜플 구조 분해를 통해 첫 번째 요소만 뽑아낼 수 있다. `infer`는 배열이나 튜플과 같은 구조에서도 강력하게 작동한다.


## 정리

`infer`는 TypeScript의 **조건부 타입(Conditional Types)** 안에서 사용되는 키워드로, **타입 추론(type inference)** 을 가능하게 해준다. 쉽게 말해, 어떤 타입으로부터 부분 타입을 추출해 이름을 붙여서 나중에 그 이름으로 활용할 수 있게 해주는 기능이다.