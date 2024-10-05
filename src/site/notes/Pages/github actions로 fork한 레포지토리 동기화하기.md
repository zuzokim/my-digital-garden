---
{"dg-publish":true,"permalink":"/pages/github-actions-fork/","created":"2024-09-16","updated":"2024-10-05T22:03:00"}
---

사이드 프로젝트 [[Projects/SqetchClub/sqetch.club\|sqetch.club]] 를 하는 중에 팀 레포지토리를 organization으로 야심차게 만들었습니다. FE는 Vercel을 연동해 main 브랜치에 머지되면 배포가 되도록 해두었는데, 개인 레포지토리가 아니라 organization 레포지토리는 유료로 배포가 되는 걸 뒤늦게 알아버렸습니다. ^^; 사실 쫌쫌따리 작고 조그마한 사이드 프로젝트를 출시도 전에 유료 배포로 사용한다는게 좀 부담스러워서 organization 레포지토리를 개인 레포지토리로 fork하고 이 레포지토리를 Vercel에 연동해 우회하는 방법을 사용하기로 했습니다. 

그런데 매번 두 레포지토리를 remote(깃헙)에서 수동으로 동기화를 해주어야하는게 좀 성가신 일이 되었습니다. ~~나중에 돈 많이 벌어서 유료 배포하고 싶네요~~ 
같이 하는 팀원의 제안으로 FE 배포를 자동화하는 github actions workflow를 만들어보기로 했습니다. 회사에서는 이미 작성된 workflow에 약간의 수정을 하는 정도로만 시도해봤는데, 아예 새롭게 yaml 파일부터 만드려니 어색하더군요. 하나씩 의미를 따져보며 작성해본 내용을 포스트해보겠습니다.


```yaml
name: Push to Forked

```
일단 workflow의 이름이 필요합니다. fork뜬 레포지토리에 push하는게 목표이므로, Push to Forked라는 이름을 지어봤습니다.

```yaml
on:
	push:
		branches:
			- main
```

언제 이 workflow를 실행할 것인지를 나타내는 on: 에 main 브랜치에 push할 때마다 실행한다고 작성합니다.


```yaml
jobs:
	push-to-forked:
		runs-on: ubuntu-latest
```
jobs: 는 Push to Forked workflow에서 실행시킬 job들의 목록을 나타냅니다. 우분투 환경에서 실행될 수 있게 작성해봤습니다.

그다음은 job들의 실행순서를 아래와 같이 steps: 로 나열할 수 있습니다. step마다 이름을 가질 수 있습니다.
```yaml
		steps:
			-name: sth
			-name: sth
			-name: sth
			....
```

여기까지는 일반적인 세팅이라고 볼 수 있습니다. 이제 이 steps에 어떤 job을 실행시켜야하는지 보겠습니다.
1. 일단 org 레포지토리의 main 브랜치에 checkout 하고
2. 거기서 fork한 레포지토리에 접근해서
3. org 레포지토리의 main브랜치를 fork한 레포지토리에 push

1번
```yaml
		steps:
			-name: Checkout repository
			uses: actions/checkout@v2
			with:
				ref: main
				fetch-depth: 0
```
ref: main 브랜치에 checkout합니다. 여기서 주의점 fetch-depth:0을 명시해줘야 모든 브랜치와 커밋 및 태그 히스토리를 fetch할 수 있습니다. 그렇지 않으면 가장 마지막 커밋만 가져옵니다. 추후에 submodules를 사용하거나 changelogs 작성을 자동화할 일이 생길 수도 있고 커밋히스토리를 보고 싶을 수 있는데, 이때 문제가 생길 수 있습니다. org 레포지토리와 forked 레포지토리의 git history를 똑같이 클론하는게 목적이기 때문에 잘 챙겨서 모든 history를 fetch하도록 작성해줍니다.
actions summary에 들어가서보면 history가 모두 잘 fetch 된 것을 볼 수 있습니다.
![Screen Shot 2024-10-05 at 11.09.26 PM.png](/img/user/Screen%20Shot%202024-10-05%20at%2011.09.26%20PM.png)

2번
```yaml
		steps:
			-name: Set up Git
			run: |
				git config user.name {user name}
				git config user.email {user email}
				git remote add forked {forked repo url}
```

uses: 로는 만들어진 job를 가져와서 쓸 수 있고, run:으로는 커맨드를 실제로 작성해서 쓸 수 있습니다.
fork한 레포에 접근하기 위해서 user정보와 url로 레포지토리를 forked라는 이름으로 remote에 추가합니다.

3번
```yaml
		steps:
			-name: Push to Forked
			run: |
				git push forked main:main
```
forked에 실제로 푸시하는 step 입니다. org main브랜치에서 fork한 레포지토리의 main브랜치로 push하도록 작성했습니다. (fork한 레포지토리 main에 변경사항이 생길때마다 Vercel배포가 되도록 설정해두었습니다.)

그런데 여기서 action이 failed 됩니다. remote forked 레포지토리에 접근권한이 없다고 하네요.

```bash
remote: Permission to zuzokim/sqetchclub-frontend.git denied

[8](https://github.com/SqetchClub/sqetchclub-frontend/actions/runs/000000000/job/0000000000#step:4:9)fatal: unable to access '[https://github.com/zuzokim/sqetchclub-fronted.git/](https://github.com/zuzokim/sqetchclub-frontend.git/)': The requested URL returned error: 403

[9](https://github.com/SqetchClub/sqetchclub-frontend/actions/runs/00000000/job/000000000#step:4:10)Error: Process completed with exit code 128.
```

권한 부여를 위해 PAT(personal access token)을 새로 발급받아서 org 레포지토리에 등록해주어야합니다. 저는 org와 개인 레포지토리에 접근할 수 있지만 workflow상으로는 sqetchclub org가 전혀 다른 레포지토리에 접근하는 것이기 때문에 어쩌면 당연히 필요한 토큰 설정인걸 빠트렸네요.

2번을 다시 작성해줍니다.
```yaml
		steps:
			-name: Set up Git
			run: |
				git config user.name {user name}
				git config user.email {user email}
				git remote add forked https://{user name}:${FORK_PAT}@github.com/{user name}/{repository name}.git
```


이제 잘 되겠지했는데 또 접근 권한 에러가 납니다. 심지어 에러 메세지도 동일합니다. 토큰도 발급받아 등록했는데 왜 안될까 구글링을 해보고 알게 된 것은 기본적으로 actions/checkout할 때 자동으로 발급된 디폴트 GITHUB_TOKEN이 사용됩니다. 그러니까 org의 토큰이 사용되는 것이고, forked url에 PAT을 명시하더라도 workflow가 실행되는 환경에서는 해당 레포지토리에 접근 권한이 없는 상태가 되는 것이죠. 

이를 해결하려면 forked PAT을 가지고 checkout할 수 있도록 1번 job을 수정할 수 있습니다.

1번 수정
```yaml
		steps:
			-name: Checkout repository
			uses: actions/checkout@v2
			with:
				token: ${{secrets.FORK_PAT}}
				ref: main
				fetch-depth: 0
```
with: 로 등록해둔 FORK_PAT을 사용하도록 해줍니다. secrets에 접근해서 가져올 수 있습니다.

드디어 성공!

![Screen Shot 2024-10-05 at 11.12.40 PM.png](/img/user/Screen%20Shot%202024-10-05%20at%2011.12.40%20PM.png)

추가로 CI환경에서 remote로 add했던 forked 레포지토리를 지워주는 step를 작성해 마무리합니다.
```yaml
		steps:
			-name: Clean up
			run: |
				git remote remote forked
```
clean up step을 작성하면 좋은 이유는 다음과 같습니다.
1. 보안 : remote 레포지토리를 지워줌으로써 PAT이 action runner config에 남아있지 않도록 합니다. 민감한 토큰정보가 노출될 위험을 방지합니다.
2. 깨끗한 상태 유지 : action이 돌 때마다 처음의 깨끗한 상태로 시작할 수 있습니다. remote를 제거하면 이후 추가적인 action이나 다른 workflows가 remote 레포지토리에 의도치 않게 접근하게 되는 사이드이펙트를 방지할 수 있습니다.
3. 리소스 관리 : 해당 workfow가 끝나면 더이상 접근할 필요없는 remote 레포지토리이므로 지워줌으로써 깔끔하게 환경을 유지해줄 수 있습니다.

Vercel 무료배포를 위해 시작한 github actions작성이었지만, 여러가지를 알게 됐네요. git history를 모두 fetching하기, 레포에서 다른 레포에 접근하기, workflow에서 필수적으로 챙기면 좋을 step 작성하기 등등. 이제 덜 수고롭게 FE 배포할 수 있게 됐습니다 :) 