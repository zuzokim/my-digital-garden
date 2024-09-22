---
{"dg-publish":true,"permalink":"/pages/element/","created":"2024-09-22","updated":"2024-09-22T23:50:00"}
---

![Screen Shot 2024-09-22 at 10.08.25 PM.png|300](/img/user/Screen%20Shot%202024-09-22%20at%2010.08.25%20PM.png) 

(https://images.app.goo.gl/fz4Vruc1TpxqwTCt5 이미지 출처)
![Screen Shot 2024-09-22 at 10.19.43 PM.png|300](/img/user/Screen%20Shot%202024-09-22%20at%2010.19.43%20PM.png)
(https://kr.pinterest.com/pin/602567625135639636/ 이미지 출처)
회사에서 이런 선긋기 학습활동을 온라인에서 할 수 있도록 개발하고 있습니다. 새로운 학습활동인만큼 본격 개발 이전에 선을 그리고 선으로 이어진 두 카드가 매칭되는지 채점이 가능한 형태로 구현이 가능한지 이틀간 poc를 진행해봤습니다.
요구사항은 아래와 같습니다.

1. 일단 위 그림처럼 짝이 맞는 두 그림카드를 클릭하면 선이 그려지는 UI 구현이 필요했습니다. 이때 각 그림 카드의 크기, 위치가 동적으로 결정되기 때문에 (유저가 어떤 크기로 어떤 순서로 그림카드를 입력할지 모르고, 반응형도 고려해야하기 때문에) 각 그림카드의 상대적인 위치를 찾고 그 둘을 상하 좌우 뿐 아니라 대각선으로도 이어줄 수 있어야 합니다. 

2. 위 그림처럼 3개의 그림카드를 이어주기도 하지만 우선 MVP로는 2개의 짝을 맞추는 정도까지만 구현하기로 했습니다. 그러면 시작카드 - 끝카드를 클릭하면 하나의 쌍이되고, 그 다음 시작카드 - 끝카드를 클릭하면 또 다른 쌍이 되어 선긋기가 되어야 합니다.

3. 2개의 카드를 선으로 이은 상태로 같은 카드를 클릭하면 선이 지워져야합니다. 

1번 구현을 위해서는 각 카드의 중앙지점을 시작과 끝지점으로 선을 그려줄 수 있어야 합니다.
1. 각 카드 엘리먼트의 중앙지점 찾기
	1. 왼쪽에 배치되는 시작카드의 우측 중앙지점 찾기
	2. 우측에 배치되는 끝카드의 좌측 중앙지점 찾기
2. 시작점과 끝점을 선으로 이어주기
	1. canvas 2d stroke으로 직접 그려주기
	2. svg line 엘리먼트를 그려주기


먼저 각 카드의 중앙지점 찾기를 해볼까요. 수도코드로 작성을 해보겠습니다.

```js
const handleStart = (e: MouseEvent<HTMLElement>) => {
	const target = e.target as HTMLElement;
	const rect = target.getBoundingClientRect();

	const newStartPos: Anchor = {
		x: rect.x,
		y: rect.y,
	};
}

...

jsx
//왼쪽에 배치되는 시작카드
<div onClick={handleStart}>a</div>
<div onClick={handleStart}>b</div>

//우측에 배치되는 끝카드
<div onClick={handleEnd}>ㄱ</div>
<div onClick={handleEnd}>ㄴ</div>

```

우선 div element로 만들어진 카드에 클릭핸들러를 부착합니다. 그리고 parameter로 넘겨받은 event 객체의 target에 접근합니다. 위 수도코드는 실제로는 react 컴포넌트로 작성되었기 때문에 e.target이 HTMLElement 타입임을 as로 알려주어야 합니다. 타입적으로 react의 MouseEvent 제네릭이 target이 항상 HTMLElement임을 보장할 수 없기 때문입니다.

target 그러니까 element에 직접 접근이 되면 [getBoundingClientRect()](https://developer.mozilla.org/ko/docs/Web/API/Element/getBoundingClientRect) 메서드를 사용할 수 있는데, 이 메서드가 바로 element의 크기 정보를 담고있는 DOMRect 객체를 반환해줍니다. 

![Screen Shot 2024-09-22 at 10.44.59 PM.png|300](/img/user/Screen%20Shot%202024-09-22%20at%2010.44.59%20PM.png)

위에서 작성한 수도코드에서 rect는 DOMRect 객체를 말하며 여기서 rect.x, rect.y 값으로 element의 좌측상단 시작 좌표에 접근할 수 있습니다. 그런데 시작카드는 좌측에 배치되고, 끝카드는 우측에 배치되어야하고 구현하고자하는 시작과 끝지점은 시작카드는 우측중앙, 끝카드는 좌측중앙입니다. 그래야 두 카드 사이를 자연스럽게 선으로 이어줄 수 있으니까요.

시작카드 element들의 오른쪽은 x좌표로부터 element의 width만큼을 더해주면 됩니다. 중앙은 y좌표에 element의 height / 2 만큼을 더해주면 되겠네요. 그러려면 element의 위치좌표는 구했으니 element의 크기인 width, height도 알아올 수 있어야 합니다. 크기정보는 rect.width, rect.height 혹은 [clientWidth](https://developer.mozilla.org/en-US/docs/Web/API/Element/clientWidth) 로도 바로 가져올 수 있습니다. 그렇게하면 아래와 같이 코드를 수정할 수 있습니다. 끝카드는 어차피 왼쪽중앙에 좌표를 찍어야하니 width는 더해주지 않아도 되겠네요.

```js
const handleStart = (e: MouseEvent<HTMLElement>) => {
	const target = e.target as HTMLElement;
	const rect = target.getBoundingClientRect();

	const newStartPos: Anchor = {
		x: rect.x + rect.width or target.clientWidth,
		y: rect.y - rect.height or target.clientHeight / 2,
	};
}

const handleEnd = (e: MouseEvent<HTMLElement>) => {
	const target = e.target as HTMLElement;
	const rect = target.getBoundingClientRect();

	const newStartPos: Anchor = {
		x: rect.x,
		y: rect.y - rect.height or target.clientHeight / 2,
	};
}

...

jsx
//왼쪽에 배치되는 시작카드
<div onClick={handleStart}>a</div>
<div onClick={handleStart}>b</div>

//우측에 배치되는 끝카드
<div onClick={handleEnd}>ㄱ</div>
<div onClick={handleEnd}>ㄴ</div>

```


이제 좌표를 구했으니 두 좌표를 잇는 선을 그려보겠습니다. 
canvas로 stroke을 그려줘도 되고 svg 엘리먼트를 렌더해도 되는데, 저는 그리는 선의 갯수만큼 svg를 렌더하는 방법을 선택해봤습니다.

아이디어는 카드들의 뒤쪽 레이어에 svg를 그려줄 컨테이너를 겹쳐그리는 것입니다. 아래처럼 카드들의 컨테이너와 똑같은 사이즈의 svg 컨테이너 레이어를 뒤쪽에 깔고, 거기에 2개의 좌표를 이용해 선을 그리를 방식입니다.
![Screen Shot 2024-09-22 at 11.16.48 PM.png|300](/img/user/Screen%20Shot%202024-09-22%20at%2011.16.48%20PM.png)

root 스타일로 position: relative를, svg는 position: absolute을 주어서 겹치게한 뒤, svg가 카드 클릭을 막지 않도록 z-index를 -1로 낮춰줍니다. 그러면 카드의 실제 x,y좌표와 뒤쪽 레이어의 svg line이 시각적으로 동일한 x,y좌표에 그려질 수 있습니다. 
```js

<div 
	style={{
		padding: '20px',
		margin: '100px 0',
		width: '100%',
		height: '100%',
		display: 'flex',
		justifyContent: 'space-between',
		position: 'relative',
	}}
>
	<svg
		ref={svgRef}
		style={{
		display: 'block',
		position: 'absolute',
		top: 0,
		left: 0,
		height: '100%',
		width: '100%',
		zIndex: -1,
		}}
>	
	{lines.map((line, index) => {
		if (!line?.start || !line?.end) {
			return null;
		}
		return (
		<line
			key={index}
			x1={line?.start?.x}
			y1={line?.start?.y}
			x2={line?.end?.x}
			y2={line?.end?.y}
			stroke="green"
			strokeWidth="2"
		/>);
	})}
	</svg>
	
	<div>
		//왼쪽에 배치되는 시작카드
		<div onClick={handleStart}>a</div>
		<div onClick={handleStart}>b</div>
		<div onClick={handleStart}c</div>
	</div>
	<div>
		//우측에 배치되는 끝카드
		<div onClick={handleEnd}>ㄱ</div>
		<div onClick={handleEnd}>ㄴ</div>
		<div onClick={handleEnd}>ㄷ</div>
	</div>

</div>
```

![Screen Shot 2024-09-22 at 11.28.57 PM.png|100%](/img/user/Screen%20Shot%202024-09-22%20at%2011.28.57%20PM.png)

poc해본 결과물입니다. 두개의 좌표(시작점x,y, 끝점x,y)를 담은 line정보는 state로 관리하고 한 쌍이 될 때마다 둘 사이를 이어줄 수 있습니다. svg [line](https://developer.mozilla.org/en-US/docs/Web/SVG/Element/line) 엘리먼트의 attribute로 x1,y1,x2,y2를 line state 데이터로 채워줄 수 있습니다. 