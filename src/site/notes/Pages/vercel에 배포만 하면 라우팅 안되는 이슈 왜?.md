---
{"dg-publish":true,"permalink":"/pages/vercel/","tags":["client-side-rendering","vercel"],"created":"2024-11-10","updated":"2024-11-10T21:25:00"}
---

최근에 새로운 프로젝트를 하나 의뢰받아 vite react ts 로 개발하던 중 만난 문제를 기록해보고자 합니다. 

클라이언트 사이드 라우팅을 위해 react-router-dom으로 3개 페이지를 라우팅하는 방식을 사용하고 있었습니다.

대략적인 구조는 이렇습니다. 

```js
import { BrowserRouter as Router, Route, Routes } from "react-router-dom";
import Header from "./components/Header";
import First from "./pages/First";
import Second from "./pages/Second";
import Third from "./pages/Third";
import Footer from "./components/Footer";

function App() {

	return (
	<Router>
		<Header />
		<Routes>
			<Route path="/" Component={First} />
			<Route path="/1" Component={First} />
			<Route path="/2" Component={Second} />
			<Route path="/3" Component={Third} />
		</Routes>
		<Footer />
	</Router>
	);

}

  
export default App;

```

로컬에서 각 페이지를 잘 개발하고, Header에서 네이게이션 버튼으로 라우팅하는 것까지 잘 확인했습니다. 그런데 잘 동작하던 라우팅이 vercel로 배포한 뒤로 루트경로인 / 경로만 접근이 되고 나머지 /1, /2, /3 경로는 404에러가 뜨면서 접근이 안되는 문제가 발생했습니다.

react-router-dom의 문제라면 로컬에서도 에러가 발생해야하는데 배포된 링크에서만 라우팅이 안되고 있었습니다. 원인은 아래와 같습니다.

정적 호스팅 서비스인 Vercel은 기본적으로 정적 파일을 제공하며, 단일 페이지 애플리케이션(SPA)의 클라이언트 사이드 라우팅을 자동으로 처리하지 않습니다. SPA는 페이지 전체를 다시 로드하지 않고 클라이언트 사이드 라우팅을 사용하여 탐색을 관리합니다. 이를 위해 모든 라우트를 동일한 HTML 파일(index.html)로 제공해야 합니다. 별도의 설정 없이는 Vercel은 URL 경로와 일치하는 파일을 찾으려고 하며, 맨 처음 로드되는 루트 경로 / 외에 /1, /2, /3과 같은 클라이언트 사이드 라우트에 대해서는 404 오류를 발생시킵니다. 


이를 올바르게 처리하려면 Vercel이 정적 파일과 일치하지 않는 모든 라우트에 대해 동일한 HTML 파일로부터 경로를 찾도록 설정해야 합니다. 

프로젝트 루트 경로에 vercel.json 파일을 만들고, 
모든 라우트의 기본 경로를 index.html로 맞춰주는 작업이 필요합니다.

```json
{

"routes": [{ "src": "/[^.]+", "dest": "/", "status": 200 }]

}

```

모든 URL path를  `src`에 `"/[^.]+"` 로, 이에 대한 루트경로를  dest에 `"/"`로 맞춰주고 이에 대한 HTTP status를 200으로 설정합니다. src regex는 / 하위의 .을 제외한 모든 경로를 뜻합니다. 

순서는 이렇습니다.
최초 index.html에서 main.tsx 스크립트를 로드하고 main.tsx는 App.tsx를 로드 그 후에 App.tsx에서 작성한 클라이언트 사이드 라우트를 통해 각 page들을 로드합니다.

이렇게 하면 모든 경로가 index.html 루트경로로부터 서빙되어 404 라우팅에러를 해결할 수 있습니다. 

작성하다보니 든 생각이 나머지 유효하지 않은 라우트 /1, /2, /3 외에 다른 라우트에 대한 404페이지도 만들어야겠네요. /asdf 를 URL에 입력했을 때 아무것도 보여주지 못하니까요.

```js
import { BrowserRouter as Router, Route, Routes } from "react-router-dom";
import Header from "./components/Header";
import First from "./pages/First";
import Second from "./pages/Second";
import Third from "./pages/Third";
import Footer from "./components/Footer";

function App() {

	return (
	<Router>
		<Header />
		<Routes>
			<Route path="/" Component={First} />
			<Route path="/1" Component={First} />
			<Route path="/2" Component={Second} />
			<Route path="/3" Component={Third} />
			<Route path="/*" Component={NotFound} />
		</Routes>
		<Footer />
	</Router>
	);

}

  
export default App;

```