---
{"dg-publish":true,"permalink":"/posts/git-https-vs-ssh/","tags":["git"],"created":"2026-04-26","updated":"2026-04-26"}
---

GitHub에서 레포를 클론할 때 두 가지 길이 있다.

```bash
git clone https://github.com/user/repo.git
git clone git@github.com:user/repo.git
```

같은 레포가 같은 모양으로 받아진다. 그래서 “둘 중 뭐가 다르지?” 싶다. 그런데 이 두 줄 안에서 일어나는 일은 완전히 다르다. 자세히 알아보자.

-----

### 목차

1. 처음 클론하기
2. 일상적 push/pull
3. 자동화 스크립트와 CI : 토큰 vs 키
4. 여러 계정 쓰기 : 회사와 개인을 나누고 싶을 때
5. 안에서 정확히 무슨 일이 일어나는가
6. 정리 : 같은 결과, 다른 신원 모델

### 1. 처음 클론하기

##### HTTPS

```bash
git clone https://github.com/user/repo.git
```

공개 저장소면 인증 없이 그냥 받아진다. 그런데 사설 저장소를 클론하려고 하면 사용자명과 비밀번호를 묻는다.

여기서 한 가지 짚고 넘어가야 한다. 2021년 8월부터 GitHub은 **비밀번호 인증을 막았다**. 대신 PAT(Personal Access Token)이라는 토큰을 발급받아 비밀번호 자리에 넣어야 한다. PAT는 GitHub Settings에서 발급받는 길고 무작위한 문자열이다.

```
Username for 'https://github.com': alice
Password for 'https://alice@github.com': ghp_xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
```

매번 이걸 입력하기 싫으니 credential helper에 저장한다. macOS면 Keychain, Windows면 Credential Manager, Linux면 `git config credential.helper store` 같은 옵션. 한 번 저장되면 그 다음부터는 묻지 않는다.

##### SSH

```bash
git clone git@github.com:user/repo.git
```

처음에 두 단계 셋업이 필요하다.

1. `ssh-keygen -t ed25519`로 키 쌍을 만든다. (한 번만)
2. 만들어진 공개키(`~/.ssh/id_ed25519.pub`)를 GitHub Settings → SSH Keys에 등록한다. (한 번만)

그 다음부터는 그냥 클론 명령만 치면 되는 방식이다.

##### HTTPS vs SSH 비교

HTTPS는 진입 장벽이 낮다. 그냥 클론하고, 묻으면 답하고, 토큰을 한 번 저장하면 끝이다. SSH는 처음에 키를 만들고 등록하는 단계가 있어 살짝 번거롭다. 그래서 입문자에게는 HTTPS가 친절하다.

### 2. 일상적 push/pull 
##### HTTPS의 미묘한 함정

credential helper에 PAT를 저장하면 평소엔 매끄럽다. push도 pull도 그냥 동작한다. 그런데 어느 날 갑자기 이런 메시지가 뜬다.

```
remote: Support for password authentication was removed on August 13, 2021.
remote: Please use a personal access token instead.
fatal: Authentication failed for 'https://github.com/user/repo.git'
```

당황스럽다. 어제까지 잘 되던 게 왜? 답은 **PAT가 만료됐기 때문**이다. PAT는 발급할 때 만료 기한을 정한다. 7일, 30일, 90일, 1년, 또는 무기한. 보안상 무기한은 권장되지 않으니 보통 90일이나 1년을 고른다. 그리고 그 기한이 다가왔을 때 알림이 잘 와도, 못 보고 지나치면 어느 날 push가 막힌다.

복구하려면 새 PAT를 발급받고, credential helper의 옛 토큰을 지우고, 새 토큰을 다시 등록해야 한다. 1년에 한두 번씩 이걸 반복한다.

##### SSH

SSH 키는 사용자가 명시적으로 회수하지 않는 한 만료되지 않는다. 한 번 등록하면 끝이다. 매 push마다 challenge-response 인증이 일어나지만, ssh-agent가 메모리에 비밀키를 들고 있어서 사용자에겐 그냥 즉시 동작한다.

“어제까지 잘 되던 push가 갑자기 막히는” 상황이 거의 없다. 키 파일이 사라지거나, GitHub에서 키를 회수하거나, 사용자가 직접 지우지 않는 한.

##### 차이점

HTTPS는 **토큰**을 쓴다. 토큰은 짧은 수명을 가진 객체다. 발급, 저장, 만료, 갱신, 폐기의 라이프사이클을 사용자가 관리해야 한다.

SSH는 **키 쌍**을 쓴다. 키는 만들어둔 시점에 영구적이다. 누가 폐기하지 않는 한 살아 있다. 라이프사이클 관리가 거의 필요 없다.

### 3. 자동화 스크립트와 CI : 토큰 vs 키

##### HTTPS로 자동화하기

CI/CD에서 사설 저장소를 클론하려면 PAT를 환경변수로 넣고 URL에 끼워야 한다.

```bash
git clone https://${GITHUB_TOKEN}@github.com/user/repo.git
```

문제가 몇 가지 있다.

∙ 토큰이 평문으로 명령어에 들어가니 로그에 노출되지 않게 조심해야 한다.  
∙ 토큰이 만료되면 CI 파이프라인이 갑자기 실패한다.  
∙ 권한이 너무 넓은 PAT는 보안 위험이다. 그래서 **fine-grained PAT**나 **GitHub App 토큰**을 따로 발급받는 운영이 필요하다.

##### SSH로 자동화하기

GitHub에는 **deploy key**라는 개념이 있다. 특정 저장소에만 접근할 수 있는 SSH 키다. CI 환경에 deploy key의 비밀키를 secret으로 넣어두면 된다.

```bash
# CI 환경
ssh-add <(echo "$DEPLOY_KEY")
git clone git@github.com:user/repo.git
```

키는 만료되지 않으니 갱신 스크립트도 필요 없다. 권한도 그 저장소로 한정되어 있어 더 깔끔하다.

(물론 모든 자동화에 SSH가 정답은 아니다. GitHub Actions에서 자기 저장소를 다룰 땐 자동 발급되는 `GITHUB_TOKEN`을 쓰는 게 맞고, 외부 API를 부르는 작업은 PAT나 App 토큰이 필요하다. 다만 단순한 클론 자동화에 한해선 SSH 쪽이 부담이 적다.)

### 4. 여러 계정 쓰기 : 회사와 개인을 나누고 싶을 때

회사 노트북에서 회사 GitHub 계정과 개인 계정을 둘 다 쓰고 싶다고 하자. HTTPS와 SSH가 어떻게 다르게 풀리는지 보자.

##### HTTPS의 어려움

credential helper는 보통 **도메인 단위**로 자격을 저장한다. `github.com`에 대해 하나의 자격만 기억한다. 회사 PAT를 저장해두면 개인 작업에서 그 PAT가 쓰이고, 그 반대도 마찬가지다.

운영체제별 credential helper에 따라 분기 설정이 가능하긴 하지만, 깔끔하게 풀리지 않는다. 결국 어떤 저장소를 클론할 때 어떤 토큰이 쓰이는지 헷갈리는 상황이 자주 생긴다.

##### SSH

`~/.ssh/config`에 호스트 별칭을 만든다.

```
Host github-personal
    HostName github.com
    User git
    IdentityFile ~/.ssh/id_personal

Host github-work
    HostName github.com
    User git
    IdentityFile ~/.ssh/id_work
```

그러면 클론 URL에 어떤 키를 쓸지 표현할 수 있다.

```bash
git clone git@github-personal:my-account/personal-repo.git
git clone git@github-work:company/work-repo.git
```

`github-personal`과 `github-work`는 SSH 클라이언트만 아는 별칭이다. 실제로는 둘 다 `github.com:22`에 연결되지만, 어떤 비밀키를 쓸지가 분기된다. 한 머신에서 두 계정이 명확하게 분리된다.

##### 왜 SSH가 더 잘 되는가

핵심은 SSH가 **연결 단위로 신원을 정한다**는 점이다. 한 번 연결을 맺는 시점에 어떤 키로 인증할지가 결정되고, 그 연결의 모든 동작은 그 신원 위에서 일어난다. 어떤 연결을 어떤 키로 맺을지는 `~/.ssh/config`로 자유롭게 라우팅할 수 있다.

HTTPS는 **요청 단위로 자격을 보낸다**. 매번 요청에 토큰을 실어 보내야 하고, 어떤 도메인에 어떤 토큰을 보낼지는 credential helper가 관리한다. 같은 도메인에 두 가지 신원이 있는 경우 helper가 헷갈린다.

### 5. 안에서 정확히 무슨 일이 일어나는가

지금까지의 차이를 만든 안쪽 동작을 짧게 정리해보자.

##### `git clone https://...`의 흐름

1. git이 HTTPS로 GitHub에 연결한다. (TLS 핸드셰이크, 서버 인증서 검증)
2. git smart protocol 요청을 보낸다.
3. 사설 저장소면 GitHub이 401 응답으로 인증을 요구한다.
4. credential helper에서 PAT를 꺼내 `Authorization: Basic` 헤더에 실어 다시 요청한다.
5. GitHub이 PAT를 검증한다. 만료됐으면 거부.
6. 통과하면 저장소 데이터를 HTTPS 위에서 받는다.

핵심은 **3~4단계가 매 요청마다 반복된다**는 점이다. push 한 번, pull 한 번이 다 별도의 HTTPS 요청이고, 매번 토큰이 실려간다.

##### `git clone git@github.com:...`의 흐름

1. SSH 클라이언트가 `github.com:22`에 TCP 연결.
2. 호스트 키 검증. 처음이면 fingerprint를 보여주고 `~/.ssh/known_hosts`에 저장. 이후엔 자동 비교.
3. 사용자 인증. GitHub이 challenge를 보내고, ssh-agent가 비밀키로 서명해 응답. 검증되면 채널이 인증된 상태로 열린다.
4. 그 채널 위에서 GitHub이 `git-upload-pack` 같은 git 프로세스를 띄워준다.
5. git 프로토콜로 저장소 데이터를 받는다.

핵심은 **3단계의 인증이 채널 단위로 한 번만 일어난다**는 점이다. 그 채널이 살아있는 동안의 모든 git 동작은 이미 인증된 신원 위에서 이루어진다.

```
[HTTPS]                           [SSH]
요청1: 토큰 첨부 ──────►          연결: 인증 (1회) ──────►
요청2: 토큰 첨부 ──────►          ─ git push ─►
요청3: 토큰 첨부 ──────►          ─ git pull ─►
...                               ─ git fetch ─►
매 요청마다 토큰 검증              한 번 인증한 채널 위에서 흐름
```


### 6. 정리 : 같은 결과, 다른 신원 모델

GitHub의 두 클론 방식은 사실상 두 가지 신원 검증 모델을 가지고 있다.

|      |HTTPS   |SSH                |
|------|--------|-------------------|
|신원의 형태|토큰 (PAT)|키 쌍                |
|수명    |만료됨     |영구 (회수 전까지)        |
|인증 시점 |매 요청마다  |연결 시 한 번           |
|첫 셋업  |토큰 발급·저장|키 생성·등록            |
|갱신 부담 |주기적     |거의 없음              |
|다중 계정 |까다로움    |`~/.ssh/config`로 깔끔|
|자동화   |토큰 관리 필요|deploy key로 단순     |

HTTPS는 진입 장벽이 낮은 대신, 토큰의 짧은 수명과 매 요청마다의 인증이라는 부담을 사용자에게 분산해서 안긴다. 한 번에 큰 고통은 없지만, 1년에 몇 번씩 작은 마찰이 있다. (토큰 만료)

SSH는 처음에 키 셋업이라는 한 번의 진입 비용이 있는 대신, 그 다음부터는 거의 없다. 키 한 쌍을 한 번 등록해두면 그 키가 살아있는 동안 모든 일이 매끄럽게 흐른다.

토큰은 끊임없이 갱신해야 하는 일회용 증명서지만, 키는 한 번 만들어 둔 신원 그 자체, “키가 곧 신원” 이다. 그래서 토큰 기반 시스템은 항상 토큰 라이프사이클 관리라는 운영 부담을 동반하고, 키 기반 시스템은 그 부담이 거의 사라진다.