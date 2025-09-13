---
{"dg-publish":true,"permalink":"/posts/dev/react-portal-draft/","tags":["React"],"created":"2024-12-29","updated":"2024-12-29T21:33:00"}
---

React에서 `createPortal`을 사용하여 렌더링된 컴포넌트는 **React 트리**의 계층 구조는 유지하면서, 실제 DOM 트리에서 다른 위치에 렌더링됩니다. 이러한 포털은 React 이벤트 시스템과의 통합 덕분에 React 트리 기반의 이벤트 전파를 유지할 수 있습니다.

### **React Portal의 이벤트 전파 원리**

React는 포털이 렌더링된 DOM 노드의 위치와 관계없이, **React 트리의 계층 구조를 기반으로 이벤트를 전파**합니다. 이 원리는 React가 SyntheticEvent와 이벤트 위임을 활용하여 이벤트를 루트 컨테이너로 전달하는 방식과 관련이 있습니다.

#### **작동 원리**

1. **React 트리와 DOM 트리의 분리**:
    - `createPortal`을 사용하면 컴포넌트가 React 트리에서는 원래 부모 컴포넌트와 연결된 상태로 유지됩니다.
    - 하지만 렌더링된 DOM은 React 트리에서 벗어난 외부 컨테이너에 추가됩니다.
2. **SyntheticEvent 시스템**:
    - React의 이벤트 시스템은 이벤트를 캡처링하고, SyntheticEvent를 생성하여 React 트리 기반으로 전파를 처리합니다.
    - 포털에서 발생한 이벤트도 React 트리를 거쳐 전파됩니다.
3. **React 내부 이벤트 핸들러**:
    - 이벤트가 루트 컨테이너에서 처리될 때, React는 이벤트의 발생 위치를 확인하고, 해당 이벤트를 React 트리 계층 구조에 맞게 전파합니다.

### **내부 구현 코드와 동작 흐름**

React의 포털 이벤트 전파는 대략적으로 다음과 같은 단계를 거칩니다:

#### 1. **Portal Node Rendering**

`createPortal` 함수는 React 트리의 노드를 특정 DOM 컨테이너에 렌더링합니다.
```js
ReactDOM.createPortal(child, container);
```

React 트리는 유지되지만, `child`는 외부`container`에 렌더링됩니다.

#### 2. **이벤트 위임과 트리 구조 연결**

React는 이벤트를 루트 컨테이너에 위임합니다. 포털에서 발생한 이벤트는 브라우저의 네이티브 이벤트 전파를 통해 루트 컨테이너로 도달합니다.

#### 3. **React Tree와 SyntheticEvent**

React는 이벤트 발생 시 DOM 노드가 아닌 **React Fiber 트리**를 기준으로 이벤트를 전파합니다. Fiber 트리에서는 다음 작업이 이루어집니다:

1. Fiber 노드에 저장된 React 핸들러를 확인.
2. 상위 컴포넌트로 이벤트를 전파 (버블링 또는 캡처링 단계).


#### 결론
React의 `createPortal`은 렌더링된 DOM 위치와 관계없이 React 트리의 계층 구조를 유지합니다. SyntheticEvent 시스템과 Fiber 트리를 기반으로 이벤트를 처리하므로, 포털로 렌더링된 컴포넌트에서도 React 트리 기반의 이벤트 전파가 가능합니다. 이는 React의 강력한 이벤트 추상화 덕분에 구현된 기능입니다.


---

React에서 이벤트 전파는 브라우저의 DOM 이벤트 전파 메커니즘(캡처링과 버블링)을 활용하면서, React의 **컴포넌트 트리**를 기반으로 한 추가적인 추상화를 통해 이루어집니다. React는 SyntheticEvent를 사용하여 이러한 전파 과정을 처리하며, 이벤트 핸들링을 더 일관되고 효율적으로 관리합니다.

### **React의 이벤트 전파 방식**

#### 1. **React에서의 이벤트 위임**

React는 성능 최적화를 위해 이벤트를 루트 컨테이너(주로 `document` 또는 React DOM 루트 노드)에 위임합니다.

React는 React 17 버전부터 이벤트 위임을 브라우저의 `document`가 아닌 React의 루트 컨테이너(root node)에 위임하는 방식으로 변경되었습니다.

- React는 루트 노드에 단일 이벤트 리스너를 등록하고, 모든 하위 컴포넌트에서 발생하는 이벤트를 해당 리스너가 처리합니다.
- 발생한 이벤트는 React의 SyntheticEvent로 래핑되어 컴포넌트 계층을 따라 전파됩니다.
#### **변경 이유**

React 17 이전에는 이벤트 위임이 브라우저의 `document`에 설정되었습니다. 이 방식은 잘 동작했지만, 다음과 같은 한계가 있었습니다:

1. **마이크로 프론트엔드 및 여러 React 애플리케이션 문제**:
    
    - 여러 React 애플리케이션이 같은 페이지에서 실행될 경우, 각각의 React 애플리케이션이 `document`에 이벤트를 등록하면서 충돌 가능성이 있었습니다.
    - 예를 들어, 이벤트 핸들러가 다른 애플리케이션의 이벤트를 처리하려고 시도하거나, 의도하지 않은 동작을 유발할 수 있었습니다.
2. **더 나은 이벤트 격리 필요**:
    
    - 각 React 애플리케이션은 자체적으로 독립된 이벤트 위임을 가지는 것이 더 안정적이며 유지보수에 유리합니다.
    - 이를 통해 특정 루트 컨테이너 외부에서 발생한 이벤트는 해당 컨테이너의 React 애플리케이션에 영향을 미치지 않도록 격리할 수 있습니다.

#### 2. **React의 트리 구조에서 이벤트 전파**

React는 브라우저의 DOM 트리가 아니라 React 자체의 **컴포넌트 트리**를 따라 이벤트를 전파합니다. 이 과정에서:

- 먼저, 이벤트가 캡처링 단계로 상위 컴포넌트로 전파됩니다.
- 그런 다음, 타겟 컴포넌트의 이벤트 핸들러가 실행됩니다.
- 마지막으로, 버블링 단계로 하위에서 상위로 전파됩니다.

---

### **React 이벤트 전파의 내부 동작**

#### 1. **이벤트 위임 설정**

17버전 이후의 React는 초기 렌더링 시 루트 DOM 노드(React DOM 컨테이너)에 이벤트 리스너를 등록합니다.

#### 2. **SyntheticEvent 생성**

이벤트가 발생하면 React는 다음 작업을 수행합니다:

1. 브라우저 이벤트를 감지.
2. 이벤트를 SyntheticEvent로 래핑하여 일관된 이벤트 객체 생성.
3. 해당 이벤트 객체를 React 컴포넌트 트리와 연관.

---

### **SyntheticEvent란?**

React의 `SyntheticEvent`는 브라우저의 네이티브 DOM 이벤트를 감싸는 추상화된 객체입니다. 이를 통해 React는 브라우저 간의 이벤트 처리 방식의 차이를 숨기고, 일관된 이벤트 처리를 제공합니다.

`SyntheticEvent`는 React가 이벤트 핸들링을 단순화하고 효율적으로 관리하기 위해 사용하는 핵심 요소 중 하나입니다.

### **SyntheticEvent의 특징 및 장점** 

1. **크로스 브라우저 호환성, 일관된 API**:
    - 브라우저 간의 이벤트 객체 차이를 감추고, 모든 브라우저에서 동일한 API를 제공합니다.
    - 예: `e.target`, `e.currentTarget`, `e.preventDefault()` 등.
2. **이벤트 풀링(Pooling)**:
    - React는 성능 최적화를 위해 `SyntheticEvent` 객체를 재사용합니다.
    - 이벤트 핸들러가 호출된 후 이벤트 객체는 초기화되고, 재사용 가능한 풀(pool)에 반환됩니다.
    - 이벤트 풀링과 위임을 통해 메모리 사용을 최소화하고, 많은 이벤트를 효율적으로 처리할 수 있습니다.
    - 초기화된 후에는 이벤트 객체의 프로퍼티에 접근하려 하면 `null`이 반환됩니다.
    - 비동기 작업에서 이벤트 객체를 바로 사용할 수 없기 때문에 필요한 경우, 이벤트 객체를 유지하려면 `event.persist()`를 호출해야 합니다. ( React 17 이후에는 이벤트 풀링의 필요성이 줄어들었습니다. React는 이전 버전과의 호환성을 위해 이벤트 풀링을 유지하고 있지만, 내부적으로는 더 이상 강제적으로 객체를 초기화하지 않고 단순하게 처리합니다. 따라서 `persist()`를 사용하지 않아도 대부분의 경우 문제없이 동작합니다. )
		```js
		function App() {
		  const handleClick = (e) => {
		    e.persist(); // 이벤트 객체를 풀에서 제거
		    console.log(e.type); // "click"
		    setTimeout(() => {
		      console.log(e.type); // 여전히 "click"
		    }, 100);
		  };
		
		  return (
		    <button onClick={handleClick}>
		      Click me
		    </button>
		  );
		}
		
		```
1. **React의 이벤트 시스템과 연동**:
    - React의 이벤트는 루트 노드에서 단일 이벤트 리스너로 처리됩니다(이벤트 위임).
    - React의 이벤트 핸들러는 기본적으로 SyntheticEvent 객체를 전달받습니다.

#### 3. **컴포넌트 트리 탐색**

React는 이벤트 발생 위치를 기준으로 컴포넌트 트리를 탐색하여, 각 컴포넌트의 캡처링 핸들러와 버블링 핸들러를 실행합니다:

1. **캡처링 단계**: 트리의 루트부터 타겟 컴포넌트까지 위쪽에서 아래로 이동하며 캡처링 핸들러 실행.
2. **타깃 단계**: 타겟 컴포넌트의 이벤트 핸들러 실행.
3. **버블링 단계**: 타겟 컴포넌트에서 루트까지 아래에서 위로 이동하며 버블링 핸들러 실행.

---

### **React의 캡처링과 버블링 예제**

```js
function App() {   

	const handleCapture = (e) => {     
		console.log("Capture Phase: ", e.currentTarget.id);   
	};    
	const handleBubble = (e) => {     
		console.log("Bubble Phase: ", e.currentTarget.id);   
	}; 
	   
	return (<div 
		id="parent" 
		onClickCapture={handleCapture} 
		onClick={handleBubble}>       
			<div 
			id="child" 
			onClickCapture={handleCapture} 
			onClick={handleBubble}>         
				<button 
				id="button" 
				onClickCapture={handleCapture} 
				onClick={handleBubble}>
					Click Me         
				</button>      
			 </div>     
		 </div>); 
}

export default App;
```

#### 출력 결과

1. 버튼 클릭 시 출력 순서:

    Capture Phase: parent 
    Capture Phase: child 
    Capture Phase: button 
    Bubble Phase: button 
    Bubble Phase: child 
    Bubble Phase: parent`
    
- **캡처링 단계**는 루트에서 타깃까지 내려오며 `onClickCapture` 실행.
- **버블링 단계**는 타깃에서 루트까지 올라가며 `onClick` 실행.

---

### **React의 이벤트 위임과 전파의 장점**

1. **성능 최적화**:
    - 모든 이벤트를 루트 컨테이너에서 처리하므로, 많은 DOM 요소에 이벤트 리스너를 개별적으로 추가하지 않아도 됩니다.
    - 이벤트 핸들링 코드가 한 곳에 집중되어 효율적으로 관리 가능.
2. **일관된 이벤트 처리**:
    - SyntheticEvent를 통해 브라우저 간의 이벤트 처리 차이를 제거하고, 모든 환경에서 동일한 이벤트 인터페이스 제공.
3. **React 컴포넌트 트리 기반 처리**:
    - React의 컴포넌트 계층 구조를 기반으로 이벤트가 처리되므로, DOM 구조와 상관없이 논리적 계층 구조를 유지.

---

### **React의 이벤트 전파와 DOM 이벤트 전파의 차이점**

| 특징               | **React 이벤트 전파**               | **DOM 이벤트 전파**              |
| ---------------- | ------------------------------ | --------------------------- |
| **전파 경로**        | React 컴포넌트 트리를 따라 전파           | DOM 요소 트리를 따라 전파            |
| **이벤트 객체**       | SyntheticEvent                 | 브라우저 네이티브 이벤트 객체            |
| **이벤트 리스너 위치**   | 루트 컨테이너에 이벤트 위임                | 각 DOM 요소에 개별 리스너 설정         |
| **브라우저 간 차이 처리** | React가 자동 처리                   | 개발자가 직접 처리해야 함              |
| **캡처링과 버블링 지원**  | `onClickCapture`, `onClick` 지원 | `addEventListener`의 옵션으로 지원 |

---

### **결론**

React의 이벤트 전파 방식은 브라우저 DOM 이벤트 전파를 기반으로 하지만, SyntheticEvent와 컴포넌트 트리를 활용하여 일관성과 성능 최적화를 제공합니다. 이를 통해 개발자는 브라우저 간 차이를 신경 쓰지 않고, React 컴포넌트 계층에서 직관적으로 이벤트를 처리할 수 있습니다.