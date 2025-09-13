---
{"dg-publish":true,"permalink":"/posts/dev/corepack-package-manager/","created":"2025-09-06","updated":"2025-09-07T00:52:00"}
---

참여하고 있는 프로젝트 B-Peach LAB 프론트엔드 프로젝트 유지보수와 고도화를 위해 Storybook과 디자인 시스템을 도입하기로 하면서 겪은 문제를 해결하고, 배운 점을 기록한다.

기존에 인수인계 받은 프로젝트에는 Storybook이 없어서 컴포넌트 유지보수, 유닛테스트가 용이하지 않은 구조였다. 그래서 올해 6월 새로운 버전9가 출시되기도 했고, 추후에 나말고도 기여하게 될 엔지니어들을 위해 Storybook9를 설치하고자 했다.

Storybook9는 20이상의 node 버전을 요구한다. 그런데 프로젝트에는 .nvmrc 가 없어서 어떤 node 버전을 사용하고 있는지 알기가 어려웠다. 협업의 관점에서도 node 버전을 고정하는 것이 좋기 때문에 먼저 .nvmrc 파일을 추가하고 node 버전을 22버전을 사용하도록 명시했다.

그리고서 로컬에서 문제없이 Storybook9을 install하고, 여러가지 프로젝트에 맞는 config를 맞춘 뒤 PR을 올렸다. 그런데 여기서 문제가 생겼다. 로컬에서 빌드, 실행 모두 성공하고 잘 동작하던 프로젝트가 Github Actions CI에서만 실패했다. 심지어 pacakge-lock.json 기준으로 의존성을 install할 때부터 실패해버렸다.

에러 로그를 살펴보니, 최근에 추가된 devDependencies 몇 개가 있었는데 그 패키지들의 peerDependencies를 resolve할 수 없다는 의존성 충돌 에러였다. 곤란하다. 직접 설치한 의존성의 경우엔 호환이 가능한 버전을 package.json에 고정적으로 명시해 올바르게 재설치하면 됐지만, 이건 의존성의 의존성, 심지어 의존성의 의존성의 의존성.. 어떤 패키지의 dependencies 인지 파악하는 것부터가 오래걸렸다. 게다가 어떻게 해결해야 좋을지도 막막했다.

차근차근 해결 과정을 기록해보자.

---

##### 로컬에서 의존성을 설치할 때는 `npm install` 을 사용 vs Github Actions CI환경에서 의존성을 설치할 때는 `npm ci` 을 사용

- 로컬에서 npm install을 하면 package.json을 기준으로 의존성을 설치하고, package-lock.json의 diff를 만든다.
- CI 환경에서는 package-lock.json을 기준으로 npm ci로 의존성을 설치한다. 

이 둘의 차이에서 문제가 발생한다.

-  `npm install`
	- package.json을 기준으로 의존성을 설치
	- 동작:  
		1. package.json 읽음
		2. package-lock.json 있으면 참고해서 최대한 맞춤
		3. 안 맞으면 lockfile 업데이트 (diff 발생 가능)
		4. 최종적으로 node_modules 생성
	- lockfile을 수정할 수 있음 → CI에서 쓰면 “빌드마다 lockfile이 달라지는” 혼란 발생
-   `npm ci` (continuous integration 모드)
	- package-lock.json을 진실(SOT, single source of truth)로 보고 설치
	- 동작:  
		1. package.json과 package-lock.json 불일치하면 바로 에러 (설치 중단)
		2. node_modules 폴더 통째로 지우고 → lockfile 그대로 재구축
		3. lockfile 절대 수정하지 않음
	-   완전히 결정적(deterministic) 설치 → 어떤 환경에서 돌려도 항상 동일한 결과

node 버전을 22로 올리면서 내장 npm 버전도 달라지게 됨 + 최근에 Storybook9 포함 새로운 패키지를 추가함에 따라

- 달라진 패키지매니저 npm 버전에 따라 의존성 트리 해석 방식이나 포맷이 달라지게 됨
- 추가된 새로운 패키지들의 복잡한 peerDependencies들이 달라지면서 package-lock.json이 업데이트됨

이렇게 되는 것이 원인이었다.

---

좀 더 정리를 해보면 이렇다.

- `npm ci` 는 package-lock.json을 진실(SOT)로 보고 의존성을 설치하며 package.json과 불일치, 충돌하면 에러를 뱉는다.
- npm 버전마다 peerDependencies를 자동 설치, 검증하는 로직이 추가되거나, package-lock.json 해석 방식이 달라 설치 결과에 차이가 발생할 수 있다.
- .nvmrc로 node 버전을 업데이트 및 고정
- 최근에 여러 엔지니어들이 각자 다른 node 버전, npm 버전을 바탕으로 새로운 패키지를 설치
	- 복잡한 peerDependencies가 생기면서 버전마다 불필요해진 의존성을 삭제하거나 업데이트,설치 순서가 달라지거나 함
	- 협업하는 엔지니어들의 각 로컬에서 npm 설정이 다를 수 있음 (--legacy-peer-deps 여부 등)
- 로컬에서 생성된 package-lock.json 변경점이 CI환경의 다른 npm 버전이 읽을 때 충돌하는 에러 발생! 


---

##### Corepack 으로 해결해보기

Corepack은 Node에 포함된 도구로, 프로젝트 package.json에 명시하는 packageManager 필드를 보고 정확한 패키지 매니저와 버전을 실행하도록 도와주는 역할을 한다.

```json
//pacakge.json
"name": "my-project",
"packageManager": "npm@10.8.2+sha512........"
...
```

그리고 CI 환경에서도 corepack을 enabled 한 뒤 npm ci로 의존성을 설치하게 되면

```yaml
- name : Set corepack enabled
  shell: bash
  run: corepack enable
```

Corepack이 정확히 명시한 패키지 매니저 버전을 사용해 설치할 수 있게 된다.

---

##### 협업을 위한 추가 장치 마련하기

1. .nvmrc를 커밋해 협업하는 엔지니어가 모두 같은 node 버전을 사용하게 한다.
2. CI 환경에서도 Github Actions jobs에서 node 버전을 .nvmrc를 따라가게 한다.
3. 의존성 설치 전 Corepack을 활성화해서 모두 같은 npm 버전 혹은 명시된 패키지 매니저를 사용하게 한다.
4. .npmrc 를 커밋해 협업하는 엔지니어가 모두 같은 npm 설정을 사용하게 한다.

```json
22.18.0
```

```yaml
name: CI

on: [push, pull_request]

jobs:
  test:
    runs-on: ubuntu-latest

    steps:
      - uses: actions/checkout@v4

      # 1) Node 설정: .nvmrc를 사용(또는 node-version 직접 지정)
      - name: Setup Node (from .nvmrc)
        uses: actions/setup-node@v4
        with:
          node-version-file: '.nvmrc'  # 또는 node-version: '20.x'

      # 2) Corepack 활성화 (필수)
      - name: Enable Corepack
        run: |
          corepack enable
          # (선택) 명시적으로 packageManager 버전 준비
          # corepack prepare --activate npm@10.8.2

      # 3) 의존성 설치: lockfile 기준으로
      - name: Install dependencies (clean, lockfile-based)
        run: npm ci

      - name: Run tests
        run: npm test
```

```json
//.npmrc
 legacy-peer-deps=false
```

---

추가로 팀원이 알려준 소식으로 Node 25 부터는 Corepack이 기본 배포에 포함되지 않는다는 것을 알게 됐다.

처음엔 Corepack이 지원중단된다는 건가?하고 놀랐는데, 그건 아니었다. 지원중단이 아닌 Node.js와 Corepack이 별도의 프로젝트로 관리된다는 방향성이라고 한다. 이는 Node.js 자체를 경량화하고 유지보수를 단순화하고자 함에 있다고 한다.

24버전 이하까지는 Corepack이 experimantal로 corepack enabled 를 해서 사용하는 방식이었다면, Node.js TSC(기술위원회회의 2025년 3월 19일 결정에 따라 25버전부터는 직접 설치를 해서 사용하는 방식으로 전환된다고 한다. 

https://news.hada.io/topic?id=19970

![7D7F57DA-91F6-4B25-ABC1-40EB44B6515C.jpeg](/img/user/7D7F57DA-91F6-4B25-ABC1-40EB44B6515C.jpeg)