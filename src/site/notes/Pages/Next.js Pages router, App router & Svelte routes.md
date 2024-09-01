---
{"dg-publish":true,"permalink":"/pages/next-js-pages-router-app-router-and-svelte-routes/","tags":["routes","Nextjs","Sveltekit"],"created":"2024-08-29","updated":"2024-09-01T16:11:00"}
---

## Intro

최근에 Sveltekit + Phoenix 조합으로 사이드 프로젝트[^sqetchclub]를 하나 만들고 있습니다. 
회사에서도 Next.js 13으로 버전업하면서 Pages router 에서 App router로 점진적으로 마이그레이션을 하고 있고, 새로운 기술스택 Sveltekit의 routes 방식을 비교해보려고 합니다.

세 가지 라우팅 방식 다 공식 문서에 아주 자세히 설명되어 있어서 따라가면서 배우기 좋은데 비슷한 점도 많고 다른 점도 있어서 각각 보기보다는 서로 비교하면서 이해하면 더 도움이 될 것 같아 정리해보기로 했습니다.

- [Next.js Pages router](https://nextjs.org/docs/pages/building-your-application/routing)
- [Next.js App router](https://nextjs.org/docs/app/building-your-application/routing)
- Sveltekit routes


### Next.js Pages Router
- #### 전통적인 file-based 라우팅 시스템입니다.
- `pages/` 디렉토리 하위로 파일을 만들면 경로가 됩니다.
```js
pages/home/page.js -> `/home`
pages/logs/page.js -> `/logs`
```
#### 동적 라우팅(Dynamic Segments)
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
- #### Layouts and Nested Routing
- <Layout /> 컴포넌트를 만들고 children으로 page들을 넣어서 공통 레이아웃을 다룰 수 있습니다.
- 네이티브한 방식으로 중첩된 레이아웃을 지원하지는 않습니다.


### Next.js App Router
- #### Next.js 13부터 도입된 라우팅 시스템입니다. React Server Components를 기반으로 보다 유연한 접근 방식을 염두에 두고 만들어졌습니다.
- `app/` 디렉토리 하위의 파일 구조가 URL 구조를 반영합니다. 
```js
app/home/page.js -> `/home`
app/logs/page.js -> `/logs`
```
- #### 동적 라우팅(Dynamic Segments)
	- App router에서 동적 라우팅 작성법은 Pages Router에서와 동일합니다.

[^sqetchclub]: [[Projects/WIP projects & workshops/SqetchClub/sqetch.club\|sqetch.club]]
