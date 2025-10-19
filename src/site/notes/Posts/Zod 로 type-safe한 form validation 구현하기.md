---
{"dg-publish":true,"permalink":"/posts/zod-type-safe-form-validation/","created":"2025-10-19","updated":"2025-10-19"}
---


프론트엔드 개발에서 아주 많이 다루는 from과 유효성 검사. [Zod](https://zod.dev/) 와 react-hook-form을 대부분 활용한다. 다른 form 라이브러리도 있는데, 가장 익숙한 것은 아무래도 react-hook-form이다.

최근에 Zod와 react-hook-form의 resolver를 활용해서 보다 선언적이고 유효성 검증이라는 관심사를 분리해 구현하는 방법을 알게 되어 시도해봤다. 기초적인 Zod의 여러 함수가 어떤 역할을 하고, 어떻게 활용하면 될지 공부하고 기록해보는 글.

---

내가 익숙했던 방식은 아래와 같이 register한 form field에 required 여부와 유효성 패턴을 각각 작성해주는 식이었다. 이렇게 하면 각 field에 필수값 여부, 메세지, 유효성 검사 로직이 작성되므로, 빠른 구현이 된다.

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


다만, field가 매우 많아지는 경우를 가정하면, 한 컴포넌트 내에 form으로 관리해야할 field 마크업과 로직이 뒤섞이게 되고, 추후 수정이 필요하면 한 컴포넌트 파일 안에서 스크롤을 하며 수정을 해줘야 한다.

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
	resolver: zodResolver(formSchema), // 유효성 검사로직을 담음 schema
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

그럼 이제 zod의 여러 함수가 어떤 역할을 하는지 자세히 알아보자.

우선, Select의 
