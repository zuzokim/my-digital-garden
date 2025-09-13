---
{"dg-publish":true,"permalink":"/posts/dev/uncaught-syntax-error-failed-to-execute-query-selector/","tags":["HTML","CSS","javascript"],"created":"2024-11-14","updated":"2024-11-14T19:07:00"}
---

최근 리치에디터 기능 개발을 하면서 div 요소에 id indentifier를 랜덤으로 제너레이트해서 부여하고 해당 id를 querySelector로 찾아 DOM을 업데이트해줄 일이 있었습니다.

자세한 스펙은 적을 수 없지만 리치에디터 컴포넌트를 tiptap을 사용해서 구현하고 있는데, 이때 커스텀한 [Node](https://tiptap.dev/docs/editor/extensions/custom-extensions/node-views)마다 id를 부여해 json content에 저장하고, DOM을 업데이트해줘야하는 경우에는 DOM element에도 해당 id를 identifier로 부착해 조작하는 방식으로 개발을 하고 있습니다.

단, 리치에디터가 화면에 여러 개 반복 렌더되는 경우, id가 고유하지 않은 문제가 발생합니다. DOM tree에 그릴 때 id가 중복이 되는 것입니다. 이런 경우 첫번 째 그려진 리치에디터의 Node id와 두번 째 그려진 Node id를 구분하기 위해 에디터마다 key를 랜덤한 값으로 부여하여 구분할 수 있게 처리해주었습니다.

그런데 전체 도큐먼트에서 특정 Node의 dom element의 포지션을 업데이트해야하는 경우가 생겨, Node의 id와 에디터의 랜덤 key를 조합한 고유한 identifier로 해당 element를 querySelector로 찾는 코드를 작성하니 이런 에러가 간헐적으로 발생했습니다.

> [!ERROR]
> Uncaught SyntaxError: Failed to execute 'querySelector' on 'Document': '#1V1StGXR8_Z5jdHi6B-my' is not a valid selector

(해당 에러는 예시입니다.)

처음에는 오타가 있는 줄 알았는데 아니었습니다.

```js
document.querySelector(`#${generatedID}`)
```

원인은 querySelector는 CSS3 셀렉터를 이용해 DOM을 쿼리하는데, CSS indentifier 자체가 올바르지 않아 쿼리할 수 없다는 syntax에러가 났기 때문이었습니다.

w3.org[^w3.org] 의 CSS syntax 스펙을 보면 아래와 같습니다.

![스크린샷 2024-11-14 오후 6.31.37.png](/img/user/%EC%8A%A4%ED%81%AC%EB%A6%B0%EC%83%B7%202024-11-14%20%EC%98%A4%ED%9B%84%206.31.37.png)
> [!NOTE] 4.1.3 Characters and case
> CSS indentifier(요소 이름, 클래스 및 선택기의 ID 포함)에는 a-zA-Z0-9 및 ISO 10646 문자 U+00A0 이상과 하이픈( - ) 및 밑줄( _ )만 포함할 수 있으며 숫자, 하이픈 2개 또는 하이픈 다음에 숫자로 시작하면 안됩니다.

id가 고유하지 않은 문제는 아니었고, Node의 id와 에디터에 랜덤으로 부여되는 key에 위의 스펙에 위반되는 숫자나 하이픈이 포함되는 경우 발생했던 것입니다. 그래서 간헐적인 것처럼 보였던 것이지요.

그럼 어떻게 해결할 수 있을까요? 일단 id를 syntax를 위반하지 않도록 하기는 어려워보입니다. 따라서 이스케이프문자열로 변환되게 해주거나, 숫자가 들어오더라도 스트링으로 인식하게끔 querySelector를 작성하면 됩니다.

저는 이런 식으로 해결했습니다. getElementById 메소드를 사용할 수도 있지만 저의 경우에는 id 선택자 외에 다른 선택자도 함께 필요한 상황이어서 querySelector를 사용하되, id를 스트링으로 찾을 수 있도록 "" 로 감싸 작성하는 방식을 택했습니다.

```js
document.querySelector(`[id="${generatedID}"]`)
```

디버깅하면서 든 생각이 애초에 HTML id attributes의 스펙이 CSS identifier 스펙과 다른게 원인이 아닐까? 하는 의아함이 들었습니다. 

html.spec[^html.spec] 의 id attributes 스펙
![스크린샷 2024-11-14 오후 6.44.59.png](/img/user/%EC%8A%A4%ED%81%AC%EB%A6%B0%EC%83%B7%202024-11-14%20%EC%98%A4%ED%9B%84%206.44.59.png)
> [!NOTE] the-id-attribute
> The [id](https://html.spec.whatwg.org/multipage/dom.html#the-id-attribute) attribute specifies its element's [unique identifier (ID)](https://dom.spec.whatwg.org/#concept-id). 
> There are no other restrictions on what form an ID can take; in particular, IDs can consist of just digits, start with a digit, start with an underscore, consist of just punctuation, etc. 
> An element's [unique identifier](https://dom.spec.whatwg.org/#concept-id) can be used for a variety of purposes, most notably as a way to link to specific parts of a document using [fragments](https://url.spec.whatwg.org/#concept-url-fragment), as a way to target an element when scripting, and as a way to style a specific element from CSS.


gpt에게 질문해본 결과, 

HTML attributes와 CSS identifier의 스펙이 다른 이유는 일단 이 둘의 목적이 다르기 때문입니다.
- HTML attributes는 document 내에서 엘리먼트를 고유하게 식별하기 위해 사용합니다.
- CSS identifier는 스타일시트에서 엘리먼트에 스타일을 적용하기 위해 사용됩니다.


하지만.. 아직도 document의 엘리먼트를 쿼리하는 데 목적이 있는 querySelector는 왜 CSS 셀렉터를 사용함에도 두 스펙이 일치하지 않는 것인지, 왜 CSS identifier 스펙이 더 엄격한 것인지 궁금함이 해소되지 않았네요. 

HTML, CSS 기본 스펙을 잘 아는 것이 중요하겠지만, 적어도 코드를 작성할 때 주의할 수 있다면 좋을 것 같습니다.



[^w3.org]: https://www.w3.org/TR/CSS21/syndata.html#characters
[^html.spec]: https://html.spec.whatwg.org/multipage/dom.html#the-id-attribute
[^참고] : https://stackoverflow.com/questions/37270787/uncaught-syntaxerror-failed-to-execute-queryselector-on-document