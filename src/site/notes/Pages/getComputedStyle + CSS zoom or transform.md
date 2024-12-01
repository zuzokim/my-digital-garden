---
{"dg-publish":true,"permalink":"/pages/get-computed-style-css-zoom-or-transform/","tags":["CSS","WebAPI"],"created":"2024-12-01","updated":"2024-12-01T22:37:00"}
---

최근 엘리먼트의 포지션을 제대로 잡지 못해 발생하는 이슈 2가지를 만났습니다. 하나는 회사 프로덕션에서, 다른 하나는 애니메이션을 적용한 개인 퍼블리싱 프로젝트에서 엘리먼트의 포지션이 어긋나 화면에서 보이지 않은 이슈였습니다.

1. 회사 프로덕션 : 화면 확대/축소 배율 조정 기능이 들어간 화면에서만 툴팁 컴포넌트의 x,y 포지션을 못잡은 이슈
2. 개인 프로젝트 : 랜덤으로 부유하는 keyframe 애니메이션을 적용한 스티커 컴포넌트를 클릭하면 부유하던 위치에 고정되어 애니메이션이 일시정지해야하는데, 클릭과 동시에 화면 밖으로 사라지는 이슈

요구사항과 증상은 달랐지만 근본적으로는 엘리먼트를 위치시켜야할 정확한 x,y좌표를 잃어버리는 것이 원인이었습니다.

1 . 
화면 확대/축소 배율 조정 기능이 들어간 화면에서는 최상위 스크린 컴포넌트에 css property로 zoom을 적용해주고 있었고, css variables로 zoom의 값을 js로 제어하고 있었습니다. 따라서 유저가 zoom 레벨을 조정할 때마다 새로 갱신되는 zoom 값이 달라지며, 그에 따라 하위 자식 컴포넌트와 엘리먼트가 브라우저에 그려질 때의 실제 포지션 값 x,y도 달라졌습니다. 

2 .
랜덤으로 부유하는 keyframe 애니메이션이 적용된 스티커 컴포넌트를 클릭해 그 자리에 정지시키려면 클릭할 당시의 포지션을 알아내어 top, left값에 새롭게 적용을 해줘야합니다. 클릭과 동시에 화면 밖으로 사라지는 원인은 애니메이션은 제거되나 최초에 적용되었던 top, left 0 값이 적용되면서 클릭한 위치에서 벗어났기 때문입니다.


이를 해결 하기 위해 현재 엘리먼트가 실제로 브라우저에서 위치한 포지션 값을 가져올 방법이 필요했습니다. 

네이티브 Web Api 중 [getComputedStyle](https://developer.mozilla.org/ko/docs/Web/API/Window/getComputedStyle) 이라는 api가 있는데, 이를 활용할 수 있었습니다.

> `Window.getComputedStyle()` 메소드는 인자로 전달받은 요소의 모든 CSS 속성값을 담은 객체를 회신합니다. 이 속성값들은, 해당 요소에 대하여 활성 스타일시트와 속성값에 대한 기본 연산이 모두 반영된 결과값입니다. 개별 CSS속성 값은 객체를 통해 제공되는 API 또는 CSS 속성 이름을 사용해서 간단히 색인화해서 액세스할 수 있습니다.

브라우저 개발자도구 Elements 탭에서도 최종적으로 적용된 스타일 값을 확인할 수 있는 Computed에서 보이는 스타일들과 같은 스타일값을 반환한다고 이해하면 됩니다. 

![Screenshot 2024-12-01 at 11.00.37 PM.png|300](/img/user/Screenshot%202024-12-01%20at%2011.00.37%20PM.png)

따라서 동적으로 변화하는 zoom 값, 그때그때 달라지는 transform 애니메이션이 적용되더라도 현재 최종적으로 유효하게 적용된 position값을 알아올 수 있습니다.


아래는 getComputedStyle를 활용해 툴팁에 정확한 포지션을 적용한 것을 수도코드로 작성해본 코드입니다.
zoom 값을 state에 저장하고, 해당 zoomFactor state로 '원래' 적용하려고 했던 x,y 포지션 값을 나누어 정확한 포지션에 툴팁이 위치하게 하는 방식입니다.
```js

const [zoomFactor, setZoomFactor] = useState<number>(1);

useEffect(() => {

const handleZoomChange = () => {
const zoom = parseFloat(getComputedStyle(document.documentElement).getPropertyValue('--zoom')) || 1;

setZoomFactor(zoom);

};

handleZoomChange();

window.addEventListener('resize', handleZoomChange);
  
return () => {
	window.removeEventListener('resize', handleZoomChange);
};

}, []);


return <svg>
	<foreignObject x={midpointX} y={midpointY} ref={containerRef}>
		<Tooltip
			placement="top"
			PopperProps={{
				anchorEl: {
					getBoundingClientRect: () => {
						const containerRect = containerRef.current?.getBoundingClientRect();
						const x = (containerRect?.x ?? 0) / zoomFactor;
						const y = (containerRect?.y ?? 0) / zoomFactor;
						return new DOMRect(x, y);

						},

					},
			}}>	
</Tooltip>
</foreignObject>
</svg>

```

아래는 클릭시 애니메이션을 멈추고 멈춘 자리에 엘리먼트를 고정하는 수도 코드입니다. 

```js

useEffect(() => {

const iconElement = iconRef.current;

if (iconElement) {

const handleClick = () => {
const computedStyle = getComputedStyle(iconElement);
const matrix = new DOMMatrix(computedStyle.transform);
iconElement.style.top = `${iconElement.offsetTop + matrix.m42}px`;
iconElement.style.left = `${iconElement.offsetLeft + matrix.m41}px`;
};

iconElement.addEventListener("click", handleClick);

return () => {
iconElement.removeEventListener("click", handleClick);
};

}

}, []);

```