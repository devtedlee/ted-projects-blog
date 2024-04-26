---
title: "[번역] React 에서 useLayoutEffect()는 어떻게 동작하나요?"
datePublished: Fri Apr 26 2024 07:37:42 GMT+0000 (Coordinated Universal Time)
cuid: clvgd1i5700050al918ithulc
slug: react-internals-deep-dive-10
cover: https://cdn.hashnode.com/res/hashnode/image/upload/v1714116987248/b6ca7122-dad8-42aa-813a-562f6996ba02.jpeg
tags: reactjs, react-internals, uselayouteffect

---

> ***영문 블로그 글을 번역했습니다. 허가를 받으면 시리즈를 이어갈 예정입니다.  
> 원문링크:***[https://jser.dev/react/2021/12/04/how-does-useLayoutEffect-work](https://jser.dev/react/2021/12/04/how-does-useLayoutEffect-work)

---

> ***ℹ️***[***React Internals Deep Dive***](https://jser.dev/series/react-source-code-walkthrough.html) ***에피소드 10,*** [***유튜브에서 제가 설명하는 것***](https://www.youtube.com/watch?v=6HLvyiYv7HI&list=PLvx8w9g4qv_p-OS-XdbB3Ux_6DMXhAJC3&index=10)***을 시청해주세요.***
> 
> ***⚠***[***React@18.2.0***](https://github.com/facebook/react/releases/tag/v18.2.0)***기준, 최신 버전에서는 구현이 변경되었을 수 있습니다.***
> 
> ***💬 역자 주석: Jser의 코멘트는 ❗❗로 표시 해뒀습니다.  
> 그 외 주석은 리액트 소스 코드 자체의 주석입니다.  
> ... 은 생략된 코드입니다.***

`useLayoutEffect()`는 DOM이 업데이트된 직후 일부 코드를 동기적으로 실행할 수 있는 기회를 제공합니다. 어떻게 작동하는지 알아봅시다.

대부분의 훅의 경우 내부적으로 `mountXXX()` 와 `updateXXX()`의 두 가지 구현이 있습니다.

## 1\. 레이아웃 이펙트는 어떻게 마운트 되나요?

```typescript
function mountLayoutEffect(
  create: () => (() => void) | void,
  deps: Array<mixed> | void | null
): void {
  let fiberFlags: Flags = UpdateEffect;
  if (enableSuspenseLayoutEffectSemantics) {
    fiberFlags |= LayoutStaticEffect;
  }
  return mountEffectImpl(fiberFlags, HookLayout, create, deps);
}
function updateLayoutEffect(
  create: () => (() => void) | void,
  deps: Array<mixed> | void | null
): void {
  return updateEffectImpl(UpdateEffect, HookLayout, create, deps);
}
```

`useEffect()`에서 동일하게 사용하는 함수인 `mountEffectImpl()`과 `updateEffectImpl()`이 차례로 사용되며, 다른 점은 두 번째 인수인 `hookFlags`입니다.

```typescript
function mountEffectImpl(fiberFlags, hookFlags, create, deps): void {
  const hook = mountWorkInProgressHook();
  const nextDeps = deps === undefined ? null : deps;
  currentlyRenderingFiber.flags |= fiberFlags;
  hook.memoizedState = pushEffect(
    HookHasEffect | hookFlags,
    create,
    undefined,
    nextDeps
  );
}
function mountWorkInProgressHook(): Hook {
  const hook: Hook = {
    memoizedState: null,
    baseState: null,
    baseQueue: null,
    queue: null,
    next: null,
  };
  if (workInProgressHook === null) {
    currentlyRenderingFiber.memoizedState = workInProgressHook = hook;
    // ❗❗ ↖ 이게 목록에 있는 첫 훅입니다
  } else {
    workInProgressHook = workInProgressHook.next = hook;
    // ❗❗ ↖ 목록의 가장 마지막에 삽입합니다.
  }
  return workInProgressHook;
}
```

`mountWorkInProgressHook()` 은 새로운 빈 훅을 생성하고 이 파이버의 훅 목록에 추가합니다. `pushEffect()`는 파이버의 `updateQueue`에 effect 객체를 생성하며, 이들은 `memoizedState`로 연결됩니다.

```typescript
function pushEffect(tag, create, destroy, deps) {
  const effect: Effect = {
    tag,
    create,
    destroy,
    deps,
    // Circular
    next: (null: any),
  };
  let componentUpdateQueue: null | FunctionComponentUpdateQueue =
    (currentlyRenderingFiber.updateQueue: any);
  if (componentUpdateQueue === null) {
    componentUpdateQueue = createFunctionComponentUpdateQueue();
    currentlyRenderingFiber.updateQueue = (componentUpdateQueue: any);
    componentUpdateQueue.lastEffect = effect.next = effect;
  } else {
    const lastEffect = componentUpdateQueue.lastEffect;
    if (lastEffect === null) {
      componentUpdateQueue.lastEffect = effect.next = effect;
    } else {
      const firstEffect = lastEffect.next;
      lastEffect.next = effect;
      effect.next = firstEffect;
      componentUpdateQueue.lastEffect = effect;
    }
  }
  return effect;
}
```

모든 훅이 이펙트 훅인 것은 아닙니다. 예를 들어 `useState()` 는 그렇지 않습니다. Effect는 실행되어야 하는 side effect를 의미하며, 다른 위치에 있을 수 없는 이유가 보이지 않기 때문에 `updateQueue`에 배치된 것 같습니다.

`Effect`의 `tag` 속성이 중요한데, 레이아웃 이펙트를 마운트하는 경우는 `HookHasEffect | HookLayout`입니다.

마운팅은 이것으로 끝이며, `updateQueue`를 설정하고 새 `hook`을 생성합니다.

## 2\. 레이아웃 이펙트는 어떻게 업데이트 되나요?

`updateLayoutEffect()` 도 이와 비슷할 겁니다.

```typescript
function updateEffectImpl(fiberFlags, hookFlags, create, deps): void {
  const hook = updateWorkInProgressHook();
  const nextDeps = deps === undefined ? null : deps;
  let destroy = undefined;
  if (currentHook !== null) {
    const prevEffect = currentHook.memoizedState;
    destroy = prevEffect.destroy;
    if (nextDeps !== null) {
      const prevDeps = prevEffect.deps;
      if (areHookInputsEqual(nextDeps, prevDeps)) {
        hook.memoizedState = pushEffect(hookFlags, create, destroy, nextDeps);
        return;
      }
    }
  }
  currentlyRenderingFiber.flags |= fiberFlags;
  hook.memoizedState = pushEffect(
    HookHasEffect | hookFlags, // ❗❗ HookHasEffect 
    create,
    destroy,
    nextDeps
  );
}
```

함수는 새로운 파이버 트리가 생성되는 조정(reconciliation) 단계에서 실행됩니다. `current`가 존재하는 항목의 접두사인 경우, `updateWorkInProgressHook()` 은 `currentWork`를 앞으로 이동하고 생성 중인 버전도 반환합니다.

props가 변경되었는지 여부에 따라 `areHookInputsEqual(nextDeps, prevDeps)`로 변경 감지가 이루어지고, `memoizedState`는 이펙트로 설정되지만 플래그가 다르며, 변경이 없는 경우 `HookHasEffect`가 추가되지 않는 것을 볼 수 있습니다.

따라서 `HookHasEffect`는 이펙트를 실행해야 하는지 여부를 나타내는 중요한 요소입니다.

## 3\. 레이아웃 이펙트는 실제로 언제 실행될까요?

좋은 질문입니다. `useEffect()`에 의한 패시브 이펙트나 레이아웃 이펙트의 경우, 전달된 함수는 실제로 생성 함수이고, 생성 함수가 반환하는 클로저는 파괴(정리) 함수입니다.

```typescript
const effect: Effect = {
  tag,
  create,
  destroy,
  deps,
  // Circular
  next: (null: any),
};
```

위는 `Effect`의 구조로, `create`와 `destory`를 볼 수 있습니다.

이펙트는 DOM 변이 후에 실행되므로, 커밋 단계에 있어야 하며, 아래가 시작되는 부분에 대한 내용입니다.

```typescript
function commitRootImpl(
  root: FiberRoot,
  recoverableErrors: null | Array<mixed>,
  transitions: Array<Transition> | null,
  renderPriorityLevel: EventPriority,
) {
    ....
    // The next phase is the mutation phase, where we mutate the host tree.
    commitMutationEffects(root, finishedWork, lanes);
    commitLayoutEffects(finishedWork, root, lanes); // ❗❗
    ...
}
```

`commitLayoutEffects()`는 보류 중인 모든 레이아웃 이펙트를 실행하려고 시도합니다. 이 역시 일종의 트리를 순회하는 것이므로 [동일한 순회 알고리즘](https://jser.dev/react/2022/01/16/fiber-traversal-in-react)이 사용되며, 완료 단계를 살펴봅시다.

```typescript
function commitLayoutMountEffects_complete(
  subtreeRoot: Fiber,
  root: FiberRoot,
  committedLanes: Lanes
) {
  while (nextEffect !== null) {
    const fiber = nextEffect;
    if ((fiber.flags & LayoutMask) !== NoFlags) {
      const current = fiber.alternate;
      setCurrentDebugFiberInDEV(fiber);
      try {
        commitLayoutEffectOnFiber(root, current, fiber, committedLanes); // ❗❗
      } catch (error) {
        captureCommitPhaseError(fiber, fiber.return, error);
      }
      resetCurrentDebugFiberInDEV();
    }
    if (fiber === subtreeRoot) {
      nextEffect = null;
      return;
    }
    const sibling = fiber.sibling;
    if (sibling !== null) {
      sibling.return = fiber.return;
      nextEffect = sibling;
      return;
    }
    nextEffect = fiber.return;
  }
}
```

기본적으로 이 파이버에 레이아웃 효과를 실행할 필요가 있는지 `commitLayoutEffectOnFiber()`를 통해 확인한 후, 형제 파이버 또는 부모 파이버로 이동합니다.

```typescript
function commitLayoutEffectOnFiber(
  finishedRoot: FiberRoot,
  current: Fiber | null,
  finishedWork: Fiber,
  committedLanes: Lanes,
): void {
  if ((finishedWork.flags & LayoutMask) !== NoFlags) {
    switch (finishedWork.tag) {
      case FunctionComponent:
      case ForwardRef:
      case SimpleMemoComponent: {
        if (
          !enableSuspenseLayoutEffectSemantics ||
          !offscreenSubtreeWasHidden
        ) {
          // At this point layout effects have already been destroyed (during mutation phase).
          // This is done to prevent sibling component effects from interfering with each other,
          // e.g. a destroy function in one component should never override a ref set
          // by a create function in another component during the same commit.
          if (
            enableProfilerTimer &&
            enableProfilerCommitHooks &&
            finishedWork.mode & ProfileMode
          ) {
            try {
              startLayoutEffectTimer();
              commitHookEffectListMount( // ❗❗
                HookLayout | HookHasEffect,
                finishedWork,
              );
            } finally {
              recordLayoutEffectDuration(finishedWork);
            }
          } else {
            commitHookEffectListMount(HookLayout | HookHasEffect, finishedWork); // ❗❗ 
          }
        }
        break;
      }
  ...
```

`commitHookEffectListMount(HookLayout | HookHasEffect, finishedWork)`를 찾을 수 있습니다. 이 중에

1. `HookLayout` -&gt; 이 이펙트는 레이아웃 이펙트입니다.
    
2. `HookHasEffect` -&gt; 이 이펙트를 실행해야 합니다.
    

```typescript
function commitHookEffectListMount(flags: HookFlags, finishedWork: Fiber) {
  const updateQueue: FunctionComponentUpdateQueue | null =
    (finishedWork.updateQueue: any);
  const lastEffect = updateQueue !== null ? updateQueue.lastEffect : null;
  if (lastEffect !== null) {
    const firstEffect = lastEffect.next;
    let effect = firstEffect;
    do {
      if ((effect.tag & flags) === flags) {
        // Mount
        const create = effect.create;
        effect.destroy = create();
      }
      effect = effect.next;
    } while (effect !== firstEffect);
  }
}
```

`commitHookEffectListMount()`는 간단합니다. 이펙트 태그를 확인하고 `create()` 함수를 실행하고 `destroy`를 설정합니다.

## 4\. 레이아웃 이펙트는 언제 정리되나요?

정리 함수 `destory`가 어떻게 설정되는지 살펴봤는데, 언제 실행될까요? 실제로는 그 이전인`commitMutationEffects()`에서 발생합니다.

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
  // to reconcilation, because those can be set on all fiber types.
  switch (finishedWork.tag) {
    case FunctionComponent:
    case ForwardRef:
    case MemoComponent:
    case SimpleMemoComponent: {
      recursivelyTraverseMutationEffects(root, finishedWork, lanes);
      // ❗❗ ↖ 이 호출에 주목하세요, 이것은 중요합니다
      commitReconciliationEffects(finishedWork);
      if (flags & Update) {
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
            commitHookEffectListUnmount( // ❗❗
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
            commitHookEffectListUnmount( // ❗❗
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
    ...
```

중간에 있는 주석을 참조하세요. `commitHookEffectListUnmount()`는 정리를 실행하는 역할을 합니다.

> 💬 주석 번역  
> 레이아웃 효과는 변이(mutation) 단계에서 소멸되므로 모든 파이버에 대한 모든 소멸 함수가 생성 함수보다 먼저 호출됩니다.  
> 이렇게 하면 형제 컴포넌트 이펙트가 서로 간섭하는 것을 방지할 수 있습니다. 예를 들어 한 컴포넌트의 destroy 함수가 동일한 커밋 중에 다른 컴포넌트의 create 함수가 설정한 참조를 재정의해서는 안 됩니다.

```typescript
function commitHookEffectListUnmount(
  flags: HookFlags,
  finishedWork: Fiber,
  nearestMountedAncestor: Fiber | null
) {
  const updateQueue: FunctionComponentUpdateQueue | null =
    (finishedWork.updateQueue: any);
  const lastEffect = updateQueue !== null ? updateQueue.lastEffect : null;
  if (lastEffect !== null) {
    const firstEffect = lastEffect.next;
    let effect = firstEffect;
    do {
      if ((effect.tag & flags) === flags) {
        // Unmount
        const destroy = effect.destroy;
        effect.destroy = undefined;
        if (destroy !== undefined) {
          safelyCallDestroy(finishedWork, nearestMountedAncestor, destroy);
        }
      }
      effect = effect.next;
    } while (effect !== firstEffect);
  }
}
```

`commitHookEffectListUnmount()`도 간단합니다. 이펙트에 대한 `destroy` 함수를 가져온 다음 실행하기만 하면 됩니다.

deps가 변경되면 이 이펙트는 이펙트의 `HookHasEffect`와 함께 설정된다는 것을 기억하세요.

## 5\. 하지만 컴포넌트가 언마운트되면 어떻게 정리가 실행되나요?

좋은 질문입니다. 정리를 트리거하는 deps 변경 외에도 구성 요소 언마운트도 정리로 이어지며, 파이버 세계에서는 파이버 제거를 의미합니다.

위의 코드 조각에서 `recursivelyTraverseMutationEffects()`가 있는데, 실제로 파이버 삭제의 경우 이펙트가 거기서 정리됩니다.

```typescript
function recursivelyTraverseMutationEffects(
  root: FiberRoot,
  parentFiber: Fiber,
  lanes: Lanes
) {
  // Deletions effects can be scheduled on any fiber type. They need to happen
  // before the children effects hae fired.
  const deletions = parentFiber.deletions;
    // ❗❗                      ↗ 삭제된 파이버는 부모 파이버에 저장됩니다.
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
  const prevDebugFiber = getCurrentDebugFiberInDEV();
  if (parentFiber.subtreeFlags & MutationMask) {
    let child = parentFiber.child;
    while (child !== null) {
      setCurrentDebugFiberInDEV(child);
      commitMutationEffectsOnFiber(child, root, lanes);
      child = child.sibling;
    }
  }
  setCurrentDebugFiberInDEV(prevDebugFiber);
}
```

`parentFiber.deletions`의 모든 하위 파이버를 탐색하고 정리 함수를 실행하는 것을 볼 수 있습니다.

삭제된 파이버는 최종 파이버 트리에 포함되지 않으므로, 참조를 위해 부모 파이버에 넣습니다.

```typescript
function deleteChild(returnFiber: Fiber, childToDelete: Fiber): void {
  if (!shouldTrackSideEffects) {
    // Noop.
    return;
  }
  const deletions = returnFiber.deletions;
  if (deletions === null) {
    returnFiber.deletions = [childToDelete]; // ❗❗
    returnFiber.flags |= ChildDeletion;
  } else {
    deletions.push(childToDelete); // ❗❗
  }
}
```

그리고 커밋하기 전에 조정하는 동안 `deletions`가 설정됩니다.

(원본 게시일: 2021-12-04)