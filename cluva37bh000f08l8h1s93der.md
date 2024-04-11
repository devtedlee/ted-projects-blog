---
title: "[번역] React useState()는 어떻게 동작하나요?"
datePublished: Thu Apr 11 2024 13:31:53 GMT+0000 (Coordinated Universal Time)
cuid: cluva37bh000f08l8h1s93der
slug: react-internals-deep-dive-5
cover: https://cdn.hashnode.com/res/hashnode/image/upload/v1712842303726/a9b1b296-5a1e-448b-adee-1dfa0fdfeb73.jpeg
tags: react-internals

---

> 영문 블로그 글을 번역했습니다. 허가를 받으면 시리즈를 이어갈 예정입니다.  
> 원문링크: [https://jser.dev/2023-06-19-how-does-usestate-work](https://jser.dev/2023-06-19-how-does-usestate-work)

---

> ℹ️ [React Internals Deep Dive](https://jser.dev/series/react-source-code-walkthrough.html) 에피소드 5, [유튜브에서 제가 설명하는 것](https://www.youtube.com/watch?v=svaUEHMuv9w&list=PLvx8w9g4qv_p-OS-XdbB3Ux_6DMXhAJC3&index=5)을 시청해주세요.
> 
> ⚠ [React@18.2.0](https://github.com/facebook/react/releases/tag/v18.2.0) 기준, 최신 버전에서는 구현이 변경되었을 수 있습니다.

> ***💬 역자 주석: Jser의 코멘트는 ❗❗로 표시 해뒀습니다.  
> 그 외 주석은 리액트 소스 코드 자체의 주석입니다.  
> ... 은 생략된 코드입니다.***

나는 당신이 React의 `useState()` 에 익숙할 거라고 생각합니다. 기본 카운터 앱은 `useState()`를 통해 컴포넌트에 상태를 추가하는 방법을 보여줍니다. 이번 에피소드에서는 소스코드를 살펴봄으로써 내부적으로 `useState()`가 어떻게 작동하는지 알아보겠습니다.

```typescript
import { useState } from 'react';

export default function App() {
  const [count, setCount] = useState(0)
  return (
    <div>
    <button onClick={() => setCount(count => count + 1)}>click {count}</button>
    </div>
  );
}
```

* \[코드 샌드박스 링크\]([https://codesandbox.io/p/sandbox/priceless-rubin-7hs2gc](https://codesandbox.io/p/sandbox/priceless-rubin-7hs2gc))
    

## 1\. 최초 렌더 혹은 마운트 시 `useState()`

초기 렌더링은 매우 간단합니다.

```typescript
function mountState<S>(
  initialState: (() => S) | S,
): [S, Dispatch<BasicStateAction<S>>] {
  const hook = mountWorkInProgressHook();
  // ❗❗         ↗ 새로운 훅이 생성됩니다.
  if (typeof initialState === 'function') {
    initialState = initialState();
  }
  hook.memoizedState = hook.baseState = initialState;
  // ❗❗    ↖ hook에 있는 memoizedState는 실제 state의 값을 보유 합니다.
  const queue: UpdateQueue<S, BasicStateAction<S>> = {
  // ❗❗ Updatequeue는 향후 상태 업데이트를 보관하는 곳입니다.
  // ❗❗ state를 설정할 때 state 값이 바로 업데이트 되는게 아니라는 것을 명심 해야 합니다.
  // ❗❗ 왜냐하면 업데이트들은 서로 다른 우선순위를 갖고 있고, 꼭 바로 처리되야 하는게 아닙니다.
  // ❗❗ 더 자세한 정보는 https://jser.dev/react/2022/03/26/lanes-in-react/ 를 참고 해주세요.
  // ❗❗ 그러므로 우리는 업데이트를 저장한 다음, 나중에 처리해야 합니다.
    pending: null,
    lanes: NoLanes,
    // ❗❗ ↖ lanes 가 우선순위입니다.
    dispatch: null,
    lastRenderedReducer: basicStateReducer,
    lastRenderedState: (initialState: any),
  };
  hook.queue = queue;
  // ❗❗   ↖ 이 queue는 훅을 위한 업데이트 큐라는 것을 명심하세요.
  const dispatch: Dispatch<
    BasicStateAction<S>,
  > = (queue.dispatch = (dispatchSetState.bind(
    // ❗❗                ↗ state 설정자(setter)에게 실제로는 dispatchSetState()가 제공되는 것입니다.
    // ❗❗                   현재 Fiber에 바인딩 되있는 것을 확인하세요.
    null,
    currentlyRenderingFiber,
    queue,
  ): any));
  return [hook.memoizedState, dispatch];
  // ❗❗          ↖ 여기는 우리가 useState()에서 얻을 수 있었던 익숙한 문법입니다.
}
```

## 2\. `setState()`에서 무슨 일이 일어날까요?

위의 코드에서 `setState()`가 실제로는 내부적으로 바인딩된 `dispatchSetState()`라는 것을 알 수 있습니다.

```typescript
function dispatchSetState<S, A>(
  fiber: Fiber,
  queue: UpdateQueue<S, A>,
  action: A,
) {
  const lane = requestUpdateLane(fiber);
  // ❗❗ ↗ 이게 업데이트의 우선순위를 정의합니다.
  // ❗❗ 더 자세한 정보는 https://jser.dev/react/2022/03/26/lanes-in-react/ 참고 하세요.
  const update: Update<S, A> = {
    lane,
    action,
    hasEagerState: false,
    eagerState: null,
    next: (null: any),
  };
  // ❗❗ ↖ 여기에 저장할 예정인 업데이트 객체가 있습니다.
  if (isRenderPhaseUpdate(fiber)) {
    // ❗❗ 렌더링 중에 setState를 할 수 있습니다. 이것도 유용한 패턴입니다. https://react.dev/reference/react/useState#storing-information-from-previous-renders
    // ❗❗ 다만 무한 렌더링으로 이어질 수 있으니 조심해야 합니다.
    enqueueRenderPhaseUpdate(queue, update);
  } else {
    const alternate = fiber.alternate;
  
    if (
      fiber.lanes === NoLanes &&
      (alternate === null || alternate.lanes === NoLanes)
      // ❗❗ ↖ 이 상태 확인은 빠른 bailout을 위한 것입니다,
      // ❗❗ 만약 동일한 state가 설정되있으면 아무것도 하지 않습니다.
      // ❗❗ Bailout은 하위 트리의 리-렌더링을 건너뛰기 위해 더 깊이 내려가지 않는 것을 의미하며,
      // ❗❗ 여기서는 리-렌더링 내부에 있으므로, 리-렌더링 예약을 피하기 위한 이른 bailout 입니다.
      // ❗❗ 하지만 여기 있는 조건은 실제로는 핵(Hack)이며,
      // ❗❗ 필요 이상으로 엄격한 규칙으로
      // ❗❗ React는 최선을 다해 리-렌더링 예약을 피하려고 하지만 보장할 수는 없습니다.
      // ❗❗ 우리는 이 내용에 대한 자세한 내용을 하단의 주의(caveat) 섹션에서 다룰 것입니다.
    ) {
      // The queue is currently empty, which means we can eagerly compute the
      // next state before entering the render phase. If the new state is the
      // same as the current state, we may be able to bail out entirely.
      const lastRenderedReducer = queue.lastRenderedReducer;
      if (lastRenderedReducer !== null) {
        let prevDispatcher;
        try {
          const currentState: S = (queue.lastRenderedState: any);
          const eagerState = lastRenderedReducer(currentState, action);
          // Stash the eagerly computed state, and the reducer used to compute
          // it, on the update object. If the reducer hasn't changed by the
          // time we enter the render phase, then the eager state can be used
          // without calling the reducer again.
          update.hasEagerState = true;
          update.eagerState = eagerState;
          if (is(eagerState, currentState)) {
            // Fast path. We can bail out without scheduling React to re-render.
            // It's still possible that we'll need to rebase this update later,
            // if the component re-renders for a different reason and by that
            // time the reducer has changed.
            // TODO: Do we still need to entangle transitions in this case?
            enqueueConcurrentHookUpdateAndEagerlyBailout(fiber, queue, update);
            return;
            // ❗❗ ↖ 이 반환은 업데이트가 예약되지 않도록 예방합니다.
          }
        } catch (error) {
          // Suppress the error. It will throw again in the render phase.
        } finally {
          if (__DEV__) {
            ReactCurrentDispatcher.current = prevDispatcher;
          }
        }
      }
    }
    const root = enqueueConcurrentHookUpdate(fiber, queue, update, lane);
    // ❗❗         ↗ 여기서 업데이트들을 보관 합니다.
    // ❗❗ 업데이트들은 실제 리-렌더링이 시작될 때 처리되어 Fiber에 첨부(attach)됩니다.
    if (root !== null) {
      const eventTime = requestEventTime();
      scheduleUpdateOnFiber(root, fiber, lane, eventTime);
      // ❗❗ ↖ 여기가 리-렌더링을 예약합니다. 리-렌더링이 즉시 수행되지 않음을 알 수 있습니다.
      // ❗❗ 실제 예약은 리액트 Scheduler에 의해 정해집니다. 참고링크 https://jser.dev/react/2022/03/16/how-react-scheduler-works/
      entangleTransitionUpdate(root, queue, lane);
    }
  }
}
```

업데이트 객체가 어떻게 처리되는지 자세히 살펴보겠습니다.

```typescript
// If a render is in progress, and we receive an update from a concurrent event,
// we wait until the current render is over (either finished or interrupted)
// before adding it to the fiber/hook queue. Push to this array so we can
// access the queue, fiber, update, et al later.
const concurrentQueues: Array<any> = [];
let concurrentQueuesIndex = 0;
let concurrentlyUpdatedLanes: Lanes = NoLanes;
export function finishQueueingConcurrentUpdates(): void {
  // ❗❗         ↗ 이 함수는 prepareFreshStack() 함수 안에서 호출됩니다,
  // ❗❗ 즉 이것은 리-렌더링의 최초 단계들 중 하나 입니다.
  // ❗❗ 이것은 실제 리-렌더링이 시작되기 전에 모든 state 업데이트가 저장된다는 것을 말합니다.
  const endIndex = concurrentQueuesIndex;
  concurrentQueuesIndex = 0;
  concurrentlyUpdatedLanes = NoLanes;
  let i = 0;
  while (i < endIndex) {
    const fiber: Fiber = concurrentQueues[i];
    concurrentQueues[i++] = null;
    const queue: ConcurrentQueue = concurrentQueues[i];
    concurrentQueues[i++] = null;
    const update: ConcurrentUpdate = concurrentQueues[i];
    concurrentQueues[i++] = null;
    const lane: Lane = concurrentQueues[i];
    concurrentQueues[i++] = null;
    if (queue !== null && update !== null) {
      const pending = queue.pending;
      if (pending === null) {
        // This is the first update. Create a circular list.
        update.next = update;
      } else {
        update.next = pending.next;
        pending.next = update;
      }
      queue.pending = update;
      // ❗❗ 이전에 언급했던 hook.queue 기억하나요?
      // ❗❗ 여기서 우리는 보관된 업데이트들이 드디어 fiber에 첨부(attach) 되는 것을 볼 수 있고,
      // ❗❗ 이것은 처리할 준비를 한다는 의미입니다.
    }
    if (lane !== NoLane) {
      markUpdateLaneFromFiberToRoot(fiber, update, lane);
      // ❗❗ 또한 파이버 노드 경로를 dirty로 표시하는 이 함수 호출에 주목하세요
      // ❗❗ 좀 더 자세히는 링크를 참조해 주세요. https://jser.dev/react/2022/01/07/how-does-bailout-work
    }
  }
}
function enqueueUpdate(
  fiber: Fiber,
  queue: ConcurrentQueue | null,
  update: ConcurrentUpdate | null,
  lane: Lane,
) {
  // Don't update the `childLanes` on the return path yet. If we already in
  // the middle of rendering, wait until after it has completed.
  concurrentQueues[concurrentQueuesIndex++] = fiber;
  concurrentQueues[concurrentQueuesIndex++] = queue;
  concurrentQueues[concurrentQueuesIndex++] = update;
  concurrentQueues[concurrentQueuesIndex++] = lane;
  // ❗❗ 내부적으로 업데이트들은 리스트에 유지되고 있습니다.
  // ❗❗ 마치 배치에서 처리될 메세지 큐처럼요.
   concurrentlyUpdatedLanes = mergeLanes(concurrentlyUpdatedLanes, lane);
  // The fiber's `lane` field is used in some places to check if any work is
  // scheduled, to perform an eager bailout, so we need to update it immediately.
  // TODO: We should probably move this to the "shared" queue instead.
  fiber.lanes = mergeLanes(fiber.lanes, lane);
  const alternate = fiber.alternate;
  if (alternate !== null) {
    alternate.lanes = mergeLanes(alternate.lanes, lane);
  }
  // ❗❗ current와 alternate Fiber들이 다 dirty라고 표시되있는 것을 볼 수 있습니다.
  // ❗❗ 이것은 우리가 하단의 주의사항(caveat)을 이해하는데 중요합니다.
}
export function enqueueConcurrentHookUpdate<S, A>(
  fiber: Fiber,
  queue: HookQueue<S, A>,
  update: HookUpdate<S, A>,
  lane: Lane,
): FiberRoot | null {
  const concurrentQueue: ConcurrentQueue = (queue: any);
  const concurrentUpdate: ConcurrentUpdate = (update: any);
  enqueueUpdate(fiber, concurrentQueue, concurrentUpdate, lane); // ❗❗ enqueueUpdate(fiber, concurrentQueue, concurrentUpdate, lane)
  return getRootForUpdatedFiber(fiber);
}
```

> 💬 역자 주석: dirty 플래그란, 변경 사항이 있는 파이버를 가리키는 플래그입니다.

```typescript
function markUpdateLaneFromFiberToRoot(
  sourceFiber: Fiber,
  update: ConcurrentUpdate | null,
  lane: Lane,
): void {
  // Update the source fiber's lanes
  sourceFiber.lanes = mergeLanes(sourceFiber.lanes, lane);
  let alternate = sourceFiber.alternate;
  if (alternate !== null) {
    alternate.lanes = mergeLanes(alternate.lanes, lane);
  }
  // ❗❗ current 파이버와 alternate 파이버 둘 다의 lanes들이 업데이트됩니다.
  // ❗❗ dispatchSetState()가 source 파이버에 바인딩되어 있다고 언급했음을 기억하세요.
  // ❗❗ 그래서 상태를 설정할 때 current 파이버 트리가 항상 업데이트되지 않을 수 있습니다.
  // ❗❗ 두 가지를 모두 설정하면 모든 것이 제대로 작동하지만 부작용이 있습니다.
  // ❗❗ 주의(caveat) 섹션에서 다시 다루겠습니다.

  // 상위 경로를 루트로 이동시키고 하위 경로를 업데이트합니다. ❗❗ <-
  // ❗❗ ↖ 더 자세한 정보는 링크를 참조 해주세요. https://jser.dev/react/2022/01/07/how-does-bailout-work/

  let isHidden = false;
  let parent = sourceFiber.return;
  let node = sourceFiber;
  while (parent !== null) {
    parent.childLanes = mergeLanes(parent.childLanes, lane);
    alternate = parent.alternate;
    if (alternate !== null) {
      alternate.childLanes = mergeLanes(alternate.childLanes, lane);
    }
    if (parent.tag === OffscreenComponent) {
      const offscreenInstance: OffscreenInstance = parent.stateNode;
      if (offscreenInstance.isHidden) {
        isHidden = true;
      }
    }
    node = parent;
    parent = parent.return;
  }
  if (isHidden && update !== null && node.tag === HostRoot) {
    const root: FiberRoot = node.stateNode;
    markHiddenUpdate(root, update, lane);
  }
}
```

```typescript
export function scheduleUpdateOnFiber(
  root: FiberRoot,
  fiber: Fiber,
  lane: Lane,
  eventTime: number,
) {
  checkForNestedUpdates();
  // Mark that the root has a pending update.
  markRootUpdated(root, lane, eventTime);
  if (
    (executionContext & RenderContext) !== NoLanes &&
    root === workInProgressRoot
  ) {
    // Track lanes that were updated during the render phase
    workInProgressRootRenderPhaseUpdatedLanes = mergeLanes(
      workInProgressRootRenderPhaseUpdatedLanes,
      lane,
    );
  } else {
    if (root === workInProgressRoot) {
      // Received an update to a tree that's in the middle of rendering. Mark
      // that there was an interleaved update work on this root. Unless the
      // `deferRenderPhaseUpdateToNextBatch` flag is off and this is a render
      // phase update. In that case, we don't treat render phase updates as if
      // they were interleaved, for backwards compat reasons.
      if (
        deferRenderPhaseUpdateToNextBatch ||
        (executionContext & RenderContext) === NoContext
      ) {
        workInProgressRootInterleavedUpdatedLanes = mergeLanes(
          workInProgressRootInterleavedUpdatedLanes,
          lane,
        );
      }
      if (workInProgressRootExitStatus === RootSuspendedWithDelay) {
        // The root already suspended with a delay, which means this render
        // definitely won't finish. Since we have a new update, let's mark it as
        // suspended now, right before marking the incoming update. This has the
        // effect of interrupting the current render and switching to the update.
        // TODO: Make sure this doesn't override pings that happen while we've
        // already started rendering.
        markRootSuspended(root, workInProgressRootRenderLanes);
      }
    }
    ensureRootIsScheduled(root, eventTime);
    // ❗❗ ↖ 이 줄이 scheduleUpdateOnFiber()에 신경 써야 하는 유일한 줄입니다.
    // ❗❗ 이 함수는 보류 중인 업데이트가 있는 경우 리-렌더링이 예약되도록 합니다.
    // ❗❗ 실제 리-렌더링이 아직 시작되지 않았기 때문에 업데이트가 아직 처리되지 않았습니다.
    // ❗❗ 실제 리-렌더링의 시작은 이벤트 카테고리, 스케줄러 상태 등 몇 가지 요인에 따라 달라집니다.
    // ❗❗ 우리는 이 기능을 여러번 만났습니다.
    // ❗❗ 이 함수에 대해 자세히 알아보려면 내부적으로 어떻게 useTransition()이 작동하는지를 참조하세요.
    // ❗❗ 참고 링크: https://jser.dev/2023-05-19-how-does-usetransition-work/#31-use-case-1---marking-a-state-update-as-a-non-blocking-transition
    if (
      lane === SyncLane &&
      executionContext === NoContext &&
      (fiber.mode & ConcurrentMode) === NoMode &&
      // Treat `act` as if it's inside `batchedUpdates`, even in legacy mode.
      !(__DEV__ && ReactCurrentActQueue.isBatchingLegacy)
    ) {
      // Flush the synchronous work now, unless we're already working or inside
      // a batch. This is intentionally inside scheduleUpdateOnFiber instead of
      // scheduleCallbackForFiber to preserve the ability to schedule a callback
      // without immediately flushing it. We only do this for user-initiated
      // updates, to preserve historical behavior of legacy mode.
      resetRenderTimer();
      flushSyncCallbacksOnlyInLegacyMode();
    }
  }
}
```

## 3\. 리-렌더링의 `useState()`

업데이트가 저장되면 이제 실제로 업데이트를 실행하고 상태 값을 업데이트할 차례입니다.  
이는 실제로 리-렌더링의 `useState()`에서 발생합니다.

```typescript
function updateState<S>(
  initialState: (() => S) | S,
): [S, Dispatch<BasicStateAction<S>>] {
  return updateReducer(basicStateReducer, (initialState: any));
}
function updateReducer<S, I, A>(
  reducer: (S, A) => S,
  initialArg: I,
  init?: I => S,
): [S, Dispatch<A>] {
  const hook = updateWorkInProgressHook();
  // ❗❗       ↗ 이렇게 하면 기존에 생성된 훅이 제공되므로 값을 가져올 수 있습니다.
  const queue = hook.queue;
  // ❗❗ 업데이트 큐가 모든 업데이트들을 보관하고 있다는것을 기억하세요.
  // ❗❗ 리-렌더링이 시작된 후 useState()가 호출되므로, 저장된 업데이트가 파이버로 이동합니다. 
  if (queue === null) {
    throw new Error(
      'Should have a queue. This is likely a bug in React. Please file an issue.',
    );
  }
  queue.lastRenderedReducer = reducer;
  const current: Hook = (currentHook: any);
  // The last rebase update that is NOT part of the base state.
  let baseQueue = current.baseQueue;
  // ❗❗                 ↗ baseQueue에 대한 설명이 필요합니다.
  // ❗❗ 가장 좋은 경우, 우리는 업데이트가 처리될 때 그냥 버리면 됩니다.
  // ❗❗ 그러나 우선순위가 다른 여러 업데이트가 있을 수 있으므로 
  // ❗❗ 나중에 처리하기 위해 일부를 건너뛰어야 할 수도 있습니다. 
  // ❗❗ 이것이 바로 baseQueue에 저장되는 이유입니다.
  // ❗❗ 또한 처리된 업데이트의 경우에도 최종 state가 올바른지 확인하려면
  // ❗❗ 업데이트가 baseQueue에 저장되면 다음 업데이트도 모두 거기에 있어야 합니다.
  // ❗❗ 예를 들어, state 값이 1이고, 업데이트가 3개 있을때, +1(low), *10(high), -2(low)
  // ❗❗ *10 이 우선순위가 높으므로, 처리하면, 1 * 10 = 10
  // ❗❗ 그 후 낮은 우선순위를 가진 것들을 처리합니다,
  // ❗❗ 만약 우리가 *10을 큐에 넣지 않으면, 1 + 1 - 2 = 0이 됩니다.
  // ❗❗ 하지만 우리에게 필요한 것은 (1 + 1) * 10 - 2 입니다.

  // The last pending update that hasn't been processed yet.
  const pendingQueue = queue.pending;
  if (pendingQueue !== null) {
    // We have new updates that haven't been processed yet.
    // We'll add them to the base queue.
    if (baseQueue !== null) {
      // Merge the pending queue and the base queue.
      const baseFirst = baseQueue.next;
      const pendingFirst = pendingQueue.next;
      baseQueue.next = pendingFirst;
      pendingQueue.next = baseFirst;
    }
    current.baseQueue = baseQueue = pendingQueue;
    queue.pending = null;
    // ❗❗     ↖ 보류 중인 큐가 정리되고 baseQueue로 병합됩니다.
  }
  if (baseQueue !== null) {
    // We have a queue to process.
    const first = baseQueue.next;
    let newState = current.baseState;
    let newBaseState = null;
    let newBaseQueueFirst = null;
    let newBaseQueueLast = null;
    // ❗❗ baseQueue를 처리한 후, 새로운 baseQueue가 생성됩니다.
    let update = first;
    do {
    // ❗❗ 이 do...while 루프는 모든 업데이트를 처리하려고 시도합니다.
      ...
      // Check if this update was made while the tree was hidden. If so, then
      // it's not a "base" update and we should disregard the extra base lanes
      // that were added to renderLanes when we entered the Offscreen tree.
      const shouldSkipUpdate = isHiddenUpdate
        ? !isSubsetOfLanes(getWorkInProgressRootRenderLanes(), updateLane)
        : !isSubsetOfLanes(renderLanes, updateLane);
      if (shouldSkipUpdate) {
        // 우선 순위가 충분하지 않습니다. 이 업데이트를 건너뛰세요.
        // 처음 건너뛴 업데이트인 경우 이전 업데이트/상태가 새로운 기본 업데이트/상태가 됩니다.
        // ❗❗ ↖ 네, 우선순위가 낮은 것
        const clone: Update<S, A> = {
          lane: updateLane,
          action: update.action,
          hasEagerState: update.hasEagerState,
          eagerState: update.eagerState,
          next: (null: any),
        };
        if (newBaseQueueLast === null) {
          newBaseQueueFirst = newBaseQueueLast = clone;
          newBaseState = newState;
        } else {
          newBaseQueueLast = newBaseQueueLast.next = clone;
        }
        // ❗❗ ↖ 업데이트는 처리되지 않았기 때문에 새 baseQueue에 저장됩니다.
        // Update the remaining priority in the queue.
        // TODO: Don't need to accumulate this. Instead, we can remove
        // renderLanes from the original lanes.
        currentlyRenderingFiber.lanes = mergeLanes(
          currentlyRenderingFiber.lanes,
          updateLane,
        );
        markSkippedUpdateLanes(updateLane);
      } else {
        // This update does have sufficient priority.
        if (newBaseQueueLast !== null) {
      
          const clone: Update<S, A> = {
            // This update is going to be committed so we never want uncommit
            // it. Using NoLane works because 0 is a subset of all bitmasks, so
            // this will never be skipped by the check above.
            lane: NoLane,
            action: update.action,
            hasEagerState: update.hasEagerState,
            eagerState: update.eagerState,
            next: (null: any),
          };
          newBaseQueueLast = newBaseQueueLast.next = clone;
        }
        // ❗❗ ↖ 앞서 baseQueue에 대해 설명했듯이, 여기서는 newBaseQueue가 비어 있지 않으면,
        // ❗❗ 나중에 사용할 수 있도록 다음의 모든 업데이트를 저장해야 합니다.

        // Process this update.
        if (update.hasEagerState) {
          // If this update is a state update (not a reducer) and was processed eagerly,
          // we can use the eagerly computed state
          newState = ((update.eagerState: any): S);
        } else {
          const action = update.action;
          newState = reducer(newState, action);
        }
      }
      update = update.next;
    } while (update !== null && update !== first);
    if (newBaseQueueLast === null) {
      newBaseState = newState;
    } else {
      newBaseQueueLast.next = (newBaseQueueFirst: any);
    }
    // Mark that the fiber performed work, but only if the new state is
    // different from the current state.
    if (!is(newState, hook.memoizedState)) {
      markWorkInProgressReceivedUpdate();
      // ❗❗ ↖ 리-렌더링하는 동안 state가 변경되지 않으면
      // ❗❗ 실제로 bailout(이른 bailout 이 아니라)됩니다.
    }
    hook.memoizedState = newState;
    // 마침내 새로운 state가 설정됩니다.
    hook.baseState = newBaseState;
    hook.baseQueue = newBaseQueueLast;
    //     ↖ 다음 라운드의 리-렌더링을 위해 새로운 baseQueue가 설정됩니다.
    queue.lastRenderedState = newState;
  }
  if (baseQueue === null) {
    // `queue.lanes` is used for entangling transitions. We can set it back to
    // zero once the queue is empty.
    queue.lanes = NoLanes;
  }
  const dispatch: Dispatch<A> = (queue.dispatch: any);
  return [hook.memoizedState, dispatch];
  // ❗❗      ↖ 이제 새로운 state가 생겼습니다! 그리고 dispatch()가 안정적입니다!
}
```

## 4\. 요약

다음은 내부를 설명하는 몇 가지 간단한 슬라이드입니다.

* 슬라이드 링크: [링크](https://jser.dev/2023-06-19-how-does-usestate-work#how-usestate---works-internally)
    

> 위 슬라이드의 경우 [실제 데모](https://jser.dev/demos/react/lanes-priority/with-schedule-api-2.html)를 사용해 볼 수 있습니다.

## 5\. 주의사항(caveats) 이해하기

React.dev에 [주의 사항](https://react.dev/reference/react/useState#setstate-returns)이 나열되어 있는데, 왜 그런 주의 사항이 존재하는지 이해해 보겠습니다.

### 5.1 state 업데이트는 동기화 되지 않습니다.

> set 함수는 다음 렌더링에 대한 state 변수만 업데이트합니다. set 함수를 호출한 후 state 변수를 읽으면 호출 전 화면에 있던 이전 값을 계속 가져옵니다.

이는 이해하기 쉬운데, 이미 `setState()`가 다음 틱에서 리-렌더링을 예약하고 동기화 동작도 아니며 state 업데이트가 `setState()`가 아닌 `useState()`에서 수행되므로 업데이트된 값은 다음 렌더링에서만 가져올 수 있다는 것을 보았습니다.

### 5.2 동일한 값을 가진 `setState()` 는 여전히 리-렌더링을 트리거할 수 있습니다.

> Object.is 비교에 의해 결정된 대로 사용자가 제공하는 새 값이 현재 state와 동일한 경우, React는 컴포넌트와 그 자식들의 리-렌더링을 생략합니다. 이것은 최적화입니다. 경우에 따라 React가 자식들을 건너뛰기 전에 컴포넌트를 호출해야 할 수도 있지만 코드에는 영향을 미치지 않습니다.

이것이 가장 까다로운 주의 사항입니다. 다음은 한 번 해볼 수 있는 퀴즈입니다.

```typescript

import React, { useState } from 'react'
import ReactDOM from 'react-dom'
import { screen } from '@testing-library/dom'
import userEvent from '@testing-library/user-event'

function A() {
  console.log('render A')
  return null
}

function App() {
  const [_state, setState] = useState(false)
  console.log('render App')
  return <div>
    <button onClick={() => {
      console.log('click')
      setState(true)
    }}>click me</button>
    <A />
  </div>
}

ReactDOM.render(<App/>, document.getElementById('root'))

userEvent.click(screen.getByText('click me'))
userEvent.click(screen.getByText('click me'))
userEvent.click(screen.getByText('click me'))

// 최종 출력 결과를 한줄 한줄씩 예상해보세요
```

* 정답 링크: [링크](https://stackoverflow.com/questions/57652176/react-hooks-usestate-setvalue-still-rerender-one-more-time-when-value-is-equal)
    

같은 state 값을 설정해도 왜 리-렌더링이 발생하는지 알아내는 데 꽤 오랜 시간이 걸렸습니다.

이를 이해하려면 더 깊이 파고들지 않은 `dispatchSetState()` 내부의 긴급 bailout 조건으로 돌아가야 합니다.

```typescript
 if (
      fiber.lanes === NoLanes &&
      (alternate === null || alternate.lanes === NoLanes)
      // ❗❗ 이러한 조건에서 state가 변경되지 않으면 리-렌더링 예약을 피하십시오.
 ) {
```

이전 슬라이드에서 설명한 것처럼, 가장 좋은 확인 방법은 *보류 중인 업데이트 큐와 훅에 대한 baseQueue가 비어 있는지* 확인하는 것입니다. 하지만 현재 구현 방식으로는 실제로 리-렌더링을 시작할 때까지는 사실 여부를 알 수 없습니다.

따라서 여기서는 파이버 노드에 업데이트가 없는지 간단히 확인하는 것으로 돌아갑니다. 업데이트가 큐에 추가되면 파이버가 dirty로 표시되므로 리-렌더링이 시작될 때까지 기다릴 필요가 없습니다.

하지만 부작용도 있습니다.

```typescript
function enqueueUpdate(
  fiber: Fiber,
  queue: ConcurrentQueue | null,
  update: ConcurrentUpdate | null,
  lane: Lane,
) {
  concurrentQueues[concurrentQueuesIndex++] = fiber;
  concurrentQueues[concurrentQueuesIndex++] = queue;
  concurrentQueues[concurrentQueuesIndex++] = update;
  concurrentQueues[concurrentQueuesIndex++] = lane;

  concurrentlyUpdatedLanes = mergeLanes(concurrentlyUpdatedLanes, lane);

  fiber.lanes = mergeLanes(fiber.lanes, lane);
  const alternate = fiber.alternate;
  if (alternate !== null) {
    alternate.lanes = mergeLanes(alternate.lanes, lane);
  }
}
```

업데이트를 큐에 넣을 때 current 및 alternate 파이버 모두 lanes가 dirty로 표시되어 있는 것을 볼 수 있습니다. 이는 `dispatchSetState()` 가 소스 파이버에 바인딩되어 있으므로 current와 alternate를 모두 업데이트하지 않으면 업데이트가 처리되는지 확인할 수 없기 때문에 필요합니다.

> current와 alternate에 대한 건, [디버깅 비디오](https://www.youtube.com/watch?v=0GM-1W7i9Tk&list=PLvx8w9g4qv_p-OS-XdbB3Ux_6DMXhAJC3&index=3&themeRefresh=1)를 참고해주세요

하지만 lanes 지우기는 실제 리-렌더링이 이루어지는 `beginWork()`에서만 이루어집니다.

```typescript
function beginWork(
  current: Fiber | null,
  workInProgress: Fiber,
  renderLanes: Lanes,
): Fiber | null {
  ...
  // Before entering the begin phase, clear pending update priority.
  // TODO: This assumes that we're about to evaluate the component and process
  // the update queue. However, there's an exception: SimpleMemoComponent
  // sometimes bails out later in the begin phase. This indicates that we should
  // move this assignment out of the common path and into each branch.
  workInProgress.lanes = NoLanes;
  ...
}
```

이로 인해 **업데이트가 예약되면 최소 2라운드의 리-렌더링을 거쳐야만 dirty lanes 플래그가 완전히 지워지는** 결과가 발생합니다.

단계는 대략 다음과 같습니다.

1. **fiber1(current, clean) / null(alternate)** → fiber1은 `useState()`의 소스 파이버입니다.
    
2. `setState(true)` → `true`는 `false`와 다르기 때문에 이른 bailout 조치가 발생하지 않습니다.
    
3. **fiber1(current, dirty) / null(대체)** → 업데이트 큐에 삽입
    
4. **fiber1(current, dirty) / fiber2(workInProgress, dirty**) → 리-렌더링 시작, 새 파이버를 workInProgress로 생성함
    
5. **fiber1(current, dirty) / fiber2(workInProgress, clean**) → `beginWork()`에서 lanes가 지워집니다.
    
6. **fiber1(alternate, dirty) / fiber2(current, clean**) → 커밋 후 React는 2가지 버전의 파이버 트리를 바꿉(swap)니다.
    
7. `setState(true)` → 파이버 중 하나가 깨끗하지 않기 때문에 이른 bailout이 여전히 발생하지 않습니다.
    
8. **fiber1(alternate, dirty)/fiber2(current, dirty**) → 업데이트 큐에 삽입
    
9. **fiber1(workInProgress, dirty) / fiber2(current, dirty)** → 리-렌더링 시작, fiber1에 fiber2의 lanes 할당
    
10. **fiber1 (workInProgress, clean) / fiber2 (current, dirty**) → `beginWork()`에서 lanes가 지워집니다.
    
11. **fiber1(workInProgress, clean) / fiber2(current, clean**) → state 변경이 발견되지 않고, `bailoutHooks()`에서 현재 파이버에 대한 lanes가 제거되고, bailout(이른 bailout 아님)가 발생합니다.
    
12. **fiber1(current, clean) / fiber2(alternate, clean**) → 커밋 후 React는 2가지 버전의 파이버 트리를 교체합니다.
    
13. `setState(true)` → 이번에는 두 파이버가 모두 깨끗해져서 실제로 이른 bailout을 할 수 있습니다!
    

이 문제를 해결할 수 있을까요? 하지만 파이버 아키텍처와 훅의 작동 방식 때문에 비용이 너무 많이 들었을 수도 있습니다. 대부분의 경우 문제가 되지 않기 때문에 React 팀에서 [수정할 의사가 없는 것으로 보이는 논의](https://github.com/facebook/react/issues/14994)가 이미 있었습니다.

**React는 필요할 경우 리-렌더링을 수행**한다는 점을 명심해야 하며,**성능 트릭이 항상 작동한다고 가정해서는 안 됩니다.**

### 5.3 React state 업데이트 배치 처리

> React는 state 업데이트를 일괄 처리합니다. 모든 이벤트 핸들러가 실행되고 설정된 함수를 호출한 후에 화면을 업데이트합니다. 이렇게 하면 단일 이벤트 중에 여러 번 리-렌더링하는 것을 방지할 수 있습니다. 드물지만 DOM에 액세스하는 등 React가 화면을 더 일찍 업데이트하도록 강제해야 하는 경우 flushSync를 사용할 수 있습니다.

이전 슬라이드에서 설명한 것처럼 업데이트는 실제로 처리되기 전에 저장된 후 함께 처리됩니다.

(원본 게시일: 2023-07-23)