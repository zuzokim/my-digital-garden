---
{"dg-publish":true,"permalink":"/posts/j-query-react/","created":"2025-06-13","updated":"2025-06-13T20:54:00"}
---

아주 오래전에 웹디자이너가 되고 싶었던 적이 있다. 그 때 동적인 인터랙션을 구현하기 위해 제이쿼리를 짧게 다뤄본 적이 있는데, 최근에 웹빌더를 기반으로 웹사이트를 만들어달라는 부탁을 받아 jQuery 기반의 코드를 커스텀해볼 일이 있었다. 사실상 바닐라 자바스크립트로 DOM을 조작하거나, React를 사용하는게 가장 익숙한 나로서는 jQuery 코드를 만지는게 참 막막했다. 그래서 정리해보는 글.

일단, 이미지 슬라이드 기능을 구현해야 했다. 버튼을 누르면 다음 슬라이드로 넘어가고, 슬라이드를 클릭해도 넘길 수 있는 간단한 기능. 그런데 이 코드를 React로 옮기면 어떤 점이 달라질까? 그전에, jQuery와 React가 **UI를 다루는 방식에서 근본적으로 어떻게 다른지** 먼저 짚고 넘어가 보자.

### jQuery는 "어떻게 할지"를 설명하고 React는 "무엇을 할지"를 선언한다

jQuery는 말 그대로 DOM(Document Object Model)을 조작하기 위한 도구이다.  
어떤 요소를 선택해서, 어떤 스타일을 입히고, 어떤 이벤트를 달고 싶은지를 **직접 조작**하는 방식이다. 예를 들면:

```js
$(".slide").hide();
$(".slide").eq(index).css("display", "flex");
```

이 코드는 슬라이드들을 모두 숨긴 뒤, 특정 인덱스의 슬라이드만 보여주는 방식이다. 전형적인 **명령형 프로그래밍**이다. 개발자가 직접 조작 순서를 정의하고, 그 흐름을 따라가야 한다.

반면 React는 **상태(state)** 를 중심으로 UI를 선언한다. 특정 상태가 되었을 때 어떤 UI를 보여줄지를 **미리 선언**해두고, 상태만 바뀌면 React가 알아서 UI를 갱신해준다. 예를 들면:

```js
{slides.map((slide, i) =>
  <div style={{ display: i === currentIndex ? "flex" : "none" }}>
    {slide}
  </div>
)}
```

#### 예제 코드: jQuery로 만든 슬라이더

```js
$(document).ready(function () {
  const $slides = $(".slide");
  let currentIndex = 0;

  function showSlide(index) {
    $slides.hide();
    $slides.eq(index).css("display", "flex");
  }

  $(".arrow.next").on("click", function (e) {
    e.stopPropagation();
    currentIndex = (currentIndex + 1) % $slides.length;
    showSlide(currentIndex);
  });

  $(".arrow.prev").on("click", function (e) {
    e.stopPropagation();
    currentIndex = (currentIndex - 1 + $slides.length) % $slides.length;
    showSlide(currentIndex);
  });

  $slides.on("click", function () {
    currentIndex = (currentIndex + 1) % $slides.length;
    showSlide(currentIndex);
  });

  showSlide(currentIndex);
});
```


- DOM 요소를 `$slides = $(".slide")`로 직접 선택하고 조작
- 이벤트(`click`)를 수동으로 등록하고,
- DOM을 수동으로 `.hide()`, `.css("display", "flex")`로 상태 변경
- `currentIndex`는 외부 변수로 관리하며 상태 변화와 DOM 조작을 명시적으로 수행

- 상태(`currentIndex`)는 전역 변수 형태이고, 이 값이 바뀌었을 때 UI 갱신을 직접 처리해야 함
- DOM을 직접 조작하므로, UI 상태와 실제 화면 간의 싱크가 깨질 수 있음
- 재사용성과 유지보수가 떨어지고, 컴포넌트화도 어렵다

이 코드는 DOM을 직접 조작한다. 버튼에 이벤트를 수동으로 등록하고, 슬라이드를 숨겼다가 다시 보여주는 로직도 개발자가 전부 처리해야 한다. 이 방식은 간단한 UI에는 적합하지만, 애플리케이션이 복잡해질수록 **상태와 DOM 사이의 일관성 유지가 어려워진다**.

예를 들어 슬라이드 외에도 모달, 로딩 스피너, 사용자 입력 등 다양한 UI 상태가 추가되면, 이 모든 것을 DOM 중심으로 관리하기엔 코드가 쉽게 엉켜버린다.

#### React로 바꾸면 어떤 점이 좋아질까?

React로 바꾸면 전체 흐름이 달라진다.  
중요한 것은 **상태만 신경 쓰면 나머지는 자동으로 그려진다는 점**이다.

```js
function Slider({ slides }) {
  const [currentIndex, setCurrentIndex] = useState(0);

  const next = () => setCurrentIndex((currentIndex + 1) % slides.length);
  const prev = () => setCurrentIndex((currentIndex - 1 + slides.length) % slides.length);

  return (
    <div>
      <button className="arrow prev" onClick={prev}>Prev</button>
      <button className="arrow next" onClick={next}>Next</button>

      {slides.map((slide, i) => (
        <div
          key={i}
          style={{ display: i === currentIndex ? "flex" : "none" }}
          onClick={next}
        >
          {slide}
        </div>
      ))}
    </div>
  );
}
```

- `useState`로 `currentIndex` 상태 관리
- `useEffect`나 조건부 렌더링으로 DOM 직접 조작 없이 슬라이드 전환
- 이벤트 핸들러는 각 컴포넌트에 선언적으로 연결
- `slides.map()`으로 슬라이드 렌더링

여기서 핵심은 두 가지이다:

1. `currentIndex`는 React가 관리하는 **상태값**이다.
2. 이 값이 바뀌면, React는 자동으로 화면을 다시 그린다.

DOM을 직접 건드릴 필요가 없다. 우리는 오직 **"현재 상태가 X일 때 UI는 이렇게 보여야 한다"** 라고만 선언하면 된다.


#### 핵심

- jQuery는 "DOM 조작 중심"
- React는 "상태 관리와 선언적 UI 중심"

jQuery는 한때 가장 강력한 도구였고, 지금도 간단한 UI 구현에는 충분히 쓸 수 있다. 하지만 UI가 복잡해질수록 상태와 화면 사이의 동기화를 유지하기 어렵고, 유지보수 비용도 높아진다.

반면 React는 구조화된 방식으로 UI를 선언하고, 상태에 따라 자동으로 UI를 갱신하므로 **복잡한 애플리케이션 개발과 유지보수에 훨씬 유리한 선택**이 된다.

이렇게 슬라이더처럼 단순해 보이는 기능도, 어떤 도구를 쓰느냐에 따라 코드 구조와 사고방식이 완전히 달라진다.

#### 정리

jQuery와 React 사이에는 단순히 라이브러리의 차원을 넘어서는 **기술적 패러다임의 변화**가 존재한다고 볼 수 있다.

##### 명령형 → 선언형 (Imperative → Declarative)

- **jQuery**는 명령형 패러다임을 따른다.  
    즉, **“무엇을 보여줄지”** 보다 **“어떻게 보여줄지”** 를 상세하게 지시해야 한다. DOM 요소를 수동으로 선택하고, 순차적으로 조작해야 원하는 결과를 얻을 수 있다.
    
    예를 들어, 슬라이드를 바꾸기 위해선 기존 슬라이드를 `hide()` 하고, 새 슬라이드를 `css("display", "flex")` 해야 한다.
- **React**는 선언형 패러다임을 따른다.  
    **“현재 상태에 따라 UI는 어떻게 보여야 하는가”** 를 선언적으로 기술하면, React가 상태 변화를 감지해 UI를 자동으로 업데이트한다.  
    DOM을 직접 건드릴 필요가 없다. 상태만 정의하고, UI는 상태에 따라 **"그려진다"**.

이 변화는 코드의 **예측 가능성**, **테스트 용이성**, **복잡도 관리** 측면에서 큰 차이를 만들어낸다.

##### DOM 중심 → 상태 중심 (DOM-centric → State-centric)

- **jQuery**에서는 **DOM**이 곧 진실의 원천(source of truth)이다. 화면에 어떤 게 있는지를 기준으로 로직이 돌아간다. 상태가 따로 없다. 그래서 DOM이 꼬이면 로직도 꼬이게 된다.
- **React**에서는 **상태(state)** 가 진실의 원천이다. 화면에 무엇이 보이느냐는 오직 상태에 따라 결정된다. 상태만 관리하면 UI는 항상 그 상태를 반영하므로, DOM을 수동으로 추적할 필요가 없다.

이 구조는 **복잡한 UI 로직**에서 유지보수성을 획기적으로 높여준다. 상태만 잘 관리하면 되기 때문이다.

##### 로직과 뷰의 분리 → 컴포넌트 기반 통합 (Separation of Concerns → Co-location)

- **jQuery** 시대에는 HTML, CSS, JS가 분리된 파일에 각각 존재했고, 이들을 하나로 연결하는 방식이었다. UI 조작 로직은 종종 HTML 구조와 멀리 떨어진 곳에서 작성되곤 했다.
- **React**는 로직, 뷰, 스타일이 **컴포넌트 단위로 응집**되어 있다. 하나의 컴포넌트 파일 안에 JSX, 상태 관리, 이벤트 핸들링이 모두 함께 존재한다. 즉, 관심사의 분리(SoC)가 아니라, 관심사의 **응집(co-location)**이 일어난 것이다.

이 방식은 **재사용성**, **컴포넌트 단위 테스트**, **협업 효율**을 극적으로 개선한다.