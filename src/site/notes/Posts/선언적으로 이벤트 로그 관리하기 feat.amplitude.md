---
{"dg-publish":true,"permalink":"/posts/feat-amplitude/","created":"2025-08-24","updated":"2025-08-24T19:34:00"}
---

회사에서는 amplitude를 이용해 이벤트를 로깅하고 있다. 이를 바탕으로 클릭률, 전환율 등의 사용자 이벤트를 트랙킹해보고 UX를 개선하는 지표로 활용하거나, 새로운 기능 출시나 개선 과제에 아이디어를 얻는다. 이전 회사에서는 이벤트 기반으로 의사결정할 기회가 많지 않았는데, 지금 회사에서는 퍼블릭하게 오픈된 B2C 플랫폼 서비스이다보니 (심지어 유저가 회원가입하지 않아도) 얻어낼 수 있는 데이터들이 많아서 앞으로 시도해볼 것들이 기대가 되는 중이다.

우선, 이벤트 로깅 작업의 순서를 간략히 설명해보면 이렇다. 데이터 분석팀에서 기능 출시를 하는 시점에 피그마 시안을 기준으로 이벤트 설계를 한 뒤, 프론트엔드팀에 넘겨주면 그 이벤트를 기능 개발과 함께 혹은 기능 개발 후에 로그를 붙여 릴리즈하는 프로세스다.

기존 프론트엔드 코드베이스에는 기능 구현을 위한 상태관리, 컴포넌트, 로직 등이 관련 컴포넌트에 작성되고, 거기에 이벤트 로그를 심는 공통 커스텀 훅을 함께 작성하는 구조였다.

이벤트의 종류는 크게는 클릭이나 데이터 패칭과 맞물린 이벤트와 뷰 이벤트, 즉 화면 진입이나 특정 UI의 노출 여부를 트랙킹하는 이벤트로 나뉘고 있었다.

다만, 이벤트 종류의 구분 없이 모두 useEventHook 이런 식의 공통 훅으로 작성되어 있어, 비즈니스 로직 중간 중간에 섞이기도하고, 더이상 필요없는 이벤트의 경우 비즈니스 로직 사이에서 분기문을 고치거나 잘 피해서(?) 삭제해야하는 경우가 많았다. 게다가 데이터 로깅만을 위해 필요한 데이터가 이벤트가 트리거 되는 컴포넌트 자체에 있지 않은 경우 추가적으로 tanstack query를 작성하거나 전역 store에서 데이터를 꺼내와야 하는 등의 오버페칭이 필요한 경우도 많았다. 

입사후 가장 먼저 제안한 협업 프로세스로는 프론트엔드 파트 위클리 싱크가 있었다. 스쿼드라는 목적 조직으로 나뉘어 있는 만큼 일주일에 한 번은 프론트엔드 기능 조직의 업무공유, 기술부채 해소 논의, 아이디어 공유나 컨벤션 논의를 주기적으로 할 필요를 느꼈기 때문이다. 아무튼 이 싱크를 통해 이벤트 로그 작업의 효율화에 관해 논의를 해볼 기회가 있었다.

이 때 주고받았던 아이디어로는, 
- 클릭이벤트나 특정 데이터 페칭 시점에만 조건에 따라 호출해야하는 이벤트의 경우는 명시적으로 호출 가능한 훅이나 함수의 형태로 작성하고, 
- 화면이나 컴포넌트의 노출 시점을 로깅하는 뷰이벤트의 경우는 리액트 컴포넌트 렌더링 주기와 맞물리게 선언적인 컴포넌트로 작성하자.
- 그렇게 해서 비즈니스 로직과 이벤트 로깅 로직이 뒤섞이지 않게 해서 이벤트 교체나 삭제가 쉽고, 컴포넌트 내 코드 가독성을 높이고, 유지보수에 도움이 되게 하자.

정도가 있었다.

퍼블릭 B2C 플랫폼인만큼 이벤트 종류가 무척 세분화 되어있고, 스크롤 이벤트나 데이터의 종류도 꽤나 다양하기 때문에 당장 모든 이벤트 로깅 방식을 개편할 수는 없지만, 새로 작업하게 되는 신규 기능부터는 차근차근 시도해보고자 했다.

```ts
import { useEffect, useRef } from 'react';

export const useViewEvent = (eventCallback: () => void, enabled = true) => {
  const executedViewEvent = useRef(false);

  useEffect(() => {
    if (enabled && !executedViewEvent.current) {
      eventCallback();
      executedViewEvent.current = true;
    }
  }, [enabled, eventCallback]);
};
```

기존 뷰이벤트를 로깅할 커스텀 훅은 이렇게 생겼다.

여기에 나는 선언적인 컴포넌트 하나를 간단히 추가하는 방식으로 구현을 했다. 

```ts
interface ViewEventTrackerProps {
  /**
   * 발생시킬 이벤트 콜백 함수
   */
  eventCallback: () => void;
  /**
   * 이벤트 발생 여부를 제어하는 조건
   * @default true
   */
  enabled?: boolean;
  children: ReactNode;
}

/**
 * 컴포넌트가 마운트될 때 한 번만 이벤트를 발생시키는 컴포넌트
 *
 * @example
 * <ViewEventTracker
 *   eventCallback={() => amplitude.viewBottomsheet({
 *     id,
 *     name,
 *   })}
 *   enabled={!isLoading}
 * >
 *   <YourComponent />
 * </ViewEventTracker>
 */
export const ViewEventTracker = ({ eventCallback, enabled = true, children }: ViewEventTrackerProps) => {
  useViewEvent(eventCallback, enabled);
  return children;
};
```

이렇게 하면 이벤트 로깅 관련 로직은 컴포넌트 내부에 작성하지 않을 수 있다. 로깅이 필요없어지면 부모인 ViewEventTracker만 지우면 된다.

추가로, 여러가지 다양한 이벤트를 한 컴포넌트에서 로깅해야할 수도 있다. 그래서 이렇게 변경해봤다.

```ts
interface ViewEventTrackerProps {
  /**
   * 발생시킬 이벤트 콜백 함수
   */
  eventCallback: () => void;
  /**
   * 이벤트 발생 여부를 제어하는 조건
   * @default true
   */
  enabled?: boolean;
  children?: ReactNode;
}

/**
 * 컴포넌트가 마운트될 때 한 번만 이벤트를 발생시키는 컴포넌트
 *
 * @example
 * <>
 * <ViewEventTracker
 *   eventCallback={() => amplitude.viewBottomsheet({
 *     id,
 *     name,
 *   })}
 *   enabled={!isLoading && firstReady }
 * />
 * <ViewEventTracker 
 *  eventCallback={() => amplitude.viewBottomsheet({
 *     id,
 *     name,
 *   })}
 *  enabled={!isLoading && secondReady }
 * />
 * <YourComponent />
 * </>
 */
export const ViewEventTracker = ({ eventCallback, enabled = true, children }: ViewEventTrackerProps) => {
  useViewEvent(eventCallback, enabled);
  return children;
};
```

children prop을 optional하게 해서 꼭 부모 컴포넌트로 감싸지 않아도 되게끔 했다. 실제로 이렇게 하니, 3-4가지 뷰이벤트를 하나의 복합 뷰 이벤트 트래커 컴포넌트로 떼어내고 기존 컴포넌트의 비즈니스로직과 격리시킬 수 있었다.

![Group 52.png](/img/user/Group%2052.png)

간단한 개선이지만, 현재의 불편함을 팀 내에서 공감할 수 있게 가시화하고 이를 개선할 수 있는 작은 변화부터 만들어가는게 좋은 팀이 되는 길이지 않을까 생각해보는 작업이었다. 미루지 말고, 작게 작게 실천하기.

추후 더 개선해볼 만한 점은, 
- eventCallback 에 아직 콜백 함수를 작성해서 넘겨줘야 하는데, 데이터 분석팀에서 정의한 이벤트명과 일치하도록 이벤트명을 key로 관리하고, 내부에서 이벤트를 자체적으로 호출할 수 있게 해도 좋을 것 같고,
- 리뷰해준 팀원의 아이디어를 바탕으로, 클릭 이벤트도 선언적인 컴포넌트로 만들어볼 수 도 있겠다. 
	- `<EventTrack.Click eventName="click_event"><button>확인</button></EventTrack.Click>`


