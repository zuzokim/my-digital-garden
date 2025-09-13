---
{"dg-publish":true,"permalink":"/posts/dev/zustand-selector-helper-state-re-render/","tags":["Typescript"],"created":"2025-08-03","updated":"2025-08-03T22:36:00"}
---

`Zustand`에서는 기본적으로 `useStore()` 훅이 **선택자(selector)** 없이 사용될 경우, **스토어의 전체 상태 객체를 반환**하기 때문에, 내부의 **어떤 state가 바뀌든 리렌더링이 발생** 한다.

예를 들어, 이렇게 사용하는 경우 
```ts
const { state1, setState1 } = useStore(); // 선택자 없이 사용
```
`state2`, `state3` 등이 업데이트되어도 이 컴포넌트는 `useStore()`를 통해 **전체 store에 대해 구독 중**이므로, **불필요하게 리렌더링** 된다.

그래서 이렇게 **Selector 패턴**으로 필요한 상태만 구독하게끔 사용해야 불필요한 리렌더를 방지해 성능상 이점을 누릴 수 있다.
```ts
const state1 = useStore((state) => state.state1);
const setState1 = useStore((state) => state.setState1);
```

체를 반환하는 경우 `shallow` 비교를 명시적으로 사용하는 것도 고려해볼 수 있다.

```ts
import shallow from 'zustand/shallow';

const { a, b } = useStore((state) => ({ a: state.a, b: state.b }), shallow);
```

근데 너무 불편하다. 그리고 실제 회사에서는 이보다 더 복잡한 상태를 관리하는 store를 구성하는데, 그런 경우 이렇게 컴포넌트 내에서 보기 안좋은 코드가 반복 작성되게 된다.

```ts
const {
		state1, 
		setState1, 
		state2,
		setState2, 
		state3,
		setState3,
		...
	  
	  } = useStore((state) => ({
			  state1: state.state1,
			  state1: state.setState1,
			  state1: state.state2,
			  state1: state.setState2,
			  state1: state.state3,
			  state1: state.setState3,
			  ...
		  }))
```

그래서 좀 더 보기 좋고, 덜 불편한 방식으로  **Selector 패턴** 을 작성할 수 없을까 고민이 되었다.

![thread 1.png](/img/user/thread%201.png)

그래서 만들어봄..

**1번.  Base store 타입 정의와 함께 생성**

```ts
//store.ts
type MyState = {
  state1: number;
  setState1: (v: number) => void;
  state2: string;
  setState2: (v: string) => void;
};

const useMyStoreBase = create<MyState>((set) => ({
  state1: 0,
  setState1: (v) => set({ state1: v }),
  state2: '',
  setState2: (v) => set({ state2: v }),
}));

```

**2번. Selector 타입 추론 helper**

```ts
//zustand-selector-helper.ts
// 넘겨받은 key 로 Selector 함수 자동 구성
export createTypedSelector<Store extends object> = (
  store: (selector: (state: Store) => any) => any
) => {
  return function <K extends readonly (keyof Store)[]>(config: {
    selectorKeys: K;
  }): Pick<Store, K[number]> {
    return store((state) => {
      const result = {} as Pick<Store, K[number]>;
      for (const key of config.selectorKeys) {
        result[key] = state[key];
      }
      return result;
    });
  };
}
```

**3번. 컴포넌트에서 사용하기**

```ts

// store.ts에서 export
export const useMyStore = createTypedSelector(useMyStoreBase);

//Component.tsx
// ✅ 컴포넌트에서 추론 잘 됨!
const { state1, setState1 } = useMyStore({ selectorKeys: ['state1', 'setState1'] as const });

```

단, 여기서 `as const`를 붙여야 **key literal 추론**이 정확히 된다.
- as const 를 사용하지 않으면 Typescript는 key를 그냥 string으로 넓은(widened) 타입으로 추론한다.

```ts
const { state1 } = useMyStore({ selectorKeys: ['state1'] as const });
// 🔴 Error: "state1" does not exist on type "MyState"
```

존재하지 않는 키를 넣으면 컴파일 에러가 난다.

++ 일단 팀원들에게 아이디어를 제안한 뒤 간단한 구현만 해보았는데, 시간을 내어 조금 더 발전시켜볼 수 있겠다.
- **as const 대신 타입을 좁혀서 추론할 수 있게 하기 with Zod.literal**
- **객체 형태의 state를 위해 shallow도 지원할 수 있게 하기**
- **실제 사용하는 state와 구독하는 selectorKey가 다른 경우 알려주기 with Eslint** 
	- https://github.com/paulschoen/eslint-plugin-zustand-rules 참고해보기

ps. 아이디어에 긍정적으로 응답해준 팀원들에게 감사하다. 나 혼자 느낀 사소한 불편함과 아이디어였지만, 나도 팀원들에게 설명을 하면서 실제 구현을 더 자세히 생각해보게 되었고 이후 발전방향도 떠올려볼 수 있었다. 