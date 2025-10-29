---
{"dg-publish":true,"permalink":"/posts/html-id-css-identifier-feat-w3-c-selector-api/","tags":["HTML","CSS"],"created":"2025-10-29","updated":"2025-10-29"}
---

[[Posts/Uncaught SyntaxError - Failed to execute 'querySelector'\|Uncaught SyntaxError - Failed to execute 'querySelector']] 글에서 이어지는 DOM API, CSS Selector, 그리고 HTML Attribute의 역사적 층위 차이 에 대해 공부해본 내용 기록.

디버깅을 하면서 위 글을 쓸 때 “`document.querySelector()`는 HTML의 id 속성을 대상으로 하지만, CSS Selector 문법을 따르기 때문에 두 스펙이 완전히 일치하지 않는다” 는 이유에 대해 의문이 계속 남아있었다. 그래서 좀 더 찾아본 내용을 추가로 기록해본다.

---

HTML의 `id` 속성과 CSS의 “identifier”는 이름이 비슷하지만 태생과 역할이 다르다. 둘을 구분해보면, HTML의 `id`는 “문서 구조적 식별자”이고, CSS의 identifier는 “스타일링 언어의 문법 토큰”이라 서로 다른 문맥의 개념이다.

HTML은 브라우저가 DOM 트리를 만들 때 유효성을 보장하기만 하면 되지만, CSS는 _문법적 파싱이 필요한 언어_ 이기 때문에 더 엄격한 규칙을 가지게 되었다. 

CSS는 “스타일시트 언어”이기 때문에 문법 파싱 단계가 존재하고, 토큰 단위로 분석한다. 예를 들어,  이런 스타일링 코드가 있다고 했을 때, CSS 파서는 `#1foo`를 “숫자로 시작하는 identifier”로 인식하지 못해 문법 에러로 처리한다. 반면 HTML은 단순히 `"1foo"`라는 문자열을 `id` 속성값으로 저장할 뿐이므로 문제가 되지 않는다.

```css
#1foo { color: red }
```

```js
document.querySelector('#1foo') // ❌ CSS identifier로서 invalid (숫자로 시작)
document.getElementById('1foo') // ✅ HTML id로서 valid
```

즉,
- CSS는 **문법적 토큰을 정의**해야 해서 제한이 생겼고,
- HTML은 **데이터 속성값만 저장**하므로 자유롭다.


웹 표준화 그룹 World Wide Web Consortium(W3C)의 웹앱스 워킹그룹(“Web Applications Working Group”)의 Selector API 스펙문서(https://www.w3.org/TR/selectors-api/) 를 더 찾아보면서 신기한 것을 알게 됐는데, 요약해보면 이렇다.

##### `querySelector()`는 **CSS 파서를 재사용**하는 문법적 구현체다.

`document.querySelector()`는 “DOM 검색 API”처럼 보이지만, 내부적으로 **CSS Selector 엔진**을 그대로 사용한다. 셀렉터에 넘긴 문자열은 브라우저의 **CSS 파서**에 전달되어, CSS 선택자 문법으로 해석된 다음 일치하는 요소를 찾는 방식이다.

HTML 팀과 DOM 팀은 “선택자 해석용 파서를 새로 만들지 않고 CSS 파서를 재활용”하는 기술적 문법 재활용을 한 것인데, 그 배경을 알아보면 이렇다.

> Selectors, which are widely used in CSS, are patterns that match against elements in a tree structure [[SELECT\|SELECT]](https://www.w3.org/TR/selectors-api/#SELECT)[[CSS21\|CSS21]](https://www.w3.org/TR/selectors-api/#CSS21). The Selectors API specification defines methods for retrieving [`Element`](https://www.w3.org/TR/selectors-api/#element) nodes from the DOM by matching against a group of selectors. It is often desirable to perform DOM operations on a specific set of elements in a document. These methods simplify the process of acquiring specific elements, especially compared with the more verbose techniques defined and used in the past.

CSS가 이미 갖고 있는 선택자 문법(태그, 클래스, id, 속성, 구조 등)을 **DOM 탐색용 API**로 재활용하자는 것이 핵심이다.

- 개발자는 별도의 새로운 문법을 익힐 필요 없이, 스타일링을 위해 쓰이던 선택자 문법을 DOM 탐색에도 쓸 수 있게 되므로, 직관적이고 일관된 방식으로 요소를 참조할 수 있다는 장점이 있다.
- 또한 브라우저 엔진 입장에서도 이미 CSS 선택자 엔진 및 파서(parser)가 구현되어 있는 상태였고, 이 파서를 재활용하거나 선택자 해석 로직을 공유하는 것이 구현 부담이 덜하다는 실용적 이점이 있다.


정리해보면, HTML과 CSS는 별개의 사양 영역이었다가 나중에 서로 연동된 셈이고, 선택자(API)가 CSS의 문법 체계를 따르도록 설계되었기 때문에 이러한 차이가 나타난 것이다. 알고나니 꽤나 합리적인 활용 방안이라는 생각이 든다. 

---

그럼 위에서 짧게 정리한 HTML과 CSS 사양의 차이를 좀 더 자세히 정리해보자.

- HTML `id`는 DOM 트리 안에서 **유일성 (unique) 식별자**로 쓰이기 위한 값이다. 브라우저는 이 값을 단순히 string으로 저장하거나 참조하고 “동일 id 이면 getElementById 같은 메소드로 찾는다” 정도의 내부 처리면 된다. 특별히 선택자 문법을 따를 필요는 없다.
- 반면 CSS 식별자는 **스타일 규칙 언어(CSS)의 토큰으로서** 파싱되어야 한다. 따라서 CSS 문법 입장에서 의미를 갖는 문자셋, 토큰화 규칙, 이스케이프 처리 등이 필요하다. (예: `-`나 `_`의 위치, 숫자 시작 여부, 이스케이프된 유니코드 등)
- Selectors API가 CSS 선택자 문법을 재활용했기 때문에, DOM 탐색 API가 CSS 파서를 재활용하는 형태가 되었다. 이 때문에 “id 값이 HTML상으론 유효하지만 CSS 선택자 문법상 유효하지 않다”는 `querySelector()`가 syntax 오류를 던지는 시나리오가 생길 수 있다.
