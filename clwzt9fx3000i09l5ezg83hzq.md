---
title: "[번역] React에서 act()는 어떻게 동작하나요?"
datePublished: Tue Jun 04 2024 02:59:06 GMT+0000 (Coordinated Universal Time)
cuid: clwzt9fx3000i09l5ezg83hzq
slug: react-internals-deep-dive-24
cover: https://cdn.hashnode.com/res/hashnode/image/upload/v1717467666955/77819efb-96aa-4229-bb16-9de9afb73736.jpeg
tags: react-internals, react-act

---

> 💬 *에피소드 22, 23은 Episode 7의 Suspense 하위에서 진행되었습니다.*
> 
> ***영문 블로그 글을 번역했습니다. 허가를 받으면 시리즈를 이어갈 예정입니다.  
> 원문링크:*** [https://jser.dev/react/2022/05/15/how-act-works](https://jser.dev/react/2022/05/15/how-act-works)

---

> ***ℹ️***[***React Internals Deep Dive***](https://jser.dev/series/react-source-code-walkthrough.html) ***에피소드 24*** [***유튜브에서 제가 설명하는 것***](https://www.youtube.com/watch?v=E7P7abZGsK0&list=PLvx8w9g4qv_p-OS-XdbB3Ux_6DMXhAJC3&index=24)***을 시청해주세요.***
> 
> ***⚠***[***React@18.2.0***](https://github.com/facebook/react/releases/tag/v18.2.0)***기준, 최신 버전에서는 구현이 변경되었을 수 있습니다.***
> 
> ***💬 역자 주석: JSer의 코멘트는 ❗❗로 표시 해뒀습니다.  
> 그 외 주석은 리액트 소스 코드 자체의 주석입니다.  
> ... 은 생략된 코드입니다.***

React에는 몇 가지 [테스트 도구가 내장](https://reactjs.org/docs/test-utils.html)되어 있으며, 보통 직접 사용하지 않고 잘 통합된 테스트 라이브러리와 프레임워크를 통해 사용합니다.

이미 React 내부에 대한 몇 가지 주제를 다루었으므로 이러한 기본 제공 테스트 도구가 내부적으로 어떻게 작동하는지 살펴보는 것도 흥미로울 것입니다.

오늘은 `act()` 에 대해 살펴보겠습니다.

## 1\. act() 데모

[공식 문서](https://reactjs.org/docs/testing-recipes.html#act)에 따르면

> UI 테스트를 작성할 때 렌더링, 사용자 이벤트 또는 데이터 불러오기와 같은 작업들은 사용자 인터페이스와의 상호작용 "units"로 간주할 수 있습니다. react-dom/test-utils는 assertion을 만들기 전에 이러한 "units"와 관련된 모든 업데이트가 처리되어 DOM에 적용되는지 확인하는 act()라는 헬퍼를 제공합니다.

간단히 말해, `act()`에서 예약된 작업은 동기적으로 실행되어야 합니다.

다음은 [데모](https://jser.dev/demos/react/act/without-act.html)입니다.

```typescript
function App() {
  useEffect(() => {
    console.log("effect");
  });
  return null;
}
const root = ReactDOM.createRoot(document.getElementById("container"));
root.render(<App />);
console.log("after render");
```

동시 모드의 렌더링은 비동기식이며 패시브 이펙트도 마찬가지이므로, 로그 순서는 다음과 같을 것으로 예상할 수 있습니다.

```typescript
after render
effect
```

[여기에서](https://jser.dev/demos/react/act/without-act.html) 시도해 볼 수 있습니다.

이제 렌더링을 실제 동작에 적용해 보겠습니다. [시도하기](https://jser.dev/demos/react/act/with-act.html)

```typescript
function App() {
  useEffect(() => {
    console.log("effect");
  });
  return null;
}
const root = ReactDOM.createRoot(document.getElementById("container"));
React.unstable_act(() => root.render(<App />));
console.log("after render");
```

이제 순서가 변경되고 효과가 동시에 플러시됩니다.

```typescript
effect
after render
```

비동기 동작이 React의 블랙박스 안에 있기 때문에 테스트 중에 유용할 수 있으며, await 등이 없는채로 assert 하기가 더 쉬워질 수 있습니다.

## 2\. act()는 어떻게 동작하나요?

[React의 이펙트 훅의 수명 주기](https://ted-projects.com/react-internals-deep-dive-16)에서 이펙트가 어떻게 실행되는지 간략하게 설명했습니다.

간단히 말해, React 런타임은 새로운 버전의 파이버 트리를 생성하고 필요한 변경 사항을 DOM에 커밋하며, 패시브 효과가 있는 경우 **스케줄러는 해당 효과를 실행하기 위해 작업을 예약합니다**. 다음은 [React 리포지토리에 있는 코드](https://github.com/facebook/react/blob/5a1e558df21bd3cafbaea01cc418fa69d14a8cab/packages/react-reconciler/src/ReactFiberWorkLoop.old.js#L2151)입니다.

스케줄러의 개념은 작업을 계속 진행하되 메인 스레드를 너무 오래 차단하지 않고 `setImmediate()` 또는 `MessageChannel` 콜백 또는 `setTimeout`을 통해 비동기적으로 작업을 스케줄링하는 것입니다. 이 글 - [스케줄러의 작동 방식](https://ted-projects.com/react-internals-deep-dive-20)에서 설명했습니다.

따라서 `act()` 가 작동하게 하려면 어떤 조건에서 동기화하도록 스케줄러의 동작을 변경해야 합니다. React는 이와 같은 작업을 수행하지만, 스케줄러의 내부를 변경하는 대신 스케줄러를 우회합니다.

[ReactAct.js](https://github.com/facebook/react/blob/5a1e558df21bd3cafbaea01cc418fa69d14a8cab/packages/react/src/ReactAct.js)는 `act()`의 모듈로 약간 긴데, 이를 분해해 보겠습니다.

1. `ReactCurrentActQueue`는 매우 중요하며, 스케줄러의 작업 대기열과 비슷한 것으로 생각할 수 있습니다.
    
2. `act(callback)` 이 호출되면 [여기](https://github.com/facebook/react/blob/5a1e558df21bd3cafbaea01cc418fa69d14a8cab/packages/react/src/ReactAct.js#L38)에 콜백이 호출됩니다.
    
3. act 큐에서 `performConcurrentWorkOnRoot()`가 예약됩니다.
    
4. 이 큐는 `flushActQueue()`에 의해 플러시(flush)됩니다. [여기](https://github.com/facebook/react/blob/5a1e558df21bd3cafbaea01cc418fa69d14a8cab/packages/react/src/ReactAct.js#L121)
    
    * `performConcurrentWorkOnRoot()` 는 `flushPassiveEffects()`의 콜백을 Act Queue로 푸시합니다.
        
    * `flushActQueue()` 는 모든 작업을 계속 플러시합니다.
        

위의 흐름은 간단해 보이지만 한 가지 큰 의문이 있습니다:

## 3\. act queue에서 콜백은 어떻게 예약되나요?

먼저 `ReactCurrentActQueue`를 정의하고 [ReactCurrentActQueue.js](https://github.com/facebook/react/blob/5a1e558df21bd3cafbaea01cc418fa69d14a8cab/packages/react/src/ReactCurrentActQueue.js#L10)에서 export 합니다.

```typescript
type RendererTask = (boolean) => RendererTask | null;
const ReactCurrentActQueue = {
  current: (null: null | Array<RendererTask>),
  // Used to reproduce behavior of `batchedUpdates` in legacy mode.
  isBatchingLegacy: false,
  didScheduleLegacyUpdate: false,
};
export default ReactCurrentActQueue;
```

단순히 배열에 대한 `current` 참조를 보관하기만 하면 됩니다.

그리고 브레이크포인트를 통해 `performConcurrentWorkOnRoot()`가 어떻게 예약되는지 콜 스택에서 쉽게 확인할 수 있습니다.

[![](https://jser.dev/static/act-2.png align="left")](https://jser.dev/static/act-2.png)

모든 마법은 `scheduleCallback()`에 있으며, 여기 [코드](https://github.com/facebook/react/blob/5a1e558df21bd3cafbaea01cc418fa69d14a8cab/packages/react-reconciler/src/ReactFiberWorkLoop.new.js#L3137)가 있습니다.

```typescript
function scheduleCallback(priorityLevel, callback) {
  if (__DEV__) {
    // If we're currently inside an `act` scope, bypass Scheduler and push to
    // the `act` queue instead.
    const actQueue = ReactCurrentActQueue.current;
    if (actQueue !== null) {
      actQueue.push(callback);
      return fakeActCallbackNode;
    } else {
      return Scheduler_scheduleCallback(priorityLevel, callback);
    }
  } else {
    // In production, always call Scheduler. This function will be stripped out.
    return Scheduler_scheduleCallback(priorityLevel, callback);
  }
}
```

우리는 분명히 알 수 있습니다.

1. act 큐가 있으면 콜백이 대기열에 푸시됩니다.
    
2. 그렇지 않으면 스케줄러로 이동합니다.
    

`performConcurrentWorkOnRoot()` 이후, 커밋 단계에서 플러시할 패시브 이펙트가 있다는 것을 알 수 있습니다. [코드](https://github.com/facebook/react/blob/5a1e558df21bd3cafbaea01cc418fa69d14a8cab/packages/react-reconciler/src/ReactFiberWorkLoop.new.js#L2137)

```typescript
if (
  (finishedWork.subtreeFlags & PassiveMask) !== NoFlags ||
  (finishedWork.flags & PassiveMask) !== NoFlags
) {
  if (!rootDoesHavePassiveEffects) {
    rootDoesHavePassiveEffects = true;
    pendingPassiveEffectsRemainingLanes = remainingLanes;
    // workInProgressTransitions might be overwritten, so we want
    // to store it in pendingPassiveTransitions until they get processed
    // We need to pass this through as an argument to commitRoot
    // because workInProgressTransitions might have changed between
    // the previous render and commit if we throttle the commit
    // with setTimeout
    pendingPassiveTransitions = transitions;
    scheduleCallback(NormalSchedulerPriority, () => {
      flushPassiveEffects();
      // This render triggered passive effects: release the root cache pool
      // *after* passive effects fire to avoid freeing a cache pool that may
      // be referenced by a node in the tree (HostRoot, Cache boundary etc)
      return null;
    });
  }
}
```

`scheduleCallback()`이 다시 호출되는 것을 볼 수 있는데, 이는 새 콜백이 액트 큐에 푸시되고 있으므로 처리되는 동안 큐가 커질 수 있음을 의미합니다.

## 4\. act queue는 어떻게 export 되나요?

흥미로운 점은 위에서 설명한 것처럼 React 런타임의 경우 대기열을 [ReactSharedInternals](https://github.com/facebook/react/blob/4c03bb6ed01a448185d9a1554229208a9480560d/packages/shared/ReactSharedInternals.js)로 가져온다는 점입니다.

```typescript
import * as React from "react";
const ReactSharedInternals =
  React.__SECRET_INTERNALS_DO_NOT_USE_OR_YOU_WILL_BE_FIRED;
export default ReactSharedInternals;
```

그리고 ... 네, 글로벌 `__SECRET_INTERNALS_DO_NOT_USE_OR_YOU_WILL_BE_FIRED`를 통해 이루어집니다. 그래요, 좋은 이름입니다.

```typescript
export {
  ...
  ReactSharedInternals as __SECRET_INTERNALS_DO_NOT_USE_OR_YOU_WILL_BE_FIRED,
  ...
};
```

[ReactSharedInternals](https://github.com/facebook/react/blob/4c03bb6ed01a448185d9a1554229208a9480560d/packages/react/src/ReactSharedInternals.js)에 별칭(alisas)으로 export합니다.

```typescript
if (__DEV__) {
 ReactSharedInternals.ReactDebugCurrentFrame = ReactDebugCurrentFrame;
 ReactSharedInternals.ReactCurrentActQueue = ReactCurrentActQueue;
}
```

따라서 런타임의 코드는 `__DEV__`에 의해 보호되며, 프로덕션 빌드에서는 제거될 것입니다.

그냥 import하면 안 되나요? 글쎄요... 아직은 코딩 관례에 가깝다고 생각하는데, 더 자세히 알고 계신다면 알려주세요.

## 5\. 요약

자, `act()`는 여기까지 입니다, 간단히 정리해 보겠습니다.

1. `act()`가 호출되면 공유 콜백 큐인 `ReactCurrentActQueue`가 초기화됩니다.
    
2. React 런타임에서 `__DEV__` 아래에 있고 `ReactCurrentActQueue`가 비어 있지 않으면 예약된 콜백이 스케줄러에서 예약되지 않고 대기열로 푸시됩니다.
    
3. `act()`에서 `ReactCurrentActQueue`가 처리되고 비워질 때까지 플러시됩니다.
    

이는 `act()`에서 이펙트가 동기적으로 플러시되는 이유에 대한 답변으로, `flushPassiveEffects()`로 예약된 작업이 큐로 이동하기 때문에 React 스케줄러가 우회(bypassed)됩니다.

(원본 게시일: 2022-05-15)