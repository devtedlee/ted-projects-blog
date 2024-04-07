---
title: "[번역] React는 어떻게 리-렌더링하나요?"
datePublished: Sun Apr 07 2024 15:07:21 GMT+0000 (Coordinated Universal Time)
cuid: clupnqkfj000208kz0peq5m82
slug: react-internals-deep-dive-3
cover: https://cdn.hashnode.com/res/hashnode/image/upload/v1712502122694/f4b34b27-47bc-441e-b72a-6f97009e5742.jpeg
tags: react-internals

---

> 영문 블로그 글을 번역했습니다. 허가를 받으면 시리즈를 이어갈 예정입니다.  
> 원문링크: [https://jser.dev/2023-07-18-how-react-rerenders](https://jser.dev/2023-07-18-how-react-rerenders/)

---

> ℹ️ [React Internals Deep Dive](https://jser.dev/series/react-source-code-walkthrough.html) 에피소드 3, [유튜브에서 제가 설명하는 것](https://www.youtube.com/watch?v=0GM-1W7i9Tk&list=PLvx8w9g4qv_p-OS-XdbB3Ux_6DMXhAJC3&index=3)을 시청해주세요.
> 
> ⚠ [React@18.2.0](https://github.com/facebook/react/releases/tag/v18.2.0) 기준, 최신 버전에서는 구현이 변경되었을 수 있습니다.

[React가 최초 마운트](https://ted-projects.com/react-internals-deep-dive-2)를 수행하고 전체 DOM을 처음부터 생성하는 방법에 대해 설명했습니다. 최초 마운트 후 리-렌더링할 때 React는 재조정(recouncile) 과정을 통해 가능한 한 DOM을 재사용하려고 합니다. 이 에피소드에서는 아래 데모에서 버튼을 클릭한 후 React가 리렌더링할 때 실제로 어떤 일이 일어나는지 알아보겠습니다.

```typescript
import {useState} from 'react'
  function Link() {
    return <a href="https://jser.dev">jser.dev</a>;
  }

  function Component() {
    const [count, setCount] = useState(0);
    return (
      <div>
      <button onClick={() => setCount((count) => count + 1)}>
        click me - {count} 
      </button> ({count % 2 === 0 ? <span>even</span> : <b>odd</b>})
      </div>
    );
  }
  export default function App() {
    return (
      <div>
        <Link />
        <br />
        <Component />
      </div>
    );
  }
```

* [코드 샌드박스 링크](https://codesandbox.io/s/tt7kwc?file=%2FApp.js&utm_medium=sandpack)
    
* [데모 링크](https://jser.dev/demos/react/overview/re-render.html)

---
- [1. Re-render: Trigger phase](#heading-1-re-render-trigger-phase)
  - [1.1 `lanes` and `childLanes`](#heading-11-lanes-and-childlanes)
- [2. Re-render : Render phase](#heading-2-re-render--render-phase)
  - [2.1 기본 렌더링 로직은 최초 마운트와 동일합니다.](#heading-21)
  - [2.2 React는 새로운 Fiber Node를 생성하기 전에 중복된 Fiber Node를 재사용합니다.](#heading-22-react-fiber-node-fiber-node)
  - [2.3 `beginWork()` 내의 업데이트 브랜치](#heading-23-beginwork)
  - [2.4 `attemptEarlyBailoutIfNoScheduledUpdate()` 내의 Bailout 로직](#heading-24-attemptearlybailoutifnoscheduledupdate-bailout)
  - [2.5 `memoizedProps` vs `pendingProps`](#heading-25-memoizedprops-vs-pendingprops)
  - [2.6 `updateFunctionComponent()` 는 함수 컴포넌트를 리-렌더링하고 자식을 조정합니다.](#heading-26-updatefunctioncomponent)
  - [2.7 \```reconcileSingleElement()```](#heading-27-reconcilesingleelement)
  - [2.8 컴포넌트가 리-렌더링되면 기본적으로 해당 하위 트리가 리-렌더링됩니다.](#heading-28)
  - [2.9 `updateHostComponent()`](#heading-29-updatehostcomponent)
  - [2.10 `reconcileChildrenArray()` 는 필요에 따라 Fiber를 생성하고 삭제합니다.](#heading-210-reconcilechildrenarray-fiber)
  - [2.11 `placeChild()` 및 `deleteChild()` 는 플래그로 Fiber를 표시합니다.](#heading-211-placechild-deletechild-fiber)
  - [2.12 `updateHostText()`](#heading-212-updatehosttext)
  - [2.13 `completeWork()` 는 HostComponent의 업데이트를 표시하고 필요한 경우 DOM 노드를 생성합니다.](#heading-213-completework-hostcomponent-dom)
- [3. Re-render: Commit Phase](#heading-3-re-render-commit-phase)
  - [3.1 `commitMutationEffectsOnFiber()` 는 Insertion/Deletion/Update의 커밋을 시작합니다.](#heading-31-commitmutationeffectsonfiber-insertiondeletionupdate)
  - [3.2 삭제가 먼저 처리된 후, 자식 및 self를 처리합니다.](#heading-32-self)
  - [3.3 삽입은 다음으로 처리됩니다.](#heading-33)
  - [3.4 업데이트는 마지막에 처리됩니다.](#heading-34)
- [4. 요약](#heading-4)
---

# 1\. Re-render: Trigger phase

React는 [최초 마운트](https://ted-projects.com/react-internals-deep-dive-2)에서 Fiber Tree와 DOM 트리를 구성하며, 완료되면 아래와 같이 두 개의 트리가 생깁니다.

[![](https://jser.dev/static/rerender/1.avif align="left")](https://jser.dev/static/rerender/1.avif)

### 1.1 `lanes` and `childLanes`

Lane은 보류 중인 작업의 우선순위입니다. Fiber Node의 경우, 이렇습니다:

1. `lanes` =&gt; 스스로에 대한 보류 중인 작업
    
2. `childLanes` =&gt; 서브트리에 대한 보류 중인 작업
    

> ℹ️ Lane 에 대한 자세한 이야기는, [EP21 - 리액트 소스 코드에서 Lane이란?](https://jser.dev/react/2022/03/26/lanes-in-react/) 을 참조 해주세요.

버튼이 클릭 됐을 때, `setState()`가 호출됩니다:

1. 루트에서 대상 Fiber 까지의 경로에는 다음 렌더링에서 확인해야 할 위치를 표시하기 위해 `lanes` 와 `childLanes`가 표시됩니다.
    
2. 업데이트는 `scheduleUpdateOnFiber()`에 의해 스케줄링되며, 이 스케줄링은 결국 `ensureRootIsScheduled()` 를 호출하고 "Scheduler"에서 `performConcurrentWorkOnRoot()` 를 스케쥴링합니다. 이는 [최초 마운트](https://ted-projects.com/react-internals-deep-dive-2)와 매우 유사합니다.
    

명심해야 할 한 가지 중요한 점은 이벤트의 우선순위에 따라 업데이트의 우선순위가 결정된다는 것입니다. `click` 이벤트의 경우 우선순위가 높은 `SyncLane`에 매핑되는 `DiscreteEventPriority`입니다.

> ℹ️ \`useState()가 어떻게 동작하는지 더 자세히 알고 싶으면, [EP5 - useState()는 어떻게 동작하나요?](https://jser.dev/2023-06-19-how-does-usestate-work) 를 참조 하세요.

자세한 내용은 여기서는 생략하겠지만 결국에는 Fiber Tree를 따라 작업하게 됩니다.

[![](https://jser.dev/static/rerender/2.avif align="left")](https://jser.dev/static/rerender/2.avif)

## 2\. Re-render : Render phase

### 2.1 기본 렌더링 로직은 [최초 마운트](https://ted-projects.com/react-internals-deep-dive-2)와 동일합니다.

`click` 이벤트에서 렌더링 lane은 차단 lane인 SyncLane이기 때문에 최초 마운트와 마찬가지로 `performConcurrentWorkOnRoot()` 내부에서는 여전히 동시 모드(concurrent mode)가 활성화되지 않습니다.

> ℹ️ 동시 모드가 켜져 있는 경우, [EP8 - 리액트에서 useTransition()은 어떻게 동작하나요?](https://jser.dev/2023-05-19-how-does-usetransition-work/) 를 참고하세요.

아래는 전체 프로세스를 요약한 코드입니다.

> ***💬 역자 주석: Jser의 코멘트는 ❗❗로 표시 해뒀습니다.  
> 그 외 주석은 리액트 소스 코드 자체의 주석입니다.  
> ... 은 생략된 코드입니다.***

```typescript
do {
  try {
    workLoopSync();
    break;
  } catch (thrownValue) {
    handleError(root, thrownValue);
  }
} while (true);

function workLoopSync() {
  while (workInProgress !== null) {
    performUnitOfWork(workInProgress);
  }
}

function performUnitOfWork(unitOfWork: Fiber): void {
  const current = unitOfWork.alternate;
  next = beginWork(current, unitOfWork, subtreeRenderLanes);
  unitOfWork.memoizedProps = unitOfWork.pendingProps;
  // ❗❗ ↖ 이 줄은 중요하므로 2.5 memoizedProps vs pendingPros에서 다루겠습니다.
  if (next === null) {
    // If this doesn't spawn new work, complete the current work.
    completeUnitOfWork(unitOfWork);
  } else {
    workInProgress = next;
  }
}
```

더 자세한 설명은 [이전 에피소드](https://ted-projects.com/react-internals-deep-dive-2#heading-32-renderrootsync)를 참고하세요. 여기서는 React가 Fiber Tree를 순회하고 필요한 경우 Fiber를 업데이트한다는 점만 기억하세요.

### 2.2 React는 새로운 Fiber Node를 생성하기 전에 중복된 Fiber Node를 재사용합니다.

[최초 마운트](https://ted-projects.com/react-internals-deep-dive-2)에서 우리는 Fiber가 처음부터 생성되는 것을 볼 수 있습니다. 하지만 실제로 React는 먼저 Fiber Node를 재사용하려고 시도합니다.

```typescript
export function createWorkInProgress(current: Fiber, pendingProps: any): Fiber {
  let workInProgress = current.alternate;
  // ❗❗               ↗ curernt 는 현재 버전입니다
  // ❗❗ alternate는 그 이전 버전입니다.
  if (workInProgress === null) {
    // ❗❗ ↖ 처음부터 새로 만들어야 하는 경우
    // We use a double buffering pooling technique because we know that we'll
    // only ever need at most two versions of a tree. We pool the "other" unused
    // node that we're free to reuse. This is lazily created to avoid allocating
    // extra objects for things that are never updated. It also allow us to
    // reclaim the extra memory if needed.
    workInProgress = createFiber(
      current.tag,
      pendingProps, // ❗❗ pendingProps
      current.key,
      current.mode,
    );
    ...
    workInProgress.alternate = current;
    current.alternate = workInProgress;
  } else {
  // ❗❗ ↖ 이전 버전을 재사용할 수 있다면
    workInProgress.pendingProps = pendingProps;
    // ❗❗ 재사용이 가능하므로 Fiber Node를 만들 필요가 없습니다.Since we can reuse, we don't need to create Fiber Node
    // ❗❗ 하지만 프로퍼티를 업데이트하여 재사용하는 것만으로
    // Needed because Blocks store data on type.
    workInProgress.type = current.type;
    // We already have an alternate.
    // Reset the effect tag.
    workInProgress.flags = NoFlags;
    // The effects are no longer valid.
    workInProgress.subtreeFlags = NoFlags;
    workInProgress.deletions = null;
  }
  // Reset all effects except static ones.
  // Static effects are not specific to a render.
  workInProgress.flags = current.flags & StaticMask;
  workInProgress.childLanes = current.childLanes;
  workInProgress.lanes = current.lanes;
  // ❗❗ lanes 와 childLanes 가 복사됩니다.

  workInProgress.child = current.child;
  workInProgress.memoizedProps = current.memoizedProps;
  workInProgress.memoizedState = current.memoizedState;
  workInProgress.updateQueue = current.updateQueue;
  ...
  return workInProgress;
}
```

FiberRootNode는 `current` 를 통해 현재 Fiber Tree를 가리키기 때문에 현재 트리에 없는 모든 Fiber Node를 재사용할 수 있습니다.

리-렌더링 프로세스에서, 중복 `HostRoot`는 `prepareFreshStack()`에서 재사용됩니다.

```typescript
function prepareFreshStack(root: FiberRoot, lanes: Lanes): Fiber {
  root.finishedWork = null;
  root.finishedLanes = NoLanes;
  ...
  workInProgressRoot = root;
  const rootWorkInProgress = createWorkInProgress(root.current, null);
  // ❗❗                                               ↗ root의 current는 HostRoot의 FiberNode 입니다.
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

따라서 다음 head로 리-렌더링을 시작합니다.

[![](https://jser.dev/static/rerender/3.avif align="left")](https://jser.dev/static/rerender/3.avif)

색을 입혀 보겠습니다.

[![](https://jser.dev/static/rerender/4.avif align="left")](https://jser.dev/static/rerender/4.avif)

### 2.3 `beginWork()` 내의 업데이트 브랜치

`beginWork()`에는 업데이트를 처리하는 중요한 브랜치가 있는데, 최초 마운트 에피소드에서는 다루지 않았습니다.

```typescript
function beginWork(
  current: Fiber | null,
  // ❗❗ ↖ current는 그려지고 있는 현재 버전입니다.
  workInProgress: Fiber,
  // ❗❗ ↖ workInProgress 그려지고 있는 새 버전입니다.
  renderLanes: Lanes,
): Fiber | null {
  if (current !== null) {
  // ❗❗ ↗ current가 널이 아니면, 최초 마운트가 아니라는 의미입니다.
  // ❗❗ 만약 이게 HostComponent면, 이전 버전의 Fiber Node와 DOM 노드도 있습니다.
  // ❗❗ 따라서 React는 하위 트리에 더 깊게 들어가는 것을 피함으로써
  // ❗❗ 최적화 할 수 있습니다. - 이게 bailout 입니다! 
    const oldProps = current.memoizedProps;
    const newProps = workInProgress.pendingProps;
    if (
      oldProps !== newProps ||
      // ❗❗ ↗ 여기서는 얕은 동등이 아닌 깊은 동등을 사용하여
      // ❗❗ 리액트 렌더링의 중요한 동작을 유도합니다.
      hasLegacyContextChanged() ||
      // Force a re-render if the implementation changed due to hot reload:
      (__DEV__ ? workInProgress.type !== current.type : false)
    ) {
      // If props or context changed, mark the fiber as having performed work.
      // This may be unset if the props are determined to be equal later (memo).
      didReceiveUpdate = true;
    } else {
      // Neither props nor legacy context changes. Check if there's a pending
      // update or context change.
      const hasScheduledUpdateOrContext = checkScheduledUpdateOrContext(
      // ❗❗                              ↗ 이건 Fiber의 lane을 확인합니다.
        current,
        renderLanes,
      );
      if (
        !hasScheduledUpdateOrContext &&
        // If this is the second pass of an error or suspense boundary, there
        // may not be work scheduled on `current`, so we check for this flag.
        (workInProgress.flags & DidCapture) === NoFlags
      ) {
        // No pending updates or context. Bail out now.
        didReceiveUpdate = false;
        return attemptEarlyBailoutIfNoScheduledUpdate(
        // ❗❗  ↗ 만약 이 Fiber에 업데이트가 없으면, 리액트는 bailout을 시도합니다
        // ❗❗ 하지만 오로지 프로퍼티나 컨텍스트의 변경이 없을 경우에만 시도합니다. 
          current,
          workInProgress,
          renderLanes,
        );
      }
     ...
    }
  } else {
    didReceiveUpdate = false;
    // ❗❗ 이 마운트 브랜치는 이전에 이미 다뤘습니다.
    ...
  }
  workInProgress.lanes = NoLanes;
  switch (workInProgress.tag) {
    case IndeterminateComponent: {
      return mountIndeterminateComponent(
        current,
        workInProgress,
        workInProgress.type,
        renderLanes,
      );
    }
    case FunctionComponent: {
      const Component = workInProgress.type;
      const unresolvedProps = workInProgress.pendingProps;
      const resolvedProps =
        workInProgress.elementType === Component
          ? unresolvedProps
          : resolveDefaultProps(Component, unresolvedProps);
      return updateFunctionComponent( // ❗❗ updateFunctionComponent
        current,
        workInProgress,
        Component,
        resolvedProps,
        renderLanes,
      );
    }
    case HostRoot:
      return updateHostRoot(current, workInProgress, renderLanes); // ❗❗ updateHostRoot
    case HostComponent:
      return updateHostComponent(current, workInProgress, renderLanes); // ❗❗ updateHostComponent
    case HostText:
      return updateHostText(current, workInProgress); // ❗❗ updateHostText
    ...
  }
}
```

### 2.4 `attemptEarlyBailoutIfNoScheduledUpdate()` 내의 Bailout 로직

이름에서 알 수 있듯, 이 함수는 불필요한 경우 렌더링을 더 빨리 중지하려고 시도합니다.

```typescript
function attemptEarlyBailoutIfNoScheduledUpdate(
  current: Fiber,
  workInProgress: Fiber,
  renderLanes: Lanes,
) {
  // This fiber does not have any pending work. Bailout without entering
  // the begin phase. There's still some bookkeeping we that needs to be done
  // in this optimized path, mostly pushing stuff onto the stack.
  // 💬 위의 주석을 한번 읽어보라고 하여 바로 하단에 번역합니다.
  // ❗❗ 이 Fiber에는 보류 중인 작업이 없습니다. 시작 단계에 들어가지 않고 Bailout 합니다.
  // ❗❗ 이 최적화된 경로에서 아직 해야 할 일이 남아있습니다.
  // ❗❗ 이 최적화된 경로에서, 대부분은 스택에 밀어 넣어(push) 줍니다.
  switch (workInProgress.tag) {
    case HostRoot:
      pushHostRootContext(workInProgress);
      const root: FiberRoot = workInProgress.stateNode;
      pushRootTransition(workInProgress, root, renderLanes);
      if (enableCache) {
        const cache: Cache = current.memoizedState.cache;
        pushCacheProvider(workInProgress, cache);
      }
      resetHydrationState();
      break;
    case HostComponent:
      pushHostContext(workInProgress);
      break;
    ...
  }
  return bailoutOnAlreadyFinishedWork(current, workInProgress, renderLanes); // ❗❗ bailoutOnAlreadyFinishedWork
}
```

```typescript
function bailoutOnAlreadyFinishedWork(
  current: Fiber | null,
  workInProgress: Fiber,
  renderLanes: Lanes,
): Fiber | null {
  if (current !== null) {
    // Reuse previous dependencies
    workInProgress.dependencies = current.dependencies;
  }
  // Check if the children have any pending work.
  if (!includesSomeLane(renderLanes, workInProgress.childLanes)) {
  // ❗❗ ↗ 여기서 우리는 childLanes가 확인 되는걸 볼 수 있습니다.
    // The children don't have any work either. We can skip them.
    // TODO: Once we add back resuming, we should check if the children are
    // a work-in-progress set. If so, we need to transfer their effects.
    if (enableLazyContextPropagation && current !== null) {
      // Before bailing out, check if there are any context changes in
      // the children.
      lazilyPropagateParentContextChanges(current, workInProgress, renderLanes);
      if (!includesSomeLane(renderLanes, workInProgress.childLanes)) {
        return null;
      }
    } else {
      return null;
      // ❗❗ ↗ 따라서 만약 Fiber 자신과 그 하위 트리에 대한 업데이트가 없는 경우
      // ❗❗ 그럴 때는 당연히, 우리는 null을 반환함으로써 트리에서 더 깊이 들어가는 것을 
      // ❗❗ 멈출 수 있습니다.
    }
  }
  // This fiber doesn't have work, but its subtree does. Clone the child
  // fibers and continue.
  cloneChildFibers(current, workInProgress);
  // ❗❗ 비록 이 함수의 이름이 clone이지만, 실제로는 새로운 자식 노드들을 만들거나,
  // ❗❗ 이전 노드들을 재사용하기도 합니다.
  return workInProgress.child;
  // ❗❗ 우린 그냥 자식을 바로 반환하고, 리액트가 다음 fiber로 이것을 처리합니다.
  // ❗❗ 좀 더 알고 싶으면, EP15 - 리액트가 Fiber tree를 어떻게 순회하는지를 알아보세요
}
export function cloneChildFibers(
  current: Fiber | null,
  workInProgress: Fiber,
): void {
  // ❗❗ if (current !== null && workInProgress.child !== current.child)
  if (current !== null && workInProgress.child !== current.child) {
    throw new Error('Resuming work not yet implemented.');
  }
  if (workInProgress.child === null) {
    return;
  }
  let currentChild = workInProgress.child;
  let newChild = createWorkInProgress(currentChild, currentChild.pendingProps);
  // ❗❗                                                          ↗↗
  // ❗❗ cloneChildFibers()에서, 자식 fiber들은 이전 버전에서 만들어지지만
  // ❗❗ 조정(reconciliation) 중에 설정되는 새로운 pendingProps를 사용하여 생성됩니다. tion
  workInProgress.child = newChild;
  newChild.return = workInProgress;
  while (currentChild.sibling !== null) {
    currentChild = currentChild.sibling;
    newChild = newChild.sibling = createWorkInProgress(
      currentChild,
      currentChild.pendingProps,
    );
    newChild.return = workInProgress;
  }
  newChild.sibling = null;
}
```

bailout 절차를 요약해 보겠습니다.

1. Fiber에 프로퍼티/컨텍스트 변경이 없고 보류 중인 작업(비어있는 `lane`)이 없는 경우
    
    1. 자식에게 보류 중인 작업(비어있는 `childLanes`)이 없는 경우, bailout이 발생하고 React는 트리에서 더 깊게 이동하지 않습니다.
        
    2. 그렇지 않으면 React는 이 Fiber를 리-렌더링하지 않고 바로 자식에게 이동합니다.
        
2. 그렇지 않으면 React가 먼저 다시 렌더링을 시도한 후 자식에게 전달합니다.
    

> ℹ bailout에 대한 자세한 정보는, [EP13 - 리액트 reconciliation에서 bailout이 작동하는 방법](https://jser.dev/react/2022/01/07/how-does-bailout-work)을 참고해주세요

### 2.5 `memoizedProps` vs `pendingProps`

`beginWork()`에서 `workInProgress`는 `current` 와 비교됩니다. props의 경우, `workInProgress.pendingProps`가 `current.memoizedProps`와 비교됩니다. `momoizedProps`는 현재 props로, `pendingProps`는 다음 버전으로 생각할 수 있습니다.

React는 "Render" 단계에서 새로운 Fiber Tree를 생성한 다음 현재 Fiber Tree와 비교(diffing)합니다. `pendingProps`가 실제로는 workInProgress 생성을 위한 매개변수임을 알 수 있습니다.

```typescript
export function createWorkInProgress(current: Fiber, pendingProps: any): Fiber {
  let workInProgress = current.alternate;
  // ❗❗                 ↗ current 현재 버전입니다.
  // ❗❗ alternate 는 이것의 이전 버전입니다.
  if (workInProgress === null) {
  // ❗❗ 처음부터 새로 만들어야 하는  경우
    // We use a double buffering pooling technique because we know that we'll
    // only ever need at most two versions of a tree. We pool the "other" unused
    // node that we're free to reuse. This is lazily created to avoid allocating
    // extra objects for things that are never updated. It also allow us to
    // reclaim the extra memory if needed.
    workInProgress = createFiber(
      current.tag,
      pendingProps, // ❗❗ pendingProps
      current.key,
      current.mode,
    );
    ...
    workInProgress.alternate = current;
    current.alternate = workInProgress;
  } else {
  // ❗❗ ↖ 만약 이전 버전을 재사용할 수 있는 경우
    workInProgress.pendingProps = pendingProps;
    // ❗❗ ↗ 재사용이 가능하므로, Fiber Node를 만들 필요가 없습니다. 
    // ❗❗ 필요한 프로퍼티를 업데이트하여 재사용 할 수 있습니다.
    // Needed because Blocks store data on type.
    workInProgress.type = current.type;
    // We already have an alternate.
    // Reset the effect tag.
    workInProgress.flags = NoFlags;
    // The effects are no longer valid.
    workInProgress.subtreeFlags = NoFlags;
    workInProgress.deletions = null;
  }
  // Reset all effects except static ones.
  // Static effects are not specific to a render.
  workInProgress.flags = current.flags & StaticMask;
  workInProgress.childLanes = current.childLanes;
  workInProgress.lanes = current.lanes;
  workInProgress.child = current.child;
  workInProgress.memoizedProps = current.memoizedProps;
  workInProgress.memoizedState = current.memoizedState;
  workInProgress.updateQueue = current.updateQueue;
  // Clone the dependencies object. This is mutated during the render phase, so
  // it cannot be shared with the current fiber.
  const currentDependencies = current.dependencies;
  workInProgress.dependencies =
    currentDependencies === null
      ? null
      : {
          lanes: currentDependencies.lanes,
          firstContext: currentDependencies.firstContext,
        };
  // These will be overridden during the parent's reconciliation
  workInProgress.sibling = current.sibling;
  workInProgress.index = current.index;
  workInProgress.ref = current.ref;
  workInProgress.refCleanup = current.refCleanup;
  return workInProgress;
}
```

실제로, 루트 FiberNode 생성자에는 `pendingProps`가 매개변수로 있습니다.

```typescript

function createFiber(
  tag: WorkTag,
  pendingProps: mixed, // ❗❗ pendingProps: mixed
  key: null | string,
  mode: TypeOfMode,
): Fiber {
  // $FlowFixMe[invalid-constructor]: the shapes are exact here but Flow doesn't like constructors
  return new FiberNode(tag, pendingProps, key, mode);
}
function FiberNode(
  this: $FlowFixMe,
  tag: WorkTag,
  pendingProps: mixed, // ❗❗ pendingProps: mixed
  key: null | string,
  mode: TypeOfMode,
) {
  ....
}
```

따라서 Fiber Node를 만드는 것이 첫 번째 단계입니다. 이 Fiber Node는 나중에 동작하게 됩니다.

그리고 Fiber에 대한 리-렌더링이 완료되면 `memoizedProps`가 `pendingProps`에 설정되며, 이는 `performUnitOfWork()` 내부에 있습니다.

```typescript
function performUnitOfWork(unitOfWork: Fiber): void {
  const current = unitOfWork.alternate;
  setCurrentDebugFiberInDEV(unitOfWork);
  let next;
  if (enableProfilerTimer && (unitOfWork.mode & ProfileMode) !== NoMode) {
    startProfilerTimer(unitOfWork);
    next = beginWork(current, unitOfWork, subtreeRenderLanes);
    stopProfilerTimerIfRunningAndRecordDelta(unitOfWork, true);
  } else {
    next = beginWork(current, unitOfWork, subtreeRenderLanes);
  }
  resetCurrentDebugFiberInDEV();
  unitOfWork.memoizedProps = unitOfWork.pendingProps;
  // ❗❗ ↗ The memoizedProps는 작업이 완료되면 업데이트 됩니다.
  if (next === null) {
    // If this doesn't spawn new work, complete the current work.
    completeUnitOfWork(unitOfWork);
  } else {
    workInProgress = next;
  }
  ReactCurrentOwner.current = null;
}
```

이제 [demo](https://jser.dev/demos/react/overview/re-render.html)를 살펴보겠습니다.

1. React는 HostRoot(lanes: 0, childLanes: 1)에서 작동합니다. HostRoot에는 props가 없고 `memoizedProps` 와 `pendingProps` 가 모두 null이므로 React는 복제된 `App`인 자식으로 바로 이동합니다.
    
2. React는 `<App/>`(lanes: 0, childLanes: 1)에서 작동합니다. App 컴포넌트는 리-렌더링되지 않으므로 `memoizedProps` 와 `pendingProps` 는 동일하므로 React는 그 자식인 복제된 `div`로 바로 이동합니다.
    
3. React는 `<div/>`(lanes: 0, childLanes: 1)에서 작동합니다. App에서 자식들을 가져오지만 App이 다시 실행되지 않으므로 자식(`<Link>,<br/>` 및 `<Component/>`)이 변경되지 않으므로 다시 React는 `<Link/>` 로 바로 이동합니다.
    
4. React는 `<Link/>`(lanes: 0, childLanes: 0)에서 작동합니다. 이번에는 React가 더 깊이 들어갈 필요도 없으므로 여기서 멈추고 형제인 `<br/>`로 이동합니다.
    
5. React가 `<br/>`(lanes: 0, childLanes: 0)에서 작동하고, bailout이 다시 발생하고, React가 `<Component/>` 로 이동합니다.
    

이제 조금 다른 점이 있습니다. `<Component/>` 에는 `1` 의 `lanes`가 있어 React가 그 자식들을 리-렌더링하고 조정해야 하는데, 이는 `updateFunctionComponent(current, workInProgress)`로 수행됩니다.

지금까지 다음과 같은 상태를 얻었습니다.

[![](https://jser.dev/static/rerender/10.avif align="left")](https://jser.dev/static/rerender/10.avif)

### 2.6 `updateFunctionComponent()` 는 함수 컴포넌트를 리-렌더링하고 자식을 조정합니다.

```typescript
function updateFunctionComponent(
  current,
  workInProgress,
  Component,
  nextProps: any,
  renderLanes,
) {
  let context;
  if (!disableLegacyContext) {
    const unmaskedContext = getUnmaskedContext(workInProgress, Component, true);
    context = getMaskedContext(workInProgress, unmaskedContext);
  }
  let nextChildren;
  let hasId;
  prepareToReadContext(workInProgress, renderLanes);
  nextChildren = renderWithHooks(
    current,
    workInProgress,
    Component,
    nextProps,
    context,
    renderLanes,
  );
  // ❗❗ 여기서는 새 자식을 생성하기 위해 컴포넌트가 실행됨을 의미합니다.

  hasId = checkDidRenderIdHook();
  if (enableSchedulingProfiler) {
    markComponentRenderStopped();
  }
  if (current !== null && !didReceiveUpdate) {
    bailoutHooks(current, workInProgress, renderLanes);
    return bailoutOnAlreadyFinishedWork(current, workInProgress, renderLanes);
  }
  // React DevTools reads this flag.
  workInProgress.flags |= PerformedWork;
  reconcileChildren(current, workInProgress, nextChildren, renderLanes);
  // ❗❗ ↖ nextChildren을 전달하고 reconcileChildren()이 호출됩니다.
  return workInProgress.child;
}
```

우리는 [React가 최초 마운트를 수행하는 방법](https://ted-projects.com/react-internals-deep-dive-2#heading-36-reconcilechildren)에서 `reconcileChildren()`을 만났습니다. 내부적으로 자식 유형에 따라 몇 가지 변형이 있습니다. 그중 3가지에 집중하겠습니다.

새로운 하위 fiber를 생성할 뿐 아니라 기존 fiber를 재사용하려고 시도한다는 점을 기억하세요.

```typescript
function reconcileChildFibersImpl(
  returnFiber: Fiber,
  currentFirstChild: Fiber | null,
  newChild: any,
  lanes: Lanes,
): Fiber | null {
  ...
  // Handle object types
  if (typeof newChild === 'object' && newChild !== null) {
    switch (newChild.$$typeof) {
      case REACT_ELEMENT_TYPE:
        return placeSingleChild(
          reconcileSingleElement(
          // ❗❗ 하나의 자식만 있는 경우
            returnFiber,
            currentFirstChild,
            newChild,
            lanes,
          ),
        );
      case REACT_PORTAL_TYPE:
       ...
      case REACT_LAZY_TYPE:
        ...
    }
    if (isArray(newChild)) {
      return reconcileChildrenArray(
        returnFiber,
        currentFirstChild,
        newChild,
        lanes,
      );
      // ❗❗ 자식이 엘레먼트의 배열일 경우
    }
    ...
  }
  if (
    (typeof newChild === 'string' && newChild !== '') ||
    typeof newChild === 'number'
  ) {
    return placeSingleChild(
      reconcileSingleTextNode(
        returnFiber,
        currentFirstChild,
        '' + newChild,
        lanes,
      ),
      // ❗❗ 자식이 텍스트일 경우if children is text
    );
  }
  // Remaining cases are all treated as empty.
  return deleteRemainingChildren(returnFiber, currentFirstChild); // ❗❗ deleteRemainingChildren
}
```

`<Component/>`의 경우 단일 `div`를 반환합니다. 따라서 `reconcileSingleElement()`로 이동하겠습니다.

### 2.7 \``` reconcileSingleElement()` ``

```typescript
function reconcileSingleElement(
  returnFiber: Fiber,
  currentFirstChild: Fiber | null,
  element: ReactElement,
  // ❗❗ ↖ 여기 이것은 <div/>의 element이자 Component()의 반환값입니다.
  lanes: Lanes,
): Fiber {
  const key = element.key;
  let child = currentFirstChild;
  while (child !== null) {
    // TODO: If key === null and child.key === null, then this only applies to
    // the first item in the list.
    if (child.key === key) {
      const elementType = element.type;
      if (elementType === REACT_FRAGMENT_TYPE) {
        ...
      } else {
        if (
          child.elementType === elementType ||
          // ❗❗ ↗↗ 만약 타입이 같다면 재활용할 수 있습니다.
          // ❗❗ 그렇지 않다면 그냥 deleteChild()를 호출합니다.
          // Keep this check inline so it only runs on the false path:
          (__DEV__
            ? isCompatibleFamilyForHotReloading(child, element)
            : false) ||
          // Lazy types should reconcile their resolved type.
          // We need to do this after the Hot Reloading check above,
          // because hot reloading has different semantics than prod because
          // it doesn't resuspend. So we can't let the call below suspend.
          (typeof elementType === 'object' &&
            elementType !== null &&
            elementType.$$typeof === REACT_LAZY_TYPE &&
            resolveLazy(elementType) === child.type)
        ) {
          deleteRemainingChildren(returnFiber, child.sibling);
          const existing = useFiber(child, element.props);
          // ❗❗             ↗ 새로운 prop들을 갖고 기존의 fiber를 사용하려고 시도합니다
          // ❗❗             element.props 은 <div />으 pops 입니다.
          existing.ref = coerceRef(returnFiber, child, element);
          existing.return = returnFiber;
          return existing;
        }
      }
      // Didn't match.
      deleteRemainingChildren(returnFiber, child);
      break;
    } else {
      deleteChild(returnFiber, child);
    }
    child = child.sibling;
  }
  if (element.type === REACT_FRAGMENT_TYPE) {
    ...
  } else {
    const created = createFiberFromElement(element, returnFiber.mode, lanes);
    created.ref = coerceRef(returnFiber, currentFirstChild, element);
    created.return = returnFiber;
    return created;
  }
}
```

그리고 `useFiber`에서 React는 이전 버전을 생성하거나 재사용합니다. 앞서 언급했듯이 `pendingProps`(자식을 포함하는)가 설정됩니다.

```typescript
function useFiber(fiber: Fiber, pendingProps: mixed): Fiber {
  // We currently set sibling to null and index to 0 here because it is easy
  // to forget to do before returning it. E.g. for the single child case.
  const clone = createWorkInProgress(fiber, pendingProps); // ❗❗ createWorkInProgress(fiber, pendingProps)
  clone.index = 0;
  clone.sibling = null;
  return clone;
}
```

따라서 컴포넌트가 다시 렌더링된 후 React는 새로운 `<div/>`인 자식으로 이동하며, 현재 버전은 빈 `lanes` 와 `childLanes` 를 모두 가지고 있습니다.

### 2.8 컴포넌트가 리-렌더링되면 기본적으로 해당 하위 트리가 리-렌더링됩니다.

`<div/>`와 그 자식들은 예정된 작업이 없으므로 bailout이 발생한다고 생각할 수 있지만 그렇지 않습니다.

`beginWork()`에 `memoizedProps` 및 `pendingProps` 검사가 있다는 것을 기억하세요.

```typescript
const oldProps = current.memoizedProps;
const newProps = workInProgress.pendingProps;
if (
  oldProps !== newProps ||
  // ❗❗ ↖ 여기서는 얕은 동등이 아닌 깊은 동등을 사용합니다
  hasLegacyContextChanged() ||
  // Force a re-render if the implementation changed due to hot reload:
  (__DEV__ ? workInProgress.type !== current.type : false)
) {
  // If props or context changed, mark the fiber as having performed work.
  // This may be unset if the props are determined to be equal later (memo).
  didReceiveUpdate = true;
}
```

컴포넌트가 렌더링될 때마다 React 엘리먼트가 포함된 새로운 객체를 생성하므로 매번 `pendingProps`가 새로 생성되는 반면, props를 비교할 때는 얕은 동등([느슨한 동등](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Operators/Equality))이 사용되지 않는다는 점에 유의하세요.

`<div/>의` 경우 `Component()` 가 실행되면 항상 새 프로퍼티를 가져오기 때문에 bailout이 전혀 일어나지 않습니다.

따라서 React는 업데이트 브랜치 - `updateHostComponent()`로 이동합니다.

### 2.9 `updateHostComponent()`

```typescript
function updateHostComponent(
  current: Fiber | null,
  workInProgress: Fiber,
  renderLanes: Lanes,
) {
  pushHostContext(workInProgress);
  if (current === null) {
    tryToClaimNextHydratableInstance(workInProgress);
  }
  const type = workInProgress.type;
  const nextProps = workInProgress.pendingProps;
  const prevProps = current !== null ? current.memoizedProps : null;
  let nextChildren = nextProps.children;
  const isDirectTextChild = shouldSetTextContent(type, nextProps);
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
  markRef(current, workInProgress);
  reconcileChildren(current, workInProgress, nextChildren, renderLanes);
  // ❗❗ reconcileChildren(current, workInProgress, nextChildren, renderLanes)
  return workInProgress.child;
}
```

여기있는 nextChildren은 아래와 같습니다:

```typescript
[
  {$$typeof: Symbol(react.element), type: 'button'},
  " (", 
  {$$typeof: Symbol(react.element), type: 'b'}, 
  ")"
]
```

따라서 내부적으로 React는 `reconcileChildrenArray()`로 이를 조정합니다.

그리고 `memoizedProps의current`는 아래와 같습니다.

```typescript
[
  {$$typeof: Symbol(react.element), type: 'button'},
  " (", 
  {$$typeof: Symbol(react.element), type: 'span'}, 
  ")"
]
```

### 2.10 `reconcileChildrenArray()` 는 필요에 따라 Fiber를 생성하고 삭제합니다.

`reconcileChildrenArray()` 는 약간 복잡합니다. 엘리먼트의 재정렬(re-order)이 있는지 확인하고 `key` 가 있는 경우 Fiber를 재사용하려고 시도하여 추가적인 최적화를 수행합니다.

> ℹ `key`의 경우, 이 주제에 대한 별도의 에피소드 - [EP19 - key는 어떻게 동작하나? 리액트의 리스트 diffing](https://jser.dev/react/2022/02/08/the-diffing-algorithm-for-array-in-react/)

하지만 데모에서는 `key`가 없으므로 그냥 기본 지점으로 이동하겠습니다.

```typescript
function reconcileChildrenArray(
    returnFiber: Fiber,
    currentFirstChild: Fiber | null,
    newChildren: Array<any>,
    lanes: Lanes,
  ): Fiber | null {
    let resultingFirstChild: Fiber | null = null;
    let previousNewFiber: Fiber | null = null;
    let oldFiber = currentFirstChild;
    let lastPlacedIndex = 0;
    let newIdx = 0;
    let nextOldFiber = null;
    for (; oldFiber !== null && newIdx < newChildren.length; newIdx++) {
      // ❗❗ ↗ 하위 엘리먼트들에 대한 현재 fiber들의 확인
      if (oldFiber.index > newIdx) {
        nextOldFiber = oldFiber;
        oldFiber = null;
      } else {
        nextOldFiber = oldFiber.sibling;
      }
      const newFiber = updateSlot(
        returnFiber,
        oldFiber,
        newChildren[newIdx],
        lanes,
      );
      // ❗❗ ↖ 여기에서 목록의 각 fiber를 새로운 props로 확인합니다.
      if (newFiber === null) {
        // TODO: This breaks on empty slots like null children. That's
        // unfortunate because it triggers the slow path all the time. We need
        // a better way to communicate whether this was a miss or null,
        // boolean, undefined, etc.
        if (oldFiber === null) {
          oldFiber = nextOldFiber;
        }
        break;
      }
      if (shouldTrackSideEffects) {
        if (oldFiber && newFiber.alternate === null) {
          // We matched the slot, but we didn't reuse the existing fiber, so we
          // need to delete the existing child.
          deleteChild(returnFiber, oldFiber);
        } // ❗❗ ↖ 
          // ❗❗ 만약 fiber를 재사용할 수 없으면, Deletion 표시가 될 것입니다.
          // ❗❗ 커밋 단계에서 해당 DOM 노드가 삭제됩니다.
      }
      lastPlacedIndex = placeChild(newFiber, lastPlacedIndex, newIdx);
      // ❗❗               ↗ 이렇게 하면 fiber를 Insertion으로 표시하려고 시도합니다
      if (previousNewFiber === null) {
        // TODO: Move out of the loop. This only happens for the first run.
        resultingFirstChild = newFiber;
      } else {
        // TODO: Defer siblings if we're not at the right index for this slot.
        // I.e. if we had null values before, then we want to defer this
        // for each null value. However, we also don't want to call updateSlot
        // with the previous one.
        previousNewFiber.sibling = newFiber;
      }
      previousNewFiber = newFiber;
      oldFiber = nextOldFiber;
    }
    if (newIdx === newChildren.length) {
      // We've reached the end of the new children. We can delete the rest.
      deleteRemainingChildren(returnFiber, oldFiber);
      if (getIsHydrating()) {
        const numberOfForks = newIdx;
        pushTreeFork(returnFiber, numberOfForks);
      }
      return resultingFirstChild;
    }
    ...
    return resultingFirstChild;
  }
```

`updateSlot()` 은 기본적으로 `key`를 고려하여 새로운 props로 Fiber를 생성하거나 재사용할 뿐입니다.

```typescript
function updateSlot(
  returnFiber: Fiber,
  oldFiber: Fiber | null,
  newChild: any,
  lanes: Lanes,
): Fiber | null {
  // Update the fiber if the keys match, otherwise return null.
  const key = oldFiber !== null ? oldFiber.key : null;
  if (
    (typeof newChild === 'string' && newChild !== '') ||
    typeof newChild === 'number'
  ) {
    // Text nodes don't have keys. If the previous node is implicitly keyed
    // we can continue to replace it without aborting even if it is not a text
    // node.
    if (key !== null) {
      return null;
    }
    return updateTextNode(returnFiber, oldFiber, '' + newChild, lanes);
  }
  if (typeof newChild === 'object' && newChild !== null) {
    switch (newChild.$$typeof) {
      case REACT_ELEMENT_TYPE: {
        if (newChild.key === key) {
          return updateElement(returnFiber, oldFiber, newChild, lanes);
          // ❗❗ ↗ updateElement 
        } else {
          return null;
        }
      }
      ...
    }
  }
  return null;
}
function updateElement(
  returnFiber: Fiber,
  current: Fiber | null,
  element: ReactElement,
  lanes: Lanes,
): Fiber {
  const elementType = element.type;
  if (elementType === REACT_FRAGMENT_TYPE) {
    return updateFragment(
      returnFiber,
      current,
      element.props.children,
      lanes,
      element.key,
    );
  }
  if (current !== null) {
    if (
      current.elementType === elementType ||
      // ❗❗ ↗↗ 재사용 할 수 있는 경우
      // Keep this check inline so it only runs on the false path:
      (__DEV__
        ? isCompatibleFamilyForHotReloading(current, element)
        : false) ||
      // Lazy types should reconcile their resolved type.
      // We need to do this after the Hot Reloading check above,
      // because hot reloading has different semantics than prod because
      // it doesn't resuspend. So we can't let the call below suspend.
      (typeof elementType === 'object' &&
        elementType !== null &&
        elementType.$$typeof === REACT_LAZY_TYPE &&
        resolveLazy(elementType) === current.type)
    ) {
      // Move based on index
      const existing = useFiber(current, element.props);
      // ❗❗            ↗ 여기서 useFiber()를 다시 볼 수 있습니다.
      existing.ref = coerceRef(returnFiber, current, element);
      existing.return = returnFiber;
      return existing;
    }
  }
  // Insert
  const created = createFiberFromElement(element, returnFiber.mode, lanes);
  // ❗❗             ↗ 유형이 달라서 재사용할 수 없는 경우, 처음부터 다시 fiber를 생성합니다.
  created.ref = coerceRef(returnFiber, current, element);
  created.return = returnFiber;
  return created;
}
```

따라서 `<div/>`에서, `updateSlot()`은 3개의 자식을 성공적으로 재사용하고, 4번째는 예외인데 왜냐하면 `current`가 `span`이지만 우리는 `b`를 원하기 때문에, span의 Fiber는 처음부터 생성되고 `deleteChild()`를 통해 `b`의 Fiber는 삭제됩니다. 새로 생성된 `span`은 `placeChild()`로 표시됩니다.

### 2.11 `placeChild()` 및 `deleteChild()` 는 플래그로 Fiber를 표시합니다.

자, `Comonent` 아래 `<div>`의 자식에는 파이버 노드를 표시하는 이 두 가지 함수가 있습니다.

```typescript
function placeChild(
  newFiber: Fiber,
  lastPlacedIndex: number,
  newIndex: number,
): number {
  newFiber.index = newIndex;
  if (!shouldTrackSideEffects) {
    // During hydration, the useId algorithm needs to know which fibers are
    // part of a list of children (arrays, iterators).
    newFiber.flags |= Forked;
    return lastPlacedIndex;
  }
  const current = newFiber.alternate;
  if (current !== null) {
    const oldIndex = current.index;
    if (oldIndex < lastPlacedIndex) {
      // This is a move.  // ❗❗
      newFiber.flags |= Placement; // ❗❗ 
      return lastPlacedIndex;
    } else {
      // This item can stay in place.
      return oldIndex;
    }
  } else {
    // This is an insertion.  // ❗❗
    newFiber.flags |= Placement; // ❗❗
    return lastPlacedIndex;
  }
}
```

```typescript
function deleteChild(returnFiber: Fiber, childToDelete: Fiber): void {
  if (!shouldTrackSideEffects) {
    // Noop.
    return;
  }
  const deletions = returnFiber.deletions;
  if (deletions === null) {
    returnFiber.deletions = [childToDelete];
    returnFiber.flags |= ChildDeletion; // ❗❗ returnFiber.flags |= ChildDeletion
  } else {
    deletions.push(childToDelete);
  }
}
```

삭제해야 하는 Fiber는 부모 Fiber의 배열에 임시로 저장됩니다. 이는 삭제 후 새 Fiber Tree에 더 이상 존재하지 않지만 "Commit" 단계에서 처리해야 하므로 어딘가에 저장해야 하기 때문에 저장이 필요합니다.

이제 `<div>가` 완성되었습니다.

[![](https://jser.dev/static/rerender/13.avif align="left")](https://jser.dev/static/rerender/13.avif)

다음 React는 `버튼`으로 이동합니다. 다시 말하지만, 스케줄이 작동하지 않는다고 생각했지만, prop이 `["click me-", "1"]` 에서 `["click me-", "2"]`로 변경되었기 때문에 React는 여전히 `updateHostComponent()` 를 사용하여 작동합니다.

HostText의 경우 프로퍼티가 문자열이므로 첫 번째 `"click me -"` 는 중단됩니다. 그리고 React는 `updateHostText()`로 텍스트를 조정하려고 시도합니다.

### 2.12 `updateHostText()`

```typescript
function updateHostText(current, workInProgress) {
  if (current === null) {
    tryToClaimNextHydratableInstance(workInProgress);
  }
  // Nothing to do here. This is terminal. We'll do the completion step
  // immediately after.
  return null;
}
```

다시 말하지만 이건 아무것도 안합니다, 왜냐하면 업데이트는 완료 단계인 `completeWork()` 에서 표시됩니다. 이는 [최초 마운트](https://ted-projects.com/react-internals-deep-dive-2#heading-36-reconcilechildren)에서도 설명 했습니다.

### 2.13 `completeWork()` 는 HostComponent의 업데이트를 표시하고 필요한 경우 DOM 노드를 생성합니다.

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
    case HostComponent: {
      popHostContext(workInProgress);
      const rootContainerInstance = getRootHostContainer();
      const type = workInProgress.type;
      if (current !== null && workInProgress.stateNode != null) {
      // ❗❗ ↗ 이게 우리가 진행하고 있는 업데이트 브랜치입니다.
        updateHostComponent(
          current,
          workInProgress,
          type,
          newProps,
          rootContainerInstance,
        );
        if (current.ref !== workInProgress.ref) {
          markRef(workInProgress);
        }
      } else {
      // ❗❗ 이건 이전에 EP에서 했던 최초 마운트 브랜치입니다.
        ...
      }
      bubbleProperties(workInProgress);
      return null;
    }
    case HostText: {
      const newText = newProps;
      if (current && workInProgress.stateNode != null) {
      // ❗❗ ↗ 이게 우리가 진행하고 있는 업데이트 브랜치입니다.
        const oldText = current.memoizedProps;
        // If we have an alternate, that means this is an update and we need
        // to schedule a side-effect to do the updates.
        updateHostText(current, workInProgress, oldText, newText);
      } else {
        // ❗❗ 이건 이전에 EP에서 했던 최초 마운트 브랜치입니다.
        ...
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

```typescript
updateHostText = function(
  // ❗❗ 이 것은 complete 단계에 있는 다른 updateHostText() 입니다.
  current: Fiber,
  workInProgress: Fiber,
  oldText: string,
  newText: string,
) {
  // If the text differs, mark it as an update. All the work in done in commitWork.
  if (oldText !== newText) {
    markUpdate(workInProgress); // ❗❗ markUpdate
  }
};
updateHostComponent = function(
  current: Fiber,
  workInProgress: Fiber,
  type: Type,
  newProps: Props,
  rootContainerInstance: Container,
) {
  // If we have an alternate, that means this is an update and we need to
  // schedule a side-effect to do the updates.
  const oldProps = current.memoizedProps;
  if (oldProps === newProps) {
    // In mutation mode, this is sufficient for a bailout because
    // we won't touch this node even if children changed.
    return;
  }
  // If we get updated because one of our children updated, we don't
  // have newProps so we'll have to reuse them.
  // TODO: Split the update API as separate for the props vs. children.
  // Even better would be if children weren't special cased at all tho.
  const instance: Instance = workInProgress.stateNode;
  const currentHostContext = getHostContext();
  // TODO: Experiencing an error where oldProps is null. Suggests a host
  // component is hitting the resume path. Figure out why. Possibly
  // related to `hidden`.
  const updatePayload = prepareUpdate(
    instance,
    type,
    oldProps,
    newProps,
    rootContainerInstance,
    currentHostContext,
  );
  // TODO: Type this specific to this type of component.
  workInProgress.updateQueue = (updatePayload: any);
  // ❗❗            ↗ 업데이트는 업데이트 큐에 저장됩니다.
  // ❗❗               실제로 이펙트 훅과 같은 훅에서에서 사용됩니다.
  // If the update payload indicates that there is a change or if there
  // is a new ref we mark this as an update. All the work is done in commitWork.
  if (updatePayload) {
    markUpdate(workInProgress); // ❗❗ markUpdate
  }
};
function markUpdate(workInProgress: Fiber) {
  // Tag the fiber with an update effect. This turns a Placement into
  // a PlacementAndUpdate.
  workInProgress.flags |= Update;
               // ❗❗ ↗↗ 맞습니다, 또다른 플래그입니다!
}
```

렌더링 단계가 끝나면 다음과 같은 결과가 나타납니다.

1. `b`에 삽입(Insertion)
    
2. `span`에서 삭제(Deletion)
    
3. HostText의 업데이트
    
4. `button` 의 업데이트(후드 아래 비어 있음)
    

한 가지 강조하고 싶은 것은 `button`과 그 부모 `div` 모두에 대해 `prepareUpdate()` 가 실행되지만, `div`에 대해서는 `null`을 생성하지만 `button`에 대해서는 `[]` 을 생성한다는 점입니다. 여기서는 다루지 않을 까다로운 에지 케이스입니다.

[![](https://jser.dev/static/rerender/25.avif align="left")](https://jser.dev/static/rerender/25.avif)

이제 커밋 단계에서 이러한 업데이트를 커밋할 차례입니다.

## 3\. Re-render: Commit Phase

### 3.1 `commitMutationEffectsOnFiber()` 는 Insertion/Deletion/Update의 커밋을 시작합니다.

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
      // ❗❗ ↖ 재귀적으로 자식들을 먼저 처리합니다
      commitReconciliationEffects(finishedWork);
      // ❗❗ ↖ 그리고 나서 삽입(Insertion)을 처리합니다.
      if (flags & Update) {
      // ❗❗ ↖ 업데이트를 마지막에 처리합니다.
        try {
          commitHookEffectListUnmount(
            HookInsertion | HookHasEffect,
            finishedWork,
            finishedWork.return,
          );
          commitHookEffectListMount(
            HookInsertion | HookHasEffect,
            finishedWork,
          );
        } catch (error) {
          captureCommitPhaseError(finishedWork, finishedWork.return, error);
        }
        ...
      }
      return;
    }
    case HostComponent: {
      recursivelyTraverseMutationEffects(root, finishedWork, lanes);
      // ❗❗ ↖ 재귀적으로 자식들을 먼저 처리합니다
      commitReconciliationEffects(finishedWork);
      // ❗❗ ↖ 그리고 나서 삽입(Insertion)을 처리합니다.

      if (supportsMutation) {
        // TODO: ContentReset gets cleared by the children during the commit
        // phase. This is a refactor hazard because it means we must read
        // flags the flags after `commitReconciliationEffects` has already run;
        // the order matters. We should refactor so that ContentReset does not
        // rely on mutating the flag during commit. Like by setting a flag
        // during the render phase instead.
        if (finishedWork.flags & ContentReset) {
          const instance: Instance = finishedWork.stateNode;
          try {
            resetTextContent(instance);
          } catch (error) {
            captureCommitPhaseError(finishedWork, finishedWork.return, error);
          }
        }
        if (flags & Update) {
        // ❗❗ ↖ 업데이트를 마지막에 처리합니다.
          const instance: Instance = finishedWork.stateNode;
          if (instance != null) {
            // Commit the work prepared earlier.
            const newProps = finishedWork.memoizedProps;
            // For hydration we reuse the update path but we treat the oldProps
            // as the newProps. The updatePayload will contain the real change in
            // this case.
            const oldProps =
              current !== null ? current.memoizedProps : newProps;
            const type = finishedWork.type;
            // TODO: Type the updateQueue to be specific to host components.
            const updatePayload: null | UpdatePayload = (finishedWork.updateQueue: any);
            finishedWork.updateQueue = null;
            if (updatePayload !== null) {
              try {
                commitUpdate(
                // ❗❗ ↖ HostComponent의 경우, props만 업데이트 처리합니다.
                  instance,
                  updatePayload,
                  type,
                  oldProps,
                  newProps,
                  finishedWork,
                );
              } catch (error) {
                captureCommitPhaseError(
                  finishedWork,
                  finishedWork.return,
                  error,
                );
              }
            }
          }
          
        }
      }
      return;
    }
    case HostText: {
      recursivelyTraverseMutationEffects(root, finishedWork, lanes);
      commitReconciliationEffects(finishedWork);
      if (flags & Update) {
        if (supportsMutation) {
          if (finishedWork.stateNode === null) {
            throw new Error(
              'This should have a text node initialized. This error is likely ' +
                'caused by a bug in React. Please file an issue.',
            );
          }
          const textInstance: TextInstance = finishedWork.stateNode;
          const newText: string = finishedWork.memoizedProps;
          // For hydration we reuse the update path but we treat the oldProps
          // as the newProps. The updatePayload will contain the real change in
          // this case.
          const oldText: string =
            current !== null ? current.memoizedProps : newText;
          try {
            commitTextUpdate(textInstance, oldText, newText);
            // ❗❗ ↖ HostText를 위해, textContent만 업데이트합니다.
          } catch (error) {
            captureCommitPhaseError(finishedWork, finishedWork.return, error);
          }
        }
      }
      return;
    }
  }
}
```

우리는 이것이 재귀적 프로세스임을 알 수 있으며 각 변형(mutation) 유형을 자세히 살펴 보겠습니다.

### 3.2 삭제가 먼저 처리된 후, 자식 및 self를 처리합니다.

```typescript
function recursivelyTraverseMutationEffects(
  root: FiberRoot,
  parentFiber: Fiber,
  lanes: Lanes,
) {
  // Deletions effects can be scheduled on any fiber type. They need to happen
  // before the children effects hae fired.
  const deletions = parentFiber.deletions; // ❗❗ 
  if (deletions !== null) {
    for (let i = 0; i < deletions.length; i++) {
      const childToDelete = deletions[i];
      try {
        commitDeletionEffects(root, parentFiber, childToDelete); // ❗❗ commitDeletionEffects
      } catch (error) {
        captureCommitPhaseError(childToDelete, parentFiber, error);
      }
    }
  }
  const prevDebugFiber = getCurrentDebugFiberInDEV();
  if (parentFiber.subtreeFlags & MutationMask) {
    let child = parentFiber.child;
    while (child !== null) {
      setCurrentDebugFiberInDEV(child);
      commitMutationEffectsOnFiber(child, root, lanes); // ❗❗ commitMutationEffectsOnFiber
      child = child.sibling;
    }
  }
  setCurrentDebugFiberInDEV(prevDebugFiber);
}
```

자식을 처리하기 전임에도, 삭제는 우선적으로 처리됩니다.

```typescript
function commitDeletionEffects(
  root: FiberRoot,
  returnFiber: Fiber,
  deletedFiber: Fiber,
) {
  if (supportsMutation) {
    // We only have the top Fiber that was deleted but we need to recurse down its
    // children to find all the terminal nodes.
    // Recursively delete all host nodes from the parent, detach refs, clean
    // up mounted layout effects, and call componentWillUnmount.
    // We only need to remove the topmost host child in each branch. But then we
    // still need to keep traversing to unmount effects, refs, and cWU. TODO: We
    // could split this into two separate traversals functions, where the second
    // one doesn't include any removeChild logic. This is maybe the same
    // function as "disappearLayoutEffects" (or whatever that turns into after
    // the layout phase is refactored to use recursion).
    // Before starting, find the nearest host parent on the stack so we know
    // which instance/container to remove the children from.
    // TODO: Instead of searching up the fiber return path on every deletion, we
    // can track the nearest host component on the JS stack as we traverse the
    // tree during the commit phase. This would make insertions faster, too.
    let parent = returnFiber;
    findParent: while (parent !== null) {
    // ❗❗ ↖ 부모 노드가 반드시 백업 DOM을 가지고 있다는 것을 의미하지 않으므로
    // ❗❗ 여기서는 백업 DOM을 가진 가장 가까운 Fiber Node를 검색합니다.
      switch (parent.tag) {
        case HostComponent: {
          hostParent = parent.stateNode;
          hostParentIsContainer = false;
          break findParent;
        }
        case HostRoot: {
          hostParent = parent.stateNode.containerInfo;
          hostParentIsContainer = true;
          break findParent;
        }
        case HostPortal: {
          hostParent = parent.stateNode.containerInfo;
          hostParentIsContainer = true;
          break findParent;
        }
      }
      parent = parent.return;
    }
    if (hostParent === null) {
      throw new Error(
        'Expected to find a host parent. This error is likely caused by ' +
          'a bug in React. Please file an issue.',
      );
    }
    commitDeletionEffectsOnFiber(root, returnFiber, deletedFiber); // ❗❗ commitDeletionEffectsOnFiber
    hostParent = null;
    hostParentIsContainer = false;
  } else {
    // Detach refs and call componentWillUnmount() on the whole subtree.
    commitDeletionEffectsOnFiber(root, returnFiber, deletedFiber); // ❗❗commitDeletionEffectsOnFiber
  }
  detachFiberMutation(deletedFiber);
}
```

```typescript
function commitDeletionEffectsOnFiber(
  finishedRoot: FiberRoot,
  nearestMountedAncestor: Fiber,
  deletedFiber: Fiber,
) {
  onCommitUnmount(deletedFiber);
  // The cases in this outer switch modify the stack before they traverse
  // into their subtree. There are simpler cases in the inner switch
  // that don't modify the stack.
  switch (deletedFiber.tag) {
    case HostComponent: {
      if (!offscreenSubtreeWasHidden) {
        safelyDetachRef(deletedFiber, nearestMountedAncestor);
      }
      // Intentional fallthrough to next branch
    }
    // eslint-disable-next-line-no-fallthrough
    case HostText: {
      // We only need to remove the nearest host child. Set the host parent
      // to `null` on the stack to indicate that nested children don't
      // need to be removed.
      if (supportsMutation) {
        const prevHostParent = hostParent;
        const prevHostParentIsContainer = hostParentIsContainer;
        hostParent = null;
        recursivelyTraverseDeletionEffects(
          finishedRoot,
          nearestMountedAncestor,
          deletedFiber,
        );
        hostParent = prevHostParent;
        hostParentIsContainer = prevHostParentIsContainer;
        if (hostParent !== null) {
          // Now that all the child effects have unmounted, we can remove the
          // node from the tree.
          if (hostParentIsContainer) {
            removeChildFromContainer( // ❗❗ removeChildFromContainer
              ((hostParent: any): Container),
              // ❗❗ ↗ 여기의 hostParent는 이전 while 루프에서 재시도됩니다.
              (deletedFiber.stateNode: Instance | TextInstance),
            );
          } else {
            removeChild( // ❗❗ removeChild
              ((hostParent: any): Instance),
              (deletedFiber.stateNode: Instance | TextInstance),
            );
          }
        }
      } else {
        recursivelyTraverseDeletionEffects(
          finishedRoot,
          nearestMountedAncestor,
          deletedFiber,
        );
      }
      return;
    }
    ...
    default: {
      recursivelyTraverseDeletionEffects(
        finishedRoot,
        nearestMountedAncestor,
        deletedFiber,
      );
      return;
    }
  }
}
```

[![](https://jser.dev/static/rerender/26.avif align="left")](https://jser.dev/static/rerender/26.avif)

### 3.3 삽입은 다음으로 처리됩니다.

이는 새로 생성된 노드를 트리 구조로 설정할 수 있도록 하기 위한 것입니다.

```typescript
function commitReconciliationEffects(finishedWork: Fiber) {
  // Placement effects (insertions, reorders) can be scheduled on any fiber
  // type. They needs to happen after the children effects have fired, but
  // before the effects on this fiber have fired.
  const flags = finishedWork.flags;
  if (flags & Placement) {
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
  if (flags & Hydrating) {
    finishedWork.flags &= ~Hydrating;
  }
}
function commitPlacement(finishedWork: Fiber): void {
  if (!supportsMutation) {
    return;
  }
  // Recursively insert all host nodes into the parent.
  const parentFiber = getHostParentFiber(finishedWork);
  // Note: these two variables *must* always be updated together.
  switch (parentFiber.tag) {
    case HostComponent: {
      const parent: Instance = parentFiber.stateNode;
      if (parentFiber.flags & ContentReset) {
        // Reset the text content of the parent before doing any insertions
        resetTextContent(parent);
        // Clear ContentReset from the effect tag
        parentFiber.flags &= ~ContentReset;
      }
      const before = getHostSibling(finishedWork);
      // ❗❗          ↗ 여기가 중요합니다. Node.insertBefore()는 형제자매 노드들이 필요합니다.
      // ❗❗             만약 우리가 찾을 수 없으면, 그 땐 끝에 추가해줍니다.
      // We only have the top Fiber that was inserted but we need to recurse down its
      // children to find all the terminal nodes.
      insertOrAppendPlacementNode(finishedWork, before, parent); // ❗❗ insertOrAppendPlacementNode
      break;
    }
    case HostRoot:
    case HostPortal: {
      const parent: Container = parentFiber.stateNode.containerInfo;
      const before = getHostSibling(finishedWork);
      insertOrAppendPlacementNodeIntoContainer(finishedWork, before, parent);
      break;
    }
    // eslint-disable-next-line-no-fallthrough
    default:
      throw new Error(
        'Invalid host parent fiber. This error is likely caused by a bug ' +
          'in React. Please file an issue.',
      );
  }
}
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
  } else if (tag === HostPortal) {
    // If the insertion itself is a portal, then we don't want to traverse
    // down its children. Instead, we'll get insertions from each child in
    // the portal directly.
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
  }
}
function insertOrAppendPlacementNode(
  node: Fiber,
  before: ?Instance,
  parent: Instance,
): void {
  const {tag} = node;
  const isHost = tag === HostComponent || tag === HostText;
  if (isHost) {
    const stateNode = node.stateNode;
    if (before) {
      insertBefore(parent, stateNode, before);
    } else {
      appendChild(parent, stateNode);
    }
  } else if (tag === HostPortal) {
    // If the insertion itself is a portal, then we don't want to traverse
    // down its children. Instead, we'll get insertions from each child in
    // the portal directly.
  } else {
    const child = node.child;
    if (child !== null) {
      insertOrAppendPlacementNode(child, before, parent);
      let sibling = child.sibling;
      while (sibling !== null) {
        insertOrAppendPlacementNode(sibling, before, parent);
        sibling = sibling.sibling;
      }
    }
  }
}
```

[![](https://jser.dev/static/rerender/30.avif align="left")](https://jser.dev/static/rerender/30.avif)

### 3.4 업데이트는 마지막에 처리됩니다.

업데이트 브랜치는 `commitMutationEffectsOnFiber(`) 안에 있습니다.

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
      commitReconciliationEffects(finishedWork);
      if (flags & Update) {
        // ❗❗ ↖ FunctionComponent일 경우, 이것은 훅을 실행해야 되는 것을 의미합니다.
        try {
          commitHookEffectListUnmount(
            HookInsertion | HookHasEffect,
            finishedWork,
            finishedWork.return,
          );
          commitHookEffectListMount(
            HookInsertion | HookHasEffect,
            finishedWork,
          );
        } catch (error) {
          captureCommitPhaseError(finishedWork, finishedWork.return, error);
        }
        // Layout effects are destroyed during the mutation phase so that all
        // destroy functions for all fibers are called before any create functions.
        // This prevents sibling component effects from interfering with each other,
        // e.g. a destroy function in one component should never override a ref set
        // by a create function in another component during the same commit.
        if (
          enableProfilerTimer &&
          enableProfilerCommitHooks &&
          finishedWork.mode & ProfileMode
        ) {
          try {
            startLayoutEffectTimer();
            commitHookEffectListUnmount(
              HookLayout | HookHasEffect,
              finishedWork,
              finishedWork.return,
            );
          } catch (error) {
            captureCommitPhaseError(finishedWork, finishedWork.return, error);
          }
          recordLayoutEffectDuration(finishedWork);
        } else {
          try {
            commitHookEffectListUnmount(
              HookLayout | HookHasEffect,
              finishedWork,
              finishedWork.return,
            );
          } catch (error) {
            captureCommitPhaseError(finishedWork, finishedWork.return, error);
          }
        }
      }
      return;
    }
    case HostComponent: {
      recursivelyTraverseMutationEffects(root, finishedWork, lanes);
      commitReconciliationEffects(finishedWork);
      if (flags & Ref) {
        if (current !== null) {
          safelyDetachRef(current, current.return);
        }
      }
      if (supportsMutation) {
        // TODO: ContentReset gets cleared by the children during the commit
        // phase. This is a refactor hazard because it means we must read
        // flags the flags after `commitReconciliationEffects` has already run;
        // the order matters. We should refactor so that ContentReset does not
        // rely on mutating the flag during commit. Like by setting a flag
        // during the render phase instead.
        if (finishedWork.flags & ContentReset) {
          const instance: Instance = finishedWork.stateNode;
          try {
            resetTextContent(instance);
          } catch (error) {
            captureCommitPhaseError(finishedWork, finishedWork.return, error);
          }
        }
        if (flags & Update) {
        // ❗❗ ↗ HostComponent의 경우, 이것은 엘리먼트 어트리뷰트가 업데이트 되야 함을 의미합니다.
          const instance: Instance = finishedWork.stateNode;
          if (instance != null) {
            // Commit the work prepared earlier.
            const newProps = finishedWork.memoizedProps;
            // For hydration we reuse the update path but we treat the oldProps
            // as the newProps. The updatePayload will contain the real change in
            // this case.
            const oldProps =
              current !== null ? current.memoizedProps : newProps;
            const type = finishedWork.type;
            // TODO: Type the updateQueue to be specific to host components.
            const updatePayload: null | UpdatePayload = (finishedWork.updateQueue: any);
            finishedWork.updateQueue = null;
            if (updatePayload !== null) {
              try {
                commitUpdate( // ❗❗ commitUpdate
                  instance,
                  updatePayload,
                  type,
                  oldProps,
                  newProps,
                  finishedWork,
                );
              } catch (error) {
                captureCommitPhaseError(
                  finishedWork,
                  finishedWork.return,
                  error,
                );
              }
            }
          }
        }
      }
      return;
    }
    case HostText: {
      recursivelyTraverseMutationEffects(root, finishedWork, lanes);
      commitReconciliationEffects(finishedWork);
      if (flags & Update) { // ❗❗ if (flags & Update)
        if (supportsMutation) {
          if (finishedWork.stateNode === null) {
            throw new Error(
              'This should have a text node initialized. This error is likely ' +
                'caused by a bug in React. Please file an issue.',
            );
          }
          const textInstance: TextInstance = finishedWork.stateNode;
          const newText: string = finishedWork.memoizedProps;
          // For hydration we reuse the update path but we treat the oldProps
          // as the newProps. The updatePayload will contain the real change in
          // this case.
          const oldText: string =
            current !== null ? current.memoizedProps : newText;
          try {
            commitTextUpdate(textInstance, oldText, newText); // ❗❗ commitTextUpdate
          } catch (error) {
            captureCommitPhaseError(finishedWork, finishedWork.return, error);
          }
        }
      }
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
export function commitUpdate(
  domElement: Instance,
  updatePayload: Array<mixed>,
  type: string,
  oldProps: Props,
  newProps: Props,
  internalInstanceHandle: Object,
): void {
  // Apply the diff to the DOM node.
  updateProperties(domElement, updatePayload, type, oldProps, newProps); // ❗❗ updateProperties
  // Update the props handle so that we know which props are the ones with
  // with current event handlers.
  updateFiberProps(domElement, newProps);
}
export function commitTextUpdate(
  textInstance: TextInstance,
  oldText: string,
  newText: string,
): void {
  textInstance.nodeValue = newText;
}
```

데모에서는 트리 구조 때문에 변형(mutation)이 아래 순서대로 처리됩니다.

1. `span`에서 삭제
    
2. HostText 업데이트
    
3. `button`의 업데이트 (후드 아래 비어 있음)
    
4. `b` 삽입
    

## 4\. 요약

휴, 정말 많네요. 리-렌더링 프로세스를 대략적으로 요약하면 다음과 같습니다.

1. 상태가 변경되면 대상 Fiber Node의 경로가 `lanes` 및 `childLanes`으로 표시되어 해당 노드 또는 하위 트리를 리-렌더링해야 하는지 여부를 나타냅니다.
    
2. 불필요한 리-렌더링을 피하기 위해 bailout을 최적화하여 전체 Fiber Tree를 리-렌더링합니다.
    
3. 컴포넌트가 리-렌더링되면 새로운 React 엘리먼트가 생성되고, 그 자식들은 모두 동일하더라도 새로운 프로퍼티를 가지므로 React는 기본적으로 전체 Fiber Tree를 리-렌더링합니다. `useMemo()` 함수가 필요한 이유는 다음과 같습니다.
    
4. "리-렌더링"을 통해 React는 현재 트리에서 새로운 Fiber Tree를 생성하고, 필요한 경우 Fiber Node에 `PlacementChildDeletion` 및 `Update` 플래그를 표시합니다.
    
5. 새 Fiber Tree가 완료되면 React는 위의 플래그로 Fiber Node를 처리하고 "Commit" 단계에서 호스트 DOM에 변경 사항을 적용합니다.
    
6. 그러면 새 Fiber Tree가 현재 Fiber Tree를 가리키게 됩니다. 이전 Fiber Tree의 노드는 다음 렌더링에 재사용 할 수 있습니다.
    

아래 슬라이드에 단계를 정리해 두었으니 도움이 되길 바랍니다.

[JSer의 슬라이드 링크](https://jser.dev/2023-07-18-how-react-rerenders/#how-react-re-renders-internally)