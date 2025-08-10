---
{"dg-publish":true,"permalink":"/posts/ime-input-method-editor/","created":"2025-08-10","updated":"2025-08-10T20:12:00"}
---

### IME(Input Method Editor)란?

IME(Input Method Editor)는 한글, 일본어, 중국어처럼 **조합형 문자**를 입력할 때 사용하는 입력기다. 영어처럼 한 글자씩 완성되는 언어와 달리,  조합형 문자는 자음·모음 또는 여러 글자 요소를 조합해 하나의 완성형 글자를 만든다. 즉, 한 글자를 완성하기 위해 여러 번의 키 입력이 발생한다.

예를 들어,
- 한글: `"ㄱ"` → `"가"`
- 일본어: `"ka"` → `"か"`
- 중국어(병음): `"han"` → `"汉"`

이 조합 과정에서 브라우저는 사용자가 어떤 글자를 “완성 중”인지 추적해야 하고,  이를 위해 **`compositionstart`**, **`compositionupdate`**, **`compositionend`** 같은 이벤트를 발생시킨다. 문제는 이 조합 과정에서 **Enter, Backspace 등 키 이벤트가 의도치 않게 처리될 수 있다는 것**이다. 예를 들어, 한글을 입력 중인데 Enter 키가 눌리면, 아직 조합이 끝나지 않았는데도 줄바꿈이나 제출이 실행될 수 있다.

### 왜 문제가 생기나?

IME 조합 중에도 `keydown` 이벤트가 그대로 발생하기 때문에,  채팅창 같은 곳에서 한글 입력 중 Enter를 누르면 조합이 끝나기 전에 메시지가 전송될 수 있다.  예를 들어 `"안녕"`을 쓰다가 `"안ㄴ"` 상태에서 Enter를 누르면 `"안ㄴ"`이 전송된다. 이걸 막으려면 **조합 상태를 감지**해서, 조합 중에는 Enter 같은 특정 키 입력을 무시해야 한다.

### 예시

3월부터 진행중인 프론트엔드 코드리뷰 스터디에서 간단한 체크리스트 에디터를 만들었었다. 노션처럼 불렛 형태의 리스트아이템을 만들고 탭와 엔터로 리스트를 만들고, 토글 기능을 붙이는 작업이었다. 그 중에서 토글(접고 펼치는 기능) 로직 구현 외에 이 IME 조합 문제를 마주쳤는데, 영어로 리스트아이템을 작성할 때는 나타나지 않다가 한글로 만들면 문제가 발생했었다.

이 문제를 해결하기 위해 작성한 코드 중에 기타 비즈니스 로직을 제외하고 IME 조합 문제를 핸들링한 부분만 가져와봤다.



```ts
import { useState } from "react";

export default function IMEInput() {
  const [value, setValue] = useState("");
  const [isComposing, setIsComposing] = useState(false);

  const handleKeyDown = (e: React.KeyboardEvent<HTMLInputElement>) => {
    if (isComposing && e.key === "Enter") {
      // 조합 중일 땐 Enter 무시
      return;
    }

    if (e.key === "Enter") {
      e.preventDefault();
      console.log("Submit:", value);
    }
  };

  return (
    <input
      type="text"
      value={value}
      onChange={(e) => setValue(e.target.value)}
      onKeyDown={handleKeyDown}
      onCompositionStart={() => setIsComposing(true)}
      onCompositionEnd={() => setIsComposing(false)}
    />
  );
}

```

#### 동작 원리
- **onCompositionStart**  
    IME 입력이 시작되면 `isComposing`을 `true`로 설정한다.
- **onCompositionEnd**  
    조합이 끝나면 `isComposing`을 `false`로 되돌린다.
- **onKeyDown**  
    `isComposing`이 `true`일 때 Enter 키가 눌리면 함수를 바로 종료해 이벤트를 무시한다.  
    조합이 끝난 후에만 실제 동작(예: 제출)이 실행된다.

##### 다른 방식 : `e.nativeEvent.isComposing` 사용하기

이벤트 핸들러를 사용하는 방식 외에 nativeEvent 객체를 사용할 수도 있다는 걸 스터디 팀원의 코드를 리뷰하면서 새로 알게 됐다. 

React의 키 이벤트 객체에는 브라우저에서 발생한 원본 이벤트(`nativeEvent`)가 들어있다.  여기에는 `isComposing`이라는 속성이 있어서, 현재 IME 조합 중인지 바로 확인할 수 있다. 이걸 쓰면 별도의 `isComposing` 상태 변수를 관리할 필요가 없어서 더 적은 코드로 처리할 수 있다는 장점이 생긴다.

```ts
import { useState } from "react";

export default function IMEInput() {
  const [value, setValue] = useState("");

  const handleKeyDown = (e: React.KeyboardEvent<HTMLInputElement>) => {
    if (e.nativeEvent.isComposing && e.key === "Enter") {
      // 조합 중이면 Enter 무시
      return;
    }

    if (e.key === "Enter") {
      e.preventDefault();
      console.log("Submit:", value);
    }
  };

  return (
    <input
      type="text"
      value={value}
      onChange={(e) => setValue(e.target.value)}
      onKeyDown={handleKeyDown}
    />
  );
}

```

다만, 일부 오래된 브라우저(특히 IE11 같은 구형 환경)에서는 `isComposing` 속성이 없을 수 있다.. ;; 일단 모던 브라우저만 지원한다면 문제가 없을 듯 하다.

#### 두 방식 비교

![스크린샷 2025-08-10 오후 8.24.47.png](/img/user/%EC%8A%A4%ED%81%AC%EB%A6%B0%EC%83%B7%202025-08-10%20%EC%98%A4%ED%9B%84%208.24.47.png)


혼자 코딩하는 것보다 역시 같이 하는게 재밌다..!

![IME_me.png](/img/user/IME_me.png)![IME_mate.png](/img/user/IME_mate.png)