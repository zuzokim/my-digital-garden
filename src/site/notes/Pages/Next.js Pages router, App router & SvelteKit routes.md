---
{"dg-publish":true,"permalink":"/pages/next-js-pages-router-app-router-and-svelte-kit-routes/","tags":["routes","Nextjs","SvelteKit","blog"],"created":"2024-08-29","updated":"2024-09-01T16:11:00"}
---

## Intro

최근에 Sveltekit + Phoenix 조합으로 사이드 프로젝트[^sqetchclub]를 하나 만들고 있습니다. 
회사에서도 Next.js 13으로 버전업하면서 Pages router 에서 App router로 점진적으로 마이그레이션을 하고 있고, 새로운 기술스택 Sveltekit의 routes 방식을 비교해보려고 합니다.

세 가지 라우팅 방식 다 공식 문서에 아주 자세히 설명되어 있어서 따라가면서 배우기 좋은데 비슷한 점도 많고 다른 점도 있어서 각각 보기보다는 서로 비교하면서 이해하면 더 도움이 될 것 같아 정리해보기로 했습니다.

- [Next.js Pages router](https://nextjs.org/docs/pages/building-your-application/routing)
- [Next.js App router](https://nextjs.org/docs/app/building-your-application/routing)
- [SvelteKit routes](https://kit.svelte.dev/docs/routing)


### Next.js Pages Router
- 전통적인 file-based 라우팅 시스템입니다.
- `pages/` 디렉토리 하위로 파일을 만들면 경로가 됩니다.
```js
pages/home/page.js -> `/home`
pages/logs/page.js -> `/logs`
```
- 동적 라우팅(Dynamic Segments)
- 폴더/파일명을 대괄호로 감싸서 만들면 됩니다.
```js
pages/logs/[slug]/page.js -> `/logs/1` , `logs/2`, `logs/digital-garden-logs`
```

- Catch-all Segments 
	- ... 를 사용하면 여러 segments를 모두 매칭시키는 경로도 만들 수 있습니다.
```js
pages/shop/[...slug].js -> `/pages/shop/clothes`, `/pages/shop/clothes/tops` , `pages/shop/clothes/tops/t-shirts`
```
- Optional Catch-all Segments
	- 대괄호 두개를 중첩해 작성하면 여러 segments를 일부만 매칭시키는 경로도 만들 수 있습니다.
	- Catch-all Segments와 다르게 slug가 undefined일 때 pages/shop에 대한 페이지도 유효한 경로로 만들 수 있습니다.  

```js
pages/shop/[[...slug]].js -> `/pages/shop`, `/pages/shop/clothes`, , `/pages/shop/clothes/tops` , `pages/shop/clothes/tops/t-shirts`
```

- Layouts and Nested Routing
	- <Layout /> 컴포넌트를 만들고 children으로 page들을 넣어서 공통 레이아웃을 다룰 수 있습니다.
	- 네이티브한 방식으로 중첩된 레이아웃을 지원하지는 않습니다.


### Next.js App Router
- Next.js 13부터 도입된 라우팅 시스템입니다. React Server Components를 기반으로 보다 유연한 접근 방식을 염두에 두고 만들어졌습니다.
- `app/` 디렉토리 하위의 파일 구조가 URL 구조를 반영합니다. 
```js
app/home/page.js -> `/home`
app/logs/page.js -> `/logs`
```
- 동적 라우팅(Dynamic Segments)
	- App router에서 동적 라우팅 작성법은 Pages Router에서와 동일합니다.

- Layouts and Nested Routing
	- 경로에 `layout.js` 파일을 만들면 페이지마다 공유하는 레이아웃을 다룰 수 있습니다. 
		- root 레이아웃은 app 디렉토리 최상위 레벨의 layout.js `app/layout.js` 에 필수로 정의하고, 모든 페이지, 컴포넌트와 레이아웃에 적용됩니다.
		- 서버에서 반환한 초기 html을 수정할 수 있도록 root layout은 반드시 `html` 과 `body`태그가 포함되어있어야 합니다.
	- 중첩된 디렉토리 하위로도 layout을 만들 수 있고, 기본적으로 부모 레이아웃을 자식 레이아웃이 상속받습니다. 
		![Screen Shot 2024-09-01 at 6.49.54 PM.png](/img/user/Screen%20Shot%202024-09-01%20at%206.49.54%20PM.png)
	- Pages router와 다른 점은 괄호로 감싼 Route Groups인데, 파일구조가 URL 구조에 영향을 미치지 않도록 하고 싶을 때 사용할 수 있습니다. 예를 들어, `(marketing)` 과 `(shop)` 은 같은 레벨의 URL 구조를 가지지만 하위에 `layout.js`를 따로 작성해 각기 다른 레이아웃을 별도로 적용할 수 있습니다.
		 ![Screen Shot 2024-09-01 at 6.31.26 PM.png](/img/user/Screen%20Shot%202024-09-01%20at%206.31.26%20PM.png)![Screen Shot 2024-09-01 at 7.01.55 PM.png](/img/user/Screen%20Shot%202024-09-01%20at%207.01.55%20PM.png)

	- 만약, root 레이아웃을 아예 상속받고 싶지 않은 경우에는 `app/`하위의 루트경로에 괄호로 감싼 route groups를 따로 만들고, 각기 다른 root layout을 만들면 됩니다. 대신 위에서 말했던 것처럼 각각 `html` 과 `body`태그가 포함되어있어야 합니다.
	- Layout은 기본적으로 Server Component이며, Client Component로 세팅할 수 있습니다.


### SvelteKit routes
- Svelte의 동시성(reactive) 패러다임을 기반으로한 file based 라우팅 시스템입니다.
	- `src/routes/` 하위로 파일을 만들면 경로가 됩니다.
```js
src/routes/+page.svelte -> `/`
src/routes/logs/+page.svelte -> `/logs`
```
- 동적 라우팅
```js
src/routes/home/+page.svelte -> `/home`
arc/routes/logs/[slug]/+page.svelte -> `/logs/digital-garden-log`
```
 - Layouts and Nested Routing
	 -  app router처럼 네이티브 레이아웃을 제공합니다. `+layout.svelte` 파일을 원하는 디렉토리 하위로 작성하면 해당 디렉토리의 모든 경로에 레이아웃을 적용할 수 있습니다.
	 - app router처럼 여러 레벨에서 각각 다른 레이아웃을 만들 수 있습니다.
	
- Page Options
	- SvelteKit는 기본적으로 서버에서 컴포넌트를 먼저 렌더 혹은 pre-render한 후 HTML로 클라이언트에 보내줍니다. 그리고 hydration[^hydration]을 거쳐 브라우저에서 컴포넌트가 상호작용 가능하도록 다시 렌더하게 됩니다. 모든 페이지가 server 혹은 client에서 동작할 수 있게 해야하는데 이를 page option을 통해 조정할 수 있습니다.
- `+page.server.js`로 작성한 페이지의 `load` function은 서버에서 실행됩니다. (db에서 데이터를 fetching하거나 하는 경우)
	- client side 네비게이션을 하는 동안에 SvelteKit는 서버에서 해당 데이터를 가져오게 됩니다.
	- 예를 들어 `npm create svelte`로 만든 스타터앱의 `sverdle/+page.server.ts` 파일에 보면 서버에서 실행되는 actions를 export하고 있는 걸 볼 수 있습니다. [^page.server.ts]
		![Screen Shot 2024-09-01 at 9.20.43 PM.png](/img/user/Screen%20Shot%202024-09-01%20at%209.20.43%20PM.png)
		그리고 `+page.svelte` 파일에도 동일한 keypress 이벤트를 받는 action 로직이 있는 걸 볼 수 있습니다. [^page.svelte]
		![Screen Shot 2024-09-01 at 9.23.34 PM.png](/img/user/Screen%20Shot%202024-09-01%20at%209.23.34%20PM.png)
		크롬브라우저 개발자 도구 - Settings - Debugger - Disable Javascript를 체크하고 js hydration을 막아둔 뒤 sverdle을 플레이하면 `+page.svelte` 의 update 액션은 실행되지 않고, 서버의 update 액션이 실행되는 것을 볼 수 있습니다. URL 경로가 `http://localhost:5173/sverdle?/update` 로 변경되어 보이면서 keypress 이벤트가 발생할 때마다 서버 요청이 발생하는 것을 발견할 수 있습니다. 
		- SvelteKit에서 form이 제출되면 form에 해당하는 action이 실행되고, URL은 action을 query parameter로 포함합니다. 
		- 클라이언트 환경이 좋지 못할 때에도 플레이 기능이 fallback으로 동작할 수 있는 이점이 생겨납니다.
	
- +layout.js & +layout.server.js
	- `+layout.server.js`로 작성한 레이아웃도 마찬가지로 서버에서 실행됩니다. 클라이언트에 노출되면 안되는 보안api나 서버사이드 인증이 필요할 때 사용할 수 있습니다.
	- SvelteKit는 두 가지 레이아웃을 다 사용할 수 있는데, 서버사이드 레이아웃의 로직을 먼저 실행하고나서 클라이언트 사이드 로직을 추가적으로 실행합니다.
	![Screen Shot 2024-09-01 at 8.41.59 PM.png](/img/user/Screen%20Shot%202024-09-01%20at%208.41.59%20PM.png)


## Outro
사이드 프로젝트 초기 스트럭쳐를 잡기로 하면서 SvelteKit 기본 구조와 Nextjs 라우팅 구조와 자연스럽게 비교하게 됐습니다. 공부도 할겸 간단하게 세 가지 라우팅 방식을 비교 정리해봤습니다. 각자 다르면서 닮은 방식을 가지고 있는데, 특히나 App router와 SvelteKit의 라우팅 방식이 멘탈모델이 비슷한 결을 가지고 있다는 느낌을 받았네요. 더 자세한 스펙과 라우팅 - data fetching, error, pre-render, optimization, middleware... - 에 관해서는 나중에 다른 포스트로 또 다뤄보도록 하겠습니다.
	![Screen Shot 2024-09-01 at 9.42.40 PM.png](/img/user/Screen%20Shot%202024-09-01%20at%209.42.40%20PM.png)


[^sqetchclub]: [[Projects/SqetchClub/sqetch.club\|sqetch.club]]
[^hydration]: https://kit.svelte.dev/docs/glossary#hydration
[^page.server.ts]: https://github.com/sveltejs/kit/blob/108cb127eb0a4c3655aeff1f749bcbd98b70324e/packages/create-svelte/templates/default/src/routes/sverdle/%2Bpage.server.ts#L29-#L49
[^page.svelte]: https://github.com/sveltejs/kit/blob/108cb127eb0a4c3655aeff1f749bcbd98b70324e/packages/create-svelte/templates/default/src/routes/sverdle/%2Bpage.svelte#L60-#L76
