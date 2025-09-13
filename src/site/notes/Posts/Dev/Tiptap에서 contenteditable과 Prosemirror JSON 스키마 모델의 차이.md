---
{"dg-publish":true,"permalink":"/posts/dev/tiptap-contenteditable-prosemirror-json/","tags":["RichTextEditor","ProsemirrorAPI"],"created":"2025-07-20","updated":"2025-07-20T22:29:00"}
---

Tiptap은 [ProseMirror API](https://prosemirror.net/)를 기반으로 한 리치 텍스트 에디터 라이브러리다. 이를 이해하려면 먼저 `contenteditable` 기반의 편집 방식과 ProseMirror의 JSON 스키마 기반 모델 간의 차이를 이해해야 한다.

## `contenteditable`은 자유롭지만, 구조화되어 있지 않다

`contenteditable`은 브라우저가 제공하는 기본 편집 기능으로, HTML 요소에 `contenteditable` 속성을 부여하면 그 영역이 직접 편집 가능해진다. 사용자가 입력하거나 스타일을 지정하면, 그 결과는 DOM 구조로 곧바로 반영된다. 이 방식은 매우 간단하고 빠르게 구현할 수 있는 장점이 있지만, 편집된 내용을 상태로서 다루기에는 몇 가지 본질적인 어려움이 있다.

## 상태 동기화가 어려운 이유

`contenteditable`은 **DOM이 곧 상태**이기 때문에, 사용자의 입력 방식이나 브라우저 환경에 따라 HTML 구조가 달라질 수 있다. 예를 들어 사용자가 굵은 글씨를 입력할 때, 다음과 같은 다양한 표현이 가능하다.

- `Ctrl + B`를 눌렀을 때: `<b>Hello</b>`
- 툴바에서 Bold 버튼을 클릭했을 때: `<strong>Hello</strong>`
- 특정 브라우저에서는: `<span style="font-weight: bold">Hello</span>`

표현은 다르지만 모두 같은 의미를 가진다. 하지만 이를 innerHTML로 추출해 상태로 관리하려고 하면, 각기 다른 마크업을 어떻게 통일시킬 것인지, 즉 normalize할 방식에 대한 추가 구현이 필요해진다.

또한, 복잡한 리치텍스트 구조나 플러그인 확장이 필요한 경우에는 DOM의 유연함이 오히려 문제가 된다. 브라우저마다 다르게 동작하거나, 깨진 HTML 구조가 생성될 위험이 있기 때문이다.

## 협업 / 동시편집에서의 한계

동시 편집 환경에서는 두 사용자가 같은 문서를 편집할 때 어떤 부분이 어떻게 바뀌었는지를 정확히 추적하고 병합해야 한다. `contenteditable` 기반에서는 DOM 자체가 상태이기 때문에 어떤 노드가 어떻게 바뀌었는지를 감지하기 위해 diff 알고리즘이나 CRDT 같은 별도 로직이 필요하다.

반면, ProseMirror는 내부적으로 모든 편집을 `transaction` 단위로 관리한다. 이로 인해 어떤 변화가 어떤 시점에 일어났는지를 명확히 추적할 수 있고, 충돌 병합(conflict merge)도 수월하게 이뤄진다.

## ProseMirror는 JSON 스키마로 문서를 정의한다

ProseMirror는 문서 구조를 **JSON 기반 스키마**로 정의한다. 예를 들어 단락 안에 텍스트 노드를 포함시키고, 특정 텍스트 범위에는 마크(bold, italic 등)를 지정하는 식이다. 이 구조는 HTML처럼 표현 방식이 자유롭지 않지만, 정확하게 구조화되어 있으며 유효성 검사가 가능하다. 덕분에 깨진 마크업이나 잘못된 중첩이 발생할 수 없다.

```json
{
  "type": "doc",
  "content": [
    {
      "type": "paragraph",
      "content": [
        { "type": "text", "text": "Hello", "marks": [{ "type": "bold" }] },
        { "type": "text", "text": " world" }
      ]
    }
  ]
}
```

이러한 구조 덕분에 문서 내용을 직렬화하여 서버에 저장하거나, 다시 불러와 복원하는 것이 안정적이고 일관된 방식으로 가능하다.

## 트랜잭션 기반 상태 관리 흐름

ProseMirror의 편집 흐름은 다음과 같다.

1. 사용자의 입력이 발생하면, 현재 상태에서 `transaction`을 생성한다.
2. 이 트랜잭션은 문서 상태를 어떻게 바꿀지를 설명하는 불변 객체(blueprint)다.
3. `state.apply(tr)` or `dispatch` 를 호출하면, 이 트랜잭션이 반영된 새로운 상태가 만들어진다.
4. 이 새로운 상태는 `view.updateState(newState)`를 통해 에디터에 반영된다.

이 과정에서 기존 상태 객체는 직접 수정되지 않고, 항상 새로운 상태가 생성된다. 이는 React의 상태 관리와 유사한 불변성(immutable) 패턴이다. 상태가 불변이라는 것은 곧 undo/redo, collaborative editing, 변경 추적 등을 손쉽게 구현할 수 있게 해준다. 


## 결론

간단한 텍스트 입력에는 `contenteditable`만으로도 충분할 수 있다. 하지만 명확한 상태 구조, 복잡한 문서 표현, 협업 기능, 다양한 플러그인 확장 등을 고려한다면, ProseMirror의 JSON 스키마 기반 모델이 훨씬 강력한 선택이 된다. Tiptap은 바로 이 ProseMirror의 장점을 잘 추상화한 라이브러리고, 무엇보다 React 친화적이다. 복잡한 리치 에디터 구현에는 1순위 선택지가 되지 않을지! (ChatGPT, Confluence, Asana도 모두 Prosemirror 기반이다ㅎㅎ)