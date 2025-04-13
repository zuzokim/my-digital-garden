---
{"dg-publish":true,"permalink":"/posts/zustand-combine/","created":"2025-04-13","updated":"2025-04-13T15:08:00"}
---

최근 참여하고 있는 프론트엔드 코드리뷰 스터디에서 간단한 토이프로젝트를 진행하고 있다. 장바구니 기능이 있는 이커머스 페이지에서 상태관리에 초점을 맞추어 구현하고 서로의 코드를 리뷰해주는 시간을 갖는다. 그 과정에서 전역적으로 상태를 관리해야할 때 많은 스터디원들이 zustand를 사용했는데, 나도 회사에서 zustand를 애용하고 있던 와중 스터디원이 소개해준 덕에 새로운 미들웨어를 알게 돼서 알게 된 내용을 정리해본다. 인터널 소스코드도 Typescript를 활용한 굉장히 짧은 코드여서 어떤식으로 스토어의 타입을 추론하는지 뜯어보았다.

---

## `combine` 미들웨어의 타입 추론 방식

Zustand의 `combine` 미들웨어는 아래와 같은 형태로 사용된다:

```
combine<T, U>(initialState: T, create: (set, get) => U): T & U
```

여기서 중요한 타입 유틸이 하나 있다:

```
type Write<T, U> = Omit<T, keyof U> & U
```

이 타입은 `T`와 `U`의 충돌되는 키를 제거하고, `U`로 덮어씌우는 역할을 한다. 결국 최종적으로 반환되는 상태는 `T & U` 타입이지만, `U`의 메서드가 우선시된다.

`combine` 함수 자체 구현은 단순하다:

```
export function combine<T, U>(
  initialState: T,
  create: (set, get) => U
): () => T & U {
  return (set, get, api) => Object.assign({}, initialState, create(set, get, api))
}
```

Zustand의 타입 시스템은 이 `combine`을 활용할 때 초기 상태와 메서드를 명확히 구분하고 타입을 추론하기 때문에, 별도의 수동 타입 선언 없이도 IDE 자동완성과 타입 검사를 완벽히 지원한다.


기존에 나는 최대한 간결하게 서트파티 라이브러리 없이 구현하고자 했기 때문에 React Context를 사용해서 아래와 같이 구현했었다.

```js
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