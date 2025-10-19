---
{"dg-publish":true,"permalink":"/posts/zod-type-safe-form-validation/","created":"2025-10-19","updated":"2025-10-19"}
---


프론트엔드 개발에서 아주 많이 다루는 from과 유효성 검사. 

[Zod](https://zod.dev/) 와 react-hook-form을 대부분 활용한다. 다른 form 라이브러리도 있는데 가장 익숙한 것은 아무래도 react-hook-form이다.

최근에 Zod와 react-hook-form의 resolver를 활용해서 보다 선언적이고, 유효성 검증이라는 하나의 관심사를 분리해 구현하는 방법을 알게 되어 시도해봤다. 기초적인 Zod의 여러 함수가 어떤 역할을 하고, 어떻게 활용하면 될지 공부하고 기록해보는 글.

---

내가 익숙했던 방식은 아래와 같이 register한 form field에 required 여부와 유효성 패턴을 각각 작성해주는 식이었다. 이렇게 하면 각 field에 필수값 여부, 메세지, 유효성 검사 로직이 작성되므로 빠른 구현이 된다.

```ts
{...register("experience", {
	required: "FE 경력 연차를 선택해주세요",
})}
```


```ts
{...register("github", {
    pattern: {
		  value:/^https:\/\/(www\.)?github\.com\/[a-zA-Z0-9-]+\/?.*$/,
		  message: "올바른 Github URL 형식으로 입력해주세요",
	},
})}
```


다만, field가 매우 많아지는 경우를 가정하면, 한 컴포넌트 내에 form으로 관리해야할 field 마크업과 로직이 뒤섞이게 되고, 추후 수정이 필요하면 한 컴포넌트 파일 안에서 스크롤을 하며 이리저리 수정을 해줘야 한다.

---

이제 그럼 zodResolver와 zod로 type-safe한 유효성 검사를 선언적으로 관리해보자.

아래처럼 유효성 검사로직을 담은 schema 파일을 별도로 만들어주자.

```ts
import z from "zod";

export type Experience = "0-3" | "4-7" | "8+";

export const formSchema = z.object({
	name: z.string().min(2, "최소 2글자 이상 입력해주세요"),
	email: z.email("올바른 이메일 형식으로 입력해주세요"),
	experience: z
		.string()
		.min(1, "FE 경력 연차를 선택해주세요")
		.refine(
		(val) => val === "0-3" || val === "4-7" || val === "8+",
		"FE 경력 연차를 선택해주세요"
	),
	github: z
		.string()
		.regex(
		/^https:\/\/(www\.)?github\.com\/[a-zA-Z0-9-]+\/?.*$/,
		"올바른 Github URL 형식으로 입력해주세요"
	)
	.optional()
	.or(z.literal("")), // 빈 문자열도 허용
});

// Zod 타입에서 자동으로 FormData 타입 생성
export type FormData = z.infer<typeof formSchema>;
```

그리고 react-hook-form의 resolver에 넘겨주면 field 마크업에는 register만 해주면 된다. 실제 유효성 검사 로직은 schema 파일 내에서 다룰 수 있게 되는 것.

```ts
import { zodResolver } from "@hookform/resolvers/zod";
import { formSchema, type FormData } from "./schema/formSchema";

const {
	register,
	handleSubmit,
	watch,
	formState: { errors },
	reset,
} = useForm<FormData>({
	resolver: zodResolver(formSchema), // 유효성 검사로직을 담은 schema
	defaultValues: {
		name: "",
		email: "",
		experience: undefined,
		github: "",
	},
});

return <Select
	id="experience"
	aria-invalid={errors.experience ? "true" : "false"}
	aria-describedby={
	errors.experience ? ERROR_EXPERIENCE : undefined
	}
	{...register("experience")} //register만 해주기
	value={watch("experience") || ""}
>	
	<option value="" disabled>
	선택해주세요
	</option>
	<option value="0-3">0-3년차</option>
	<option value="4-7">4-7년차</option>
	<option value="8+">8년차 이상</option>
</Select>
...
```

##### 그럼 이제 zod의 여러 함수가 어떤 역할을 하는지 자세히 알아보자.

우선, Select에서 아무것도 선택하지 않으면 첫번째 option이 기본 선택된다. 하지만, 요구사항은 
- 기본으로는 선택되지 않고, 문구를 "선택해주세요"로 보여준다.
- option이 선택됐을 때에만 form 제출할 수 있게 유효성 검사를 한다.
라면 어떻게 처리할 수 있을까?

처음에는 아래와 같이 빈문자열을 기본값으로 하고 enum을 옵션으로 선택하고 타입 검사를 하도록 처리를 해줬었다.

```ts
  experience: z
	  .enum(["0-3", "4-7", "8+"], "FE 경력 연차를 선택해주세요")
	  .or(z.literal("")),
```

이러면 form type상 기본값을 빈 문자열로 줘도 타입에러가 발생하지 않는다. 그러나 이렇게 하면 실제 제출되는 값도 빈 문자열 리터럴을 허용하기 때문에 실제 option을 선택하지 않아도 form이 제출되어버리는 문제가 있다.

---

##### 올바르게 유효성 검사하기

그럼 이렇게 할 수 있다.

```ts
experience: z
    .string()
    .min(1, "FE 경력 연차를 선택해주세요")
    .refine(
      (val) => val === "0-3" || val === "4-7" || val === "8+",
      "FE 경력 연차를 선택해주세요"
    ),
```

#### Zod의 `.refine()`는 어떻게 동작하는 것일까?

겉보기엔 단순히 “조건을 하나 더 추가하는 후처리 함수”처럼 보이지만, 실제로는 타입 시스템과 런타임 검증 사이의 경계를 다루는 중요한 메커니즘이다. 

`.refine()`는 말 그대로 “정제(refine)”하는 함수이다.  기존 스키마가 통과시킨 값을 다시 한 번 걸러내고, 그 값이 원하는 조건을 만족하지 않으면 에러를 발생시킨다.

`.refine()` 의 시그니쳐를 짚어보자. 

```ts
schema.refine(
  (value) => boolean | Promise<boolean>, // 조건식
  { message?: string, path?: (string | number)[] } // 옵션
);

```

- 첫 번째 인자 `(value)`는 현재 검증 중인 값이다.  
    이 함수가 `true`를 반환하면 통과, `false`면 실패이다.
- 두 번째 인자는 에러 메시지와 위치를 지정하는 옵션이다.  
    `path`는 어느 필드에 에러를 표시할지를 지정할 때 쓴다.
- 비동기 함수도 지원하기 때문에, 서버에 요청해 중복 여부를 확인하는 검증 같은 것도 가능하다.  
    이 경우 `parseAsync` 또는 `safeParseAsync`를 사용해야 한다.


`.refine()` 이 후처리 방식이라고 했는데, 그럼 Zod는 어떤 순서로 스키마를 검증하는지도 보자.

Zod는 스키마를 검증할 때 내부적으로 아래 순서대로 실행한다.

1. **기본 타입 검사**: 예를 들어 `z.string()`이라면 값이 문자열인지 확인한다.
2. **내장 제약 검사**: `.min()`, `.max()`, `.email()` 같은 내장 제약을 순서대로 검사한다.
3. **사용자 정의 검사 (`refine`, `superRefine`)**: 우리가 직접 정의한 조건 함수를 호출한다.
4. **에러 수집**: 실패한 검사는 모두 `ZodError.issues` 배열에 추가된다.

즉, `.refine()`는 “기본 검증을 통과한 다음, 그 값에 추가 조건을 검증하는 후처리 단계” 에서 호출된다.


---

##### Zod의 `.superRefine()` 과 비교

`.refine()`는 “boolean 기반의 단일 조건 검사”를 위한 간단한 도구이다.  하지만 복수의 조건을 한 번에 검사하거나, 다양한 필드에 에러를 나눠서 표시하고 싶을 때는 `.superRefine()`를 써야 한다.

```ts
z.string().superRefine((val, ctx) => {
  if (!/^[0-9]+$/.test(val)) {
	  //콜백의 두 번째 인자로 `ctx`를 받아 `ctx.addIssue()`로 여러 검증 결과를 추가할 수 있다.
    ctx.addIssue({
      code: z.ZodIssueCode.custom,
      message: "숫자만 입력해야 합니다",
    });
  }

  if (val.length < 3) {
    ctx.addIssue({
      code: z.ZodIssueCode.too_small,
      message: "길이가 너무 짧습니다",
    });
  }
});
```

---

##### 타입시스템 관점에서의 한계

`.refine()`는 런타임 검증만 수행한다.  즉, TypeScript 타입은 여전히 기존 타입 그대로 유지된다.

```ts
const experience = z
  .string()
  .refine((v) => v === "0-3" || v === "4-7" || v === "8+");

type Experience = z.infer<typeof experience>;
// 여전히 string

```

이건 타입 시스템 입장에서는 `string` 전체 중 일부만 허용하는 걸 알 수 없기 때문이다.  만약 타입까지 제한하고 싶다면 `z.enum()`을 사용해야 한다.

```ts
const experience = z.enum(["0-3", "4-7", "8+"]);
type Experience = z.infer<typeof experience>;
// "0-3" | "4-7" | "8+"
```

`.refine()`는 어디까지나 런타임 검증을 위한 장치일 뿐, 타입을 좁히지 않는다는 점을 기억해야 한다.  따라서 **타입 안정성과 런타임 안전성**을 모두 챙기고 싶다면, 가능한 한 `z.enum`, `z.literal`, `z.union` 같은 선언적 스키마를 우선적으로 고려하자.

---

##### 비동기 함수도 지원하는 `.refine()`

```ts
const schema = z.string().refine(async (val) => {
  const exists = await checkIfUsernameExists(val);
  return !exists;
}, "이미 존재하는 아이디입니다");
```

이렇게 하면 비동기로 서버에서 유효성을 검증할 수 있다. 단, 이 경우에는 반드시 `parseAsync()`나 `safeParseAsync()`를 사용해야 한다.  동기 `parse()`를 쓰면 `Promise`가 resolve되는 것을 기다려주지 않기 때문에 검증이 제대로 작동하지 않는다.

react-hook-form의 resolver는 Promise를 반환하면 **기본적으로 await 처리** 해주므로 문제가 없다. 정확히 말하면, 디폴트로 parseAsync를 사용한다.

https://github.com/react-hook-form/resolvers/blob/e95721d3c8c6d6e555508b0e7b21c6ac801360cf/zod/src/zod.ts#L224

> - The function gives resolverOptions a default empty object: the parameter line (lines ~241–245) sets resolverOptions: { mode?: 'async' | 'sync'; raw?: boolean; } = {}.
> - The code then chooses the async parser whenever mode is not the string 'sync':
>   - Zod 3 path: schema[resolverOptions.mode === 'sync' ? 'parse' : 'parseAsync'](https://github.com/react-hook-form/resolvers/blob/e95721d3c8c6d6e555508b0e7b21c6ac801360cf/zod/src/zod.ts#L250) (lines ~249–251)
>   -Zod 4 path: const parseFn = resolverOptions.mode === 'sync' ? z4.parse : z4.parseAsync (lines ~283–285)
> 
> Because resolverOptions.mode is undefined by default, the checks treat it as not 'sync' and therefore use the async parser — effectively making 'async' the default.

