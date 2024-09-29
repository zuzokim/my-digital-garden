---
{"dg-publish":true,"permalink":"/pages/pointer-events-elements/","tags":["CSS"],"created":"2024-09-29","updated":"2024-09-29T16:19:00"}
---

지난 포스트 [[Pages/element의 중앙을 찾아 선긋기\|element의 중앙을 찾아 선긋기]] 에서 svg 레이어를 보기카드 레이어보다 낮은 z-index로 겹쳐 그려 보기카드의 위치와 선이 그려지는 위치를 일치시키는 방식으로 '선긋기'를 완성했었습니다. 그런데 만약 svg로 그려진 선을 직접 클릭해서 이어진 두 카드의 연결을 끊는 인터랙션을 만들기 위해서는 어떻게 할 수 있을까요? 

기본적으로 모든 elements는 z-index상 가장 위에 배치된 element만 클릭이벤트를 호출할 수 있습니다. 지금까지의 구현으로는 가장 위에 배치된 보기카드만 클릭이 가능하고, 뒤 쪽에 그려진 svg line들은 클릭이 어렵습니다. 또, x자로 line들이 겹쳐그려진 경우 line들도 겹쳐진 상태이기 때문에 개별적으로 클릭이 어렵습니다.

정리하면 아래 두 가지 태스크를 해결해야 합니다.
1. 보기카드 레이어보다 낮은 z-index의 svg 레이어를 클릭 가능하게 하기
2. svg 레이어에 그려질 line들을 개별적으로 클릭 가능하게 하기

##### 1. 낮은 z-index elements를 클릭 가능하게 하기

![layers.png|400](/img/user/layers.png)

###### 2. svg line들을 개별적으로 클릭 가능하게 하기

![svglines.png|400](/img/user/svglines.png)