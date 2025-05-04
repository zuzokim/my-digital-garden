---
{"dg-publish":true,"permalink":"/posts/hook/","tags":["React"],"created":"2025-05-04","updated":"2025-05-04T22:30:00"}
---

최근에 React 기본기를 아주 크게 놓친 코드를 작성했던 일이 있다. 처음엔 음.. 뭐가 문제지 싶었는데, 아주아주 치명적인 실수여서 기록해본다. 역시 모든건 기본이 중요하다. ㅠㅠ

```ts

const [data, setData] =
  STORAGE === "local-storage"
    ? useLocalStorage(...)
    : useState(...);

```

맨 처음에 내가 작성했던 코드는 이런 식이다. 

요구사항은 로컬스토리지에 저장하겠다는 변수가 있다면 로컬스토리지에 저장하고 아니라면 in-memory 에 저장한다는 것이었다. 뭐가 문제인지 바로 알아챘다면 당신은 React 기본기 충실맨이므로, 박수 받아 마땅하다.

하지만 나는 처음에 음 삼항 연산자 때문에 좀 가독성이 떨어지나? 하는 정도만 걱정했지 이 방식의 치명적인 문제가 있는 줄 인지하지 못했었다. 제일 처음 React 배울 때 다루던 React 공식문서에도 명시된 사용 금지 패턴이다.... (럴수..ㅜ)


> Hooks must be called unconditionally at the top level of your component.
https://react.dev/reference/rules/rules-of-hooks#only-call-hooks-at-the-top-level

- **문제점**: `useLocalStorage`도 훅이고, `useState`도 훅이다. 그런데 `STORAGE` 값에 따라 조건부로 훅을 다르게 호출하고 있다.
- **왜 안티패턴인가?**
    - 리액트 훅은 **컴포넌트가 실행될 때 항상 동일한 순서로 호출**되어야 한다.
    - 조건문 안에서 훅을 호출하면, 리렌더링 중에 훅 순서가 꼬일 수 있어서 **예측 불가능한 버그**가 발생한다.

평소에 [`eslint-plugin-react-hooks`](https://www.npmjs.com/package/eslint-plugin-react-hooks) 린트를 사용하는데 무엇때문인지 IDE에서도 잡지 못해서 이런 치명적인 안티패턴을 만들게 됐다. 

그럼 어떻게 작성해야 될까?

```ts

function useDataStorage(
  useLocal: boolean,
  key: string,
  defaultValue: (Data & { key: number })[]
) {
  const [localValue, setLocalValue] = useLocalStorage<typeof defaultValue>(key, defaultValue);
  const [fallbackValue, setFallbackValue] = useState<typeof defaultValue>(defaultValue);

  return useLocal
    ? [localValue, setLocalValue] as const
    : [fallbackValue, setFallbackValue] as const;
}

...

const [data, setData] = useDataStorage<(Data & { key: number })[]>(
  STORAGE === "local-storage",
  STORAGE_KEY,
  [...DEFAULT_DATA]
);


```

이런식으로 리팩토링해봤다. 이렇게 하면 훅이 항상 **정해진 순서로 한 번만 호출**되므로 리액트 규칙을 지키면서도 조건에 따라 동작을 다르게 할 수 있다.

### 핵심

핵심은 "**훅의 호출 순서는 항상 동일해야 한다**"는 React의 규칙 때문인데, enabled의 값에 따라 실행 순서가 달라지는 게 아니라, **호출 여부 자체가 달라지는 것**이 문제다.

React는 훅의 호출 순서를 컴파일러나 런타임에서 추적해서 상태를 유지한다.  그런데 조건에 따라 어떤 훅은 호출되고, 어떤 훅은 아예 호출되지 않으면 내부적으로 상태가 꼬인다.

또다른 예시

```ts

function MyComponent({ enabled }) {
  if (enabled) {
    useEffect(() => {
      console.log("enabled!");
    }, []);
  }
  useState(0);
}

```

이런 경우는 "아, if 조건문 안에서 useEffect를 호출하면 안되지" 하고 바로 알아차렸을텐데, 평소에 이에 관해 신중하게 생각을 못했던게 아닌가 싶다.

- `enabled`가 `true`일 땐 `useEffect` → `useState` 순으로 훅이 두 개 호출
- `enabled`가 `false`일 땐 `useEffect`는 호출되지 않고, `useState`만 호출

훅 호출 개수와 순서가 달라져서 React가 어떤 상태가 어떤 훅에 해당하는지 헷갈려 하게 되는 것.

#### 결론

**훅은 호출 여부가 조건에 따라 달라지면 안 되고, 호출은 항상 이뤄져야 한다.** 조건은 그 내부에서 처리해야 하도록 한다.
- 훅(`useState`, `useEffect`, `useMemo`, `useCustomHook` 등)은 **컴포넌트가 렌더링될 때 항상 호출돼야 한다**.
