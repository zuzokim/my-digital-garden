---
{"dg-publish":true,"permalink":"/posts/zustand-combine/","tags":["type-safe"],"created":"2025-04-13","updated":"2025-04-13T15:08:00"}
---

최근 참여하고 있는 프론트엔드 코드리뷰 스터디에서 간단한 토이프로젝트를 진행하고 있다. 장바구니 기능이 있는 이커머스 페이지에서 상태관리에 초점을 맞추어 구현하고 서로의 코드를 리뷰해주는 시간을 갖는다. 그 과정에서 전역적으로 상태를 관리해야할 때 많은 스터디원들이 zustand를 사용했는데, 나도 회사에서 zustand를 애용하고 있던 와중 스터디원이 소개해준 덕에 새로운 미들웨어를 알게 돼서 알게 된 내용을 정리해본다. 인터널 소스코드도 Typescript를 활용한 굉장히 짧은 코드여서 어떤식으로 스토어의 타입을 추론하는지 뜯어보았다.

---

## `combine`미들웨어의 타입 추론 방식

Zustand의 `combine` 미들웨어 함수 자체 구현 코드가 매우 짧아서 놀랐다. 짧고 단순하지만 타입추론 방식이 Generic을 활용한 방식이라 조금 더 들여다 보았다.

소스코드 : https://github.com/pmndrs/zustand/blob/main/src/middleware/combine.ts#L3

```ts
import type { StateCreator, StoreMutatorIdentifier } from '../vanilla.ts'

type Write<T, U> = Omit<T, keyof U> & U

export function combine<
  T extends object,
  U extends object,
  Mps extends [StoreMutatorIdentifier, unknown][] = [],
  Mcs extends [StoreMutatorIdentifier, unknown][] = [],
>(
  initialState: T,
  create: StateCreator<T, Mps, Mcs, U>,
): StateCreator<Write<T, U>, Mps, Mcs> {
  return (...args) => Object.assign({}, initialState, (create as any)(...args))
}
```

여기서 중요한 타입 유틸이 하나 있다:

```ts
// T와 U를 합치되, U에 있는 키는 T에서 덮어쓴다
type Write<T, U> = Omit<T, keyof U> & U
```

이 타입은 `T`와 `U`의 충돌되는 키를 제거하고, `U`로 덮어씌우는 역할을 한다. 결국 최종적으로 반환되는 상태는 `T & U` 타입이지만, `U`의 메서드가 우선시된다.

```ts
// 예시
Write<{ count: number }, { count: () => void }> 
// 결과: { count: () => void }
```

- `Write<T, U>`
	- `T`와 `U`를 병합하되, `U`가 `T`의 필드를 덮어쓸 수 있도록 설계됨
	- `Omit<T, keyof U>`는 `U`와 겹치는 키를 `T`에서 제거
	- 최종적으로는 `T & U` 와 비슷하지만, **`U`가 우선**인 타입

```ts
export function combine<
  T extends object,                       // 초기 상태 (InitialState)
  U extends object,                       // 추가 상태 (AdditionalState)
  Mps extends [StoreMutatorIdentifier, unknown][] = [], // Middleware Pipe from previous
  Mcs extends [StoreMutatorIdentifier, unknown][] = [], // Middleware Compose for this layer
>(
  initialState: T,
  create: StateCreator<T, Mps, Mcs, U>,   // 상태 생성자 함수
): StateCreator<Write<T, U>, Mps, Mcs> {  // 병합된 타입의 상태 생성자 반환
  return (...args) => Object.assign({}, initialState, (create as any)(...args))
}
```

여기서 미들웨어 체이닝을 할 수 있다는 말이 처음에 좀 이해가 안됐는데, 설명해보면 이렇다.

```ts
type StateCreator<
  T,
  Mps extends MutatorIdentifier[] = [],
  Mcs extends MutatorIdentifier[] = [],
  U = T
> = (
  set: ..., 
  get: ..., 
  api: ...
) => U
```

- `StateCreator`는 위와 같은 시그니처를 가진 Zustand의 상태 생성자 함수 타입이다.
- 즉, `combine`은 다음과 같은 형태를 가진 상태 생성자를 기대함:
	- `create` 함수는 `set`, `get`, `api`를 받아서 `U` 타입의 상태 조각을 리턴
	- 이 상태 조각은 `initialState` (`T`)와 병합될 예정

- <span style="background:rgba(240, 200, 0, 0.2)">TypeScript의 강력한 제네릭 + 유틸리티 타입(`Omit`) + intersection을 조합해서 만들어진 고급 API다.</span>
	- 함수 시그니처를 보면 `combine`이 반환하는 건 `StateCreator<Write<T, U>, Mps, Mcs>`다. 이 덕분에 `create(...)` 함수 내부에서는:
		- `set`과 `get`이 `T & U`를 기준으로 타입이 잡힘
		- 사용자는 `initialState`와 `create`에서 정의한 상태를 모두 사용할 수 있고, 타입도 자동 추론됨

-  `Mps`, `Mcs`: 미들웨어 파이프를 추적하기 위한 타입 체이닝 시스템이다.
	- `StoreMutatorIdentifier`는 특정 미들웨어를 식별하기 위한 타입 태그로 쓰인다. 예를 들면 다음과 같은 튜플 형태로 사용된다.
```ts
// 체이닝 할 수 있는 미들웨어
['zustand/immer', unknown]
['zustand/persist', unknown]
...
```
```ts
type StoreMutatorIdentifier = string
// indentifier를 기반으로 정의한 타입
type MutatorIdentifier = [StoreMutatorIdentifier, any]
//전체 미들웨어 스택을 타입으로 표현하면
type MiddlewareStack = [ ['zustand/immer', any], ['zustand/persist', any] ]
``` 
- Zustand는 내부적으로 이걸 통해 “어떤 미들웨어가 적용됐는지”를 추적하고, 각 미들웨어가 `set`, `get`, `api`에 주입한 기능(예: `setImmer`)이 있는지를 타입 수준에서 안전하게 관리할 수 있다.

결론적으로 타입이 추론되는 구조를 정리해보면 이렇다.
- `create` 함수는 `StateCreator<T, Mps, Mcs, U>` 타입임
- 상태 조각 U를 만들어 반환하고
- 이를 initialState T와 병합해서 `Write<T, U>` 반환
- 미들웨어 체이닝 정보는 `Mps`, `Mcs`로 전달됨

Zustand의 타입 시스템은 이 `combine`을 활용할 때 초기 상태와 메서드를 명확히 구분하고 타입을 추론하기 때문에, 별도의 수동 타입 선언 없이도 IDE 자동완성과 타입 검사를 완벽히 지원하게 되는 것이다. 

createStore 할때 매번 타입을 정의해서 사용했던 나는 스터디원의 코드를 보고 매우 편리할 것 같다는 리뷰를 남겼었다. ㅎㅎ

![Screenshot 2025-04-14 at 12.03.39 AM.png](/img/user/Screenshot%202025-04-14%20at%2012.03.39%20AM.png)


---

기존에 나는 최대한 간결하게 서트파티 라이브러리 없이 구현하고자 했기 때문에 React Context를 사용해서 아래와 같이 구현했었다.

```ts
//기존 CartContext.ts
import { ReactNode, useState, createContext } from 'react';
import { Product } from '../products';

export interface CartContext {
	products: Product[];
	addProduct: (product: Product) => void;
	removeProduct: (product: Product) => void;
}

export const CartContext = createContext<CartContext>({
	products: [],
	addProduct: (product) => {},
	removeProduct: (product) => {},
});

export const CartProvider = ({ children }: { children: ReactNode }) => {

const [products, setProducts] = useState<Product[]>([]);

const addProduct = (product: Product) => {
	setProducts((prev) => [...prev, product]);
};

const removeProduct = (product: Product) => {
	const index = products.findIndex((p) => p.id === product.id);
	if (index !== -1) {
		setProducts((prev) => [
		...prev.slice(0, index),
		...prev.slice(index + 1),
		]);
	}
};

	return ( <CartContext.Provider value={{ products, addProduct, removeProduct }}>{children}</CartContext.Provider>
	);

};
```

이 코드는 다음과 같은 단점이 발생할 가능성이 있다:

- `useState` 기반이라 비즈니스 로직이 `CartProvider` 내부에 갇힘
- `setProducts`가 여러 메서드에서 반복 사용됨
- 비동기 로직 추가 시 훨씬 복잡해짐
- 테스트/디버깅/분리 어려움

Zustand와 combine 미들웨어를 사용해 다시 구현해보면 이렇게 된다.

```js
// 리팩토링해본 cartStore.ts
import { create } from 'zustand';
import { combine } from 'zustand/middleware';
import { Product } from '../products';

export const useCartStore = create(
  combine({ products: [] as Product[] }, (set, get) => ({
    addProduct: (product: Product) => {
      set({ products: [...get().products, product] });
    },
    removeProduct: (product: Product) => {
      const index = get().products.findIndex((p) => p.id === product.id);
      if (index !== -1) {
        const newProducts = [...get().products];
        newProducts.splice(index, 1);
        set({ products: newProducts });
      }
    },
  }))
);
```

Zustand의 `combine` 미들웨어를 활용하면, 상태와 로직을 분리하면서도 타입 안정성을 유지할 수 있다. 특히 `React Context`로 관리하던 전역 상태를 리팩토링할 때 매우 강력한 선택지다.

Zustand가 제공하는 `persist`, `immer`, `devtools` 등과 조합하면 더 확장성 있는 상태 관리를 할 수 있다는 장점도 갖는다.

