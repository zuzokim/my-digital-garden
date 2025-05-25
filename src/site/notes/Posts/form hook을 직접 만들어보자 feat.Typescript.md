---
{"dg-publish":true,"permalink":"/posts/form-hook-feat-typescript/","tags":["React","type-safe"],"created":"2025-05-25","updated":"2025-05-25T20:59:00"}
---

최근에 우연히 form hook을 직접 만들어볼 일이 있었다. 평소에는 주로 react-hook-form 라이브러리를 사용했었는데, 라이브러리의 도움없이 form 데이터와 상태를 다룰 수 있는 추상화된 React 커스텀 훅을 직접 구현해보았다. 사실 react-hook-form 에도 버그가 있고, form 기능 자체도 다루기 까다로운 편이라고 생각해왔어서, 아주 기본적인 기능에 집중하는 대신 타입 안전성을 보장하는 간단한 훅을 만드는 것을 목표로 했다.

- form value 상태 관리 : 중첩된 객체 형태의 상태도 관리할 수 있게 하자.
- error 상태 관리 : 추상화된 인터페이스로 유효성 검사나 에러 상태를 공통적으로 관리할 수 있게 하자.
- Typescript를 적극 활용해 form 상태와 값 접근을 모두 type-safe하게 하자.

이렇게 3가지 목표를 중심으로 구현을 했다.

폼 상태가 중첩된 객체 형태라면, `user.address.street` 같은 경로를 통해 특정 값을 읽거나 업데이트할 수 있어야 한다. 이를 위해 먼저 중첩된 경로를 타입으로 표현할 필요가 있다.

#### 1. 중첩 경로 타입 만들기 — `DotPath<T>`

```ts
// 중첩 깊이 제한을 위한 배열
export type PrevArr = [never, 0, 1, 2, 3, 4, 5, 6, 7, 8, 9, 10];

// 중첩 객체 경로 타입 생성기
export type DotPath<T, Depth extends number = 5, Prefix extends string = ""> = [
  Depth
] extends [never]
  ? never
  : Depth extends 0
  ? never
  : T extends readonly (infer U)[]
  ? // 배열인 경우: 인덱스에 대한 경로도 허용
	| Prefix
    | `${Prefix}${number}`
    | DotPath<U, PrevArr[Depth], `${Prefix}${number}.`>
  : {
      [K in keyof T & string]: T[K] extends object
        ? `${Prefix}${K}` | DotPath<T[K], PrevArr[Depth], `${Prefix}${K}.`>
        : `${Prefix}${K}`;
    }[keyof T & string];
```

- `DotPath<T>`는 타입 `T` 내에서 접근 가능한 모든 경로 문자열(예: `"user"`, `"user.address"`, `"user.address.street"`)를 만들어낸다.
- 배열도 지원하여, 예를 들어 `"users.0.name"` 같은 경로도 포함한다.
- 최대 재귀 깊이를 제한하는 `Depth` 매개변수로 무한 재귀를 방지한다.

#### 2. 경로에 따른 실제 값 타입 추출 — `DotPathValue<T, P>`

```ts
export type DotPathValue<T, P extends string> = T extends readonly (infer U)[]
  ? P extends `${number}.${infer Rest}`
    ? DotPathValue<U, Rest>
    : P extends `${number}`
    ? U
    : never
  : P extends `${infer K}.${infer Rest}`
  ? K extends keyof T
    ? DotPathValue<T[K], Rest>
    : never
  : P extends keyof T
  ? T[P]
  : never;
```

- `DotPathValue<T, P>`는 경로 문자열 `P`에 해당하는 타입을 `T`에서 찾아낸다.
- 예를 들어, `T`가 `{ user: { name: string } }`이고, `P`가 `"user.name"`이라면 반환 타입은 `string`이 된다.
- 이 덕분에 `setValue("user.name", "Alice")` 호출 시 타입 체크가 된다.

#### 3. 경로에 따른 실제 값 타입 추출 — `DotPathValue<T, P>`

```ts
function setNestedValue<T, P extends DotPath<T>>(
  obj: T,
  path: P,
  value: DotPathValue<T, P>
): T {
  const keys = path.split(".");
  const lastKey = keys.pop()!;
  const newObj = { ...obj };
  let curr: any = newObj;
  for (const key of keys) {
    if (!(key in curr)) curr[key] = {};
    curr[key] = { ...curr[key] };
    curr = curr[key];
  }
  curr[lastKey] = value;
  return newObj;
}
```

- 불변성을 지키면서 중첩된 객체의 특정 경로에 값을 설정하는 함수다.
- 경로 문자열을 `.` 기준으로 쪼개고, 단계별로 객체를 복사해가며 새 객체를 만들어낸다.
- `value` 타입도 `DotPathValue`로 타입 안전하게 받는다.

#### 4. `useForm` 커스텀 훅 구조
```ts
export function useForm<T extends Record<string, unknown>>({
  defaultValues,
  validate,
}: {
  defaultValues: T;
  validate?: Validator<T>;
}) {
  const [values, setValues] = useState(defaultValues);
  const [errors, setErrors] = useState<Partial<Record<keyof T, string>>>({});

  // 값 변경 함수
  const handleSetValue = <P extends DotPath<T>>(
    path: P,
    value: DotPathValue<T, P>
  ) => {
    setValues((prev) => setNestedValue(prev, path, value));
    setErrors((prevErrors) => {
      if (!prevErrors) return prevErrors;
      const newErrors = { ...prevErrors };
      if (path in newErrors) {
        delete newErrors[path as keyof T];
      }
      return newErrors;
    });
  };

  // 유효성 검사 함수
  const handleValidate = () => {
    const validationErrors = validate?.(values) ?? {};
    setErrors(validationErrors);
    return Object.keys(validationErrors).length === 0;
  };

  // onSubmit 핸들러 생성 함수
  function handleSubmit(onValid: (values: T) => void) {
    return (e?: FormEvent) => {
      if (e) e.preventDefault();
      const isValid = handleValidate();
      if (isValid) {
        onValid(values);
      }
    };
  }

  return {
    values,
    setValue: handleSetValue,
    validate: handleValidate,
    formState: { errors },
    onSubmit: handleSubmit,
  };
}

```

#### 5. 실제 사용 케이스

```ts

import React from "react";
import { useForm } from "./useForm"; // 앞서 작성한 훅

const UserForm = () => {
  const {
    values: formValues,
    setValue,
    validate,
    onSubmit,
    formState: { errors },
  } = useForm<TYPE>({
    defaultValues: {
      user: {
        name: "",
        age: 0,
        address: {
          street: "",
          city: "",
          zipCode: "",
        },
      },
    },
    validate: validateForm,
  });

  return (
    <form onSubmit={onSubmit((values) => console.log("폼 제출됨", values))}>
      <div>
        <label>이름</label>
        <input
          value={formValues.user.name}
          onChange={(e) => setValue("user.name", e.target.value)}
        />
      </div>

      <div>
        <label>나이</label>
        <input
          type="number"
          value={formValues.user.age}
          onChange={(e) => setValue("user.age", Number(e.target.value))}
        />
      </div>

      <div>
        <label>도로명 주소</label>
        <input
          value={formValues.user.address.street}
          onChange={(e) => setValue("user.address.street", e.target.value)}
        />
      </div>

      <div>
        <label>도시</label>
        <input
          value={formValues.user.address.city}
          onChange={(e) => setValue("user.address.city", e.target.value)}
        />
      </div>

      <div>
        <label>우편번호</label>
        <input
          value={formValues.user.address.zipCode}
          onChange={(e) => setValue("user.address.zipCode", e.target.value)}
        />
      </div>

      {errors.user && <p style={{ color: "red" }}>{errors.user}</p>}

      <button type="submit">제출</button>
    </form>
  );
};

export default UserForm;

```

![Screenshot 2025-05-25 at 9.50.16 PM.png](/img/user/Screenshot%202025-05-25%20at%209.50.16%20PM.png)
![Screenshot 2025-05-25 at 9.50.25 PM.png](/img/user/Screenshot%202025-05-25%20at%209.50.25%20PM.png)

