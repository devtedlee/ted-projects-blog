---
title: "[번역] 리액트 동시성 모드에서 Suspense는 어떻게 동작하는가 1 - Reconciling flow"
datePublished: Mon Apr 15 2024 12:43:44 GMT+0000 (Coordinated Universal Time)
cuid: clv0y4owu000h08jw7hkw8op9
slug: react-internals-deep-dive-7-1
cover: https://cdn.hashnode.com/res/hashnode/image/upload/v1713184926097/c07871fa-e29d-4e24-acc4-34a02923fe8e.jpeg
tags: react-internals

---

> 영문 블로그 글을 번역했습니다. 허가를 받으면 시리즈를 이어갈 예정입니다.  
> 원문링크: [https://jser.dev/react/2022/04/02/suspense-in-concurrent-mode-1-reconciling](https://jser.dev/react/2022/04/02/suspense-in-concurrent-mode-1-reconciling)

---

> ℹ️ [React Internals Deep Dive](https://jser.dev/series/react-source-code-walkthrough.html) 에피소드 22, [유튜브에서 제가 설명하는 것](https://www.youtube.com/watch?v=tnuS4cfMhF8&list=PLvx8w9g4qv_p-OS-XdbB3Ux_6DMXhAJC3&index=22)을 시청해주세요. 💬 역자 주석: 에피소드가 건너뛰게 된 건 시리즈 순서상 7번째가 맞지만 에피소드는 22이기 때문입니다. 원 글의 순서대로 진행하는 중입니다.
> 
> ⚠ [React@18.2.0](https://github.com/facebook/react/releases/tag/v18.2.0) 기준, 최신 버전에서는 구현이 변경되었을 수 있습니다.

제가 Suspense가 어떻게 작동하는지 알아내려고 노력한 적이 있는데, [유튜브 동영상](https://www.youtube.com/watch?v=4Ippewm6AXk)을 보시면 아시겠지만 매우 거칠고 React 18의 최신 로직도 반영하지 못했습니다.

이제 **서스펜스가 동시 모드에서 어떻게 작동하는지** 좀 더 자세히 살펴보겠습니다. 매우 복잡함으로 다음 단계에 걸쳐 몇 개의 에피소드로 나누어 설명할 계획입니다.

1. 조정(reconciling) - Suspense가 조정하는 방법
    
2. Offscreen 컴포넌트 - Suspense 컴포넌트가 사용하는 내부 컴포넌트
    
3. Suspense Context - ??
    
4. Ping & Retry - Promise가 해결된 후 멋지게 리-렌더링하세요.
    

이 에피소드는 1은 조정(reconciling)에 관한 내용입니다.

## Susponse 데모

[기초 Suspense 데모](https://jser.dev/demos/react/suspense/basic.html)를 열어보세요.

[![](https://jser.dev/static/basic-suspense.gif align="left")](https://jser.dev/static/basic-suspense.gif)

코드는 매우 간단하며, 데이터가 준비되지 않았을 때 Promise를 던지는 기본적인 구현일 뿐입니다.

```typescript
const getResource = (data, delay = 1000) => ({
  _data: null,
  _promise: null,
  status: "pending",
  get data() {
    if (this.status === "ready") {
      return this._data;
    } else {
      if (this._promise == null) {
        this._promise = new Promise((resolve) => {
          setTimeout(() => {
            this._data = data;
            this.status = "ready";
            resolve();
          }, delay);
        });
      }
      throw this._promise;
    }
  },
});
function App() {
  const [resource, setResource] = React.useState(null);
  return (
    <div className="app">
      <button
        onClick={() => {
          setResource(getResource("JSer"));
        }}
      >
        start
      </button>
      <React.Suspense fallback={<p>loading...</p>}>
        <Child resource={resource} />
      </React.Suspense>
    </div>
  );
}
```

예상대로, 리소스가 로딩중일 때 Fallback이 표시됩니다.

## 먼저 Suspense 컴포넌트가 어떻게 렌더링 되는지 살펴봅시다

`beginWork()`내에 있는 이 코드 조각을 찾을 수 있습니다. [소스](https://github.com/facebook/react/blob/b8cfda15e1232554487c7285fb464f22705a23ce/packages/react-reconciler/src/ReactFiberBeginWork.new.js#L3925)

```typescript
case SuspenseComponent:
  return updateSuspenseComponent(current, workInProgress, renderLanes);
```

Suspense의 최초 렌더링과 업데이트가 모두 `updateSuspenseComponent`에 있다는 의미로, 거대한 코드 [소스](https://github.com/facebook/react/blob/b8cfda15e1232554487c7285fb464f22705a23ce/packages/react-reconciler/src/ReactFiberBeginWork.new.js#L2343) 덩어리(chunk)인데, 이를 분석해 보겠습니다.

```typescript
function updateSuspenseComponent(current, workInProgress, renderLanes) {
  const nextProps = workInProgress.pendingProps;
  let suspenseContext: SuspenseContext = suspenseStackCursor.current;
  let showFallback = false;
  const didSuspend = (workInProgress.flags & DidCapture) !== NoFlags;
  if (
    didSuspend ||
    shouldRemainOnFallback(suspenseContext, current, workInProgress, renderLanes)
  ) {
    // Something in this boundary's subtree already suspended. Switch to
    // rendering the fallback children.
    showFallback = true;
    workInProgress.flags &= ~DidCapture;
  }
```

먼저 `SuspenseContext`가 무엇인지, 이 부분은 다음 에피소드(Suspense 2편)에서 알아볼 예정입니다. 지금은 건너뛰겠습니다.

`showFallback`은 매우 간단하며, 폴백 표시 여부를 결정하는 변수이며 기본값은 false입니다.

`showFallback`은 `DidSuspend`에 의존하고, 다시 `DidCapture`에 의존하는 것을 볼 수 있으며, 이는 매우 중요한 플래그이므로 유의 해주세요.

`shouldRemainOnFallback()`은 Suspense Context와 관련된 것이므로 다음 에피소드에서 다루겠습니다.

향후 리-렌더링에서 올바른 콘텐츠를 얻기 위해 `DidCapture`가 제거되었음을 알 수 있으며, 이는 또한 Promise가 다시 throw됐음을 의미합니다. (이 [데모](https://jser.dev/demos/react/suspense/rethrow.html)를 사용해 보세요)

### 최초 마운트

```typescript
if (current === null) {
  const nextPrimaryChildren = nextProps.children;
  const nextFallbackChildren = nextProps.fallback;
  if (showFallback) {
    const fallbackFragment = mountSuspenseFallbackChildren(
      workInProgress,
      nextPrimaryChildren,
      nextFallbackChildren,
      renderLanes
    );
    const primaryChildFragment: Fiber = (workInProgress.child: any);
    primaryChildFragment.memoizedState =
      mountSuspenseOffscreenState(renderLanes);
    workInProgress.memoizedState = SUSPENDED_MARKER;
    return fallbackFragment;
  } else {
    return mountSuspensePrimaryChildren(
      workInProgress,
      nextPrimaryChildren,
      renderLanes
    );
  }
}
```

`current === null`은 최초 렌더링을 의미합니다. `mountSuspenseFallbackChildren()`은 primary children(content)과 폴백을 모두 마운트하지만 폴백 프래그먼트를 반환합니다.

`memoizedState`도 초기화되며, 이는 이 Suspense가 폴백을 렌더링하고 있음을 나타내는 마커입니다.

폴백을 렌더링하지 않는 경우, `mountSuspenseFallbackChildren()`이 자식을 마운트합니다.

이 에피소드의 뒷부분에서 `mountSuspenseFallbackChildren()` 및 `mountSuspensePrimaryChildren()`에 대해 다시 살펴보겠습니다.

### 업데이트

그리고 업데이트의 경우, 로직은 실제로 비슷하며, 현재 상태와 상태에 따라 네 가지 분기가 있습니다. 이에 대해 자세히 다룰 것입니다.

```typescript
} else {
  // This is an update.
  // If the current fiber has a SuspenseState, that means it's already showing
  // a fallback.
  const prevState: null | SuspenseState = current.memoizedState;
  if (prevState !== null) {
    // The current tree is already showing a fallback
    if (showFallback) {
      // prev: fallback, now: fallback
      ...
    } else {
      // prev: fallback, now: content
      ...
    }
  } else {
    if (showFallback) {
     // prev: content, now: callback
     ...
    } else {
      // prev: content, now: content
      ...
    }
  }
}
```

#### prev: fallback, now: fallback

```typescript
prev: fallback, now: fallback#
const nextFallbackChildren = nextProps.fallback;
const nextPrimaryChildren = nextProps.children;
const fallbackChildFragment = updateSuspenseFallbackChildren(
  current,
  workInProgress,
  nextPrimaryChildren,
  nextFallbackChildren,
  renderLanes
);
const primaryChildFragment: Fiber = (workInProgress.child: any);
const prevOffscreenState: OffscreenState | null = (current.child: any)
  .memoizedState;
primaryChildFragment.memoizedState =
  prevOffscreenState === null
    ? mountSuspenseOffscreenState(renderLanes)
    : updateSuspenseOffscreenState(prevOffscreenState, renderLanes);
primaryChildFragment.childLanes = getRemainingWorkInPrimaryTree(
  current,
  renderLanes
);
workInProgress.memoizedState = SUSPENDED_MARKER;
return fallbackChildFragment;
```

둘 다 폴백을 렌더링하지만 폴백 자체가 변경될 수 있습니다. `updateSuspenseFallbackChildren()`이 조정(reconciling)합니다. OffscreenState 부분은 약간 혼란스러운데, Suspense Cache와 관련이 있으므로 다음 편을 위해 남겨 두겠습니다.

#### prev: fallback, now: content

```typescript
const nextPrimaryChildren = nextProps.children;
const primaryChildFragment = updateSuspensePrimaryChildren(
  current,
  workInProgress,
  nextPrimaryChildren,
  renderLanes
);
workInProgress.memoizedState = null;
return primaryChildFragment;
```

여기는 간단히, 그냥 자식 파트를 조정(reconcile)합니다.

#### prev: content, now: callback

코드가 `prev: fallback, now: content` 과 비슷합니다. 넘어갑니다.

#### prev: content, now: content

코드가 `prev: fallback, now: content` 과 비슷합니다.

## Suspense 내부 래퍼(Wrapper)

Suspense 컴포넌트는 단순한 컴포넌트가 아니라 Offscreen 컴포넌트 같은 것으로 자식들을 감싸서 멋진 무언가를 만들어냅니다.

지금부터 Offscreen에 대해 간략하게 살펴보고 향후 에피소드에서 자세히 알아보겠습니다.

### mountSuspenseFallbackChildren()

좋습니다, 이제부터 `mountSuspenseFallbackChildren()`에 실제로 무슨 일이 생기는지 봐봅시다.

```typescript
function mountSuspenseFallbackChildren(
  workInProgress,
  primaryChildren,
  fallbackChildren,
  renderLanes
) {
  const mode = workInProgress.mode;
  const progressedPrimaryFragment: Fiber | null = workInProgress.child;
  const primaryChildProps: OffscreenProps = {
    mode: "hidden",
    children: primaryChildren,
  };
  let primaryChildFragment;
  let fallbackChildFragment;
  primaryChildFragment = mountWorkInProgressOffscreenFiber(
    primaryChildProps,
    mode,
    NoLanes
  );
  fallbackChildFragment = createFiberFromFragment(
    fallbackChildren,
    mode,
    renderLanes,
    null
  );
  primaryChildFragment.return = workInProgress;
  fallbackChildFragment.return = workInProgress;
  primaryChildFragment.sibling = fallbackChildFragment;
  workInProgress.child = primaryChildFragment;
  return fallbackChildFragment;
}
```

1. primary child가 Offscreen Fiber로 래핑되고 모드가 `hidden`으로 설정됨
    
2. 폴백은 Fragment로 래핑됩니다.
    
3. primary child와 폴백은 모두 자식으로 배치됩니다.
    

왜 폴백을 프래그먼트로 감싸는 걸까요?

제가 추측하기로 폴백은 `ReactNodeList`의 타입이며 숫자나 문자열일 수 있고 일반적으로 문자열은 특별한 처리를 해야 하기 때문에 Fragment로 감싸는 것이 처리하기 쉬워서 일 것 입니다.

```typescript
export type ReactNode =
  | React$Element<any>
  | ReactPortal
  | ReactText
  | ReactFragment
  | ReactProvider<any>
  | ReactConsumer<any>;
export type ReactEmpty = null | void | boolean;
export type ReactFragment = ReactEmpty | Iterable<React$Node>;
export type ReactNodeList = ReactEmpty | React$Node;
export type ReactText = string | number;
```

자, 여기 Suspense에 대한 파이버 구조 다이어그램입니다.

[![](https://jser.dev/static/suspense-fiber-structure-hidden.png align="left")](https://jser.dev/static/suspense-fiber-structure-hidden.png)

`mountWorkInProgressOffscreenFiber`의 특별한 점이 무엇일까요?

```typescript
function mountWorkInProgressOffscreenFiber(
  offscreenProps: OffscreenProps,
  mode: TypeOfMode,
  renderLanes: Lanes
) {
  // The props argument to `createFiberFromOffscreen` is `any` typed, so we use
  // this wrapper function to constrain it.
  return createFiberFromOffscreen(offscreenProps, mode, NoLanes, null);
}
export function createFiberFromOffscreen(
  pendingProps: OffscreenProps,
  mode: TypeOfMode,
  lanes: Lanes,
  key: null | string
) {
  const fiber = createFiber(OffscreenComponent, pendingProps, key, mode);
  fiber.elementType = REACT_OFFSCREEN_TYPE;
  fiber.lanes = lanes;
  const primaryChildInstance: OffscreenInstance = {};
  fiber.stateNode = primaryChildInstance;
  return fiber;
}
```

화려하진 않지만, `mode` 속성을 통해 `hidden` 또는 `visible` 여부를 표시할 수 있습니다.

### mountSuspensePrimaryChildren()

```typescript
function mountSuspensePrimaryChildren(
  workInProgress,
  primaryChildren,
  renderLanes
) {
  const mode = workInProgress.mode;
  const primaryChildProps: OffscreenProps = {
    mode: "visible",
    children: primaryChildren,
  };
  const primaryChildFragment = mountWorkInProgressOffscreenFiber(
    primaryChildProps,
    mode,
    renderLanes
  );
  primaryChildFragment.return = workInProgress;
  workInProgress.child = primaryChildFragment;
  return primaryChildFragment;
}
```

여기에서도 Offscreen Fiber를 사용하지만 이번에는 폴백이 없고 모드가 "visible"입니다.

[![](https://jser.dev/static/suspense-fiber-structure-visible.png align="left")](https://jser.dev/static/suspense-fiber-structure-visible.png)

참고로 `workInProgress`도 `mode`를 가졌지만, 다른 모드인 `TypeOfMode`가 있습니다.

```typescript
export type TypeOfMode = number;
export const NoMode = /*                         */ 0b000000;
// TODO: Remove ConcurrentMode by reading from the root tag instead
export const ConcurrentMode = /*                 */ 0b000001;
export const ProfileMode = /*                    */ 0b000010;
export const DebugTracingMode = /*               */ 0b000100;
export const StrictLegacyMode = /*               */ 0b001000;
export const StrictEffectsMode = /*              */ 0b010000;
export const ConcurrentUpdatesByDefaultMode = /* */ 0b100000;
```

왜 primary children을 파이버0 트리에 남겨두는지 궁금할 수 있습니다. 왜 그냥 제거하지 않을까요? 훌륭한 질문입니다, 간단히 말하자면 파이버의 state를 유지하기 위해서이며, 폴백에서 다시 전환한 후 모든 것이 새것으로 바뀌는 것을 원하지 않기 때문입니다. 자세한 내용은 다음 Offscreen 에피소드에서 확인할 수 있습니다.

이제 Promise가 어떻게 작동하는지 알아보겠습니다.

## Suspense에서 Promise가 어떻게 잡히고 업데이트가 트리거 되나요?

우리는 이미 에러 처리의 일부인 promise가 throw 될 때 서스펜스가 반응한다는 것을 알고 있으므로 먼저 `handleError`로 가보겠습니다. [(소스](https://github.com/facebook/react/blob/5a1e558df21bd3cafbaea01cc418fa69d14a8cab/packages/react-reconciler/src/ReactFiberWorkLoop.new.js#L1553))

```typescript
function handleError(root, thrownValue): void {
  do {
    let erroredWork = workInProgress;
    try {
      // Reset module-level state that was set during the render phase.
      resetContextDependencies();
      resetHooksAfterThrow();
      // TODO: I found and added this missing line while investigating a
      // separate issue. Write a regression test using string refs.
      ReactCurrentOwner.current = null;
      throwException(
        root,
        erroredWork.return,
        erroredWork,
        thrownValue,
        workInProgressRootRenderLanes
      );
      completeUnitOfWork(erroredWork);
    } catch (yetAnotherThrownValue) {
      // Something in the return path also threw.
      thrownValue = yetAnotherThrownValue;
      if (workInProgress === erroredWork && erroredWork !== null) {
        // If this boundary has already errored, then we had trouble processing
        // the error. Bubble it to the next boundary.
        erroredWork = erroredWork.return;
        workInProgress = erroredWork;
      } else {
        erroredWork = workInProgress;
      }
      continue;
    }
    // Return to the normal work loop.
    return;
  } while (true);
}
```

따라서 핵심 부분은 다음 두 가지 함수 호출입니다.

1. `throwException`
    
2. `completeUnitOfWork`
    

### throwException

[소스](https://github.com/facebook/react/blob/b8cfda15e1232554487c7285fb464f22705a23ce/packages/react-reconciler/src/ReactFiberThrow.new.js#L430)

거대한 코드 덩어리이므로 세분화해 보겠습니다. 먼저 던지는 파이버는 Incomplete로 표시됩니다.

```typescript
// The source fiber did not complete.
sourceFiber.flags |= Incomplete;
```

그런 다음 오류가 then 호출이 가능한지 확인하고, 만약 then 호출이 가능하면 컴포넌트가 일시 중단됩니다.

```typescript
if (
  value !== null &&
  typeof value === 'object' &&
  typeof value.then === 'function'
) {
  // This is a wakeable. The component suspended.
  const wakeable: Wakeable = (value: any);
  ...
} else {
  // regular error
}
```

`wakeable` 은 그냥 던져지는 Promise라고 생각하면 됩니다. Promise가 아니라면 Error Boundary에서 처리해야 하는 일반 에러일 뿐입니다([ErrorBoundary에 대한 동영상](https://www.youtube.com/watch?v=0TnuJKLjMyg) 보기).

이제 Suspense 브랜치에 집중해 보겠습니다.

```typescript
// Schedule the nearest Suspense to re-render the timed out view.
const suspenseBoundary = getNearestSuspenseBoundaryToCapture(returnFiber);
```

먼저 가장 가까운 Suspense를 가져옵니다. Suspense Boundary라고 부르는데, Error Boundary와 매우 유사하다는 것을 알 수 있습니다.

`getNearestSuspenseBoundaryToCapture`는 `return`을 보고 조상 파이버 노드를 재귀적으로 역추적하는 간단한 함수이므로 생략하겠습니다. [소스](https://github.com/facebook/react/blob/b8cfda15e1232554487c7285fb464f22705a23ce/packages/react-reconciler/src/ReactFiberThrow.new.js#L277).

```typescript
if (suspenseBoundary !== null) {
  suspenseBoundary.flags &= ~ForceClientRender;
  markSuspenseBoundaryShouldCapture(
    suspenseBoundary,
    returnFiber,
    sourceFiber,
    root,
    rootRenderLanes
  );
  // We only attach ping listeners in concurrent mode. Legacy Suspense always
  // commits fallbacks synchronously, so there are no pings.
  if (suspenseBoundary.mode & ConcurrentMode) {
    attachPingListener(root, wakeable, rootRenderLanes);
  }
  attachRetryListener(suspenseBoundary, root, wakeable, rootRenderLanes);
  return;
}
```

Suspense Boundary를 찾은 후에, 우리가 할 것은 3가지 입니다

1. `markSuspenseBoundaryShouldCapture()`
    
2. `attachPingListener()`
    
3. `attachRetryListener()`
    

분명히 `markSuspenseBoundaryShouldCapture()` 는 Suspense가 폴백을 렌더링하기 위한 것이고, 나머지 두 개는 어떻게든 콜백을 프로미스에 첨부(attach)하는 것인데, 왜냐하면 폴백들이 안정(setteled)되면 content를 렌더링해야 하기 때문입니다.

2와 3에 대해서는 향후 Ping & Retry 에피소드에서 자세히 설명할 예정입니다.

### Suspense를 찾지 못하면 어떻게 할까요?

코드를 계속 진행하면 SyncLane이 아니라면 Suspense Boundary가 없어도 괜찮다는 것을 알 수 있습니다.

```typescript
else {
  // No boundary was found. Unless this is a sync update, this is OK.
  // We can suspend and wait for more data to arrive.
  if (!includesSyncLane(rootRenderLanes)) {
    // This is not a sync update. Suspend. Since we're not activating a
    // Suspense boundary, this will unwind all the way to the root without
    // performing a second pass to render a fallback. (This is arguably how
    // refresh transitions should work, too, since we're not going to commit
    // the fallbacks anyway.)
    //
    // This case also applies to initial hydration.
    attachPingListener(root, wakeable, rootRenderLanes);
    renderDidSuspendDelayIfPossible();
    return;
  }
  // This is a sync/discrete update. We treat this case like an error
  // because discrete renders are expected to produce a complete tree
  // synchronously to maintain consistency with external state.
  const uncaughtSuspenseError = new Error(
    "A component suspended while responding to synchronous input. This " +
      "will cause the UI to be replaced with a loading indicator. To " +
      "fix, updates that suspend should be wrapped " +
      "with startTransition."
  );
  // If we're outside a transition, fall through to the regular error path.
  // The error will be caught by the nearest suspense boundary.
  value = uncaughtSuspenseError;
}
```

간단히 말해, 서스펜스가 사용자 행동(action)으로 인해 발생하는 것이라면, Suspense Boundary가 있어야만 합니다.

사용자 행동(action)이 아니거나 전환(transition) 중이면 `attachPingListener()` 와 `renderDidSuspendDelayIfPossible()` 이 복구를 시도합니다.

다음은 [전환(transition)을 사용하지만 Suspense Boundary가 없는 데모](https://jser.dev/demos/react/suspense/transition.html)로, 여전히 작동하는 것을 확인할 수 있습니다.

#### markSuspenseBoundaryShouldCapture()

[소스](https://github.com/facebook/react/blob/b8cfda15e1232554487c7285fb464f22705a23ce/packages/react-reconciler/src/ReactFiberThrow.new.js#L297)

`markSuspenseBoundaryShouldCapture()`에서는 동시(Concurrent) 모드 이전에 사용된 Legacy Suspense를 처리하는데, 이는 제가 이전에 만난 버전이므로 무시하고 동시 모드에만 집중해 보겠습니다.

```typescript
suspenseBoundary.flags |= ShouldCapture;
```

`ShouldCapture`가 여기에 설정되어 있으면 `DidCapture`로 변환되는 단계가 있을 것입니다. 잠시만 기다려 보겠습니다.

```typescript
sourceFiber.flags |= ForceUpdateForLegacySuspense;
// We're going to commit this fiber even though it didn't complete.
// But we shouldn't call any lifecycle methods or callbacks. Remove
// all lifecycle effect tags.
sourceFiber.flags &= ~(LifecycleEffectMask | Incomplete);
```

source 파이버의 경우 이미 Incomplete로 표시했지만 여기서는 플래그가 제거되었습니다.

```typescript
export const LifecycleEffectMask =
  Passive | Update | Callback | Ref | Snapshot | StoreConsistency;
```

`LifecycleEffectMask`에는 모든 부수 효과(side effects)가 포함되어 있습니다.

#### **실제로는 완료되지 않은 가짜 완료로 취급하고** 있습니다.

```typescript
// The source fiber did not complete. Mark it with Sync priority to
// indicate that it still has pending work.
sourceFiber.lanes = mergeLanes(sourceFiber.lanes, SyncLane);
```

이는 Suspense 렌더링 시 DidCapture를 제거하는 것과 관련이 있습니다. 리-렌더링할 때 오류가 발생한 컴포넌트가 리-렌더링되도록 하려면 `lanes`를 설정하여 [bailout](https://jser.dev/react/2022/01/07/how-does-bailout-work)을 피해야 합니다.

그런 다음 `completeUnitOfWork(erroredWork)`로 이동합니다.

# completeUnitOfWork

throwException()이 완료된 후 `completeUnitOfWork()`가 호출됩니다. [(소스](https://github.com/facebook/react/blob/b8cfda15e1232554487c7285fb464f22705a23ce/packages/react-reconciler/src/ReactFiberWorkLoop.new.js#L1858))

서스펜스에서는 작업이 Incomplete이므로 Incomplete 브랜치만 살펴보겠습니다.

```typescript
function completeUnitOfWork(unitOfWork: Fiber): void {
  // Attempt to complete the current unit of work, then move to the next
  // sibling. If there are no more siblings, return to the parent fiber.
  let completedWork = unitOfWork;
  do {
    // The current, flushed, state of this fiber is the alternate. Ideally
    // nothing should rely on this, but relying on it here means that we don't
    // need an additional field on the work in progress.
    const current = completedWork.alternate;
    const returnFiber = completedWork.return;
    // Check if the work completed or if something threw.
    if ((completedWork.flags & Incomplete) === NoFlags) {
      ...
    } else {
      // This fiber did not complete because something threw. Pop values off
      // the stack without entering the complete phase. If this is a boundary,
      // capture values if possible.
      const next = unwindWork(current, completedWork, subtreeRenderLanes);
      // Because this fiber did not complete, don't reset its lanes.
      if (next !== null) {
        // If completing this work spawned new work, do that next. We'll come
        // back here again.
        // Since we're restarting, remove anything that is not a host effect
        // from the effect tag.
        next.flags &= HostEffectMask;
        workInProgress = next;
        return;
      }
      if (returnFiber !== null) {
        // Mark the parent fiber as incomplete and clear its subtree flags.
        returnFiber.flags |= Incomplete;
        returnFiber.subtreeFlags = NoFlags;
        returnFiber.deletions = null;
      } else {
        // We've unwound all the way to the root.
        workInProgressRootExitStatus = RootDidNotComplete;
        workInProgress = null;
        return;
      }
    }
    const siblingFiber = completedWork.sibling;
    if (siblingFiber !== null) {
      // If there is more work to do in this returnFiber, do that next.
      workInProgress = siblingFiber;
      return;
    }
    // Otherwise, return to the parent
    completedWork = returnFiber;
    // Update the next thing we're working on in case something throws.
    workInProgress = completedWork;
  } while (completedWork !== null);
  // We've reached the root.
  if (workInProgressRootExitStatus === RootInProgress) {
    workInProgressRootExitStatus = RootCompleted;
  }
}
```

[순회 알고리즘에 대한 포스팅](https://jser.dev/react/2022/01/16/fiber-traversal-in-react)에서 설명했듯이, completeUnitWork는 파이버 노드를 조정(reconcile)하는 마지막 단계입니다.

Incomplete 파이버 노드의 경우

```typescript
const next = unwindWork(current, completedWork, subtreeRenderLanes);
// Because this fiber did not complete, don't reset its lanes.
if (next !== null) {
  // If completing this work spawned new work, do that next. We'll come
  // back here again.
  // Since we're restarting, remove anything that is not a host effect
  // from the effect tag.
  next.flags &= HostEffectMask;
  workInProgress = next;
  return;
}
```

unwindWork에서 반환된 경우 작업을 계속할 수 있는 기회를 제공한다는 것을 알 수 있습니다.

또한 재귀적으로 조상 노드를 Incomplete로 표시합니다.

이름과 같이 `unwindWork`는 컨텍스트 등을 정리합니다. [소스](https://github.com/facebook/react/blob/b8cfda15e1232554487c7285fb464f22705a23ce/packages/react-reconciler/src/ReactFiberUnwindWork.new.js#L52)

```typescript
case SuspenseComponent: {
  popSuspenseContext(workInProgress);
  const flags = workInProgress.flags;
  if (flags & ShouldCapture) {
    workInProgress.flags = (flags & ~ShouldCapture) | DidCapture;
    // Captured a suspense effect. Re-render the boundary.
    if (
      enableProfilerTimer &&
      (workInProgress.mode & ProfileMode) !== NoMode
    ) {
      transferActualDuration(workInProgress);
    }
    return workInProgress;
  }
  return null;
}
```

Suspense로 언와인드 됐을 때, 우리는 다음과 같은 것을 볼 수 있습니다.

1. 서스펜스 컨텍스트가 튀어 나옵니다(pop), 다음 에피소드에서 다룰 예정입니다.
    
2. `ShouldCapture`를 찾으면 `DidCapture`를 수행하도록 설정하고 스스로 반환합니다.
    

맞아요, `ShouldCapture`는 완료 단계에서 `DidCapture`로 변환됩니다.

## 요약

긴 여정을 요약하면, 다음과 같습니다.

1. Suspense는 폴백 또는 contents(primary children)를 렌더링할지 여부를 결정하기 위해 `DidCapture` 플래그를 사용합니다.
    
2. Suspense는 contents를 Offscreen 컴포넌트로 래핑하여 폴백이 렌더링되더라도 파이버 트리에서 contents가 제거되지 않고 내부 state를 유지하도록 합니다.
    
3. 조정하는 동안, Suspense는 `DidCapture` 플래그를 기반으로 Offscreen 건너뛰기 여부를 결정하며, 이는 "일부 파이버를 숨기는" effect를 생성합니다.
    
4. promise가 던져질 때
    
    * 가장 가까운 Suspense Boundary가 발견되고 `ShouldCapture`로 플래그가 설정되면, promise는 ping & retry 리스너로 연결됩니다.
        
    * 에러 이후, 작업을 완료하기 시작하면 에러가 발생한 컴포넌트에서 Suspense까지 모든 파이버가 Incomplete로 완료됩니다.
        
    * 가장 가까운 Suspense를 완료하려고 할 때, `ShouldCapture`는 `DidCapture`로 표시되고 Suspense 자체를 반환합니다.
        
    * 작업루프(workloop)는 Suspense를 계속 조정하고, 이번에는 폴백 브랜치를 렌더링합니다.
        
5. promise가 해결(resolved)되면
    
    * ping & retry 리스너가 리-렌더링이 발생하게 합니다. (자세한 내용은 다음 에피소드에서 설명합니다.)
        

(원글 작성일: 2022-04-02)