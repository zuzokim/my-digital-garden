---
{"dg-publish":true,"permalink":"/posts/react-property-descriptor/","created":"2026-03-01","updated":"2026-03-01"}
---

프론트엔드 개발에서 사용자 인터랙션을 감지하고 상태를 동기화하는 것은 핵심 과제이다. 대부분의 경우 브라우저가 제공하는 네이티브 이벤트(`input`, `change`, `keydown` 등)에 의존하면 되지만, 이 전제가 깨지는 순간이 있다. 바로 **외부 시스템이 DOM을 직접 조작하는 경우**이다.

대표적인 예가 보안키패드이다. 보안키패드는 키로깅 공격을 방지하기 위해 사용되는 가상 키보드로, 주로 금융권 웹사이트에서 비밀번호, 계좌번호, 주민등록번호 등 민감한 정보를 입력받을 때 사용된다. 국내에서는 은행, 증권사, 카드사 등 대부분의 금융 서비스에서 필수적으로 적용되어 있으며, 공공기관 웹사이트나 본인인증이 필요한 서비스에서도 흔히 볼 수 있다.

하지만 이 문제는 금융권에만 국한되지 않는다. 모바일 앱을 웹뷰로 구현할 때 네이티브 키패드와 연동하는 경우, 수식 입력기나 이모지 피커 같은 커스텀 가상 키보드를 직접 만드는 경우, 또는 서드파티 라이브러리가 input 값을 프로그래머틱하게 조작하는 경우에도 동일한 문제가 발생할 수 있다. 결국 **“이벤트 없이 값이 바뀌는 상황”** 을 어떻게 감지할 것인가의 문제이다.

이 글에서는 왜 일반적인 이벤트 리스너로는 이러한 입력을 감지할 수 없는지 살펴보고, `Object.defineProperty`를 활용한 해결책을 다룬다.

-----

## 문제 상황

보안키패드(가상 키보드)를 통해 입력된 값이 React 컴포넌트의 상태에 반영되지 않는 현상이 발생했다. 분명히 input 필드에는 값이 들어가 있는데, React는 이 변화를 전혀 인지하지 못하고 있었다.

처음 시도한 방법은 다음과 같다.

```js
['keypress', 'focus', 'change'].forEach(event => {
  document.addEventListener(event, () => {
    // input 값 변경 감지 후 React 상태 업데이트 시도
    triggerInputUpdate(input);
  });
});
```

하지만 이 코드는 전혀 동작하지 않았다.

-----

## 이벤트 버블링의 이해

먼저 DOM 이벤트가 어떻게 동작하는지 살펴보자.

### 이벤트 전파의 3단계

DOM 이벤트는 세 단계를 거쳐 전파된다.

```markdown
1. Capturing Phase (캡처링)
   window → document → html → body → ... → target

2. Target Phase (타겟)
   이벤트가 실제 대상 요소에 도달

3. Bubbling Phase (버블링)
   target → ... → body → html → document → window
```

`addEventListener`의 세 번째 인자로 캡처링/버블링 단계 중 언제 핸들러를 실행할지 지정할 수 있다.

```js
// 버블링 단계에서 처리 (기본값)
element.addEventListener('click', handler);
element.addEventListener('click', handler, false);

// 캡처링 단계에서 처리
element.addEventListener('click', handler, true);
```

### 사용자 상호작용과 이벤트 발생

일반적인 키보드 입력 시 발생하는 이벤트 순서는 다음과 같다.

```
keydown → keypress → (값 변경) → input → keyup → change(blur 시)
```

이 이벤트들은 브라우저가 사용자의 물리적 상호작용을 감지했을 때 자동으로 발생시킨다. 그리고 이벤트 버블링을 통해 상위 요소까지 전파되므로, `document`에 리스너를 등록해도 하위 요소의 이벤트를 감지할 수 있다.

```js
// document에서 모든 클릭 감지 가능 (버블링 덕분)
document.addEventListener('click', (e) => {
  console.log('클릭된 요소:', e.target);
});
```

-----

## 보안키패드는 왜 다른가

보안키패드는 키로깅 방지를 위해 의도적으로 DOM 이벤트를 발생시키지 않는다. (솔루션 제품이나 구현방식에 따라 조금씩 다르다.)

### 일반 키보드 입력

```markdown
사용자 키 입력
    ↓
브라우저가 keydown, keypress, input 이벤트 발생
    ↓
이벤트 버블링으로 document까지 전파
    ↓
등록된 이벤트 리스너 실행
```

### 보안키패드 입력

```
보안키패드 UI에서 버튼 클릭
    ↓
내부적으로 input.value = '암호화된값' 직접 할당
    ↓
끝 (이벤트 없음)
```

보안키패드는 JavaScript로 `input.value`를 직접 조작한다. 이 방식은 `keypress`, `keydown`, `input`, `change` 등 어떤 네이티브 이벤트도 발생시키지 않는다.

|구분      |일반 키보드|보안키패드   |
|--------|------|--------|
|keydown |✅ 발생  |❌ 발생 안 함|
|keypress|✅ 발생  |❌ 발생 안 함|
|input   |✅ 발생  |❌ 발생 안 함|
|change  |✅ 발생  |❌ 발생 안 함|
|value 변경|✅     |✅       |

따라서 아무리 `document`에 이벤트 리스너를 등록해도, 발생하지 않는 이벤트는 감지할 수 없다.

-----

## 해결책: Property Descriptor 활용

이벤트가 발생하지 않는다면, 값의 변경 자체를 가로채야 한다. JavaScript의 `Object.defineProperty`를 사용하면 가능하다.

### Property Descriptor란?

그간 나의 경험으로는 브라우저 네이티브 이벤트를 활용하면 대부분의 이슈들은 해결이 되었기 때문에, 이 속성이 익숙하지 않았다. MDN을 읽으며 기본적인 이해부터 해보자.

JavaScript의 모든 객체 속성(property)은 내부적으로 descriptor를 가지고 있다. descriptor는 해당 속성의 동작 방식을 정의한다.

```js
const obj = { name: 'Kim' };

console.log(Object.getOwnPropertyDescriptor(obj, 'name'));
// {
//   value: 'Kim',
//   writable: true,      // 값 수정 가능 여부
//   enumerable: true,    // for...in 순회 대상 여부
//   configurable: true   // 삭제/재정의 가능 여부
// }
```

### Data Descriptor vs Accessor Descriptor

속성은 두 가지 방식으로 정의할 수 있다.

```js
// Data Descriptor: 값을 직접 저장
Object.defineProperty(obj, 'age', {
  value: 25,
  writable: true
});

// Accessor Descriptor: getter/setter 함수로 값 접근
Object.defineProperty(obj, 'age', {
  get: function() { return this._age; },
  set: function(value) { this._age = value; }
});
```

Accessor Descriptor를 사용하면 속성에 값을 할당하거나 읽을 때마다 정의한 함수가 실행된다.

### input.value의 Setter 재정의

이 원리를 `input.value`에 적용한다.

```js
function defineSecureInput(inputElement) {
  // HTMLInputElement.prototype에서 원본 getter/setter 가져오기
  const descriptor = Object.getOwnPropertyDescriptor(
    HTMLInputElement.prototype,
    'value'
  );
  const originalGetter = descriptor.get;
  const originalSetter = descriptor.set;

  // 이 input 요소의 value 속성만 재정의
  Object.defineProperty(inputElement, 'value', {
    get: function() {
      return originalGetter.call(this);
    },
    set: function(newValue) {
      // 1. 원본 setter로 실제 값 설정
      originalSetter.call(this, newValue);

      // 2. 커스텀 이벤트 발생
      const event = new CustomEvent('custom-keyboard-input', {
        detail: { value: newValue },
        bubbles: true
      });
      this.dispatchEvent(event);
    }
  });
}
```

### 왜 prototype에서 가져오는가?

```js
Object.getOwnPropertyDescriptor(HTMLInputElement.prototype, 'value')
```

`input.value`는 인스턴스 자체가 아닌 `HTMLInputElement.prototype`에 정의되어 있다. 프로토타입 체인을 통해 모든 input 요소가 이 getter/setter를 공유한다.

개별 요소에 `defineProperty`를 하면, 프로토타입의 속성을 “가리는(shadow)” 새로운 인스턴스 속성이 생성된다. 이렇게 하면 해당 요소에서만 커스텀 동작이 적용되고, 다른 input 요소들은 영향받지 않는다.

-----

## 전체 구현

```js
function onReadySecureInput(input) {
  // input 요소 설정
  input.id = 'custom-keyboard-input';
  input.type = 'password';
  input.setAttribute('maxlength', '7');

  // 보안키패드 초기화 (각 솔루션마다 다름)
  window.secureKeypad.init();

  // 커스텀 이벤트 리스너 등록
  document.addEventListener('custom-keyboard-input', (e) => {
    const value = e.detail.value;

    // React 상태 업데이트를 위한 input 이벤트 발생
    input.dispatchEvent(new Event('input', { bubbles: true }));
  });

  // value setter 재정의
  defineSecureInput(input);
}

function defineSecureInput(inputElement) {
  if (!inputElement) return;

  const descriptor = Object.getOwnPropertyDescriptor(
    HTMLInputElement.prototype,
    'value'
  );
  const originalGetter = descriptor.get;
  const originalSetter = descriptor.set;

  Object.defineProperty(inputElement, 'value', {
    get: function() {
      return originalGetter.call(this);
    },
    set: function(newValue) {
      originalSetter.call(this, newValue);

      const event = new CustomEvent('custom-keyboard-input', {
        detail: { value: originalGetter.call(this) },
        bubbles: true
      });
      document.dispatchEvent(event);
    }
  });
}
```

### 동작 흐름

```markdown
보안키패드에서 값 입력
    ↓
input.value = '값' 할당 시도
    ↓
재정의된 setter 실행
    ↓
원본 setter로 실제 값 설정
    ↓
CustomEvent 발생 및 dispatch
    ↓
등록된 이벤트 리스너에서 감지
    ↓
React input 이벤트 발생 → 상태 업데이트
```

-----

## 주의사항

### 1. 원본 동작 보존

setter를 재정의할 때 반드시 원본 setter를 호출해야 한다. 그렇지 않으면 실제 값이 설정되지 않는다.

```js
set: function(newValue) {
  // 이 줄이 없으면 값이 실제로 저장되지 않음
  originalSetter.call(this, newValue);
  // ...
}
```

### 2. this 바인딩

`call(this, ...)`를 사용해 올바른 컨텍스트를 유지해야 한다. 화살표 함수를 사용하면 `this`가 달라질 수 있으니 주의가 필요하다.

### 3. 성능 고려

모든 값 할당에서 추가 로직이 실행되므로, 필요한 요소에만 적용해야 한다.

### 4. React와의 통합

React는 자체적으로 input의 value를 관리한다. 커스텀 이벤트 발생 후 `input` 이벤트를 dispatch해야 React가 변경을 인지한다.

```js
input.dispatchEvent(new Event('input', { bubbles: true }));
```

-----

## 정리

|방식                                |동작 여부|이유                     |
|----------------------------------|-----|-----------------------|
|이벤트 리스너 (keypress, change 등)      |❌    |보안키패드는 이벤트를 발생시키지 않음   |
|MutationObserver                  |❌    |value 변경은 DOM 속성 변경이 아님|
|Object.defineProperty (setter 재정의)|✅    |값 할당 자체를 가로챔           |

보안키패드처럼 이벤트 없이 값을 직접 조작하는 경우, 이벤트 버블링에 의존하는 일반적인 패턴은 동작하지 않는다. `Object.defineProperty`로 property descriptor를 재정의하면 값의 변경 시점을 정확히 포착할 수 있고, 이를 통해 React 등 프레임워크의 상태 관리 시스템과 연동할 수 있다.

이 패턴은 보안키패드 외에도 서드파티 라이브러리가 DOM을 직접 조작하는 상황에서 유용하게 활용할 수 있다.

가상키보드의 값 핸들링을 해보면서 모든 디버깅은 동작 원리의 기본 이해부터 출발한다는 것을 배웠다. 이벤트 버블링, DOM, JS 프로토타입.. 

그리고 React 패러다임과 그 안의 상태관리 방식에만 익숙해져 있었는데, 넓은 시야로 vanilla JS 생태계를 바라보고 더 잘 활용해야겠다는 생각도 들었다.