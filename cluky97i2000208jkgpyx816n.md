---
title: "[번역] React Internals Deep Dive 개요"
datePublished: Thu Apr 04 2024 08:02:56 GMT+0000 (Coordinated Universal Time)
cuid: cluky97i2000208jkgpyx816n
slug: react-internals-deep-dive-1
cover: https://cdn.hashnode.com/res/hashnode/image/upload/v1712205380423/568b9c52-925e-425c-b7a8-c795c9fdece7.jpeg
tags: react-internals

---

> 영문 블로그 글을 번역했습니다. 허가를 받으면 시리즈를 이어갈 예정입니다.  
> 원문링크: [https://jser.dev/2023-07-11-overall-of-react-internals/#1-tips-on-learning-react-internals](https://jser.dev/2023-07-11-overall-of-react-internals/#1-tips-on-learning-react-internals)

---

> ℹ️ [React Internals Deep Dive](https://jser.dev/series/react-source-code-walkthrough.html) 에피소드 1, [유튜브에서 제가 설명하는 것](https://www.youtube.com/watch?v=fgxxoaeKQWo&list=PLvx8w9g4qv_p-OS-XdbB3Ux_6DMXhAJC3&index=32)을 시청해주세요.
> 
> ⚠ [React@18.2.0](https://github.com/facebook/react/releases/tag/v18.2.0) 기준, 최신 버전에서는 구현이 변경되었을 수 있습니다.

더 나은 React 코드를 작성하기 위해 React 내부에 대해 더 많이 알고 싶지만, 뛰어들고 싶지만 물이 너무 넓고 깊은 것처럼 어렵고 어디서부터 시작해야 할지 모릅니다.

저는 2021년에 똑같은 어려움을 겪었지만, 어려움을 잘 관리하여 React 내부의 작동 방식을 설명하는 30개 이상의 에피소드가 포함된 [React Internals Deep Dive](https://jser.dev/series/react-source-code-walkthrough) 시리즈를 만들 수 있었습니다.

이 첫 번째 에피소드에서는 React Internals 에 대한 대략적인 개요와 React Internals를 배우는 방법에 대한 몇 가지 팁을 알려드리겠습니다.

## 1\. React Internals 배우기 팁

### 1.1 공식 자원 [Grok](https://en.wikipedia.org/wiki/Grok)하기

[React.dev](https://react.dev/)는 React API를 배울 수 있을 뿐만 아니라 React 핵심 팀의 생각도 배울 수 있는 멋진 곳입니다. 그들은 왜 그런 선택을 했는지 설명하는 데 많은 노력을 기울였습니다.

[리액트 워킹 그룹](https://github.com/reactwg)은 사람들이 멋진 새 아이디어에 대해 토론하는 공간입니다.

### 1.2 React 팀 따르기

[리스트의 모든 멤버들](https://react.dev/community/team)을 팔로우하여 React 팀이 하는 일을 계속 업데이트하고, 인터넷에서 그들의 토론을 통해 코드에서 찾을 수 없는 독특한 관점을 얻을 수 있습니다.

### 1.3 리액트 저장소 따르기

[React repo](https://github.com/facebook/react)는 코드뿐만 아니라 PR과 코드 리뷰도 볼 수 있는 곳입니다. 대부분의 경우 코드의 주석보다 더 나은 설명이 포함되어 있습니다.

### 1.4 게시글이 아닌, 코드를 신뢰하기

인터넷에서 React 내부에 대한 많은 글을 보지만 도움이 되는 글은 거의 없습니다. 대부분은 코드를 다루지 않고 일반적인 아이디어만 설명하기 때문에 읽고 나서도 [React 퀴즈](https://bigfrontend.dev/react-quiz)를 풀지 못하기 때문에 가치가 떨어집니다.

시간이 있다면 React 소스 코드를 살펴보는 것도 도움이 될 것입니다.

### 1.5 핵심 경로 찾기

하지만 문제는 React 코드 베이스가 방대하고 위협적이라는 것입니다. 제 조언과 실제로 제가 한 일은 **핵심 경로를 찾는** 것이었습니다. 모든 것을 한꺼번에 이해할 필요는 없고, 어떻게 작동하는지 대략적으로 파악한 다음 퍼즐을 하나씩 풀면서 이해의 네트워크를 형성하는 것도 괜찮다는 것이었습니다.

그래서 [디버깅 영상](https://www.youtube.com/watch?v=OcB3rTln-fI&list=PLvx8w9g4qv_p-OS-XdbB3Ux_6DMXhAJC3&index=1)을 만든 지 2년이 지나서야 이 첫 번째 개요 에피소드를 완성할 수 있었는데, 그 동안 작은 퍼즐을 푸는 데 시간을 보냈고 이제야 큰 그림을 설명할 수 있을 것 같아서입니다.

## 2\. 중단점을 사용하여 React Internals 개요를 디버깅하는 방법

### 2.1 중단점 설정

다음은 디버깅 프로세스를 시연하는 데 사용할 코드입니다.

[![](https://cdn.hashnode.com/res/hashnode/image/upload/v1712205575957/f3dfea1f-1464-4811-ba9d-227ce500a806.png align="center")](https://cdn.hashnode.com/res/hashnode/image/upload/v1712205575957/f3dfea1f-1464-4811-ba9d-227ce500a806.png)

> ℹ️ [위의 데모](https://jser.dev/demos/react/overview/index.html)를 사용해보고 아래의 설명을 따라해 보세요.

React는 UI 라이브러리이므로 중요한 작업 중 하나는 DOM이 조작되는 코드를 찾은 다음 호출 스택을 읽고 무슨 일이 일어나고 있는지 파악하는 것입니다. 여기서는 아래와 같이 DOM 컨테이너에 DOM 중단점을 만들면 됩니다.

[![](https://jser.dev/static/react-overview-debug-1.png align="left")](https://jser.dev/static/react-overview-debug-1.png)

이제 디버깅할 차례입니다.

### 2.2 첫 번째 일시정지: 컴포넌트 렌더링

다음은 호출 스택의 몇 가지 중요한 함수입니다.

1. `ReactDOMRoot.render()` → 이것은 우리가 작성한 user-side 코드이며, 먼저 `createRoot()` 를 호출한 다음 `render()`를 호출합니다.
    
2. `scheduleUpdateOnFiber()` → React에 렌더링 할 위치를 알려주는데, 초기 마운트에는 이전 버전이 없으므로 루트에서 호출됩니다.
    
3. `ensureRootIsScheduled()` → 이 중요한 호출은 `performConcurrentWorkOnRoot()` 가 예약되어 있는지 '확인'하는 것입니다.
    
4. `scheduleCallback()` → 실제 스케줄링은 [React 스케줄러](https://jser.dev/react/2022/03/16/how-react-scheduler-works/)의 일부이며, 스크린샷에서 `postMessage()`에 의해 비동기화되는 것을 확인할 수 있습니다.
    
5. `workLoop()` → [React 스케줄러](https://jser.dev/react/2022/03/16/how-react-scheduler-works/#44-workloop---the-core-of-scheduler)가 작업을 처리하는 방법
    
6. `performConcurrentWorkOnRoot()` → 이제 예약된 작업이 실행되고 있으며, App 컴포넌트가 실제로 렌더링됩니다.
    

이런 과정을 통해 React가 실제로 렌더링을 수행하는 방식에 대한 몇 가지 단서를 얻을 수 있습니다.

### 2.3 두 번째 일시정지: DOM 조작

[![](https://jser.dev/static/react-overview-debug-3.png align="left")](https://jser.dev/static/react-overview-debug-3.png)

UI 라이브러리로서, 목표는 DOM 업데이트를 관리하는 것입니다. 이는 실제로 상단의 "렌더링" 단계 이후의 "커밋" 단계에 해당합니다.

1. `commitRoot()` → 이전 렌더링 단계에서 파생됐고, 필요한 DOM 업데이트들을 커밋하며, 그 외에도 물론 이펙트 처리와 같은 더 많은 작업을 수행합니다.
    
2. `commitMutationEffects()` → 호스트 DOM의 실제 업데이트입니다.
    

> 💬 역자 주석: 두 번째 일시정지가 데모상에서는 위처럼 동작을 안해서 overview/react-dom-18.2.0.development.js에서 `appendChildToContainer` 함수를 직접 브레이크 포인트 거셔서 멈추면 동일하게 보실 수 있습니다.

### 2.4 세 번째 일시정지: 이펙트 실행

[![](https://jser.dev/static/react-overview-debug-4.png align="left")](https://jser.dev/static/react-overview-debug-4.png)

이제 `useEffect()` 호출에서 일시 중지되는 것을 볼 수 있습니다.

1. `flushPassiveEffects()` → `useEffect(`)로 생성된 모든 패시브 이펙트를 플러시합니다.
    

또한 `postMessage()`에 의해 비동기화되어 즉시 실행되지 않고 예약되어 있다는 것을 알 수 있습니다. `flushPassiveEffects()`에 중단점을 추가하면 `commitRoot()` 내부에 있다는 것을 쉽게 알 수 있습니다.

### 2.5 컴포넌트 렌더링에서 다시 일시정지

[![](https://jser.dev/static/react-overview-debug-5.png align="left")](https://jser.dev/static/react-overview-debug-5.png)

`useEffect()` 에서 `setState()` 를 호출하여 리-렌더링을 트리거 시키는데, 콜 스택을 보면 전체 리-렌더링이 첫 번째 중단점 일시 정지와 매우 유사하다는 것을 알 수 있지만, `performConcurrentWorkOnRoot()` 내부에서는 `mountIndeterminateComponent()` 가 아닌`updateFunctionComponent()` 를 호출한다는 점이 다릅니다.

React 소스 코드에서, `마운트`는 초기 렌더링을 의미하는데, 초기 렌더링에서는 이전 버전과 비교해야 할 이전 버전이 없기 때문입니다.

## 3\. React Internals 개요

사실 위의 스크린샷은 이미 React internals의 기본을 다루고 있습니다. 개요로서 너무 자세한 내용은 다루지 않겠지만, 이미 세부적인 내용을 파악했기 때문에 아래와 같이 React 내부의 개요를 4단계로 나눠서 그려보겠습니다.

[![](https://jser.dev/static/react-internals-overview-light.png align="left")](https://jser.dev/static/react-internals-overview-light.png)

### 3.1 Trigger

모든 작업이 여기서 시작되기 때문에 "Trigger"라는 이름을 붙였습니다. 초기 마운트든 상태 훅으로 인한 리-렌더링이든 상관없습니다. 이 단계에서는 앱의 어느 부분을 렌더링(`scheduleUpdateOnFiber()`) 해야 하는지, 어떻게 렌더링해야 하는지 React 런타임에 알려줍니다.

이 단계를 "작업 생성"이라고 생각할 수 있으며, `ensureRootIsScheduled()` 는 이러한 작업 생성의 마지막 단계이고, 이후 `scheduleCallback()`에 의해 작업이 "Scheduler"로 전송됩니다.

관련 주제는 다음을 참조하세요:

1. [EP5 - useState()는 어떻게 작동하나요?](https://jser.dev/2023-06-19-how-does-usestate-work/)
    

### 3.2 Scheduler

이것은 React 스케줄러로, 기본적으로 우선순위에 따라 작업을 처리하는 [우선순위 큐](https://bigfrontend.dev/problem/create-a-priority-queue-in-JavaScript)입니다. 런타임 코드에서 `scheduleCallback()`을 호출하여 렌더링이나 이펙트 실행과 같은 작업을 예약(=schedule)합니다. "Scheduler" 내부의`workLoop()`는 작업이 실제로 실행되는 방식입니다.

스케줄러에 대한 자세한 내용은 다음을 참조하세요:

1. [EP20 - 스케쥴러는 어떻게 작동하나요?](https://jser.dev/react/2022/03/16/how-react-scheduler-works/)
    

### 3.3 Render

"Render"는 예약된 작업(`performConcurrentWorkOnRoot()`)으로, 새로운 파이버 트리를 계산하고 호스트 DOM에 적용하기 위해 필요한 업데이트를 파악하는 것을 의미합니다.

파이버 트리는 기본적으로 앱의 현재 상태를 나타내는 내부 트리와 같은 구조이기 때문에 여기서 자세히 설명할 필요는 없습니다. 이전에는 가상 DOM이라고 불렸지만, 지금은 DOM에만 사용되는 것이 아니며 React 팀에서는 더 이상 가상 DOM이라고 부르지 않습니다.

따라서 `performConcurrentWorkOnRoot()`는 "Trigger" 단계에서 생성되고 "Scheduler" 단계 에서 우선순위가 지정된 다음 실제로 여기서 실행됩니다. 마치 작은 사람이 파이버 트리를 돌아다니며 리-렌더링해야 하는지 확인하고 호스트 DOM에서 필요한 업데이트들을 알아내는 것처럼 생각하면 됩니다.

동시 모드로 인해 '렌더링' 단계가 중단되었다가 다시 시작될 수 있으므로, 상당히 복잡한 단계가 됩니다.

자세한 내용은 다음 에피소드를 참조하세요:

1. [EP15 - Fiber Tree를 어떻게 순회하나요?](https://jser.dev/react/2022/01/16/fiber-traversal-in-react/)
    
2. [EP13 - bail out은 조정(reconciliation)에서 어떻게 작동하나요?](https://jser.dev/react/2022/01/07/how-does-bailout-work/)
    
3. [EP19 - 'key'는 어떻게 작동하나요? 리스트 diffing](https://jser.dev/react/2022/02/08/the-diffing-algorithm-for-array-in-react/)
    
4. [EP21 - React 소스 코드에서 Lanes란 무엇인가요?](https://jser.dev/react/2022/03/26/lanes-in-react/)
    

### 3.4 Commit

새 파이버 트리가 구성되고 최소 업데이트가 도출되면 업데이트를 호스트 DOM에 'Commit'할 차례입니다.

물론 DOM을 조작하는 것(`commitMutationEffects()`) 만이 아닙니다. 예를 들어 모든 종류의 Effects도 여기서 처리됩니다(`flushPassiveEffects()`, `commitLayoutEffects()`).

관련 에피소드:

1. [EP10 - useLayoutEffect()는 어떻게 작동하나요?](https://jser.dev/react/2021/12/04/how-does-useLayoutEffect-work/)
    
2. [EP4 - useEffect()는 어떻게 작동하나요?](https://jser.dev/2023-07-08-how-does-useeffect-work/)
    
3. [EP8 - useTransition()은 어떻게 작동하나요?](https://jser.dev/2023-05-19-how-does-usetransition-work/)
    
4. [EP16 - Effect Hooks의 생명 주기](https://jser.dev/react/2022/01/19/lifecycle-of-effect-hook/)
    

## 4\. 요약

여기까지 [React Internals Deep Dive](https://jser.dev/series/react-source-code-walkthrough)의 첫 번째 에피소드입니다. 몇 가지 디버거를 설정하는 것만으로도 얼마나 많은 정보를 얻을 수 있는지 확인할 수 있습니다.

겁내지 말고 한 번에 한 퍼즐씩 풀어나가다 보면 매일 리액트를 더 잘 알게 될 것입니다. 막히는 부분이 있으면 [Discord](https://discord.com/invite/6Kr9HvRzBT)로 저에게 핑을 보내주세요.

이 게시물이 도움이 되었기를 바랍니다.

(원본 게시일: 2023-07-11Jul 11, 2023)