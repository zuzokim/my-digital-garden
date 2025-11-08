---
{"dg-publish":true,"permalink":"/posts/typescript-generic-1/","tags":["Typescript"],"created":"2025-11-08","updated":"2025-11-08"}
---

[[Posts/Typescript 제네릭, infer\|Typescript 제네릭, infer]] 에서도 Genrice 타입과 infer 키워드 개념과 활용을 다루었었다. 그런데 아직도 문법 자체에 익숙해지지가 않아서 연습해볼겸 셀프 연습문제를 만들고, 오래 기억하기 위한 접근방법을 기록해보기로 한다.

보다 제너럴한 컴포넌트 및 훅, 함수 등을 설계할 때 컴파일 타임에 타입을 고정시키지 않고 여러 가지 '가능한' 타입을 허용할 수 있도록 자유자재로 타입을 넓히고 좁힐 수 있는 숙련도가 필요하다는 생각이 최근들어 더 많이 든다. AI 없이 타입스크립트 정의, 인터페이스 정의를 잘 해보자.

---
### Typsectip Generic에 익숙해지기 위해 Generic을 **함수**처럼 생각하고 변형해보는 연습을 해보자.

#### 1단계: 기본기 다지기 (Generic 함수)

##### 연습 문제 1
다음 함수를 Generic으로 바꿔서, `number`, `string`, `boolean` 모두 사용할 수 있게 해보자.

```js
function identity(value: number): number {
  return value;
}
```

목표 : 
```ts
identity<string>("hello")` → `"hello"`
```

솔루션 : 
```ts
function identity<T>(value: T): T {
  return value;
}
```

##### 연습문제 2
배열의 첫 번째 요소를 반환하는 함수를 Generic으로 작성해보자.

```js
function first(arr: any[]): any {
  return arr[0];
}
```

목표 : 
```js
const a = first([1, 2, 3]);       // a: number
const b = first(["a", "b", "c"]); // b: string
```

솔루션 : 
```ts
function first<T>(arr: T[]): T {
  return arr[0];
}

//빈배열인 경우도 가드하려면
function first<T>(arr: T[]): T | undefined {
  return arr[0];
}

//“각 요소 타입을 따로 다루고 싶다”면 `T`를 유니언이 아닌 튜플로 유지해야 한다.
function first<T extends readonly unknown[]>(arr: T): T[number] {
	return arr[0];
}

//as const를 활용하면 T[number] 튜플로 고정할 수 있다. (true | 1 | "two")
*여기서 잠깐!*
// 튜플이 뭘까?
- 튜플이란 요소의 갯수와 각 인덱스 요소의 타입이 고정된 배열 (모양이 정해진 배열)
  ex. Array : `[1, 2, 3]`, `string[]`, tuple : `[string, number]`, `[boolean, string, number]`
  
const arr1 = [1, "two", true];
// 타입: (string | number | boolean)[]

const arr2 = [1, "two", true] as const;
// 타입: readonly [1, "two", true]

// 적용해보면
const arr = [1, "two", true] as const;
const firstElFromMixedTypes = first(arr);

//그러나 튜플로 추론하게 되면 정확히 첫번째 요소의 타입만을 좁혀 추론할 수 없다.
//따라서 첫 번째 요소의 타입을 정확히 추출하도록 조건부 타입을 활용하자. (T extends [] ? undefined : T[0])

function first<T extends readonly unknown[]>(arr: T): T extends [] ? undefined : T[0] {
  return arr[0] as any;
}


// `T[0]`은 첫 번째 요소의 타입을 정확히 추출한다.
const a = first([1, "two", true] as const); // 1
const b = first(["a", "b", "c"]);           // string
const c = first([]);                        // undefined


```

#### 2단계: 제약조건 (Constraints)

##### 연습문제 3
`length` 프로퍼티가 있는 값만 받을 수 있는 함수를 작성해보자. (2번 문제에서 이미 다룬 것)

```ts
function getLength<T>(value: T): number {
  return value.length; // 에러 발생!
}
```

목표 : 
```js
getLength("hello") ✅    
getLength([1, 2, 3])  ✅
getLength(123) ❌
```

솔루션 : 
```ts
//타입변수 T를 extends하는 이유는 파라미터 arr가 항상 인덱싱이 가능한 배열임을 보장하기 우한 가드다. 
// 만약 이런 제약이 없다면 에러가 발생한다. (Property 'length' does not exist on type 'T'.)
T extends readonly unknown[]

function getLength<T extends readonly unknown[]>(value: T): number {
  return value.length;
}
```

##### 연습문제 4
객체에서 특정 키의 값을 안전하게 가져오는 함수를 만들어보자.
```ts
function getProperty(obj: any, key: string): any {
  return obj[key];
}
```

목표 : 
```ts
const address = { city: "Seoul", postCode: 2954 };
const city = getProperty(address, "city"); // string
const postCode = getProperty(address, "postCode");   // number
getProperty(address, "country");             // ❌ "country"은 존재하지 않음
```

솔루션 : 
```ts
function getProperty<T, K extends keyof T>(obj: T, key: K): T[K] {
  return obj[key];
}
```

---
Mapped Type은 이어서 또 알아보자.