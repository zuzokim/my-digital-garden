---
{"dg-publish":true,"permalink":"/posts/cors-feat-proxy/","created":"2025-02-16","updated":"2025-02-16T22:24:00"}
---

글을 시작하기 전에 CORS에러란 무엇인지 작성해보겠습니다.

## CORS (Cross-Origin-Resource-Sharing) 에러란?
CORS에러는 **웹 브라우저의 보안 정책(Same-Origin Policy, SOP)** 때문에 발생하는 에러입니다. 웹에서 보안상의 이유로, 한 도메인(origin)의 웹 페이지가 다른 도메인의 리소스에 접근하는 것을 제한하는 정책이 존재하는데, 이를 **동일 출처 정책(SOP)**이라고 합니다.

CORS(Cross-Origin Resource Sharing)는 이러한 제한을 완화하기 위한 메커니즘이지만, 적절한 설정이 없을 경우 브라우저가 차단하면서 CORS 에러가 발생합니다.

그럼 **origin** 은 정확히 뭐냐?
웹 브라우저는 기본적으로 **다른 출처(Origin)의 요청**을 차단합니다. 여기서 출처(Origin)는 다음 세 가지 요소로 결정됩니다.

- **프로토콜 (Protocol)** → `http://`, `https://`
- **호스트 (Host)** → `example.com`, `api.example.com`
- **포트 번호 (Port)** → `:3000`, `:8080`

즉, 출처(`Origin`)이 다르면 CORS 정책이 적용되며, 서버가 CORS 요청을 허용하지 않으면 요청이 차단됩니다. 예를 들어, 다음과 같은 상황에서 CORS 에러가 발생할 수 있습니다.

✅ **출처가 같은 경우 (CORS 에러 없음)**

> 프론트엔드: `http://example.com`  
> 백엔드: `http://example.com`  

❌ **출처가 다른 경우 (CORS 에러 발생 가능)**

> 프론트엔드: `http://example.com` 
> 백엔드: `http://api.example.com` (서브도메인 다름) 
> 
> 프론트엔드: `http://localhost:3000` 
> 백엔드: `http://localhost:5000` (포트 번호 다름)

---

브라우저가 데이터를 로딩하는 과정에서 **CORS와 Origin 헤더**가 어떻게 작동하는지 조금 더 자세히 단계별로 작성해보겠습니다.

#### 📌 1. 브라우저가 HTML 문서를 요청 (초기 로딩 과정)

웹 브라우저에서 유저가 특정 웹사이트(`https://frontend.com`)를 방문하면, 먼저 해당 사이트의 HTML 문서를 요청합니다.

ex:
- **유저가 `https://frontend.com`에 접속**
- **웹 서버가 HTML 문서를 반환**

브라우저의 HTTP 요청:
```http
GET / HTTP/1.1
Host: frontend.com
```

서버의 응답(HTML 문서 반환)
```http
HTTP/1.1 200 OK
Content-Type: text/html

<!DOCTYPE html>
<html lang="ko">
<head>
    <title>My Frontend App</title>
</head>
<body>
    <script src="/app.js"></script>
</body>
</html>
```

이 시점에서 브라우저는 **자신의 Origin을 `https://frontend.com`으로 설정**합니다.  
즉, 이후 브라우저에서 실행되는 모든 JavaScript는 기본적으로 **`Origin: https://frontend.com`** 을 가지게 됩니다.

#### 📌 2. 브라우저가 추가 리소스 (CSS, JS, API) 요청

이제 브라우저는 HTML을 해석하고 추가적인 리소스(스타일, 스크립트, 이미지 등)를 로드합니다. 
예를 들어, `<script src="/app.js">`태그가 있으면 브라우저는 다음과 같은 요청을 보냅니다.

```http
GET /app.js HTTP/1.1
Host: frontend.com
Origin: https://frontend.com
```

이때, CSS, JS 파일은 같은 Origin에서 로드되기 때문에 CORS 이슈가 없습니다.

#### 📌 3. 브라우저에서 API 요청 (CORS 적용)

이제 **프론트엔드의 JavaScript 코드가 백엔드 API 요청**을 보낼 수 있습니다.

```js
fetch('https://api.backend.com/data', {
    method: 'GET'
})
```

이 요청은 브라우저에서 실행되므로 **자동으로 `Origin` 헤더가 포함**됩니다.

```http
GET /data HTTP/1.1
Host: api.backend.com
Origin: https://frontend.com
```

여기서 `Origin: https://frontend.com`은 **브라우저가 HTML을 처음 받은 출처**를 기반으로 설정된 것입니다.  
즉, **웹사이트가 처음 로드될 때 설정된 Origin이 이후 모든 요청에 영향을 미치게 됩니다.**

#### 📌 4. 백엔드의 CORS 응답 처리

보통은 백엔드(`https://api.backend.com`)가 응답을 보낼 때, 브라우저가 해당 요청을 허용할지 결정할 수 있도록 **CORS 헤더를 추가합니다.**

```js
const cors = require('cors');
const express = require('express');
const app = express();

app.use(cors()); // 모든 요청 허용

// 특정 도메인만 허용할 수도 있음
app.use(cors({ origin: 'https://frontend.com' }));

app.listen(5000, () => console.log('Server running on port 5000'));
```

```http
HTTP/1.1 200 OK
Access-Control-Allow-Origin: https://frontend.com
Content-Type: application/json 

{"message": "Success"}
```

이렇게 하면 브라우저는 백엔드에서 반환된 데이터를 정상적으로 처리할 수 있습니다. 
하지만 만약 백엔드가 `Access-Control-Allow-Origin` 헤더를 포함하지 않으면, 브라우저는 응답을 차단하고 **CORS 에러**를 발생시키게 됩니다.

```http
HTTP/1.1 403 Forbidden
```

```http
Access to fetch at 'https://api.backend.com/data' from origin 'https://frontend.com'
has been blocked by CORS policy: No 'Access-Control-Allow-Origin' header is present 
on the requested resource.
```

---

## 그런데 백엔드 개발자가 없거나, CORS 설정 권한이 없다면?

그럴 때는 어떻게 해결할 수 있을까요? 실제로 최근에 외부 API에 요청을 보내 로컬에서 개발할 일이 있었는데, 해당 백엔드 서버를 수정할 수 있는 권한이 없는 경우가 있었습니다. 이런 경우 프론트엔드에서 해결할 수 있는 방법이 필요했는데요, 관련해서 구글링과 ai의 도움을 받아 해결한 내용을 작성해보겠습니다.

### Proxy로 우회하기

두괄식으로 해결한 방법을 먼저 작성해봤습니다. 백엔드 코드를 수정할 수 없는 경우 프론트엔드에서 **프록시 서버 설정**으로 CORS 에러를 우회할 수 있는데요, 이는 브라우저가 백엔드 API가 아닌 프록시 서버를 대상으로 요청하도록 만들어서 CORS 정책을 피하는 접근방식입니다.

일단 제가 작업하던 개발 환경을 기준으로 과정을 설명해보겠습니다.

프로젝트는 Vite + React로 생성해 작업중이었고, `vite.config.ts` 설정 파일에 프록시 서버를 설정할 수 있었습니다.

```js
import { defineConfig } from 'vite';

export default defineConfig({
  server: {
    proxy: {
      '/api': {
	    //target: 요청을 전달할 원본 서버 (백엔드)
        target: 'https://api.backend.com',
        //changeOrigin: 원본 서버의 호스트 헤더를 변경
        changeOrigin: true,
        // rewrite: 경로 변경
        rewrite: (path) => path.replace(/^\/api/, ''),
        // secure: HTTPS 인증서 무시
        secure: false,
      }
    }
  }
});
```

이렇게 설정후 브라우저에서 요청을 날려보면 

```http

GET http://localhost:5173/api/data // 브라우저 요청

GET https://api.backend.com/api/data // Vite가 변환한 요청 (백엔드로 전달)
```

Vite가 자동으로 `https://api.backend.com/data`로 요청을 전달해주는 방식으로 동작하는 걸 볼 수 있습니다.

```http
// changeOrigin: false (원래 요청 헤더 유지)
Host: localhost:5173

// changeOrigin: true (대상 서버의 호스트로 변경)
Host: api.backend.com
```

`changeOrigin 을 true` 로 설정하면 원래 요청의 Host 헤더를 `target` 서버 도메인으로 변경해줍니다.
따라서 프록시 서버가 직접 타겟 백엔드 서버로 요청하는 것처럼 동작하게 되는 것입니다.

`rewrite` 부분의 정규식은 `/api` 로 시작하는 경로를 빈스트링(' ') 으로 바꿔주는 역할을 합니다. 

```js
// 프론트엔드에서 요청
fetch('/api/data')

// rewrite 적용 전 (잘못된 요청)
GET https://api.backend.com/api/data

// rewrite 적용 후 (정상 요청)
GET https://api.backend.com/data

```

`secure 을 false` 로 설정하면 HTTPS 연결시 SSL 인증서를 검사하지 않겠다는 뜻입니다. false는 최대한 개발환경에서만 사용하고 프로덕션 환경에서는 기본값인 true로 유지하는게 좋다고 합니다.

#### 잠깐, 🔐 SSL(HTTPS) 인증서 개념 & `secure` 옵션 이해하기

##### **✅ SSL/TLS란? (HTTPS와의 관계)**

SSL(Secure Sockets Layer)과 TLS(Transport Layer Security)는 **인터넷에서 데이터를 암호화하여 안전하게 주고받기 위한 보안 프로토콜**입니다.  우리가 흔히 사용하는 **HTTPS(보안 HTTP)**가 이 SSL/TLS를 이용하여 데이터를 보호합니다.

브라우저에서 `https://`로 시작하는 웹사이트에 접속하면, 그 사이트는 **SSL/TLS 인증서를 가지고 있다**는 의미입니다.

##### **✅ SSL 인증서의 역할**

SSL 인증서는 다음과 같은 세 가지 주요 기능을 수행합니다.

1️⃣ **데이터 암호화**
- 클라이언트(브라우저)와 서버 간 전송되는 데이터를 **암호화**하여, 중간에서 해커가 가로채더라도 내용을 알아볼 수 없도록 합니다.
- 예) 로그인 정보, 신용카드 정보 등
2️⃣ **서버 인증 (신뢰성 보장)**
- 사용자가 접속한 서버가 **진짜 서버인지 확인**하는 역할을 합니다.
- 예) 가짜 은행 사이트(피싱 사이트)에 속지 않도록 방지합니다.
3️⃣ **데이터 무결성**
- 데이터를 주고받을 때 **중간에서 변조되지 않았음을 보장**합니다.


---

이렇게 정상적인 외부 API로의 요청을 해결했습니다. 항상 CORS 설정은 백엔드 개발자와 소통해서 해결해서 프론트엔드에서는 할 수 있는 것이 없다고 생각했는데, 우회할 수 있는 방법이 있다는 걸 알게 됐네요.

### 스토리북(Storybook)에서 또 CORS 에러가?

이제 로컬에서 실제 앱 실행 후 요청까지 성공했는데, 컴포넌트 관리를 위해 사용하고 있는 스토리북에서 또 CORS에러를 마주하게 됐습니다. 앞선 내용에서 배웠든 오리진 중 포트넘버가 6006으로 달라서 생기는 문제였습니다.

스토리북에서도 마찬가지로 설정이 필요했는데, 스토리북 내부의 자체적인 동작방식의 영향으로 살짝 다른 방식으로 config를 작성해줘야했습니다. 아래는 작성한 `.storybook/main.ts` 입니다.

```js

const config: StorybookConfig = {
	stories: ["../src/**/*.stories.tsx"],
	framework: {
		name: "@storybook/react-vite",
	},
	async viteFinal(config, { configType = "DEVELOPMENT" }) {
	  const env = loadEnv(configType, process.cwd(), "");
	  
	  return {
	    ...config,
	    server: {
	      ...config.server,
	      proxy: {
	        "/api": {
	          target:
	            env.VITE_API_BASE_URL ||
	            "https://api.backend.com",
	          changeOrigin: true,
	          rewrite: (path) => path.replace(/^\/api/, ""),
	          secure: false,
	        },
	      },
	    },
	  };
	}
}
```

스토리북은 기본적으로 Webpack을 사용하여 개발서버를 실행합니다. 그러나 Vite를 사용할 때는 framework에 Vite를 사용한다고 패키지를 설치한 후 명시해줘야합니다. 그리고 viteFinal을 사용하여 최종적으로 Webpack -> Vite 설정으로 변경해줄 수 있습니다.


그럼 스토리북에서도 CORS 해결!
