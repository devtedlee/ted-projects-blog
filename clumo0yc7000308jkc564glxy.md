---
title: "[번역] React는 최초 마운트를 어떻게 수행하나요?"
datePublished: Fri Apr 05 2024 12:52:07 GMT+0000 (Coordinated Universal Time)
cuid: clumo0yc7000308jkc564glxy
slug: react-internals-deep-dive-2
cover: https://cdn.hashnode.com/res/hashnode/image/upload/v1712321357762/81b5c175-fe98-4c2a-b427-6fc627dbc794.jpeg
tags: react-internals

---

> 영문 블로그 글을 번역했습니다. 허가를 받으면 시리즈를 이어갈 예정입니다.  
> 원문링크: [https://jser.dev/2023-07-14-initial-mount/](https://jser.dev/2023-07-14-initial-mount/)

---

> ℹ️ [React Internals Deep Dive](https://jser.dev/series/react-source-code-walkthrough.html) 에피소드 2, [유튜브에서 제가 설명하는 것](https://www.youtube.com/watch?v=b7rrXJl5o5I&list=PLvx8w9g4qv_p-OS-XdbB3Ux_6DMXhAJC3&index=37)을 시청해주세요.
> 
> ⚠ [React@18.2.0](https://github.com/facebook/react/releases/tag/v18.2.0) 기준, 최신 버전에서는 구현이 변경되었을 수 있습니다.\\

[React Internals 개요](https://ted-projects.com/react-internals-deep-dive-1)에서 React는 내부적으로 트리-like 구조(Fiber Tree)를 사용하여 최소 DOM 업데이트를 계산하고 "Commit" 단계에서 이를 커밋한다고 간략하게 언급했습니다. 이 글에서는 React가 초기 마운트(최초 렌더링)를 정확히 어떻게 수행하는지 알아보겠습니다. 좀 더 구체적으로, 아래 코드를 통해 React가 DOM을 어떻게 구성하는지 알아보겠습니다.

```typescript
import {useState} from 'react'
function Link() {
  return <a href="https://jser.dev">jser.dev</a>
}
export default function App() {
  const [count, setCount] = useState(0)
  return (
    <div>
      <p>
        <Link/>
        <br/>
        <button onClick={() => setCount(count => count + 1)}>click me - {count}</button>
      </p>
    </div>
  );
}
```

* [코드 샌드박스 링크](https://codesandbox.io/p/sandbox/musing-lalande-xgm29g?file=%2Findex.js)
    

## 1\. 파이버 아키텍처에 대한 간략한 소개

[![파이버 아키텍쳐 다이어그램 ](https://cdn.hashnode.com/res/hashnode/image/upload/v1712231381102/725558eb-24a4-4210-b2b5-3459e7f7f7a8.png align="center")](https://cdn.hashnode.com/res/hashnode/image/upload/v1712231381102/725558eb-24a4-4210-b2b5-3459e7f7f7a8.png?auto=compress,format&format=webp)

Fiber는 React가 앱 상태를 내부적으로 표현하는 방식의 아키텍처입니다. FiberRootNode와 FiberNode로 구성된 트리-like 구조입니다. Fiber에는 모든 종류의 FiberNode가 있으며, 그 중 일부는 백업 DOM 노드인 HostComponent를 가지고 있습니다.

React 런타임은 파이버 트리를 유지 및 업데이트하고 최소한의 업데이트로 호스트 DOM을 동기화하기 위해 최선을 다합니다.

### 1.1 `FiberRootNode`

FiberRootNode는 React 루트 역할을 하는 특별한 노드로, 전체 앱에 대한 필요한 메타 정보를 보유합니다. 이 노드의 `current`는 실제 파이버 트리를 가리키며, 새로운 파이버 트리가 생성될 때마다 `current`는 새로운 `HostRoot`를 다시 가리킵니다.

### 1.2 `FiberNode`

FiberNode는 FiberRootNode를 제외한 모든 노드를 의미하며, 몇 가지 중요한 속성은 다음과 같습니다:

1. `tag`: FiberNode에는 `tag`별로 구분되는 많은 하위 유형이 있습니다. 예를 들어, FunctionComponent, HostRoot, ContextConsumer, MemoComponent, SuspenseComponent 등이 있습니다.
    
2. `stateNode`: 다른 백업 데이터를 가리키며, `HostComponent`의 경우`stateNode`는 실제 백업 DOM 노드를 가리킵니다.
    
3. `child`, `sibling`, 그리고 `return`: 이들은 함께 트리-like 구조를 형성합니다.
    
4. `elementType`: 우리가 제공하는 컴포넌트 함수 또는 고유 HTML 태그입니다.
    
5. `flags`: "Commit" 단계에서 적용할 업데이트들을 나타냅니다. `subtreeFlags`는 `flags`의 하위 트리입니다.
    
6. `lanes`: 보류 중인 업데이트들의 우선 순위를 나타냅니다. `lanes`의 하위 트리는 `childLanes`입니다.
    
7. `memoizedState`: 중요한 데이터를 가리키며, FunctionComponent의 경우 훅을 의미합니다.
    

## 2\. 최초 마운트: Trigger 단계

`createRoot()`는 `current`로 React 루트(생성된 더미 HostRoot FiberNode를 가진)를 생성합니다.

> 💬역자 주석: Jser의 코멘트는 ❗❗로 표시 해뒀습니다.  
> 그 외 주석은 리액트 소스 코드 자체의 주석입니다.  
> ... 은 생략된 코드입니다.

```typescript
export function createRoot(
  container: Element | Document | DocumentFragment,
  options?: CreateRootOptions,
): RootType {
  let isStrictMode = false;
  let concurrentUpdatesByDefaultOverride = false;
  let identifierPrefix = '';
  let onRecoverableError = defaultOnRecoverableError;
  let transitionCallbacks = null;
  
  // ❗❗ 이게 FiberRootNode를 반환합니다.
  const root = createContainer(
    container,
    ConcurrentRoot,
    null,
    isStrictMode,
    concurrentUpdatesByDefaultOverride,
    identifierPrefix,
    onRecoverableError,
    transitionCallbacks,
  );
  markContainerAsRoot(root.current, container);
  Dispatcher.current = ReactDOMClientDispatcher;
  const rootContainerElement: Document | Element | DocumentFragment =
    container.nodeType === COMMENT_NODE
      ? (container.parentNode: any)
      : container;
  listenToAllSupportedEvents(rootContainerElement);
  return new ReactDOMRoot(root); // ❗❗ ReactDOMRoot
}
```

```typescript
export function createContainer(
  containerInfo: Container,
  tag: RootTag,
  hydrationCallbacks: null | SuspenseHydrationCallbacks,
  isStrictMode: boolean,
  concurrentUpdatesByDefaultOverride: null | boolean,
  identifierPrefix: string,
  onRecoverableError: (error: mixed) => void,
  transitionCallbacks: null | TransitionTracingCallbacks,
): OpaqueRoot {
  const hydrate = false;
  const initialChildren = null;
  return createFiberRoot( // ❗❗ createFiberRoot
    containerInfo,
    tag,
    hydrate,
    initialChildren,
    hydrationCallbacks,
    isStrictMode,
    concurrentUpdatesByDefaultOverride,
    identifierPrefix,
    onRecoverableError,
    transitionCallbacks,
  );
}
```

```typescript
export function createFiberRoot(
  containerInfo: Container,
  tag: RootTag,
  hydrate: boolean,
  initialChildren: ReactNodeList,
  hydrationCallbacks: null | SuspenseHydrationCallbacks,
  isStrictMode: boolean,
  concurrentUpdatesByDefaultOverride: null | boolean,
  identifierPrefix: string,
  onRecoverableError: null | ((error: mixed) => void),
  transitionCallbacks: null | TransitionTracingCallbacks,
): FiberRoot {
  // $FlowFixMe[invalid-constructor] Flow no longer supports calling new on functions
  const root: FiberRoot = (new FiberRootNode( // ❗❗ FiberRootNode
    containerInfo,
    tag,
    hydrate,
    identifierPrefix,
    onRecoverableError,
  ): any);
  // Cyclic construction. This cheats the type system right now because
  // stateNode is any.
  const uninitializedFiber = createHostRootFiber(
    tag,
    isStrictMode,
    concurrentUpdatesByDefaultOverride,
  );
  root.current = uninitializedFiber;
  // ❗❗ HostRoot의 FiberNode가 생성되고 React 루트의 current로 할당됩니다.
  uninitializedFiber.stateNode = root;
  ...
  initializeUpdateQueue(uninitializedFiber);
  return root;
}
```

`root.render()` 는 HostRoot에서 업데이트를 예약합니다. 엘리먼트의 인수는 업데이트 페이로드에 저장됩니다.

```typescript
function ReactDOMRoot(internalRoot: FiberRoot) {
  this._internalRoot = internalRoot;
}
ReactDOMHydrationRoot.prototype.render = ReactDOMRoot.prototype.render = // ❗❗ render 
function (children: ReactNodeList): void {
  const root = this._internalRoot;
  if (root === null) {
    throw new Error('Cannot update an unmounted root.');
  }
  updateContainer(children, root, null, null); // ❗❗ updateContainer
};
```

```typescript
export function updateContainer(
  element: ReactNodeList,
  container: OpaqueRoot,
  parentComponent: ?React$Component<any, any>,
  callback: ?Function,
): Lane {
  const current = container.current;
  const lane = requestUpdateLane(current);
  if (enableSchedulingProfiler) {
    markRenderScheduled(lane);
  }
  const context = getContextForSubtree(parentComponent);
  if (container.context === null) {
    container.context = context;
  } else {
    container.pendingContext = context;
  }
  const update = createUpdate(lane);
  // Caution: React DevTools currently depends on this property
  // being called "element".
  update.payload = {element}; // ❗❗ render()의 인수는 update payload에 저장됩니다.
  // ❗❗ 그러면 업데이트가 대기열에 추가됩니다. 이 작업이 어떻게 수행되는지는 자세히 설명하지 않겠습니다.
  // ❗❗ 업데이트가 처리되기를 기다리고 있다는 점만 기억하세요.
  const root = enqueueUpdate(current, update, lane);
  if (root !== null) {
    scheduleUpdateOnFiber(root, current, lane);
    entangleTransitions(root, current, lane);
  }
  return lane;
}
```

## 3\. 최초 마운트: Render 단계

### 3.1 `performConcurrentWorkRoot()`

[React Internals 개요](https://ted-projects.com/react-internals-deep-dive-1)에서 언급했듯[,](https://jser.dev/2023-07-11-overall-of-react-internals)`performConcurrentWorkRoot()` 는 최초 마운트 및 리-렌더링 모두의 렌더링 시작 진입점 입니다.

한 가지 명심해야 할 점은 `concurrent` 라는 이름이 붙어 있더라도 내부적으로는 필요할 때 `sync` 모드로 돌아간다는 것입니다. 최초 마운트는 DefaultLane이 blocking lane이기 때문에 발생하는 케이스들 중 하나입니다.(💬 최초 마운트도 동기화 모드다)

```typescript
function performConcurrentWorkOnRoot(root, didTimeout) {
  ...
  // Determine the next lanes to work on, using the fields stored
  // on the root.
  let lanes = getNextLanes(
    root,
    root === workInProgressRoot ? workInProgressRootRenderLanes : NoLanes,
  );
  ...
  // We disable time-slicing in some cases: if the work has been CPU-bound
  // for too long ("expired" work, to prevent starvation), or we're in
  // sync-updates-by-default mode.
  // TODO: We only check `didTimeout` defensively, to account for a Scheduler
  // bug we're still investigating. Once the bug in Scheduler is fixed,
  // we can remove this, since we track expiration ourselves.
  const shouldTimeSlice =
    !includesBlockingLane(root, lanes) &&
    !includesExpiredLane(root, lanes) &&
    (disableSchedulerTimeoutInWorkLoop || !didTimeout);
  let exitStatus = shouldTimeSlice
    ? renderRootConcurrent(root, lanes) // ❗❗ renderRootConcurrent
    : renderRootSync(root, lanes); // ❗❗ renderRootSync
  ...
}
```

```typescript
// ❗❗ Blocking의 의미는 중요하므로 방해되서 안된다는 의미입니다.
export function includesBlockingLane(root: FiberRoot, lanes: Lanes) {
  const SyncDefaultLanes =
    InputContinuousHydrationLane |
    InputContinuousLane |
    DefaultHydrationLane |
    DefaultLane; // ❗❗ 기본 차선이 Blocking lane입니다.

  return (lanes & SyncDefaultLanes) !== NoLanes;
}
```

> ℹ️ lane에 대한 자세한 내용은 [React의 lane이란 무엇인가를](https://jser.dev/react/2022/03/26/lanes-in-react/) 참조하십시오.

위의 코드를 보면 최초 마운트의 경우 concurrent 모드가 실제로 사용되지 않는다는 것을 알 수 있습니다. 이는 최초 마운트의 경우 가능한 한 빨리 UI를 고생시켜야 하며 지연(defer)시키는 것은 도움이 되지 않는다는 것을 의미합니다.

### 3.2 `renderRootSync()`

`renderRootSync()`는 내부적으로 단지 while 루프일 뿐입니다.

```typescript
function renderRootSync(root: FiberRoot, lanes: Lanes) {
  const prevExecutionContext = executionContext;
  executionContext |= RenderContext;
  const prevDispatcher = pushDispatcher();
  // If the root or lanes have changed, throw out the existing stack
  // and prepare a fresh one. Otherwise we'll continue where we left off.
  if (workInProgressRoot !== root || workInProgressRootRenderLanes !== lanes) {
    if (enableUpdaterTracking) {
      if (isDevToolsPresent) {
        const memoizedUpdaters = root.memoizedUpdaters;
        if (memoizedUpdaters.size > 0) {
          restorePendingUpdaters(root, workInProgressRootRenderLanes);
          memoizedUpdaters.clear();
        }
        // At this point, move Fibers that scheduled the upcoming work from the Map to the Set.
        // If we bailout on this work, we'll move them back (like above).
        // It's important to move them now in case the work spawns more work at the same priority with different updaters.
        // That way we can keep the current update and future updates separate.
        movePendingFibersToMemoized(root, lanes);
      }
    }
    workInProgressTransitions = getTransitionsForLanes(root, lanes);
    prepareFreshStack(root, lanes);
  }
  do {
    try {
      workLoopSync();
      break;
    } catch (thrownValue) {
      handleError(root, thrownValue);
    }
  } while (true); // ❗❗ 여기서 루프를 돌립니다.
  resetContextDependencies();
  executionContext = prevExecutionContext;
  popDispatcher(prevDispatcher);
  // Set this to null to indicate there's no in-progress render.
  workInProgressRoot = null;
  workInProgressRootRenderLanes = NoLanes;
  return workInProgressRootExitStatus;
}
// The work loop is an extremely hot path. Tell Closure not to inline it.
/** @noinline */
function workLoopSync() {
  // Already timed out, so perform work without checking if we need to yield.
  while (workInProgress !== null) {
    // ❗❗ 이 while 루프는 workInProgress 존재하면
    // ❗❗ 계속 performUnitOfWork()를 수행한다는 의미입니다.
    performUnitOfWork(workInProgress);
    // ❗❗ performUnitOfWork라는 이름과 같이, Fiber Node 유닛 하나에서 동작합니다.
  }
}
```

여기서 `workInProgress`가 무엇을 의미하는지 설명할 필요가 있습니다. React 코드 베이스에서 `current`와 `workInProgress`의 접두사는 어디에나 있습니다. React는 내부적으로 현재 상태를 표현하기 위해 파이버 트리를 사용하기 때문에 업데이트가 있을 때마다 새로운 트리를 생성하고 이전 트리와 비교해야 합니다. **따라서**`current`**는 UI에 표시되는 현재 버전을 의미하고**, `workInProgress`**는 빌드 중이며 다음**`current`**버전으로 사용될 버전을 의미합니다.**

### 3.3 `performUnitOfWork()`

여기에서 React가 단일 파이버 노드에서 작동하여 수행해야 할 작업이 있는지 확인합니다.

이 섹션을 더 쉽게 이해하려면 먼저 제 에피소드 - [15편 - React가 내부적으로 파이버 트리를 횡단하는 방법](https://jser.dev/react/2022/01/16/fiber-traversal-in-react/)부터 확인하시기 바랍니다.

```typescript
function performUnitOfWork(unitOfWork: Fiber): void {
  // The current, flushed, state of this fiber is the alternate. Ideally
  // nothing should rely on this, but relying on it here means that we don't
  // need an additional field on the work in progress.
  const current = unitOfWork.alternate;
  let next;
  if (enableProfilerTimer && (unitOfWork.mode & ProfileMode) !== NoMode) {
    startProfilerTimer(unitOfWork);
    next = beginWork(current, unitOfWork, subtreeRenderLanes); // ❗❗ beginWork
    stopProfilerTimerIfRunningAndRecordDelta(unitOfWork, true);
  } else {
    next = beginWork(current, unitOfWork, subtreeRenderLanes); // ❗❗ beginWork
  }
  resetCurrentDebugFiberInDEV();
  unitOfWork.memoizedProps = unitOfWork.pendingProps;
  if (next === null) {
    // If this doesn't spawn new work, complete the current work.
    completeUnitOfWork(unitOfWork);
  } else {
    workInProgress = next;
    // ❗❗ 앞서 언급했듯이 workLoopSync( )는 단지 동안 루프일 뿐입니다.
    // ❗❗ workInProgress에서 completeUnitOfWork( )를 계속 실행합니다.
    // ❗❗ 따라서 여기서 작업 workInProgress를 할당한다는 것은 다음 작업할 파이버 노드를 설정하는 것을 의미합니다.
  }
  ReactCurrentOwner.current = null;
}
```

`beginWork()` 가 실제 렌더링이 이루어지는 곳입니다.

```typescript
function beginWork(
  current: Fiber | null,
  workInProgress: Fiber,
  renderLanes: Lanes,
): Fiber | null {
  if (current !== null) {
    // ❗❗ 현재가 null이 아닌 경우, 즉 최초 마운트가 아님을 의미합니다.
    ...
  } else {
    didReceiveUpdate = false
    // ❗❗ 그렇지 않으면 초기 마운트이므로 당연히 업데이트가 없습니다.
    ...
  }
  switch (workInProgress.tag) {
    // ❗❗ 다양한 유형의 요소(elements)를 다르게 처리합니다.
    case IndeterminateComponent: {
      // ❗❗ IndeterminateComponent는 아직 인스턴스화 되지 않은 클래스 컴포넌트 또는 함수 컴포넌트
      // ❗❗ 일단 렌더링되면 올바른 태그로 결정됩니다. 곧 다시 설명하겠습니다.
      return mountIndeterminateComponent(
        current,
        workInProgress,
        workInProgress.type,
        renderLanes,
      );
    }
    case FunctionComponent: {
      // ❗❗ 우리가 작성하는 사용자 정의 함수 컴포넌트
      const Component = workInProgress.type;
      const unresolvedProps = workInProgress.pendingProps;
      const resolvedProps =
        workInProgress.elementType === Component
          ? unresolvedProps
          : resolveDefaultProps(Component, unresolvedProps);
      return updateFunctionComponent(
        current,
        workInProgress,
        Component,
        resolvedProps,
        renderLanes,
      );
    }
    case HostRoot:
      // ❗❗ 이것은 HostRoot 아래의 FiberRootNode 입니다.
      return updateHostRoot(current, workInProgress, renderLanes);
    case HostComponent:
      // ❗❗ p, div 등과 같은 내재적 HTML 태그입니다.
      return updateHostComponent(current, workInProgress, renderLanes);
    case HostText:
      // ❗❗ 이것은 HTML 텍스트 노드입니다.
      return updateHostText(current, workInProgress);
    case SuspenseComponent:
      ... // ❗❗ 더 많은 유형들이 있습니다.
  }
}
```

이제 렌더링 단계를 살펴볼 차례입니다.

### 3.4 `prepareFreshStack()`

`renderRootSync()`안에는, `prepareFreshStack()`라는 중요한 호출이 있습니다.

```typescript
function prepareFreshStack(root: FiberRoot, lanes: Lanes): Fiber {
  root.finishedWork = null;
  root.finishedLanes = NoLanes;
  ...
  workInProgressRoot = root;
  // ❗❗ 여기서 `root.current`의 `current`는 HostRoot의 FiberNode입니다.
  const rootWorkInProgress = createWorkInProgress(root.current, null);
  workInProgress = rootWorkInProgress;
  workInProgressRootRenderLanes = subtreeRenderLanes = workInProgressRootIncludedLanes = lanes;
  workInProgressRootExitStatus = RootInProgress;
  workInProgressRootFatalError = null;
  workInProgressRootSkippedLanes = NoLanes;
  workInProgressRootInterleavedUpdatedLanes = NoLanes;
  workInProgressRootRenderPhaseUpdatedLanes = NoLanes;
  workInProgressRootPingedLanes = NoLanes;
  workInProgressRootConcurrentErrors = null;
  workInProgressRootRecoverableErrors = null;
  finishQueueingConcurrentUpdates();
  return rootWorkInProgress;
}
```

따라서 새 렌더링이 시작될 때마다 현재 HostRoot에서 새 `workInProgress`가 생성됩니다. 이것은 새 파이버 트리의 루트로 작동합니다.

따라서 `beginWork()` 내부의 가지들에 대해 먼저 `HostRoot`로 이동하고 `updateHostRoot()` 는 다음 단계입니다.

### 3.5 `updateHostRoot()`

```typescript
function updateHostRoot(
  current: null | Fiber,
  workInProgress: Fiber,
  renderLanes: Lanes,
) {
  pushHostRootContext(workInProgress);
  const nextProps = workInProgress.pendingProps;
  const prevState = workInProgress.memoizedState;
  const prevChildren = prevState.element;
  cloneUpdateQueue(current, workInProgress);
  processUpdateQueue(workInProgress, nextProps, null, renderLanes);
  // ↖ ❗❗ 이 호출은 이 글의 시작 부분에 언급된 업데이트를 처리합니다.
  // ❗❗ 예약된 업데이트가 처리된다는 점만 기억하세요.
  // ❗❗ 페이로드가 추출되면 요소는 memoizedState로 할당됩니다.
  const nextState: RootState = workInProgress.memoizedState;
  const root: FiberRoot = workInProgress.stateNode;
  pushRootTransition(workInProgress, root, renderLanes);
  if (enableTransitionTracing) {
    pushRootMarkerInstance(workInProgress);
  }
  // Caution: React DevTools currently depends on this property
  // being called "element".
  const nextChildren = nextState.element;
  // ❗❗ ↗ ReactDOMRoot.render()의 인수를 얻을 수 있습니다!
  if (supportsHydration && prevState.isDehydrated) {
    ...
  } else {
    // Root is not dehydrated. Either this is a client-only root, or it
    // already hydrated.
    resetHydrationState();
    if (nextChildren === prevChildren) {
      return bailoutOnAlreadyFinishedWork(current, workInProgress, renderLanes);
    }
    reconcileChildren(current, workInProgress, nextChildren, renderLanes);
    // ❗❗ ↖ 여기서 현재 및 작업 진행 중 모두 하위 항목이 없습니다.
    // ❗❗ 그리고 nextChildren 은 <App/> 입니다.
  }
  return workInProgress.child;
  // ❗❗                 ↗  reconcileChildren 실행 후, workInProgress에 대한 새 하위 항목이 생성됩니다. 
  // ❗❗ 여기서 반환한다는 것은 다음 처리는 workLoopSync()에 맡긴다는 것입니다.
}
```

### 3.6 `reconcileChildren()`

이것은 React internals에서 매우 중요한 임포트 함수입니다. 이름을 대략 `reconcile`에서 `diff`로 바꿔 생각할 수 있습니다. 이 함수는 새 자식과 이전 자식을 비교하고 올바른`child`를 `workInProgress`에 설정합니다.

```typescript
export function reconcileChildren(
  current: Fiber | null,
  workInProgress: Fiber,
  nextChildren: any,
  renderLanes: Lanes,
) {
  if (current === null) {
    // ❗❗ ↖ current가 없는 것은, 최초 마운트라는 의미입니다.
    // If this is a fresh new component that hasn't been rendered yet, we
    // won't update its child set by applying minimal side-effects. Instead,
    // we will add them all to the child before it gets rendered. That means
    // we can optimize this reconciliation pass by not tracking side-effects.
    workInProgress.child = mountChildFibers( // ❗❗ mountChildFibers
      workInProgress,
      null,
      nextChildren,
      renderLanes,
    );
  } else {
    // ❗❗ ↖ 만약 current가 있으면, 리-렌더라는 의미입니다. 그러므로 reconcile 합니다. 
    // If the current child is the same as the work in progress, it means that
    // we haven't yet started any work on these children. Therefore, we use
    // the clone algorithm to create a copy of all the current children.
    // If we had any progressed work already, that is invalid at this point so
    // let's throw it out.
    workInProgress.child = reconcileChildFibers( // ❗❗ reconcileChildFibers
      workInProgress,
      current.child,
      nextChildren,
      renderLanes,
    );
  }
}
```

위에서 언급했듯이 FiberRootNode는 항상 `current`를 가지고 있으므로 두 번째 브랜치인 `reconcideChildFibers`로 이동합니다. 그러나 이것은 최초 마운트이므로, 이것의 자식인 `current.child`는 null입니다.

또한 `workInProgress`는 구성중 이며 아직 `child`가 없으므로 우리가 `workInProgress`에 `child`를 설정하고 있음을 알 수 있습니다.

### 3.7 `reconcileChildFibers()` vs `mountChildFibers()`

`reconcile`의 목표는 이미 가지고 있는 것을 재사용하는 것이며, `mount`는 '모든 것을 새로 고치는 `reconcile`'의 특별한 원시(primitive) 버전으로 취급할 수 있습니다.

사실 코드에서 이 둘은 크게 다르지 않으며, 동일한 클로저이지만 `shouldTrackSideEffects` 플래그가 약간 다릅니다.

```typescript
export const reconcileChildFibers: ChildReconciler =
  createChildReconciler(true);
export const mountChildFibers: ChildReconciler = createChildReconciler(false);
function createChildReconciler(
  shouldTrackSideEffects: boolean,
  // ❗❗↖ 이 플래그는 삽입 등의 동작들을 추적해야 하는지 여부를 제어합니다.
): ChildReconciler {
  ...
  function reconcileChildFibers(
    returnFiber: Fiber,
    currentFirstChild: Fiber | null,
    newChild: any,
    lanes: Lanes,
  ): Fiber | null {
    // This indirection only exists so we can reset `thenableState` at the end.
    // It should get inlined by Closure.
    thenableIndexCounter = 0;
    const firstChildFiber = reconcileChildFibersImpl(
      returnFiber,
      currentFirstChild,
      newChild,
      lanes,
    );
    thenableState = null;
    // Don't bother to reset `thenableIndexCounter` to 0 because it always gets
    // set at the beginning.
    return firstChildFiber;
    // ❗❗ ↗ 자식들을 조정한 후 첫번째 자식 fiber가 반환됩니다
    // ❗❗ 그리고 workInProgress의 자식으로 설정됩니다.
  }
  return reconcileChildFibers;
}
```

전체 Fiber 트리를 구성해야 한다면, 조정 후 모든 노드가 "삽입 필요"로 표시되어야 한다고 상상해 보세요. 하지만 그럴 필요 없이, 루트만 삽입하면 끝입니다! 따라서 이 `mountChildFibers`는 사실 내부적으로 개선한 것으로, 보다 명확하게 하기 위한 것입니다.

```typescript
function reconcileChildFibersImpl(
  returnFiber: Fiber,
  currentFirstChild: Fiber | null,
  newChild: any,
  lanes: Lanes,
): Fiber | null {
  // This function is not recursive.
  // If the top level item is an array, we treat it as a set of children,
  // not as a fragment. Nested arrays on the other hand will be treated as
  // fragment nodes. Recursion happens at the normal flow.
  // Handle top level unkeyed fragments as if they were arrays.
  // This leads to an ambiguity between <>{[...]}</> and <>...</>.
  // We treat the ambiguous cases above the same.
  // TODO: Let's use recursion like we do for Usable nodes?
  const isUnkeyedTopLevelFragment =
    typeof newChild === 'object' &&
    newChild !== null &&
    newChild.type === REACT_FRAGMENT_TYPE &&
    newChild.key === null;
  if (isUnkeyedTopLevelFragment) {
    newChild = newChild.props.children;
  }
  // Handle object types
  if (typeof newChild === 'object' && newChild !== null) {
    switch (newChild.$$typeof) {
      // ❗❗            ↗ 이 $$typeof React Element의 타입 입니다.
      case REACT_ELEMENT_TYPE:
        return placeSingleChild(
          reconcileSingleElement(
          // ❗❗ ↗ 우리는 이 2개의 함수(placeSingleChild, reconcileSingleElement) 들에 대해 자세히 알아보겠습니다
            returnFiber,
            currentFirstChild,
            newChild,
            lanes,
          ),
        );
        // ❗❗ <App/> 과 같이, 자식이 React Element 인 경우
      case REACT_PORTAL_TYPE:
        return placeSingleChild(
          reconcileSinglePortal(
            returnFiber,
            currentFirstChild,
            newChild,
            lanes,
          ),
        );
      case REACT_LAZY_TYPE:
        const payload = newChild._payload;
        const init = newChild._init;
        // TODO: This function is supposed to be non-recursive.
        return reconcileChildFibers(
          returnFiber,
          currentFirstChild,
          init(payload),
          lanes,
        );
    }
    if (isArray(newChild)) {
      // ❗❗ ↖ 만약 자식이 배열 이라면 
      return reconcileChildrenArray(
        returnFiber,
        currentFirstChild,
        newChild,
        lanes,
      );
    }
    if (getIteratorFn(newChild)) {
      return reconcileChildrenIterator(
        returnFiber,
        currentFirstChild,
        newChild,
        lanes,
      );
    }
    if (typeof newChild.then === 'function') {
      const thenable: Thenable<any> = (newChild: any);
      return reconcileChildFibersImpl(
        returnFiber,
        currentFirstChild,
        unwrapThenable(thenable),
        lanes,
      );
    }
    if (
      newChild.$$typeof === REACT_CONTEXT_TYPE ||
      newChild.$$typeof === REACT_SERVER_CONTEXT_TYPE
    ) {
      ...
    }
    throwOnInvalidObjectType(returnFiber, newChild);
  }
  if (
    (typeof newChild === 'string' && newChild !== '') ||
    typeof newChild === 'number'
  ) {
    return placeSingleChild(
      reconcileSingleTextNode(
      // ❗❗ ↖ 가장 원시적인 경우 Text Node를 업데이트 처리합니다.
        returnFiber,
        currentFirstChild,
        '' + newChild,
        lanes,
      ),
    );
  }

  // Remaining cases are all treated as empty.
  return deleteRemainingChildren(returnFiber, currentFirstChild);
```

두 단계가 있음을 알 수 있습니다. `reconcileXXX()` 를 사용하여 차이(diff)를 조정하고 `placeSingleChild()`를 사용하여 DOM에 fiber가 삽입되어야 함을 표시합니다.

### 3.8 `reconcileSingleElement()`

```typescript
function reconcileSingleElement(
  returnFiber: Fiber,
  currentFirstChild: Fiber | null,
  element: ReactElement,
  lanes: Lanes,
): Fiber {
  const key = element.key;
  let child = currentFirstChild;
  while (child !== null) {
    // ❗❗ ↗ 이미 자식이 있는 경우 업데이트를 처리합니다.
    // ❗❗ 그러나 최초 마운트에서는 그렇지 않으므로 지금은 무시하겠습니다.
    ...
  }
  if (element.type === REACT_FRAGMENT_TYPE) {
    ...
  } else {
    const created = createFiberFromElement(element, returnFiber.mode, lanes);
    // ❗❗          ↗ 이전 버전이 없으므로 요소에서 새 파이버를 생성하기만 하면 됩니다.
    created.ref = coerceRef(returnFiber, currentFirstChild, element);
    created.return = returnFiber;
    return created;
  }
}
```

최초 마운트를 위한 `reconcileSingleElement()`는 보시다시피 매우 간단합니다. 새로 생성된 Fiber Node가 `workInProgress` 의 `child` 노드가 되는 것을 확인할 수 있습니다.

한 가지 주목할 점은 사용자 정의 컴포넌트에서 Fiber Node를 생성할 때 해당 태그가 아직 `FunctionComponent`가 아닌 `IndeterminateComponent`라는 점입니다.

```typescript
export function createFiberFromElement(
  element: ReactElement,
  mode: TypeOfMode,
  lanes: Lanes,
): Fiber {
  let owner = null;
  const type = element.type;
  const key = element.key;
  const pendingProps = element.props;
  const fiber = createFiberFromTypeAndProps(
    type,
    key,
    pendingProps,
    owner,
    mode,
    lanes,
  );
  return fiber;
}
export function createFiberFromTypeAndProps(
  type: any, // React$ElementType
  key: null | string,
  pendingProps: any,
  owner: null | Fiber,
  mode: TypeOfMode,
  lanes: Lanes,
): Fiber {
  let fiberTag = IndeterminateComponent; // ❗❗ IndeterminateComponent
  // The resolved type is set if we know what the final type will be. I.e. it's not lazy.
  let resolvedType = type;
  ...
  const fiber = createFiber(fiberTag, pendingProps, key, mode);
  fiber.elementType = type;
  fiber.type = resolvedType;
  fiber.lanes = lanes;
  return fiber;
}
```

### 3.9 `placeSingleChild()`

`reconcideSingleElement()`는 Fiber Node 조정만 수행하며, `placeSingleChild()`는 자식 Fiber Node가 DOM에 삽입되도록 표시하는 곳입니다.

```typescript
function placeSingleChild(newFiber: Fiber): Fiber {
  // This is simpler for the single child case. We only need to do a
  // placement for inserting new children.
  if (shouldTrackSideEffects && newFiber.alternate === null) {
    // ❗❗ ↗  예, 이 플래그는 여기에서 사용됩니다(다른 곳에서도 마찬가지입니다).
    newFiber.flags |= Placement | PlacementDEV;
    // ❗❗     ↗ Placement 는 DOM sub-tree를 삽입해야 함을 의미합니다.
  }
  return newFiber;
}
```

이 작업은 `child`에서 수행되므로 최초 마운트에서 `HostRoot`의 자식은 `Placement`로 표시됩니다. 데모 코드에서는 `<App/>`입니다.

### 3.10 `mountIndeterminateComponent()`

`beginWork()` 에서 다음으로 살펴볼 분기는 `IndeterminateComponent`입니다. `<App/>` 이 HostRoot 아래에 있고, 앞서 언급했듯이 사용자 정의 컴포넌트는 처음에 `IndeterminateComponent` 로 표시되므로, `<App/>` 이 처음 조정될 때 여기로 올 것입니다.

```typescript
function mountIndeterminateComponent(
  _current: null | Fiber,
  workInProgress: Fiber,
  Component: $FlowFixMe,
  renderLanes: Lanes,
) {
  resetSuspendedCurrentOnMountInLegacyMode(_current, workInProgress);
  const props = workInProgress.pendingProps;
  let context;
  if (!disableLegacyContext) {
    const unmaskedContext = getUnmaskedContext(
      workInProgress,
      Component,
      false,
    );
    context = getMaskedContext(workInProgress, unmaskedContext);
  }
  prepareToReadContext(workInProgress, renderLanes);
  let value;
  let hasId;
  value = renderWithHooks(
    // ❗❗ ↗ 함수 컴포넌트를 실행하고 자식 요소를 반환합니다.
    null,
    workInProgress,
    Component,
    props,
    context,
    renderLanes,
  );
  hasId = checkDidRenderIdHook();
  // React DevTools reads this flag.
  workInProgress.flags |= PerformedWork;
  if (
    // Run these checks in production only if the flag is off.
    // Eventually we'll delete this branch altogether.
    !disableModulePatternComponents &&
    typeof value === 'object' &&
    value !== null &&
    typeof value.render === 'function' &&
    value.$$typeof === undefined
  ) {
    // Proceed under the assumption that this is a class instance
    workInProgress.tag = ClassComponent;
    // ❗❗ ↗ 일단 렌더링되면, 더 이상 IndeterminateComponent 이 아닙니다
    ...
  } else {
    // Proceed under the assumption that this is a function component
    workInProgress.tag = FunctionComponent;
    // ❗❗ ↗ 일단 렌더링되면, 더 이상 IndeterminateComponent 이 아닙니다
    if (getIsHydrating() && hasId) {
      pushMaterializedTreeId(workInProgress);
    }
    reconcileChildren(null, workInProgress, value, renderLanes);
    // ❗❗ ↗ 여기서는 현재가 null이므로 마운트 mountChildFibers()가 사용됩니다.
    return workInProgress.child;
  }
}
```

앞서 언급했듯이 `<App/>` 렌더링 시에는 항상 최신 버전이 있는 HostRoot와 달리 `<App/>` 에는 이전 버전이 없고 `placeSingleChild()`가 삽입 플래그를 무시하기 때문에 `mountChildFibers()`가 사용됩니다.

`App()`은 `<div/>`를 반환하며, 나중에 `beginWork()`의 `HostComponent` 브랜치에서 처리합니다.

## 3.11 `updateHostComponent()`

```typescript
function updateHostComponent(
  current: Fiber | null,
  workInProgress: Fiber,
  renderLanes: Lanes,
) {
  pushHostContext(workInProgress);
  if (current === null) {
    tryToClaimNextHydratableInstance(workInProgress);
    // ❗❗ ↗ EP27 - React에서 내부적으로 기본 hydration이 작동하는 방식을 참조 하세요. 하단 참조
  }
  const type = workInProgress.type;
  const nextProps = workInProgress.pendingProps;
  // ❗❗                           ↗ pendingProps 는 <div />의 자식들(<p>)을 보유합니다.
  const prevProps = current !== null ? current.memoizedProps : null;
  let nextChildren = nextProps.children;
  const isDirectTextChild = shouldSetTextContent(type, nextProps);
  // ❗❗ ↗ <a />와 같이 자식이 정적 텍스트인 경우 개선이 적용된 기능입니다.
  if (isDirectTextChild) {
    // We special case a direct text child of a host node. This is a common
    // case. We won't handle it as a reified child. We will instead handle
    // this in the host environment that also has access to this prop. That
    // avoids allocating another HostText fiber and traversing it.
    nextChildren = null;
  } else if (prevProps !== null && shouldSetTextContent(type, prevProps)) {
    // If we're switching from a direct text child to a normal child, or to
    // empty, we need to schedule the text content to be reset.
    workInProgress.flags |= ContentReset;
  }
  ...
  markRef(current, workInProgress);
  reconcileChildren(current, workInProgress, nextChildren, renderLanes);
  return workInProgress.child;
}
```

> [EP27 - React에서 내부적으로 기본 hydration이 작동하는 방식](https://jser.dev/react/2023/03/17/how-does-hydration-work-in-react/)

위의 프로세스는 `<p/>` 에서 반복되지만, 여기의 `nextChildren`이 배열이므로 `reconcileChildrenArray()`가 `reconcileChildFibers()` 내부에서 시작된다는 점을 제외하면 동일합니다.

`reconcileChildrenArray()` 는 `key`가 존재하기 때문에 조금 더 복잡합니다. 자세한 내용은 제 다른 블로그 포스트 - [EP19 - 'key'는 내부적으로 어떻게 작동할까요? React의 리스트 diffing](https://jser.dev/react/2022/02/08/the-diffing-algorithm-for-array-in-react/).

`key` 처리 외에는 기본적으로 첫 번째 자식 Fiber를 반환하고 계속되며, React는 트리 구조를 linked list로 평평하게 만들기 때문에 형제 자매(siblings)는 나중에 처리됩니다. 자세한 내용은 [EP15 - React가 내부적으로 fiber 트리를 순회하는 방법](https://jser.dev/react/2022/01/16/fiber-traversal-in-react/) 문서를 참고하세요.

`<Link/>의` 경우, `<App/>`으로 이 과정을 반복합니다.

`<a/>` 및 `<button/>`의 경우 해당 텍스트를 더 자세히 살펴보겠습니다.

하지만 조금 다른 점이 있는데, `<a/>`에는 정적 텍스트가 자식인 반면 `<button/>` 에는 JSX 표현식 `{count}` 가 있습니다. 그래서 위 코드에서 `<a/>` 에는`nextChildren`이 null로 있지만 `<button/>`에는 자식으로 계속 이어집니다.

### 3.12 `updateHostTest()`

`<button/>` 의 경우 그 자식은 `["click me - ", "0"]` 배열이고, `updateHostText()` 는 `beginWork()`에서 두 가지 모두에 대한 분기입니다.

```typescript
function updateHostText(current, workInProgress) {
  if (current === null) {
    tryToClaimNextHydratableInstance(workInProgress);
  } // Nothing to do here. This is terminal. We'll do the completion step
  // immediately after.
  return null;
}
```

하지만 hydration 처리 외에는 아무것도 하지 않습니다. `<a/>` 및 `<button/>` 의 텍스트가 처리되는 방식은 "Commit" 단계에 있습니다.

### 3.13 DOM 노드는 `completeWork()`내에서, 즉 화면 외부에서 생성됩니다.

[EP15 - React가 내부적으로 Fiber 트리를 순회하는 방법](https://jser.dev/react/2022/01/16/fiber-traversal-in-react/)에서 언급했듯이, 파이버에서 `completeWork()`가 호출된 후 그 형제(siblings)가 `beginWork()`로 시도되기 전에 호출됩니다.

Fiber Node에는 한 가지 중요한 프로퍼티인 `stateNode` 가 있는데, 이는 내재적 HTML 태그의 경우 실제 DOM 노드를 참조합니다. 그리고 DOM 노드의 실제 생성은 `completeWork()`에서 이루어집니다.

```typescript
function completeWork(
  current: Fiber | null,
  workInProgress: Fiber,
  renderLanes: Lanes,
): Fiber | null {
  const newProps = workInProgress.pendingProps;
  // Note: This intentionally doesn't check if we're hydrating because comparing
  // to the current tree provider fiber is just as fast and less error-prone.
  // Ideally we would have a special version of the work loop only
  // for hydration.
  popTreeContext(workInProgress);
  switch (workInProgress.tag) {
    case IndeterminateComponent:
    case LazyComponent:
    case SimpleMemoComponent:
    case FunctionComponent:
    case ForwardRef:
    case Fragment:
    case Mode:
    case Profiler:
    case ContextConsumer:
    case MemoComponent:
      bubbleProperties(workInProgress);
      return null;
    case ClassComponent: {
      const Component = workInProgress.type;
      if (isLegacyContextProvider(Component)) {
        popLegacyContext(workInProgress);
      }
      bubbleProperties(workInProgress);
      return null;
    }
    case HostRoot: {
      ...
      return null;
    }
    ...
    case HostComponent: {
    // ❗❗ ↗ HTML 태그들을 위한 것
      popHostContext(workInProgress);
      const type = workInProgress.type;
      if (current !== null && workInProgress.stateNode != null) {
        // ❗❗ ↗ 만약 current 버전이면, 브랜치를 업데이트 합니다.
        updateHostComponent(
          current,
          workInProgress,
          type,
          newProps,
          renderLanes,
        );
        if (current.ref !== workInProgress.ref) {
          markRef(workInProgress);
        }
      } else {
        // ❗❗ ↖ 하지만 우린 아직 current 버전이 없습니다, 그래서 우린 마운트 브랜치로 갑니다.
        ...
        if (wasHydrated) {
          ...
        } else {
          const rootContainerInstance = getRootHostContainer();
          const instance = createInstance(
            // ❗❗ ↖ 실제 DOM 노드 
            type,
            newProps,
            rootContainerInstance,
            currentHostContext,
            workInProgress,
          );
          appendAllChildren(instance, workInProgress, false, false);
          // ❗❗ ↖ 이게 중요합니다. DOM 노드가 만들어지면
          // ❗❗ 하위 트리에 있는 직접 연결된 모든 DOM 노드의 부모가 되어야 합니다.
          workInProgress.stateNode = instance;
         if (
            finalizeInitialChildren(
            // ❗↗ 이 내용은 곧 다룰겁니다!
              instance,
              type,
              newProps,
              rootContainerInstance,
              currentHostContext,
            )
          ) {
            markUpdate(workInProgress);
          }
        }
        if (workInProgress.ref !== null) {
          // If there is a ref on a host node we need to schedule a callback
          markRef(workInProgress);
        }
      }
      bubbleProperties(workInProgress);
      ...
      return null;
    }
    case HostText: {
      const newText = newProps;
      if (current && workInProgress.stateNode != null) {
        const oldText = current.memoizedProps;
        // If we have an alternate, that means this is an update and we need
        // to schedule a side-effect to do the updates.
        updateHostText(current, workInProgress, oldText, newText);
      } else {
        ...
        const rootContainerInstance = getRootHostContainer();
        const currentHostContext = getHostContext();
        const wasHydrated = popHydrationState(workInProgress);
        if (wasHydrated) {
          if (prepareToHydrateHostTextInstance(workInProgress)) {
            markUpdate(workInProgress);
          }
        } else {
          workInProgress.stateNode = createTextInstance(
            newText,
            rootContainerInstance,
            currentHostContext,
            workInProgress,
          );
        }
      }
      bubbleProperties(workInProgress);
      return null;
    }
    ...
  }
}
```

한 가지 질문은 `<a/>` 와 `<button/>`이 텍스트 노드를 다르게 처리한다는 점이었습니다.

`<button/>`의 경우 위의 `HostText` 브랜치로 이동하고 `createTextInstance()`가 새 텍스트 노드를 생성하는 것을 볼 수 있습니다. 하지만 `<a/`의 경우는 조금 다릅니다. 더 자세히 살펴봅시다.

위 코드에서 `HostComponent`에 `finalizeInitialChildren()` 함수가 있음을 알 수 있습니다.

```typescript
export function finalizeInitialChildren(
  domElement: Instance,
  type: string,
  props: Props,
  hostContext: HostContext,
): boolean {
  setInitialProperties(domElement, type, props); // ❗❗ setInitialProperties
  ...
}
export function setInitialProperties(
  domElement: Element,
  tag: string,
  rawProps: Object,
  rootContainerElement: Element | Document | DocumentFragment,
): void {
  ...
  setInitialDOMProperties( // ❗❗ setInitialDOMProperties
    tag,
    domElement,
    rootContainerElement,
    props,
    isCustomComponentTag,
  );
  ...
}
function setInitialDOMProperties(
  tag: string,
  domElement: Element,
  rootContainerElement: Element | Document | DocumentFragment,
  nextProps: Object,
  isCustomComponentTag: boolean,
): void {
  for (const propKey in nextProps) {
    if (!nextProps.hasOwnProperty(propKey)) {
      continue;
    }
    const nextProp = nextProps[propKey];
    if (propKey === STYLE) {
      // Relies on `updateStylesByID` not mutating `styleUpdates`.
      setValueForStyles(domElement, nextProp);
    } else if (propKey === DANGEROUSLY_SET_INNER_HTML) {
      const nextHtml = nextProp ? nextProp[HTML] : undefined;
      if (nextHtml != null) {
        setInnerHTML(domElement, nextHtml);
      }
    } else if (propKey === CHILDREN) {
      if (typeof nextProp === 'string') {
        // Avoid setting initial textContent when the text is empty. In IE11 setting
        // textContent on a <textarea> will cause the placeholder to not
        // show within the <textarea> until it has been focused and blurred again.
        // https://github.com/facebook/react/issues/6731#issuecomment-254874553
        const canSetTextContent = tag !== 'textarea' || nextProp !== '';
        if (canSetTextContent) {
          setTextContent(domElement, nextProp);
        }
      } else if (typeof nextProp === 'number') {
        setTextContent(domElement, '' + nextProp);
      }
      // ❗❗ 따라서 문자열(string) 또는 숫자(number)의 자식은 구성 요소의 텍스트 콘텐츠로 취급됩니다.
      // ❗❗ 표현식이 있는 자식은 배열이 되므로 이 분기에 포함되지 않습니다.
    } else if (
      propKey === SUPPRESS_CONTENT_EDITABLE_WARNING ||
      propKey === SUPPRESS_HYDRATION_WARNING
    ) {
      // Noop
    } else if (propKey === AUTOFOCUS) {
      // We polyfill it separately on the client during commit.
      // We could have excluded it in the property list instead of
      // adding a special case here, but then it wouldn't be emitted
      // on server rendering (but we *do* want to emit it in SSR).
    } else if (registrationNameDependencies.hasOwnProperty(propKey)) {
      if (nextProp != null) {
        if (propKey === 'onScroll') {
          listenToNonDelegatedEvent('scroll', domElement);
        }
      }
    } else if (nextProp != null) {
      setValueForProperty(domElement, propKey, nextProp, isCustomComponentTag);
    }
  }
}
```

## 4\. 최초 마운트: Commit 단계

By now:

1. Fiber Tree의 workInProgress 버전이 드디어 완성되었습니다!
    
2. 백업 DOM 노드도 생성 및 구성됩니다!
    
3. 플래그들이 안내가 필요한 파이버들에 설정되어 DOM 조작을 가이드합니다!
    

이제 React가 DOM을 어떻게 조작하는지 실제로 살펴볼 차례입니다.

### 4.1 `commitMutationEffects()`

[React Internals 개요](https://ted-projects.com/react-internals-deep-dive-1)에서 "Commit" 단계에 대해 간략하게 설명한 것을 기억하실 텐데요, 이번에는 DOM 변형을 처리하는 `commitMutationEffects()`에 대해 자세히 알아보겠습니다.

```typescript
export function commitMutationEffects(
  root: FiberRoot,
  finishedWork: Fiber,
  // ❗❗ ↖ 새로 구축된 Fiber Tree를 보유한 HostRoot의 Fiber Node
  committedLanes: Lanes,
) {
  inProgressLanes = committedLanes;
  inProgressRoot = root;
  commitMutationEffectsOnFiber(finishedWork, root, committedLanes); // ❗❗ commitMutationEffectsOnFiber
  inProgressLanes = null;
  inProgressRoot = null;
}
```

```typescript
function commitMutationEffectsOnFiber(
  finishedWork: Fiber,
  root: FiberRoot,
  lanes: Lanes,
) {
  const current = finishedWork.alternate;
  const flags = finishedWork.flags;
  // The effect flag should be checked *after* we refine the type of fiber,
  // because the fiber tag is more specific. An exception is any flag related
  // to reconciliation, because those can be set on all fiber types.
  switch (finishedWork.tag) {
    case FunctionComponent:
    case ForwardRef:
    case MemoComponent:
    case SimpleMemoComponent: {
      recursivelyTraverseMutationEffects(root, finishedWork, lanes);
      // ❗❗ ↖ 이 재귀 호출은 하위 트리가 먼저 처리되도록 합니다.
      commitReconciliationEffects(finishedWork);
      // ❗❗ ↖ ReconciliationEffects 는 삽입 등을 의미합니다.
      ...
      return;
    }
    ...
    case HostComponent: {
      recursivelyTraverseMutationEffects(root, finishedWork, lanes);
      commitReconciliationEffects(finishedWork);
      ...
      return;
    }
    case HostText: {
      recursivelyTraverseMutationEffects(root, finishedWork, lanes);
      commitReconciliationEffects(finishedWork);
      ...
      return;
    }
    case HostRoot: {
      if (enableFloat && supportsResources) {
        prepareToCommitHoistables();
        const previousHoistableRoot = currentHoistableRoot;
        currentHoistableRoot = getHoistableRoot(root.containerInfo);
        recursivelyTraverseMutationEffects(root, finishedWork, lanes);
        currentHoistableRoot = previousHoistableRoot;
        commitReconciliationEffects(finishedWork);
      } else {
        recursivelyTraverseMutationEffects(root, finishedWork, lanes);
        commitReconciliationEffects(finishedWork);
      }
      ...
      return;
    }
    ...
    default: {
      recursivelyTraverseMutationEffects(root, finishedWork, lanes);
      commitReconciliationEffects(finishedWork);
      return;
    }
  }
}
```

```typescript
function recursivelyTraverseMutationEffects(
  root: FiberRoot,
  parentFiber: Fiber,
  lanes: Lanes,
) {
  // Deletions effects can be scheduled on any fiber type. They need to happen
  // before the children effects hae fired.
  const deletions = parentFiber.deletions;
  if (deletions !== null) {
    for (let i = 0; i < deletions.length; i++) {
      const childToDelete = deletions[i];
      try {
        commitDeletionEffects(root, parentFiber, childToDelete);
      } catch (error) {
        captureCommitPhaseError(childToDelete, parentFiber, error);
      }
    }
  }
  // ❗❗ 삭제는 다르게 처리됩니다.

  const prevDebugFiber = getCurrentDebugFiberInDEV();
  if (parentFiber.subtreeFlags & MutationMask) {
    let child = parentFiber.child;
    while (child !== null) {
      setCurrentDebugFiberInDEV(child);
      commitMutationEffectsOnFiber(child, root, lanes); // ❗❗ 재귀!
      child = child.sibling;
    }
  }
  setCurrentDebugFiberInDEV(prevDebugFiber);
}
```

### 4.2 `commitReconciliationEffects()`

`commitReconciliationEffects()는` 삽입, 재정렬 등을 처리합니다.

```typescript
function commitReconciliationEffects(finishedWork: Fiber) {
  // Placement effects (insertions, reorders) can be scheduled on any fiber
  // type. They needs to happen after the children effects have fired, but
  // before the effects on this fiber have fired.
  const flags = finishedWork.flags;
  if (flags & Placement) {
    // ❗❗          ↖ 맞습니다, 이 플래그는 여기서 체크 됩니다!
    try {
      commitPlacement(finishedWork); // ❗❗ commitPlacement
    } catch (error) {
      captureCommitPhaseError(finishedWork, finishedWork.return, error);
    }
    // Clear the "placement" from effect tag so that we know that this is
    // inserted, before any life-cycles like componentDidMount gets called.
    // TODO: findDOMNode doesn't rely on this any more but isMounted does
    // and isMounted is deprecated anyway so we should be able to kill this.
    finishedWork.flags &= ~Placement;
  }
  ...
}
```

따라서 데모에서는 `<App/>의` Fiber 노드가 실제로 커밋됩니다.

### 4.3 `commitPlacement()`

```typescript
function commitPlacement(finishedWork: Fiber): void {
  ...
  // Recursively insert all host nodes into the parent.
  const parentFiber = getHostParentFiber(finishedWork);
  switch (parentFiber.tag) {
     // ❗❗ ↗ 여기에서 부모 fiber의 유형을 확인하고 있습니다.
     // ❗❗ 왜냐하면 삽입은 부모 노드에 의해 완료되기 때문입니다.
    case HostSingleton: {
      if (enableHostSingletons && supportsSingletons) {
        const parent: Instance = parentFiber.stateNode;
        const before = getHostSibling(finishedWork);
        // We only have the top Fiber that was inserted but we need to recurse down its
        // children to find all the terminal nodes.
        insertOrAppendPlacementNode(finishedWork, before, parent);
        break;
      }
      // Fall through
    }
    case HostComponent: {
      // ❗❗ ↗ 최초 마운트에서 이 브랜치는 건드리지 않습니다. 
      const parent: Instance = parentFiber.stateNode;
      if (parentFiber.flags & ContentReset) {
        // Reset the text content of the parent before doing any insertions
        resetTextContent(parent);
        // Clear ContentReset from the effect tag
        parentFiber.flags &= ~ContentReset;
      }
      const before = getHostSibling(finishedWork);
      // We only have the top Fiber that was inserted but we need to recurse down its
      // children to find all the terminal nodes.
      insertOrAppendPlacementNode(finishedWork, before, parent);
      break;
    }
    case HostRoot:
    // ❗❗ 최초 마운트에서 Placement 플래그를 가진 Fiber Node는 <App/> 입니다.
    // ❗❗ 이것의 부모 fiber는 HostRoot 입니다.
    case HostPortal: {
      const parent: Container = parentFiber.stateNode.containerInfo;
             // ❗❗                        ↗ HostRoot의 stateNode는 FiberRootNode를 가리킵니다.
      const before = getHostSibling(finishedWork);
      insertOrAppendPlacementNodeIntoContainer(finishedWork, before, parent);
      break;
    }
    default:
      throw new Error(
        'Invalid host parent fiber. This error is likely caused by a bug ' +
          'in React. Please file an issue.',
      );
  }
}
```

`finishedWork`의 DOM을 부모 컨테이너의 맞는 위치에 삽입하거나 추가하는 것이 아이디어(=관건 or 핵심)입니다.

```typescript
function insertOrAppendPlacementNodeIntoContainer(
  node: Fiber,
  before: ?Instance,
  parent: Container,
): void {
  const {tag} = node;
  const isHost = tag === HostComponent || tag === HostText;
  if (isHost) {
    const stateNode = node.stateNode;
    if (before) {
      insertInContainerBefore(parent, stateNode, before);
    } else {
      appendChildToContainer(parent, stateNode);
    }
    // ❗❗ 만약 DOM 엘리먼트면, 그냥 삽입(insert)합니다.
  } else if (
    tag === HostPortal ||
    (enableHostSingletons && supportsSingletons ? tag === HostSingleton : false)
  ) {
    // If the insertion itself is a portal, then we don't want to traverse
    // down its children. Instead, we'll get insertions from each child in
    // the portal directly.
    // If the insertion is a HostSingleton then it will be placed independently
  } else {
    const child = node.child;
    if (child !== null) {
      insertOrAppendPlacementNodeIntoContainer(child, before, parent);
      let sibling = child.sibling;
      while (sibling !== null) {
        insertOrAppendPlacementNodeIntoContainer(sibling, before, parent);
        sibling = sibling.sibling;
      }
    }
    // ❗❗ non-DOM 엘리먼트면, 재귀적으로 자식들을 처리합니다.
  }
}
```

이것이 최종적으로 DOM이 삽입되는 방식입니다.

## 5\. 요약

DOM이 어떻게 생성되고 컨테이너에 삽입되는지 확인했습니다. 최초 마운트의 경우입니다,

1. Fiber Tree는 조정(reconciliation)하는 동안 느리게(lazily) 생성되며, 백업 DOM 노드는 동시에 생성되고 구성됩니다.
    
2. `HostRoot`의 직계 자식은 `Plcement`로 표시됩니다.
    
3. "Commit" 단계에서는 `Placement`로 Fiber를 찾습니다. 부모가 HostRoot이므로, 해당 DOM 노드가 컨테이너에 삽입됩니다.
    

> 💬 역자주석: JSer의 블로그에 특수하게 만든 그래프가 있으니 보면서 위에서 본 내용을 시각적으로 복기해보시길 권합니다. [https://jser.dev/2023-07-14-initial-mount/#5-summary](https://jser.dev/2023-07-14-initial-mount/#5-summary)

(원본 게시일: 2023-07-14)