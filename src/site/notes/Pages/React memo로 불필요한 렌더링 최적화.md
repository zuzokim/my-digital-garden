---
{"dg-publish":true,"permalink":"/pages/react-memo/","created":"2024-10-20","updated":"2024-10-20T20:49:00"}
---

React 처음 배울때 제일 먼저 배우는 것
1. state가 변경될때마다 리렌더링된다.
2. props가 변경될때마다 리렌더링된다.
3. 부모 컴포넌트가 리렌더링되면 자식 컴포넌트도 리렌더링된다.
4. context value가 변경되면 context를 사용하는 컴포넌트가 리렌더링된다.
5. forceUpdate 메서드를 호출하면 강제로 리렌더링된다.

회사에서 excalidraw를 fork해서 셀프호스팅해 화이트보드 기능을 개발하고 있습니다.
excalidraw의 public api로는 initialData로 넘긴 elements를 canvas위에 렌더해주는 방식이나, 정적인 학습 컨텐츠를 canvas 뒤쪽에 깔고 그 위에 인터랙티브한 필기를 할 수 있게 제공하는 것이 요구사항이었기 때문에 별도의 content prop을 열어 excalidraw 내부에서 static, interactive canvas 뒤쪽에 배치되는 자식 컴포넌트로 넘겨주는 방식으로 커스텀을 해서 사용하고 있습니다.

그러나 excalidraw의 특성상 appState와 elements가 변경되면 전체 excalidraw app 컴포넌트가 렌더됩니다. 그럼 위에서 설명한 '부모 컴포넌트가 리렌더링되면 자식 컴포넌트도 리렌더링 된다' 는 조건에 따라 content prop으로 넘긴 자식 학습 컨텐츠 컴포넌트도 리렌더됩니다. (내부에서 static canvas는 리렌더를 방지하거나 좀더 정교하게 렌더링 성능을 최적화해둔 부분이 있는 걸로 아는데 코드를 자세히 뜯어보진 못했습니다.) 

최근에 화이트보드 컴포넌트 안에 리치에디터를 활용한 학습 콘텐츠 컴포넌트를 자식 컴포넌트로 그리고 또 다시 그 학습 콘텐츠 컴포넌트가 화이트보드 컴포넌트를 자식으로 가지고 있는 다소 복잡한 구조의 기능을 개발했습니다.

이때, excalidraw content prop으로 넘긴 '화이트보드를 품은 리치에디터 컴포넌트' 자체가 너무 무거운 컴포넌트가 되어버려 화이트보드 canvas에 조금만 선을 그리더라도 onUpdate가 호출되면서 content 컴포넌트가 깜빡깜빡거리는 리렌더링 현상이 생겨났습니다. 

사실상 화이트보드에 선을 그릴때 canvas 뒤쪽에 깔린 '화이트보드를 품은 리치에디터 컴포넌트'는 정적인 콘텐츠이므로, 이 컴포넌트의 state, prop 그 어떤 것도 변경될 가능성이 없습니다. (유저의 인터랙션도 발생하지 않는 readonly 상태)

따라서 부모인 화이트보드 컴포넌트의 state가 excalidraw 내부에서 실시간으로 변경되어도 자식 컴포넌트인 content -  '화이트보드를 품은 리치에디터 컴포넌트'는 리렌더될 필요가 없습니다. 불필요한 리렌더를 방지하기 위해 React.memo()로 content 컴포넌트를 감싸서 prop으로 넘겨주었더니 깜빡임 현상이 해소되었습니다.

그럼 React.memo()는 어떤 역할을 해준 걸까요?
React.memo()를 사용하면 
1. 부모컴포넌트가 리렌더되더라도 자식 컴포넌트가 불필요하게 리렌더링되지 않도록 컴포넌트를 메모이제이션해줍니다.
2. 컴포넌트의 props가 변경되지 않는한 리렌더되지 않게 해줍니다.


결국
1. excalidraw 화이트보드 컴포넌트에 선을 그려 리렌더되어도 content prop으로 넘겨준 '화이트보드를 품은 리치에디터' 자식 컴포넌트는 불필요하게 리렌더되지 않습니다.
2. '화이트보드를 품은 리치에디터'의 props는 canvas뒤쪽에 배치되어 유저 인터랙션이 발생하지 않는 정적인 readonly 콘텐츠이므로 렌더되지 않습니다.


```js

import {memo} from 'react';

const MemoizedRichEditor = (content) => {
	return <RichEditor content={content} />
}

export default memo(MemoizedRichEditor);

```

```js

import MemoizedRichEditor from './MemoizedRichEditor';

const RichEditor = (content) => {

	return <Whiteboard
				content={<MemoizedRichEditor content={content} />}
			/>
}
```