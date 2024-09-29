---
{"dg-publish":true,"permalink":"/pages/pointer-events-element/","tags":["CSS"],"created":"2024-09-29","updated":"2024-09-29T16:19:00"}
---

지난 포스트 [[Pages/element의 중앙을 찾아 선긋기\|element의 중앙을 찾아 선긋기]] 에서 svg 레이어를 보기카드 레이어보다 낮은 z-index로 겹쳐 그려 보기카드의 위치와 선이 그려지는 위치를 일치시키는 방식으로 '선긋기'를 완성했었습니다. 그런데 만약 svg로 그려진 선을 직접 클릭해서 이어진 두 카드의 연결을 끊는 인터랙션을 만들기 위해서는 어떻게 할 수 있을까요? 

기본적으로 모든 elements는 z-index상 가장 위에 배치된 element만 클릭이벤트를 호출할 수 있습니다. 지금까지의 구현으로는 가장 위에 배치된 보기카드만 클릭이 가능하고, 뒤 쪽에 그려진 svg line들은 클릭이 어렵습니다. 또, x자로 line들이 겹쳐그려진 경우 line들도 겹쳐진 상태이기 때문에 개별적으로 클릭이 어렵습니다.

정리하면 아래 두 가지 태스크를 해결해야 합니다.
1. 보기카드 레이어보다 낮은 z-index의 svg 레이어를 클릭 가능하게 하기
2. svg 레이어에 그려질 line들을 개별적으로 클릭 가능하게 하기

##### 1. 낮은 z-index elements를 클릭 가능하게 하기

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

한 번에 모든게 해결되면 좋겠지만, <span style="background:rgba(140, 140, 140, 0.12)"><font color="#646a73">회색 svg 선 레이어</font></span>를 클릭할 수 있게 된 대신 보기카드들을 감싸고 있는 <span style="background:rgba(5, 117, 197, 0.2)"><font color="#245bdb">파란색 보기카드 레이어</font></span> 의 모든 클릭 이벤트가 비활성화되고나니 children인 보기카드들도 클릭이 막혀버립니다.

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
		//// ** reset to be clickable*/
		pointer-events: 'auto'
	}}>a</div>
	...생략
<div>
```
##### 2. svg line들을 개별적으로 클릭 가능하게 하기

![svglines.png|400](/img/user/svglines.png)

자 그럼 이제 첫 번째 태스크는 해결을 했으니 두 번째 태스크를 해결해볼까요. 유저가 line을 잇는 순서대로 마크업tree가 만들어질 겁니다. 즉, 3번->2번->1번 순서대로 선을 이었다면 아래와 같이 elements가 만들어집니다. 
```html
<svg>
	<lines /> 3번
	<lines /> 2번
	<lines /> 1번
	...
</svg>

```
첫 번째 태스크의 경우와 다르게 z-index 조절을 별도로 하지 않았더라도 마크업의 순서상 가장 위에 쌓일 <span style="background:rgba(255, 183, 139, 0.55)"><font color="#d83931">1번 line</font></span>에만 클릭이벤트가 호출될 것입니다.

![Screen Shot 2024-09-29 at 6.38.01 PM.png|200](/img/user/Screen%20Shot%202024-09-29%20at%206.38.01%20PM.png)

