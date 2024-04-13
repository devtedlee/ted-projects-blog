---
title: "[번역] React ErrorBoundary는 어떻게 동작하나요?"
datePublished: Fri Apr 12 2024 15:33:31 GMT+0000 (Coordinated Universal Time)
cuid: cluwtvh5q000508l6dgui2mia
slug: react-internals-deep-dive-6
cover: https://cdn.hashnode.com/res/hashnode/image/upload/v1712935908689/c51c39f2-67d0-4529-9d6a-860fb8529e5f.jpeg
tags: react-internals

---

> 영문 블로그 글을 번역했습니다. 허가를 받으면 시리즈를 이어갈 예정입니다.  
> 원문링크: [https://jser.dev/2023-05-26-how-does-errorboundary-work/](https://jser.dev/2023-05-26-how-does-errorboundary-work/)

---

> ℹ️ [React Internals Deep Dive](https://jser.dev/series/react-source-code-walkthrough.html) 에피소드 6, [유튜브에서 제가 설명하는 것](https://www.youtube.com/watch?v=0TnuJKLjMyg&list=PLvx8w9g4qv_p-OS-XdbB3Ux_6DMXhAJC3&index=6)을 시청해주세요.
> 
> ⚠ [React@18.2.0](https://github.com/facebook/react/releases/tag/v18.2.0) 기준, 최신 버전에서는 구현이 변경되었을 수 있습니다.

> ***💬 역자 주석: Jser의 코멘트는 ❗❗로 표시 해뒀습니다.  
> 그 외 주석은 리액트 소스 코드 자체의 주석입니다.  
> ... 은 생략된 코드입니다.***

## 1\. ErrorBoundary는 리액트 파이버 트리에 대한 선언적 try...catch 입니다.

```typescript
<ErrorBoundary fallback={<p>Something went wrong</p>}>
  <Profile />
</ErrorBoundary>
```

`Profile`이 렌더링 중 에러를 throw하면 ErrorBoundary는 fallback을 렌더링 하도록 시도하는데, [React가 Fiber Tree를 순회하는 방법](https://jser.dev/react/2022/01/16/fiber-traversal-in-react)에서 설명한 대로, 파이버 노드의 렌더링 순서는 아래와 같습니다.

[![](https://jser.dev/static/errorboundary-1.png align="left")](https://jser.dev/static/errorboundary-1.png)

`static getDerivedStateFromError(error)`를 구현한 모든 클래스 컴포넌트는 ErrorBoundary입니다.

## 2\. ErrorBoundary는 내부적으로 어떻게 동작하나요?

### 2.1 에러가 발생하면 가장 가까운 ErrorBoundary에 플래그(`ShouldCapture`)로 표시되고 state 업데이트가 예약됩니다.

```typescript
function renderRootSync(root: FiberRoot, lanes: Lanes) {
  ...
  do {
    try {
      workLoopSync();
      break;
    } catch (thrownValue) {
      handleError(root, thrownValue); // ❗❗ handleError
    }
  } while (true);
  ...
  return workInProgressRootExitStatus;
}
function renderRootConcurrent(root: FiberRoot, lanes: Lanes) {
  ...
  do {
    try {
      workLoopConcurrent();
      break;
    } catch (thrownValue) {
      handleError(root, thrownValue); // ❗❗ handleError
    }
  } while (true);
  ...
  return workInProgressRootExitStatus;
}
```

동기화(Sync) 모드 또는 동시(Concurrent) 모드의 경우 작업 루프에 큰 `try...catch`가 적용되고 `handleError()`가 throw된 값을 처리합니다.

> ℹ 참고로 동기화 모드와 동시 모드의 차이점은 메인 스레드에 yield해야 하는지 여부를 지속적으로 확인하는 `workLoopConcurrent()`에 있습니다. [React 스케줄러의 작동 방식](https://jser.dev/react/2022/03/16/how-react-scheduler-works/)에 설명되어 있습니다.

```typescript
function handleError(root, thrownValue): void {
  do {
    let erroredWork = workInProgress;
    try {
      ...
      throwException( // ❗❗ throwException
        root,
        erroredWork.return,
        erroredWork,
        thrownValue,
        workInProgressRootRenderLanes,
      );
      completeUnitOfWork(erroredWork); // ❗❗ completeUnitOfWork
    } catch (yetAnotherThrownValue) {
      ...
    }
    // Return to the normal work loop.
    return;
  } while (true);
}
```

`handleError()` 는 내부적으로 에러를 던지고 파이버 노드에서 작업을 완료하기 시작합니다. [React는 파이버 트리를 어떻게 순회하나요?](https://jser.dev/react/2022/01/16/fiber-traversal-in-react/) 에서 설명한 것처럼 에러가 발생하면 더 이상 자식으로 내려갈 필요가 없으므로 `complete()` 단계를 시작해야 합니다.

```typescript
function throwException(
  root: FiberRoot,
  returnFiber: Fiber,
  sourceFiber: Fiber,
  value: mixed,
  rootRenderLanes: Lanes,
): void {
  // The source fiber did not complete.
  sourceFiber.flags |= Incomplete;
  // ❗❗                ↗ 여기 InComplete 플래그는 completeUnitOfWork()에게 중요합니다.
  if (
    value !== null &&
    typeof value === 'object' &&
    typeof value.then === 'function'
  ) {
    // ❗❗ 이 브랜치는 Suspense를 위한 것입니다.
    ...
  } else {
    // ❗❗ 이 브랜치는 일반적인 에러 바운더리를 위한 것입니다.
    ...
  value = createCapturedValueAtFiber(value, sourceFiber);
  renderDidError(value);
  let workInProgress: Fiber = returnFiber;
  do {
  // ↖ 이 do...while 루프는 루트 경로를 따라서 가장 가까운 에러 바운더리를 찾습니다.
    switch (workInProgress.tag) {
      case HostRoot: {
        ...
      }
      case ClassComponent: // ❗❗ ClassComponent
        // Capture and retry
        const errorInfo = value;
        const ctor = workInProgress.type;
        const instance = workInProgress.stateNode;
        if (
          (workInProgress.flags & DidCapture) === NoFlags &&
          (typeof ctor.getDerivedStateFromError === 'function' ||
                        // ❗❗ ↖ 우린 바운더리를 찾았습니다!
            (instance !== null &&
              typeof instance.componentDidCatch === 'function' &&
              !isAlreadyFailedLegacyErrorBoundary(instance)))
        ) {
          workInProgress.flags |= ShouldCapture;
                            // ↗ ❗❗ ShouldCapture 는 곧 DidCapture로 바뀔 예정입니다.
          const lane = pickArbitraryLane(rootRenderLanes);
          workInProgress.lanes = mergeLanes(workInProgress.lanes, lane);
          // Schedule the error boundary to re-render using updated state
          const update = createClassErrorUpdate(
                // ❗❗    ↗ 이 에러 바운더리는 지금 당장 처리되야 하므로 업데이트를 예약합니다.
            workInProgress,
            errorInfo,
            lane,
          );
          enqueueCapturedUpdate(workInProgress, update); // ❗❗ enqueueCapturedUpdate(workInProgress, update);
          return;
        }
        break;
      default:
        break;
    }
    // $FlowFixMe[incompatible-type] we bail out when we get a null
    workInProgress = workInProgress.return;
  } while (workInProgress !== null);
}
```

업데이트 내부에서 실제로 `getDerivedStateFromError()`가 호출되는 곳입니다.

```typescript
function createClassErrorUpdate(
  fiber: Fiber,
  errorInfo: CapturedValue<mixed>,
  lane: Lane,
): Update<mixed> {
  const update = createUpdate(lane);
  update.tag = CaptureUpdate;
  const getDerivedStateFromError = fiber.type.getDerivedStateFromError;
  if (typeof getDerivedStateFromError === 'function') {
    const error = errorInfo.value;
    update.payload = () => {
      return getDerivedStateFromError(error); // ❗❗ getDerivedStateFromError
    };
    update.callback = () => {
      logCapturedError(fiber, errorInfo);
    };
  }
```

그리고 `enqueueCapturedUpdate()`는 업데이트를 오류가 발생한 파이버의 `updateQueue`로 설정한 다음, `mountClassInstance()`와 `updateClassInstance()` 내부의 `processUpdateQueue()`를 통해 다음 렌더링에 처리되도록 합니다.

이는 Class Components에 대한 세부 사항으로 오래된 내용이므로 여기서는 자세히 다루지 않고, 이 이후에는 가장 가까운 에러 바운더리가 다음 렌더링에 오류를 반영하는 새로운 state를 갖게 된다는 점만 기억하세요.

### 2.2 언와인딩 중 `ShouldCapture`플래그는 `DidCapture`플래그로 바뀝니다.

[React에서 컨텍스트는 어떻게 작동하나요?](https://jser.dev/react/2021/07/28/how-does-context-work/) 에서 React 런타임은 경로를 따라 많은 정보를 저장하므로 각 파이버에 대한 정리 작업이 엉망이 되지 않도록 하는 것이 중요하다고 언급했습니다.

`throwException()` 에서 가장 가까운 ErrorBoundary를 찾았지만 아직 파이버 트리의 **current 커서**를 그 경계로 이동하지 않았습니다. 가장 가까운 에러 바운더리로 거슬러 올라가서 거기서부터 다시 렌더링을 시도하는 것이 *언와인딩* 과정입니다.

그리고 *언와인딩*은 이 글의 앞 부분에서 언급했던 `completeUnitOfWork()` 내부에서 이루어집니다.

> ℹ complete() 단계의 작업에 대한 자세한 내용은 [React는 파이버 트리를 어떻게 순회 할까요?](https://jser.dev/react/2022/01/16/fiber-traversal-in-react/) 를 참고하세요.

```typescript
function completeUnitOfWork(unitOfWork: Fiber): void {
  let completedWork = unitOfWork;
  do {
    // The current, flushed, state of this fiber is the alternate. Ideally
    // nothing should rely on this, but relying on it here means that we don't
    // need an additional field on the work in progress.
    const current = completedWork.alternate;
    const returnFiber = completedWork.return;
    // Check if the work completed or if something threw.
    if ((completedWork.flags & Incomplete) === NoFlags) {
        // ❗❗                 ↗ 여기서 Incomplete 플래그가 중요한 이유를 확인하세요.
        // ❗❗                    아래 브랜치는 일반 브랜치입니다.
      let next;
      if (
        !enableProfilerTimer ||
        (completedWork.mode & ProfileMode) === NoMode
      ) {
        next = completeWork(current, completedWork, subtreeRenderLanes);
      } else {
        startProfilerTimer(completedWork);
        next = completeWork(current, completedWork, subtreeRenderLanes);
        // Update render duration assuming we didn't error.
        stopProfilerTimerIfRunningAndRecordDelta(completedWork, false);
      }
      resetCurrentDebugFiberInDEV();
      if (next !== null) {
        // Completing this fiber spawned new work. Work on that next.
        workInProgress = next;
        return;
      }
    } else {
    // ❗❗ ↖ 여기가 우리가 언와인드를 해야 할 Incomplete 브랜치입니다
      // This fiber did not complete because something threw. Pop values off
      // the stack without entering the complete phase. If this is a boundary,
      // capture values if possible.
      const next = unwindWork(current, completedWork, subtreeRenderLanes);
      // Because this fiber did not complete, don't reset its lanes.
      if (next !== null) {
    // ❗❗ ↗ 이 브랜치는 completeWork()가 파이버 노드를 반환하면,
    // ❗❗ React가 부모에서 completeWork()를 계속하는 대신 해당 노드에서 리-렌더링하는 것을 보여줍니다.
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
      // ❗❗ ↖ InComplete 파이버의 모든 조상 노드들은 모두 InComplete 합니다.
      // ❗❗ 이렇게 하면 가장 가까운 ErrorBoundary가 IncComplete 인지 확인하여,
      // ❗❗ 바운더리에 대한 unwindWork()가 호출됩니다.
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

그리고 `unwindWork()`에서는, `DidCapture`플래그가 플레이됩니다.

```typescript
function unwindWork(
  current: Fiber | null,
  workInProgress: Fiber,
  renderLanes: Lanes,
) {
  // Note: This intentionally doesn't check if we're hydrating because comparing
  // to the current tree provider fiber is just as fast and less error-prone.
  // Ideally we would have a special version of the work loop only
  // for hydration.
  popTreeContext(workInProgress);
  switch (workInProgress.tag) {
    case ClassComponent: {
      const Component = workInProgress.type;
      if (isLegacyContextProvider(Component)) {
        popLegacyContext(workInProgress);
      }
      const flags = workInProgress.flags;
      if (flags & ShouldCapture) {
        workInProgress.flags = (flags & ~ShouldCapture) | DidCapture;
        // ❗❗ ↖ ShouldCapture 가 DidCapture로 바뀌는 것을 보세요.
        return workInProgress;
        // ❗❗ 앞서 설명했듯이, 이 리턴은 가장 가까운 바운더리인 이 파이버에서
        // ❗❗ 리-렌더링을 한다는 의미입니다.
      }
      return null;
    }
    ...
  }
}
```

### 2.3 DidCapture 플래그는 ErrorBoundary에서 새 자식으로 리-렌더링 하도록 강제합니다.

```typescript
function finishClassComponent(
  current: Fiber | null,
  workInProgress: Fiber,
  Component: any,
  shouldUpdate: boolean,
  hasContext: boolean,
  renderLanes: Lanes,
) {
  const didCaptureError = (workInProgress.flags & DidCapture) !== NoFlags; 
                                                 // ❗❗ ↖
  ...
  const instance = workInProgress.stateNode;
  // Rerender
  ReactCurrentOwner.current = workInProgress;
  let nextChildren;
  if (
    didCaptureError &&
    typeof Component.getDerivedStateFromError !== 'function'
  ) {
    // If we captured an error, but getDerivedStateFromError is not defined,
    // unmount all the children. componentDidCatch will schedule an update to
    // re-render a fallback. This is temporary until we migrate everyone to
    // the new API.
    // TODO: Warn in a future release.
    nextChildren = null;
  } else {
    nextChildren = instance.render();
    // ❗❗ 이 때, 에러가 throw되면 예약된 업데이트가 생성되기 때문에 
    // ❗❗ render()가 다른 자식을 반환합니다.
  }
  // React DevTools reads this flag.
  workInProgress.flags |= PerformedWork;
  if (current !== null && didCaptureError) {
    // 오류에서 복구하는 경우 기존 자식을 재사용하지 않고 재조정(reconcile)합니다.
    // 개념적으로 정상 자식과 오류로 표시되는 자식은 서로 다른 두 세트이므로
    // 아이덴티티(identities)가 일치하더라도 일반 자식을 재사용해서는 안 됩니다.
    forceUnmountCurrentAndReconcile(
    // ❗❗ 이 이름과 위에 달린 주석들이 모든 것을 설명해줍니다.(💬 주석 번역함)
      current,
      workInProgress,
      nextChildren,
      renderLanes,
    );
  } else {
    reconcileChildren(current, workInProgress, nextChildren, renderLanes);
  }
  return workInProgress.child;
    // ❗❗             ↗ 자식에게 더 깊이 들어가서 조정(reconciling)을 계속합니다.
}
```

코드는 매우 간단합니다.

## 3\. 요약

이제 몇 가지 다이어그램을 통해 배운 내용을 요약해 보겠습니다.

[![](https://cdn.hashnode.com/res/hashnode/image/upload/v1712935804086/cdf9a395-3f08-4f7c-b253-71654a2b463b.png align="center")](https://jser.dev/2023-05-26-how-does-errorboundary-work/#how-errorboundary-works-internally)

> 💬 위 이미지를 클릭하세요

## 4\. 코딩 챌린지

오늘 배운 내용을 강화하기 위해 다음 코딩 퀴즈를 풀어보세요.

[![](https://cdn.hashnode.com/res/hashnode/image/upload/v1712935856251/57768265-10da-4193-a450-a36f87304645.png align="center")](https://jser.dev/2023-05-26-how-does-errorboundary-work/#4-coding-challenge)

💬 위 이미지를 클릭하세요

(원본 게시일: 2023-05-26)