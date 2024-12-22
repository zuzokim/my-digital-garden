---
{"dg-publish":true,"permalink":"/pages/z-index/","tags":["CSS"],"created":"2024-12-22","updated":"2024-12-22T23:23:00"}
---

최근 모달 컴포넌트를 개발하고 해당 컴포넌트를 헤더에서 한 번, 푸터에서 한 번 열도록 한 케이스가 있었습니다.
우선 모달 컴포넌트는 뒤쪽에 백드롭(전체 화면이 어둡께 깔리는 UI)이 깔리고 뷰포트 중앙에 모달이 배치되는 글로벌(?)한 UI 컴포넌트로 제작을 했습니다. 구현된 방식은 fixed position에 뷰포트 중앙으로 transition을 잡아주는 평범한 방식에 화면 위쪽으로 올라오도록 z-index를 1로 높여 놓았습니다.

그런데 헤더에서 모달 UI를 오픈했을 때와 푸터에서 오픈했을 때 다른 결과가 나타났습니다. 헤더에서 오픈할 때는 문제가 되지 않았지만, 푸터에서 오픈하면 백드롭이 깔리지 않고, 모달 뒤쪽 버튼이나 input 컴포넌트 인터랙션을 막을 수 없었습니다. 

처음에는 'z-index상 모달과 모달의 백드롭이 더 높게 설정되어있어서 UI도 잘 보이고, 뒤쪽에 배치된 다른 컴포넌트 인터랙션도 막혀야하는게 아닌가?' 하고 뭔가 오타가 있거나, position 레이아웃이 잘못되었거나, z-index가 더 높아야 하는것이 아닌가 하는 생각을 했습니다.

원인은 다음과 같았습니다. 헤더는 마크업 순서상, 그리고 dom tree상 모달 컴포넌트보다 먼저 그려지고, 푸터는 모달보다 다음에 그려집니다. 따라서 아무리 모달 UI의 z-index를 높여도 모달이 푸터보다 위쪽에 올 수 없습니다. 헤더, 모달, 푸터 컴포넌트를 모두 같은 플로우 레이아웃, [stacking context](https://developer.mozilla.org/ko/docs/Web/CSS/CSS_positioned_layout/Understanding_z-index/Stacking_context)에 두는 게 아니라면요.

https://csshell.dev/posts/z-index-hell/ 을 읽고 다시 한번 z-index가 제대로 동작하기 위한 조건들을 상기시켜 봤습니다. 그리고 소소하지만 새로운 사실도 알게됐네요. 

1. z-index는 `absolute`, `fixed`, `relative` or `sticky` `positions`에만 사용할 수 있습니다.