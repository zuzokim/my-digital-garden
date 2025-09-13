---
{"dg-publish":true,"permalink":"/posts/dev/type-safe-api/","tags":["git","type-safe"],"created":"2024-09-09","updated":"2024-09-09T23:15:00"}
---

현재 [[Projects/SqetchClub/sqetch.club\|sqetch.club]] 프로젝트는 phoenix [Socket](https://hexdocs.pm/phoenix/js/#socket) , [Channel](https://hexdocs.pm/phoenix/js/#channel) 을 이용해서 Sveltekit app과 소켓통신을 하는 구조입니다. 프로젝트 poc단계에서 임시로 만들어둔 endpoint이긴 하지만 모든 타입들이 any 혹은 Object 타입인 상태로, 어떤 api에 어떤 payload data가 전송되는지 알 수가 없는 상태입니다.

phoenix에서 Socket api endpoint를 정의하고 이를 기반으로 trpc 혹은 openapi-ts로 필요한 데이터들의 type을 generate하고, Sveltekit에서 이 타입을 가져다 쓰도록 하면 graphql schema처럼 FE, BE 양쪽에서 single-source-of-truth 즉, 하나의 소스로부터 뽑아낸 type들을 기반으로 안정성이 보장된 api 연동개발을 할 수 있을 듯 합니다.

swagger 같은 솔루션으로 일부 자동화하거나, wiki 나 gitbook, docusaurus 등으로 별도의 문서로 관리해서 한 곳에서 api 스펙을 확인할 수 있는 방식이 있지만, 개발되는 실제 코드와 항상 싱크를 맞추고 관리하는 비용, 다른 플랫폼을 통하지 않고 코드단에서 좀 더 자동화해 기능 개발에 더 집중하고 싶은 니즈가 있는 상태입니다. 그렇게 찾아보던 여러 옵션 가운데 공부도 해볼겸 시도해볼만한 방법으로 [openapi-ts](https://openapi-ts.pages.dev/)가 끌려 리서치해보는 중에 있습니다.

현재 BE, FE가 각기 다른 레포를 쓰고 있어서 generate한 타입을 client측에서 가져올 방법이 필요해보입니다. 여기서부터 문제네요. 일단 떠올려본 방법은 3가지 정도입니다.
###### 1. 현재 분리된 FE, BE 레포를 모노레포로 통합한다. 같은 레포 안에 있으니 직접 필요한 type 파일 경로에 접근할 수 있다.
###### 2. BE에서 generate한 type을 expose할 private npm package를 만들고 FE에서 dependencies 모듈로 설치해서 사용한다.
###### 3-1. git submodule를 사용해서 type 파일 및 코드를 각 FE, BE레포에서 공유할 수 있게 한다.
###### 3-2. git submodule대신 git subtree를 사용한다.

###### 1번은 다음과 같은 이유로 일단 제외
- 모노레포 환경 구성하는데에 리소스가 많이 든다.
- FE, BE 스택이 같은 js, node 기반이라면 시도해봤을 법 한데, FE는 ts, node, BE는 elixir 환경 자체가 다르기도 하고 각자의 기술 스택을 스터디하고자하는 목적의 사이드프로젝트여서 딱히 끌리지는 않는 느낌.
- api를 type-safe하게 만들고 싶을 뿐인데, type을 얻기 위해 개발 환경이나 프로젝트 코드 자체를 한 곳에 모아야하는 큰 구조 변경이 꺼려짐. FE, BE가 서로 의존성 없이 자유롭고 싶음.

###### 2번은 npm package 퍼블리시를 해볼 수 있는 좋은 공부 기회가 될수도?
- 단, 아직까지 사이드 프로젝트에서 그렇게까지 복잡하고 큰 규모의 api를 만들고 있지 않음.
- 릴리즈 버전 관리도 해야함.

###### 3-1번 오 해볼만 한데?
- git submodule은 하나의 레포지토리 하위에 서브 레포지토리를 관리할 수 있게 해준다.
- share할 디렉토리를 git submodule로 만들고 remote와 싱크하는 식의 일반적인 git pull, push 방식으로 코드를 동기화할 수 있기 때문에 위의 1,2방식보다 비교적 리소스를 덜 들이고 필요한 파일들이나 설정들을 공유할 수 있다.
- 대신 업데이트가 있으면 꼭 remote에 push를 해서 싱크를 맞춰야한다. 근데 이건 단점이라고는 할 수 없는게 피쳐 브랜치에서 충분히 테스트한 버전을 main 브랜치로 병합하는 식으로 일종의 버저닝을 할 수도 있을 듯 하다. 

```bash
# Create a new directory for the shared repository
mkdir shared-code
cd shared-code

# Initialize a new Git repository
git init

# Add a README file
echo "# Shared Code" > README.md

# Commit the initial files
git add .
git commit -m "Initial commit"

# Push to your remote repository
git remote add origin <remote-repository-url>
git push -u origin master
```
```bash
# Navigate to the client repository
cd path/to/client-repo

# Add the shared repository as a submodule
git submodule add <remote-repository-url> shared

# Commit the changes
git add .
git commit -m "Add shared code submodule"
```

```ts
// Import shared types
import { SharedType } from '../shared/types';
```

```bash
# Navigate to the server repository
cd path/to/server-repo

# Add the shared repository as a submodule
git submodule add <remote-repository-url> shared

# Commit the changes
git add .
git commit -m "Add shared code submodule"

```
 (elixir에서 type을 사용할 수 있는지는 잘 모르겠음..타입스크립트의 타입을 어떤식으로 다루는지 언어의 특징을 잘 모르는 상태. BE 팀원분께 물어봐야겠음)

###### 3-2 submodule과 subtree의 차이
- subtree를 사용하면 submodule보다 조금 더 간단히 관리가 가능하다. -> 커맨드가 좀 더 간단해지는 정도라고 생각
- submodule은 별도로 git history가 관리되기 때문에 api 체인지만 관리한다는 차원에서 격리시켜두고 사용하기 좋아보임.
- subtree는 커맨드 자체는 더 단순해지지만 git history가 parent repository와 섞이게 된다는 단점이 있다.

```bash
git subtree add --prefix=<path> <repository-url> <branch> --squash

git subtree pull --prefix=<path> <repository-url> <branch> --squash

git subtree push --prefix=<path> <repository-url> <branch>
```

- Comparison of Git Submodules and Git Subtrees:

| **Feature**      | **Git Submodules**                                                                    | **Git Subtrees**                                                                    |
| ---------------- | ------------------------------------------------------------------------------------- | ----------------------------------------------------------------------------------- |
| **Pros**         |                                                                                       |                                                                                     |
| Isolation        | Submodules maintain their own history, offering isolation from the parent repository. | Subtrees merge the entire repository into the parent, but without full isolation.   |
| Version Control  | Allows locking submodules to specific commits, ensuring consistency.                  | Unified with the parent repository, no need to lock to specific commits.            |
| Ease of Use      |                                                                                       | Easier to manage than submodules, with no special commands for cloning/updating.    |
| **Cons**         |                                                                                       |                                                                                     |
| Complexity       | Managing submodules can be complex, requiring extra commands and considerations.      | Simpler but might lead to history bloat in the parent repository.                   |
| Separate Commits | Changes to submodules require separate commits and pushes.                            | All changes and history are unified with the parent repository.                     |
| History Bloat    | Minimal history impact on the parent repository.                                      | The parent repository's history can become large if the subtree has a long history. |
| Isolation Level  | Higher isolation between the parent and submodule.                                    | Less isolation, as subtree is more tightly integrated with the parent.              |


일단 이런 정도로 정리를 해봤습니다. 회사에서 graphql을 사용하면서 single endpoint, type-safe schema에 적응되어있었는데 REST, websocket api를 관리하고 개발하려니 역시 그냥 되는건 없구나 싶습니다. 개발환경 자체도 만들기 나름이고 이런 저런 솔루션과 툴을 찾아보다보니 사람들이 다 비슷한 고민을 하는구나 싶네요. 실제 type을 뽑아내는 과정은 다음 포스트에서 다뤄보겠습니다. 


---
참고링크
- [How Managing Multiple Git Repositories](https://medium.com/@saverio3107/how-managing-multiple-git-repositories-8c864d653afa)
- [openapi-ts-sveltekit-example](https://github.com/openapi-ts/openapi-typescript/tree/main/packages/openapi-fetch/examples/sveltekit)








