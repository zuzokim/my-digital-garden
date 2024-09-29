---
{"dg-publish":true,"permalink":"/pages/pointer-events-element/","tags":["CSS","svg"],"created":"2024-09-29","updated":"2024-09-29T19:43:00"}
---

지난 포스트 [[Pages/element의 중앙을 찾아 선긋기\|element의 중앙을 찾아 선긋기]] 에서 svg 레이어를 보기카드 레이어보다 낮은 z-index로 겹쳐 보기카드와 선이 그려지는 위치를 일치시키는 방식으로 '선긋기'를 완성했었습니다. 그런데 만약 svg로 그려진 선을 직접 클릭해서 이어진 두 카드의 연결을 끊는 인터랙션을 구현하려면 어떻게 할 수 있을까요? 

기본적으로 모든 elements는 z-index상 가장 위에 배치된 element만 클릭이벤트를 호출할 수 있습니다. 지금까지의 구현으로는 가장 위에 배치된 보기카드만 클릭이 가능하고, 뒤 쪽에 그려진 svg line들은 클릭이 어렵습니다. 또, x자로 line들이 겹쳐그려진 경우 각각의 line들도 겹쳐진 상태이기 때문에 개별적으로 클릭이 어렵습니다.

정리하면 아래 두 가지 태스크를 해결해야 합니다.
1. 보기카드 레이어보다 낮은 z-index의 svg 레이어를 클릭 가능하게 하기
2. svg 레이어에 그려질 line들을 개별적으로 클릭 가능하게 하기

#### 1. 낮은 z-index elements를 클릭 가능하게 하기

![layers.png|400](/img/user/layers.png)

<span style="background:rgba(5, 117, 197, 0.2)"><font color="#245bdb">파란색 보기카드 레이어</font></span> 보다 뒤에 있는 <span style="background:rgba(140, 140, 140, 0.12)"><font color="#646a73">회색 svg 선 레이어</font></span> 를 클릭할 수 있게 해보겠습니다. 앞에 가로막힌 <span style="background:rgba(5, 117, 197, 0.2)"><font color="#245bdb">파란색 보기카드 레이어</font></span> 에 `pointer-events: none` css 속성을 주면 해당 레이어를 통과해 클릭 이벤트를 전달할 수 있습니다. 아래 수도코드로 작성해보겠습니다.

```html
<div 
	style={{
		position: 'relative';	  
	}}
>
	//svg 선 레이어
	<svg
		style={{
			position: 'absolute',
			top: 0,
			left: 0,
			height: '100%',
			width: '100%',
			z-index: -1,
		}}
	>
		<line />
		<line />
	</svg>
	
	//보기카드 레이어
	<div
		style={{
			pointer-events: 'none'
		}}
	>
		
		//좌측에 배치되는 시작카드
		<div onClick={handleStart}>a</div>
		<div onClick={handleStart}>b</div>

		//우측에 배치되는 끝카드
		<div onClick={handleEnd}>ㄱ</div>
		<div onClick={handleEnd}>ㄴ</div>
	<div>

</div>
```

이렇게 하면 <span style="background:rgba(5, 117, 197, 0.2)"><font color="#245bdb">파란색 보기카드 레이어</font></span>의 클릭 이벤트는 비활성화하고 아래에 있는 <span style="background:rgba(140, 140, 140, 0.12)"><font color="#646a73">회색 svg 선 레이어</font></span>를 클릭할 수 있게 됩니다. [pointer-events](https://developer.mozilla.org/en-US/docs/Web/CSS/pointer-events)는 클릭 뿐만 아니라 터치, 드래그 등 포인터 장치로 발생시킬 수 있는 모든 이벤트를 컨트롤할 수 있는 속성입니다. 

한 번에 모든게 해결되면 좋겠지만, <span style="background:rgba(140, 140, 140, 0.12)"><font color="#646a73">회색 svg 선 레이어</font></span>를 클릭할 수 있게 된 대신 보기카드들을 감싸고 있는 <span style="background:rgba(5, 117, 197, 0.2)"><font color="#245bdb">파란색 보기카드 레이어</font></span> 의 모든 클릭 이벤트가 비활성화되고나니 children인 보기카드들(시작카드, 끝카드)도 클릭이 막혀버립니다.

parent의 `pointer-events`만 비활성화하고 children의 이벤트는 호출하고 싶다면 children에 초깃값인 `pointer-events: auto`를 주어 속성을 상속받지 않도록 리셋해줄 수 있습니다. 실제 회사 프로덕션 코드에서는 아래 수도코드보다는 마크업 tree가 복잡해서 좀 더 많은 elements를 리셋해주어야 했네요. 

```html

//보기카드 레이어
<div
	style={{
		pointer-events: 'none'
	}}
>
	//좌측에 배치되는 시작카드
	<div 
		onClick={handleStart} 		
		style={{
		pointer-events: 'auto';
		}}
	>
	a
	</div>
	...생략
<div>
```
#### 2. svg line들을 개별적으로 클릭 가능하게 하기

자 그럼 이제 첫 번째 태스크는 해결을 했으니 두 번째 태스크를 해결해볼까요. 유저가 line을 잇는 순서대로 마크업tree가 만들어질 겁니다. 즉, 3번->2번->1번 순서대로 선을 이었다면 아래와 같이 elements가 만들어집니다. 

![svglines.png|400](/img/user/svglines.png)

```html
<svg>
	<lines /> 3번
	<lines /> 2번
	<lines /> 1번
	...
</svg>

```
첫 번째 태스크의 경우와 다르게 z-index 조절을 별도로 하지 않았더라도 마크업의 순서상 가장 위에 쌓일 <span style="background-color: #FF7002"><font color="#fff">1번 line</font></span>에만 클릭이벤트가 호출될 것입니다.

![Screen Shot 2024-09-29 at 6.38.01 PM.png|200](/img/user/Screen%20Shot%202024-09-29%20at%206.38.01%20PM.png)

그럼 뒤쪽에 있는 <span style="background:#d3f8b6">2번 line</span> 과 <span style="background:#fdbfff">3번 line</span> 도 클릭 가능하려면 어떻게 해야할까요? 매번 자신의 위쪽에 오버랩된 element에 `pointer-events: none`을 줄 수도 없고, 설령 가능하다 하더라도 겹쳐진 좌표상 마우스가 어느 element를 타겟팅하는지 구분할 수도 없습니다. 또, 정확하게 element를 클릭하려면 점선으로 그려진 line element의 bounding-box가 아니라 실제로 그려진 <font color="#c00000">빨간색 line stroke</font> 영역을 구분해서 타겟팅할 수 있어야 합니다. 유저는 투명한 bounding-box는 클릭이 가능한 영역이라고 기대하지 않을테니까요.

`pointer-events` 속성의 다른 값을 사용해서 해결할 수 있습니다. 아래와 같은 svg element에 적용할 수 있는 여러 값 중 하나를 사용할 수 있습니다.[^pointer-events-syntax]
`
```css
/* Values used in SVGs */
pointer-events: visiblePainted;
pointer-events: visibleFill;
pointer-events: visibleStroke;
pointer-events: visible;
pointer-events: painted;
pointer-events: fill;
pointer-events: stroke;
pointer-events: bounding-box;
pointer-events: all;

````

`pointer-events: stroke;`을 사용하면 stroke attribute로 그린 line, 즉 정확히 선이 그어진 <font color="#c00000">빨간색 stroke영역</font>에서만 클릭 이벤트를 호출할 수 있습니다. 필요하다면 `visible` 이나 `visibleStroke` 로 element의 [visibility](https://developer.mozilla.org/en-US/docs/Web/CSS/visibility)
 상태에 따라 조건을 넣어줄 수도 있겠습니다.
```html
<line
	style={{pointer-events: 'stroke'}}
	x1={line?.start?.x}
	y1={line?.start?.y}
	x2={line?.end?.x}
	y2={line?.end?.y}
	stroke="red"
	strokeWidth="2"
/>
```

가장 위쪽에 그려진 선이나 겹치지 않은 선만 클릭이 가능했는데 이제 z-index와 무관하게, 그리고 x자로 겹쳐진 line 사이 좁은 영역을 타겟팅하더라도 원하는 선을 정확하게 클릭할 수 있게 되었습니다. 


#### 클릭 사용성 개선
추가로 조금 더 사용성을 높이기 위해 UI를 디자인해준 디자이너에게 제안한 내용과 구현하면서 더 알게 된 내용도 붙여보겠습니다.

line stroke가 2px 이다보니 마우스 커서를 2px 영역 안으로 매우 정확히 가져다놓고 클릭해야만 클릭 이벤트가 호출되고 Remove 툴팁을 띄워줄 수 있습니다. 작은 화면 유저나 저시력자, 마우스 사용이 미숙한 어린 학생 유저에게는 다소 어려운 사용성이 될겁니다. 클릭영역을 line 좌우로 더 넓히는 아이디어로 이를 보완해보겠습니다. 

line element 2개를 겹쳐서 하나는 두꺼운 stroke를 이용해 클릭영역으로 만들고, 그 중앙에 시각적으로 보이는 기존의 line을 그려주는 방식입니다. 

```html
//클릭영역
<line
	style={{pointer-events: 'stroke'}}
	x1={line?.start?.x}
	y1={line?.start?.y}
	x2={line?.end?.x}
	y2={line?.end?.y}
	stroke="red"
	strokeWidth="2"
/>
//실제 선
<line
	style={{pointer-events: 'stroke'}}
	x1={line?.start?.x}
	y1={line?.start?.y}
	x2={line?.end?.x}
	y2={line?.end?.y}
	stroke="orange"
	strokeWidth="12"
/>

```
![Screen Shot 2024-09-29 at 7.29.49 PM.png|200](/img/user/Screen%20Shot%202024-09-29%20at%207.29.49%20PM.png)

이때 <span style="background-color: #FF7002"><font color="#fff">클릭 영역</font></span>이 <span style="background:#ff4d4f"><font color="#fff">실제 선</font></span>보다 뒤쪽에 그려져야 했는데 svg에는 z-index를 사용할 수 없다는 걸 알게 됐습니다. 그럼 어떻게 해야할까요? 단순합니다. 마크업의 기본 규칙대로 element의 마크업 순서를 바꿔주면 됩니다.

```html
//클릭영역
<line
	style={{pointer-events: 'stroke'}}
	x1={line?.start?.x}
	y1={line?.start?.y}
	x2={line?.end?.x}
	y2={line?.end?.y}
	stroke="orange"
	strokeWidth="12"
/>
 //실제 선
<line
	style={{pointer-events: 'stroke'}}
	x1={line?.start?.x}
	y1={line?.start?.y}
	x2={line?.end?.x}
	y2={line?.end?.y}
	stroke="red"
	
	strokeWidth="2"
/>

```

svg의 [렌더링 순서 스펙](https://www.w3.org/TR/SVG11/render.html#RenderingOrder)을 좀 더 자세히 보면 다음과 같습니다.
*"svg elements는 암시적인 그리기 순서가 있으며, 첫 번째 element 가 먼저 페인트 됩니다. 후속 elements는 이전에 그려진 element 위에 그려집니다."*




[^pointer-events-syntax]:https://developer.mozilla.org/en-US/docs/Web/CSS/pointer-events#syntax