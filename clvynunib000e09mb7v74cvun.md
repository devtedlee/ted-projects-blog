---
title: "[번역] React 이펙트 훅의 수명주기"
datePublished: Thu May 09 2024 03:00:09 GMT+0000 (Coordinated Universal Time)
cuid: clvynunib000e09mb7v74cvun
slug: react-internals-deep-dive-16
cover: https://cdn.hashnode.com/res/hashnode/image/upload/v1715176895095/8c072980-fa62-4e65-9158-95965b5facc3.jpeg
tags: react-internals, useeffect-hook, lifecycle-in-react

---

> ***영문 블로그 글을 번역했습니다. 허가를 받으면 시리즈를 이어갈 예정입니다.  
> 원문링크:***[https://jser.dev/react/2022/01/19/lifecycle-of-effect-hook](https://jser.dev/react/2022/01/19/lifecycle-of-effect-hook)

---

> ***ℹ️***[***React Internals Deep Dive***](https://jser.dev/series/react-source-code-walkthrough.html)***에피소드 16,***[***유튜브에서 제가 설명하는 것***](https://www.youtube.com/watch?v=-0-pCZvvwaM&list=PLvx8w9g4qv_p-OS-XdbB3Ux_6DMXhAJC3&index=16)***을 시청해주세요.***
> 
> ***⚠***[***React@18.2.0***](https://github.com/facebook/react/releases/tag/v18.2.0)***기준, 최신 버전에서는 구현이 변경되었을 수 있습니다.***
> 
> ***💬 역자 주석: JSer의 코멘트는 ❗❗로 표시 해뒀습니다.  
> 그 외 주석은 리액트 소스 코드 자체의 주석입니다.  
> ... 은 생략된 코드입니다.***

[이 비디오](https://www.youtube.com/watch?v=Ggmdo7TORNc)에서 `useEffect()`가 어떻게 작동하는지에 대해 이야기했지만 내용이 약간 지저분했습니다. 또한 제가 사용한 React 버전이 최신 버전이 아니었고 그 이후로 상황이 바뀌었으므로 Effect 훅의 라이프사이클에 대해 다시 설명하겠습니다.

"Effect hook"이란 아래와 같이 "useEffect()"를 의미합니다.

```typescript
function A() {
  useEffect(function create() {
    console.log("create effect");
    return function cleanup() {
      console.log("destroy effect");
    };
  }, []);
  return <div />;
}
```

저는 함수를 구분하기 위해 `create()`와 `cleanup()`이라는 이름을 붙였습니다.

이 세 가지 질문에 답해 보겠습니다.

1. `useEffect()` 가 처음 호출되면 어떤 일이 발생하나요?
    
2. `useEffect()`의 `deps`가 변경되면 어떻게 되나요?
    
3. `cleanup()`은 언제 호출되나요?
    

## `useEffect()`가 처음 호출되면 어떤 일이 일어날까요?

`useEffect()`는 Effect 훅을 만들기 위한 방법이며, 훅은 파이버에 첨부(attach)된 무언가를 의미합니다.

[소스 코드](https://github.com/facebook/react/blob/e62a8d754548a490c2a3bcff3b420e5eedaf11c0/packages/react/src/ReactHooks.js#L101)를 보면 `useEffect()`가 첫 번째 호출에서는 [mountEffect](https://github.com/facebook/react/blob/e62a8d754548a490c2a3bcff3b420e5eedaf11c0/packages/react-reconciler/src/ReactFiberHooks.old.js#L1663)를 해결하고, 이후 업데이트에서는 [updateEffect](https://github.com/facebook/react/blob/e62a8d754548a490c2a3bcff3b420e5eedaf11c0/packages/react-reconciler/src/ReactFiberHooks.old.js#L1675)로 해결되는 것을 알 수 있습니다.

`mountEffect()`에 무엇이 있는지 살펴봅시다.

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
    // This is the first hook in the list
    currentlyRenderingFiber.memoizedState = workInProgressHook = hook;
  } else {
    // Append to the end of the list
    workInProgressHook = workInProgressHook.next = hook;
  }
  return workInProgressHook;
}
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

복잡해 보이지만 실제로는 매우 간단하며 기본적으로 두 가지 작업을 수행합니다:

1. `mountWorkInProgressHook()`에 새 훅을 생성하여 파이버의 훅 목록(`memoizedState`)에 첨부합니다.
    
2. 크리에이터 함수로 업데이트 이펙트를 설정하고, 파이버의 `updateQueue`에 연결하고, `memoizedState`를 통해 훅에서 이펙트를 추적합니다. 크리에이터 함수는 아직 호출(invoke)되지 않았습니다.
    

따라서 파이버는 가질 수 있습니다:

1. `updateQueue`에 나열 해 놓은, 업데이트(이펙트)들.
    
2. 이펙트 훅을 위한 `memoizedState`의 훅 목록, 이것들은 이펙트도 추적합니다.
    

> ℹ 훅은 파이버의 내부 링크된 states와 같다고 생각할 수 있으므로, `memoizedState`가 사용됩니다. 또한 `useEffect()` 는 `updateQueue`에 이펙트를 푸시하므로 Effect 훅이라고 합니다.

### Effect.tag는 이펙트 실행 여부를 표시합니다

`Effect는` 부작용(side effect)을 의미하며, 파이버에 `updateQueue`를 넣으면 React가 변경 사항을 커밋한 후에 실행됩니다.

`pushEffect`의 첫 번째 인수가 보이시죠? `Effect.tag`를 제어하기 위한 것으로, 마운팅 단계에서는 `HookHasEffect | hookFlags`와 함께 전달되며, **여기서**`HookHasEffect`**는 실행되어야 함을 의미합니다**.

이것은 매우 중요한데, `updateEffect`에서 이 플래그가 `deps`가 변경되었는지 확인하여 토글될 것임을 예상할 수 있습니다.

## flushPassiveEffects()

이전 동영상에서 설명한 것처럼 [flushPassiveEffects()](https://github.com/facebook/react/blob/e62a8d754548a490c2a3bcff3b420e5eedaf11c0/packages/react-reconciler/src/ReactFiberWorkLoop.old.js#L2353)는 `useEffect()`로 생성된 이펙트를 실행하는 함수입니다.

이 함수는 여러 곳에서 호출(invoke)되지만 가장 중요한 호출은 조정(reconciliation) 후 커밋 단계인 [commitRoot()](https://github.com/facebook/react/blob/e62a8d754548a490c2a3bcff3b420e5eedaf11c0/packages/react-reconciler/src/ReactFiberWorkLoop.old.js#L1957)에서 호출될 때입니다.

```typescript
if (
  (finishedWork.subtreeFlags & PassiveMask) !== NoFlags ||
  (finishedWork.flags & PassiveMask) !== NoFlags
) {
  if (!rootDoesHavePassiveEffects) {
    rootDoesHavePassiveEffects = true;
    pendingPassiveEffectsRemainingLanes = remainingLanes;
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

`flushPassiveEffects()` 는 `scheduleCallback`에 의해 스케줄링되므로, DOM이 변경된 직후에 동기적으로 실행되는 것이 아니라 다음 틱에 실행됩니다. 오늘은 스케줄러가 주제가 아니므로 `flushPassiveEffects()`에 대해 자세히 살펴보겠습니다.

> ℹ 약간 다른 이야기인 [useLayoutEffect()](https://www.youtube.com/watch?v=E7dZM6ZndfA)에 대한 또 다른 동영상이 있습니다.

[소스 코드를보면 기본적](https://github.com/facebook/react/blob/e62a8d754548a490c2a3bcff3b420e5eedaf11c0/packages/react-reconciler/src/ReactFiberWorkLoop.old.js#L2435-L2436)으로 id가 두 가지 작업을 수행한다는 것을 알 수 있습니다.

```typescript
commitPassiveUnmountEffects(root.current);
commitPassiveMountEffects(root, root.current);
```

이펙트의 정리는 다시 실행하기 전에 먼저 실행해야 하므로 `unmount`가 `mount`보다 먼저 발생합니다.

이 2가지 함수는 루트에서 모든 파이버에 미치는 영향을 확인하며, 알고리즘은 [이전 게시물에서 다룬 내용](https://ted-projects.com/react-internals-deep-dive-15)입니다.

트리를 반복해서 순회하는 것은 효율적이지 않다고 생각할 수 있습니다. 맞습니다. 그렇기 때문에 React는 `finishedWork.subtreeFlagsfinishedWork.flags` 등으로 불필요한 검사를 피하는 최적화가 있습니다.

## commitPassiveUnmountEffects()

[지난 글](https://ted-projects.com/react-internals-deep-dive-15)의 알고리즘에서 설명했듯이 `commitPassiveUnmountEffects()`에는 `begin()` 및 `complete()` 함수가 포함됩니다.

```typescript
function commitPassiveUnmountEffects_begin() {
  while (nextEffect !== null) {
    const fiber = nextEffect;
    const child = fiber.child;
    if ((nextEffect.flags & ChildDeletion) !== NoFlags) {
      const deletions = fiber.deletions;
      if (deletions !== null) {
        for (let i = 0; i < deletions.length; i++) {
          const fiberToDelete = deletions[i];
          nextEffect = fiberToDelete;
          commitPassiveUnmountEffectsInsideOfDeletedTree_begin(
            fiberToDelete,
            fiber
          );
        }
        if (deletedTreeCleanUpLevel >= 1) {
          // A fiber was deleted from this parent fiber, but it's still part of
          // the previous (alternate) parent fiber's list of children. Because
          // children are a linked list, an earlier sibling that's still alive
          // will be connected to the deleted fiber via its `alternate`:
          //
          //   live fiber
          //   --alternate--> previous live fiber
          //   --sibling--> deleted fiber
          //
          // We can't disconnect `alternate` on nodes that haven't been deleted
          // yet, but we can disconnect the `sibling` and `child` pointers.
          const previousFiber = fiber.alternate;
          if (previousFiber !== null) {
            let detachedChild = previousFiber.child;
            if (detachedChild !== null) {
              previousFiber.child = null;
              do {
                const detachedSibling = detachedChild.sibling;
                detachedChild.sibling = null;
                detachedChild = detachedSibling;
              } while (detachedChild !== null);
            }
          }
        }
        nextEffect = fiber;
      }
    }
    if ((fiber.subtreeFlags & PassiveMask) !== NoFlags && child !== null) {
      ensureCorrectReturnPointer(child, fiber);
      nextEffect = child;
    } else {
      commitPassiveUnmountEffects_complete();
    }
  }
}
```

기본적으로 **삭제된 파이버의 이펙트를 정리하는** 한 가지 작업을 수행합니다.

왜 그럴까요?

일부 파이버가 삭제되면 파이버 트리에 더 이상 존재하지 않기 때문에, 정리할 때 참조할 수 있도록 React는 `deletions`를 통해 부모 파이버에서 해당 파이버를 계속 추적합니다. 앞으로 자세히 다루겠지만, 어쨌든 우리는 지금 React가 어떻게 하는지 알고 있습니다.

```typescript
function commitPassiveUnmountEffects_complete() {
  while (nextEffect !== null) {
    const fiber = nextEffect;
    if ((fiber.flags & Passive) !== NoFlags) {
      commitPassiveUnmountOnFiber(fiber);
    }
    const sibling = fiber.sibling;
    if (sibling !== null) {
      ensureCorrectReturnPointer(sibling, fiber.return);
      nextEffect = sibling;
      return;
    }
    nextEffect = fiber.return;
  }
}
function commitPassiveUnmountOnFiber(finishedWork: Fiber): void {
  switch (finishedWork.tag) {
    case FunctionComponent:
    case ForwardRef:
    case SimpleMemoComponent: {
      commitHookEffectListUnmount(
        HookPassive | HookHasEffect,
        finishedWork,
        finishedWork.return
      );
      break;
    }
  }
}
```

`complete()`에서 `commitHookEffectListUnmount()`는 핵심 로직이며, 첫 번째 인수는 `HookPassive | HookHasEffect`로, 처음에 `effect.tag`의 기능을 설명한 것처럼 **실행해야 하는패시브 이펙트**를 다시 실행해야 함을 의미합니다.

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

코드는 매우 간단합니다. 모든 연결된 이펙트를 반복(loop)하고, `tag`를 확인하여 `HookPassive | HookHasEffect`와 일치하는지 확인한 다음 `destroy`를 실행합니다.

하지만 이펙트 훅 `pushEffect(HookHasEffect | hookFlags, create, undefined,nextDeps)`를 만들었을 때, `undefined`를 `destroy`로 전달했습니다.

그렇다면 언제 `destroy`가 설정될까요? 이미 알고 계시겠지만, `pushEffect`를 사용할 때는 아직 마운트되지 않았으므로 `commitPassiveMountEffects()`에서 `destroy`를 생성해야 합니다.

## commitPassiveMountEffects()

[트리 탐색 알고리즘은](https://jser.dev/react/2022/01/16/fiber-traversal-in-react) 동일하므로 여기서는 생략하겠습니다.

유사하게, `commitHookEffectListMount(HookPassive | HookHasEffect, finishedWork)`를 트리거합니다.

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
        effect.destroy = create(); // ❗❗
      }
      effect = effect.next;
    } while (effect !== firstEffect);
  }
}
```

간단히 말해서, `effect.destroy = create()`로 파괴를 설정했는데, 이는 여기서 크리에이터 함수가 실행된다는 뜻입니다. 드디어!!

휴, 긴 여정이네요! 두 가지 질문이 더 남았으니 계속 지켜봐 주세요.

## `useEffect()`의 `deps`가 변경되면 어떻게 되나요?

첫 번째 마운트 후, 이제 업데이트 디스패처(dispatcher)를 사용합니다. 컴포넌트 `A()` 가 다시 실행되면, `useEffect()`도 다시 실행되어 [updateEffect()](https://github.com/facebook/react/blob/e62a8d754548a490c2a3bcff3b420e5eedaf11c0/packages/react-reconciler/src/ReactFiberHooks.old.js#L1727)로 이어집니다.

> A()가 다시 실행되는 시점에 대해서는 블로그 포스트 [React bailout은 조정에서 어떻게 작동하는지](https://ted-projects.com/react-internals-deep-dive-13?source=more_series_bottom_blogs)를 참고하세요.

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
    HookHasEffect | hookFlags,
    create,
    destroy,
    nextDeps
  );
}
```

상황이 조금 복잡합니다,

1. `updateWorkInProgressHook()`에서 수행되는 작업은 무엇인가요?
    
2. `currentHook`이란 무엇인가요?
    
3. `areHookInputsEqual()`이 통과하면 어떻게 될까요? 그리고 그렇지 않으면 어떻게 될까요?
    

질문에 답하기 시작하려면 기본적으로 조정(reconciliation)에 대해 알고 있는 것을 상기할 필요가 있습니다. 기본적으로 React에는 각 파이버에 `alternate` 복사본이 있는 파이버 트리(`current`)가 있으며, 이는 `alternate` 파이버 트리가 있음을 의미합니다.

조정은 `alternate` 트리에서 업데이트를 수행하거나, `workInProgress` 파이버 트리라고 말한 다음 이 업데이트된 트리로 전환하는 것을 의미합니다.

> ℹ [이 동영상](https://www.youtube.com/watch?v=0GM-1W7i9Tk)에서 조정에 대한 설명을 참조하세요.

먼저 [updateWorkInProgressHook()](https://github.com/facebook/react/blob/51947a14bb24bd151f76f6fc0acdbbc404de13f7/packages/react-reconciler/src/ReactFiberHooks.old.js#L649)에 사용되는 두 개의 전역 변수를 살펴봅시다,

```typescript
// Hooks are stored as a linked list on the fiber's memoizedState field. The
// current hook list is the list that belongs to the current fiber. The
// work-in-progress hook list is a new list that will be added to the
// work-in-progress fiber.
let currentHook: Hook | null = null;
let workInProgressHook: Hook | null = null;
```

따라서 현재 트리 `currentHook`에서 처리 중인 훅과 `workInProgress` 트리의 훅인 `workInProgressHook`을 추적하고 있습니다.

이 두 가지를 추적하는 이유는 두 가지를 비교하여 deps가 변경되었는지 알기 위해서입니다.

`updateWorkInProgressHook()`의 내부를 살펴보겠습니다.

```typescript
let nextCurrentHook: null | Hook;
if (currentHook === null) {
  const current = currentlyRenderingFiber.alternate;
  if (current !== null) {
    nextCurrentHook = current.memoizedState;
  } else {
    nextCurrentHook = null;
  }
} else {
  nextCurrentHook = currentHook.next;
}
```

먼저 `current tree`에서 currentHook의 다음 훅인 `nextCurrentHook`을 업데이트합니다. 훅을 다시 생성할 것이므로 이전의 존재하는 훅을 찾아야 하는 것이 합리적입니다.

여기서 훅이 안정적이어야 하는 이유를 알 수 있는데, 모든 것이 순서(order)에 따라 달라지기 때문입니다.

```typescript
let nextWorkInProgressHook: null | Hook;
if (workInProgressHook === null) {
  nextWorkInProgressHook = currentlyRenderingFiber.memoizedState;
} else {
  nextWorkInProgressHook = workInProgressHook.next;
}
```

`nextWorkInProgressHook`에 대해서도 동일하게 수행합니다.

```typescript
if (nextWorkInProgressHook !== null) {
  // There's already a work-in-progress. Reuse it.
  workInProgressHook = nextWorkInProgressHook;
  nextWorkInProgressHook = workInProgressHook.next;
  currentHook = nextCurrentHook;
}
```

`nextWorkInProgressHook`을 찾으면, `currentHook`과 `workInProgressHook`을 업데이트하여 안전하게 한 단계 앞으로 나아갈 수 있습니다.

```typescript
else {
  if (nextCurrentHook === null) {
    throw new Error('Rendered more hooks than during the previous render.');
  }
  currentHook = nextCurrentHook;
  const newHook: Hook = {
    memoizedState: currentHook.memoizedState,
    baseState: currentHook.baseState,
    baseQueue: currentHook.baseQueue,
    queue: currentHook.queue,
    next: null,
  };
  if (workInProgressHook === null) {
    // This is the first hook in the list.
    currentlyRenderingFiber.memoizedState = workInProgressHook = newHook;
  } else {
    // Append to the end of the list.
    workInProgressHook = workInProgressHook.next = newHook;
  }
}
```

`nextWorkInProgressHook`을 찾지 못하면, 새 훅을 만들어야 한다는 뜻입니다.

왜 때때로 `nextWorkInProgressHook`이 null이고 어떨때는 null이 아닐까요?

좋은 질문입니다. 컴포넌트 `A()`가 [renderWithHooks()](https://github.com/facebook/react/blob/51947a14bb24bd151f76f6fc0acdbbc404de13f7/packages/react-reconciler/src/ReactFiberHooks.old.js#L366)에서 다시 실행될 때, `memoizedState`와 `updateQueue`가 실제로 재설정(reset)됩니다.

```typescript
workInProgress.memoizedState = null;
workInProgress.updateQueue = null;
workInProgress.lanes = NoLanes;
```

그래서 항상 null이라고 생각하는데, 제가 틀렸다면 바로잡아 주세요.

### deps는 비교됩니다

deps가 같다면, 즉, 이 이펙트 훅을 실행할 필요가 없다면, `hasEffect` 플래그 없이 마치 마운트된 것처럼 `pushEffect`를 실행하면 됩니다.

현재 훅에서 `memoizedState`를 사용하지 않는 이유는 무엇인가요? 좋은 질문이지만 잘 모르겠습니다. 제가 아는 한 가지는 이펙트가 완료된 후에는 `tag`가 재설정(reset)되지 않으므로 태그를 재설정하고 `next` 작업을 정리하지 않으면 쉽게 재사용할 수 없기 때문에 `pushEffect()`를 사용하면 노력을 절약할 수 있다는 것입니다.

deps가 변경되면, 마지막 코드인 이펙트 훅을 실행해야 합니다.

```typescript
hook.memoizedState = pushEffect(
  HookHasEffect | hookFlags,
  create,
  destroy,
  nextDeps
)
```

이제 `HookHasEffect` 플래그가 있는데, 이는 `flushPassiveEffects()`에서 실행될 것임을 의미합니다.

좋아요, 두 번째 질문에 대한 답변이 나온 것 같고 세 번째 질문에 대한 답변도 나온 것 같습니다.

## 요약

1. `Effect`는 `useEffect`에 전달된 함수를 계속 추적하며, 다음과 같은 속성을 가집니다.
    
    * `tag`: 실행이 필요한지 표시하는 `HasEffect`를 포함할 수 있습니다.
        
    * `create`: 우리가 전달한 함수입니다,
        
    * `destroy`: `create`에서 반환된 정리 함수
        
2. `Effect`는 순서대로 목록에 연결되고 `updateQueue`의 파이버에 첨부(attach)됩니다.
    
3. `useEffect()`가 처음 실행될 때
    
    * 새 훅이 생성되고 `memoizedState`에 첨부됩니다.
        
    * `updateQueue`의 파이버에 첨부된, `HasEffect` 태그가 있는, 새 `Effect`가 생성됩니다.
        
4. 이 이펙트가 실행(마운트)되면 `create`가 호출되고 `destroy`가 반환 값으로 설정됩니다.
    
    * 이펙트에 `HasEffect` 플래그가 있고 `destroy`도 있는 경우, destroy를 꼭 호출해야 합니다.
        
5. `flushPassiveEffects`에서는 파이버 트리를 2번의 패스로 순회합니다.
    
    * 먼저 `HasEffect`가 있는 이펙트를 검색하여 `destory`하고 정리합니다.
        
        * 또한 파이버 삭제로 인한 정리도 처리합니다.
            
    * `HasEffect`가 있는 효과를 검색하고, 다시 실행합니다.
        
6. 컴포넌트가 리렌더링되면 workInProgress 파이버에 `memoizedState`와 `emptyQueue`가 비어 있습니다. `useEffect()` 를 다시 실행하면 `deps`가 비교됩니다. 변경 사항이 발견되면 `hasEffect`를 사용하여 새 이펙트가 생성됩니다.
    

`flushPassiveEffects()` 에서 `HasEffect`가 취소되지 않는 이유는 잘 모르겠습니다. 아마도 커밋 단계에서 마운트/언마운트만 하기 때문에 이미 조정이 완료되었으므로 취소할 필요가 없는 것일 수 있습니다. 다음에 렌더링 단계로 넘어가면 이펙트가 재구성됩니다.

이펙트 훅의 라이프사이클을 이해하는 데 도움이 되었기를 바랍니다. React에 대한 더 많은 포스팅이 있을 예정이니 기대해주세요.

(원본 게시일: 2022-01-19)