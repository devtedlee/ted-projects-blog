---
title: "[번역] React 소스 코드에서 Lane이란 무엇인가요?"
datePublished: Thu May 30 2024 02:59:28 GMT+0000 (Coordinated Universal Time)
cuid: clwso2nqb000h0al50c5840mg
slug: react-internals-deep-dive-21
cover: https://cdn.hashnode.com/res/hashnode/image/upload/v1717034326739/645b46cf-3473-420e-8cbf-5b615a7b3b33.jpeg
tags: react-internals, lanes

---

> ***영문 블로그 글을 번역했습니다. 허가를 받으면 시리즈를 이어갈 예정입니다.  
> 원문링크:*** [https://jser.dev/react/2022/03/26/lanes-in-react](https://jser.dev/react/2022/03/26/lanes-in-react)

---

> ***ℹ️***[***React Internals Deep Dive***](https://jser.dev/series/react-source-code-walkthrough.html) ***에피소드 21,*** [***유튜브에서 제가 설명하는 것***](https://www.youtube.com/watch?v=yy8HRyhA_oQ&list=PLvx8w9g4qv_p-OS-XdbB3Ux_6DMXhAJC3&index=21)***을 시청해주세요.***
> 
> ***⚠***[***React@18.2.0***](https://github.com/facebook/react/releases/tag/v18.2.0)***기준, 최신 버전에서는 구현이 변경되었을 수 있습니다.***
> 
> ***💬 역자 주석: JSer의 코멘트는 ❗❗로 표시 해뒀습니다.  
> 그 외 주석은 리액트 소스 코드 자체의 주석입니다.  
> ... 은 생략된 코드입니다.***

앞서 React가 우선순위에 따라 작업을 예약하는 방법을 살펴봤는데, 그 예로 단일 파이버의 작업 수준이 아닌 파이버 트리 전체를 대상으로 하는 `performConcurrentWorkOnRoot()` 작업을 살펴보았습니다.

동시 모드는 React가 우선순위에 따라 각 파이버마다 다른 작업을 수행할 수 있기 때문에 쿨한데, 이 낮은 수준의 우선순위는 "Lane"이라는 개념으로 구현됩니다. 전문 용어로 들릴 수 있지만 걱정하지 마세요. 자세히 설명해드리고 마지막에 몇 가지 예제가 있습니다.

## 1\. 세 가지 우선순위 시스템

스케줄링 메서드가 호출되는 항목 중 하나인 `ensureRootIsScheduled()`를 다시 한 번 살펴보죠.

```typescript
// We use the highest priority lane to represent the priority of the callback.
const newCallbackPriority = getHighestPriorityLane(nextLanes);
if (newCallbackPriority === SyncLane) {
  ...
} else {
  let schedulerPriorityLevel;
  switch (lanesToEventPriority(nextLanes)) {
    case DiscreteEventPriority:
      schedulerPriorityLevel = ImmediateSchedulerPriority;
      break;
    case ContinuousEventPriority:
      schedulerPriorityLevel = UserBlockingSchedulerPriority;
      break;
    case DefaultEventPriority:
      schedulerPriorityLevel = NormalSchedulerPriority;
      break;
    case IdleEventPriority:
      schedulerPriorityLevel = IdleSchedulerPriority;
      break;
    default:
      schedulerPriorityLevel = NormalSchedulerPriority;
      break;
  }
  newCallbackNode = scheduleCallback(
    schedulerPriorityLevel,
    performConcurrentWorkOnRoot.bind(null, root),
  );
}
```

흥미롭게도 위의 코드에서 스케줄러 우선 순위가 다음과 같이 도출된다는 것을 알 수 있습니다.

1. 가장 우선순위가 높은 lane 가져오기 - `getHighestPriorityLane()`
    
2. SyncLane이 아닌 경우 lane을 이벤트 우선순위에 매핑한 다음 스케줄러 우선순위에 매핑합니다.
    

따라서 3가지 우선순위 시스템이 있습니다.

1. 스케줄러 우선순위 - 스케줄러에서 작업의 우선순위를 지정하는 데 사용됩니다.
    
2. 이벤트 우선순위 - 사용자 이벤트의 우선순위를 표시합니다.
    
3. Lane 우선순위 - 작업의 우선순위를 표시합니다.
    

목적이 다르지만 위와 같은 매핑 로직을 가지고 있기 때문에 위와 같이 분리되어 있으며, 이벤트 시스템은 다음 편에서 다룰 예정이므로 여기서는 자세히 다루지 않습니다.

## 2\. "Lane"은 무엇인가요?

[`setState()`](https://www.youtube.com/watch?v=svaUEHMuv9w)에 대한 유튜브 동영상을 보면 파이버가 링크드 리스트 자료구조에 보관된 훅들을 보유하고 있으며, 상태 훅의 경우 업데이트(리렌더링) 중에 실행되는 업데이트 큐가 있다는 것을 알 수 있습니다.

업데이트가 생성되는 코드는 다음과 같습니다. ([소스](https://github.com/facebook/react/blob/5a1e558df21bd3cafbaea01cc418fa69d14a8cab/packages/react-reconciler/src/ReactFiberHooks.old.js#L2242))

```typescript
const lane = requestUpdateLane(fiber);
const update: Update<S, A> = {
  lane, // ❗❗
  action,
  hasEagerState: false,
  eagerState: null,
  next: (null: any),
};
```

네, `lane`이라는 필드가 보이시죠? `Lane`은 **업데이트의 우선순위를 표시하는 것으로, 작업의 우선순위를 표시한다고도 말할 수 있습니다**.

다음은 React의 모든 레인을 이진 형식으로 이해하기 쉽도록 숫자로 표현한 것입니다. `1`을 찾아보세요.

```typescript
// Lane values below should be kept in sync with getLabelForLane(), used by react-devtools-timeline.
// If those values are changed that package should be rebuilt and redeployed.
export const TotalLanes = 31;
export const NoLanes: Lanes = /*                        */ 0b0000000000000000000000000000000;
export const NoLane: Lane = /*                          */ 0b0000000000000000000000000000000;
export const SyncLane: Lane = /*                        */ 0b0000000000000000000000000000001;
export const InputContinuousHydrationLane: Lane = /*    */ 0b0000000000000000000000000000010;
export const InputContinuousLane: Lanes = /*            */ 0b0000000000000000000000000000100;
export const DefaultHydrationLane: Lane = /*            */ 0b0000000000000000000000000001000;
export const DefaultLane: Lanes = /*                    */ 0b0000000000000000000000000010000;
const TransitionHydrationLane: Lane = /*                */ 0b0000000000000000000000000100000;
const TransitionLanes: Lanes = /*                       */ 0b0000000001111111111111111000000;
const TransitionLane1: Lane = /*                        */ 0b0000000000000000000000001000000;
const TransitionLane2: Lane = /*                        */ 0b0000000000000000000000010000000;
const TransitionLane3: Lane = /*                        */ 0b0000000000000000000000100000000;
const TransitionLane4: Lane = /*                        */ 0b0000000000000000000001000000000;
const TransitionLane5: Lane = /*                        */ 0b0000000000000000000010000000000;
const TransitionLane6: Lane = /*                        */ 0b0000000000000000000100000000000;
const TransitionLane7: Lane = /*                        */ 0b0000000000000000001000000000000;
const TransitionLane8: Lane = /*                        */ 0b0000000000000000010000000000000;
const TransitionLane9: Lane = /*                        */ 0b0000000000000000100000000000000;
const TransitionLane10: Lane = /*                       */ 0b0000000000000001000000000000000;
const TransitionLane11: Lane = /*                       */ 0b0000000000000010000000000000000;
const TransitionLane12: Lane = /*                       */ 0b0000000000000100000000000000000;
const TransitionLane13: Lane = /*                       */ 0b0000000000001000000000000000000;
const TransitionLane14: Lane = /*                       */ 0b0000000000010000000000000000000;
const TransitionLane15: Lane = /*                       */ 0b0000000000100000000000000000000;
const TransitionLane16: Lane = /*                       */ 0b0000000001000000000000000000000;
const RetryLanes: Lanes = /*                            */ 0b0000111110000000000000000000000;
const RetryLane1: Lane = /*                             */ 0b0000000010000000000000000000000;
const RetryLane2: Lane = /*                             */ 0b0000000100000000000000000000000;
const RetryLane3: Lane = /*                             */ 0b0000001000000000000000000000000;
const RetryLane4: Lane = /*                             */ 0b0000010000000000000000000000000;
const RetryLane5: Lane = /*                             */ 0b0000100000000000000000000000000;
export const SomeRetryLane: Lane = RetryLane1;
export const SelectiveHydrationLane: Lane = /*          */ 0b0001000000000000000000000000000;
const NonIdleLanes = /*                                 */ 0b0001111111111111111111111111111;
export const IdleHydrationLane: Lane = /*               */ 0b0010000000000000000000000000000;
export const IdleLane: Lanes = /*                       */ 0b0100000000000000000000000000000;
export const OffscreenLane: Lane = /*                   */ 0b1000000000000000000000000000000;
```

도로의 차선(lane)처럼, 속도에 따라 레인을 달리하는 것이 규칙이므로 레인이 작을수록 긴급한 작업의 우선 순위가 높습니다. 따라서 여기서 `SyncLane`은 `1`입니다.

많은 레인이 있습니다. 이 에피소드에서는 각각의 레인에 대해 자세히 설명하기보다는 전반적으로 어떻게 작동하는지에 대해 설명합니다.

### 2.1 비트 단위 연산자

레인은 숫자에 불과하며, React 소스 코드에는 비트 단위 연산이 많이 있으므로 이에 익숙해지도록 합시다.

다음은 몇 가지 예시입니다.

```typescript
export function includesSomeLane(a: Lanes | Lane, b: Lanes | Lane) {
  return (a & b) !== NoLanes;
}
export function isSubsetOfLanes(set: Lanes, subset: Lanes | Lane) {
  return (set & subset) === subset;
}
export function mergeLanes(a: Lanes | Lane, b: Lanes | Lane): Lanes {
  return a | b;
}
export function removeLanes(set: Lanes, subset: Lanes | Lane): Lanes {
  return set & ~subset;
}
export function intersectLanes(a: Lanes | Lane, b: Lanes | Lane): Lanes {
  return a & b;
}
```

### 2.2 `childLanes`를 기억하나요?

[리액트 bailout이 조정에서 어떻게 작동하는지](https://ted-projects.com/react-internals-deep-dive-13)에 대한 에피소드에서는 파이버의 `lanes`와 `childeLanes`를 약간 다뤘습니다. 각 파이버는 이것들을 알고 있습니다:

1. 자체 작업의 우선 순위 - `lanes`
    
2. 자손의 작업 우선순위 - `childLanes`
    

## 3\. `performConcurrentWorkOnRoot()`를 다시 살펴봅시다

다음은 작업이 예약되고 실행되는 기본적인 흐름입니다.

1. 파이버 트리의 `nestLanes`를 가져옵니다.
    
2. 스케줄러 우선순위에 매핑
    
3. 조정(reconcile)할 작업 예약
    
4. 조정이 발생하면, 루트에서 작업을 처리합니다
    

레인 정보가 사용되는 곳에서 실제로 조정하는 것이 매직이라고 생각합니다.

```typescript
let lanes = getNextLanes(
  root,
  root === workInProgressRoot ? workInProgressRootRenderLanes : NoLanes
);
...
prepareFreshStack(root, lanes);
```

`prepareFreshStack()`은 조정을 다시 시작한다는 의미이며, 현재 파이버를 추적하는 커서(`workInProgress`)가 있다는 것을 기억하세요. 보통 React는 일시 중지했다가 이전 위치에서 다시 시작하지만 오류나 이상한 경우 현재 완료된 작업을 포기하고 처음부터 다시 실행해야 하는데, 이것이 바로 `fresh`의 의미입니다.

```typescript
function prepareFreshStack(root: FiberRoot, lanes: Lanes) {
  root.finishedWork = null;
  root.finishedLanes = NoLanes;
  const timeoutHandle = root.timeoutHandle;
  if (timeoutHandle !== noTimeout) {
    // The root previous suspended and scheduled a timeout to commit a fallback
    // state. Now that we have additional work, cancel the timeout.
    root.timeoutHandle = noTimeout;
    // $FlowFixMe Complains noTimeout is not a TimeoutID, despite the check above
    cancelTimeout(timeoutHandle);
  }
  if (workInProgress !== null) {
    let interruptedWork = workInProgress.return;
    while (interruptedWork !== null) {
      unwindInterruptedWork(interruptedWork, workInProgressRootRenderLanes);
      interruptedWork = interruptedWork.return;
    }
  }
  workInProgressRoot = root;
  workInProgress = createWorkInProgress(root.current, null);
  workInProgressRootRenderLanes = // <❗❗
    subtreeRenderLanes =
    workInProgressRootIncludedLanes =
      lanes;
  workInProgressRootExitStatus = RootIncomplete;
  workInProgressRootFatalError = null;
  workInProgressRootSkippedLanes = NoLanes;
  workInProgressRootInterleavedUpdatedLanes = NoLanes;
  workInProgressRootRenderPhaseUpdatedLanes = NoLanes;
  workInProgressRootPingedLanes = NoLanes; // ❗❗/>

  enqueueInterleavedUpdates();
}
```

`prepareFreshStack()`에서 일부 변수가 방금 리셋된 것을 볼 수 있습니다. 그 중 상당수가 레인에 관한 변수입니다.

1. `workInProgressRootRenderLanes`
    
2. `subtreeRenderLanes`
    
3. `workInProgressRootIncludedLanes`
    
4. `workInProgressRootSkippedLanes`
    
5. `workInProgressRootInterleavedUpdatedLanes`
    
6. `workInProgressRootRenderPhaseUpdatedLanes`
    
7. `workInProgressRootPingedLanes`
    

좋습니다, 현재로서는 정체를 알 수 없지만, `workInProgressRootRenderLanes`은 간단하게 살펴볼 수 있습니다.

```typescript
// The lanes we're rendering ❗❗
let workInProgressRootRenderLanes: Lanes = NoLanes;
```

주석 자체에서 알 수 있듯이 이것이 우리가 렌더링하는 레인입니다.

예를 들어 여기와 같이 몇 군데에서 사용됩니다:

```typescript
export function requestUpdateLane(fiber: Fiber): Lane {
  // Special cases
  const mode = fiber.mode;
  if ((mode & ConcurrentMode) === NoMode) {
    return (SyncLane: Lane);
  } else if (
    !deferRenderPhaseUpdateToNextBatch &&
    (executionContext & RenderContext) !== NoContext &&
    workInProgressRootRenderLanes !== NoLanes // ❗❗
  ) {
    // This is a render phase update. These are not officially supported. The
    // old behavior is to give this the same "thread" (lanes) as
    // whatever is currently rendering. So if you call `setState` on a component
    // that happens later in the same render, it will flush. Ideally, we want to
    // remove the special case and treat them as if they came from an
    // interleaved event. Regardless, this pattern is not officially supported.
    // This behavior is only a fallback. The flag only exists until we can roll
    // out the setState warning, since existing code might accidentally rely on
    // the current behavior.
    return pickArbitraryLane(workInProgressRootRenderLanes); // ❗❗
  }
  // Updates originating inside certain React methods, like flushSync, have
  // their priority set by tracking it with a context variable.
  //
  // The opaque type returned by the host config is internally a lane, so we can
  // use that directly.
  // TODO: Move this type conversion to the event priority module.
  const updateLane: Lane = (getCurrentUpdatePriority(): any);
  if (updateLane !== NoLane) {
    return updateLane;
  }
  // This update originated outside React. Ask the host environment for an
  // appropriate priority, based on the type of event.
  //
  // The opaque type returned by the host config is internally a lane, so we can
  // use that directly.
  // TODO: Move this type conversion to the event priority module.
  const eventLane: Lane = (getCurrentEventPriority(): any);
  return eventLane;
}
```

아하, `requestUpdateLane()`이라는 것이 보이시죠? 위의 함수에서 무슨 일이 일어나는지 이해하기는 조금 어렵지만, 현재 렌더링 레인이 렌더링 중에 예약된 레인에 어느 정도 영향을 미친다는 것은 분명합니다.

`performConcurrentWorkOnRoot()`로 돌아가 보겠습니다.

```typescript
// Determine the next lanes to work on, using the fields stored
// on the root.
let lanes = getNextLanes(
  root,
  root === workInProgressRoot ? workInProgressRootRenderLanes : NoLanes
);
```

이것은 작업할 레인을 결정합니다.`getNextLanes()` 은 매우 복잡하므로 여기서는 생략하고, 아주 기본적인 경우 `getNextLanes()`가 우선순위가 가장 높은 레인을 선택한다는 점만 기억하세요.

```typescript
if (lanes === NoLanes) {
  // Defensive coding. This is never expected to happen.
  return null;
}
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
  ? renderRootConcurrent(root, lanes)
  : renderRootSync(root, lanes);
```

흥미로운 점은, 동시 모드에서도 경우에 따라 동기화 모드로 돌아갈 수 있다는 것입니다. 예를 들어 레인에 차단(blocking) 레인이 포함되어 있거나 일부 레인이 만료된 경우입니다.

자세한 내용은 건너뛰고 계속 진행하겠습니다.

## 4\. `updateReducer()`

앞서 설명했듯이, `useState()`는 최초 렌더링에서는 `mountState()`에 매핑되고 다음 업데이트에서는 `updateState()`에 매핑됩니다.

상태 업데이트는 `updateState()`에서 이루어집니다.

```typescript
function updateState<S>(
  initialState: (() => S) | S
): [S, Dispatch<BasicStateAction<S>>] {
  return updateReducer(basicStateReducer, (initialState: any));
}
```

내부적으로 `useReducer()`가 사용됩니다. ([소스](https://github.com/facebook/react/blob/5a1e558df21bd3cafbaea01cc418fa69d14a8cab/packages/react-reconciler/src/ReactFiberHooks.old.js#L756))

```typescript
function updateReducer<S, I, A>(
  reducer: (S, A) => S,
  initialArg: I,
  init?: (I) => S
): [S, Dispatch<A>] {
  const hook = updateWorkInProgressHook();
  const queue = hook.queue;
  queue.lastRenderedReducer = reducer;
  const current: Hook = (currentHook: any);
  // The last rebase update that is NOT part of the base state.
  let baseQueue = current.baseQueue;
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
  }
  if (baseQueue !== null) {
    // We have a queue to process.
    const first = baseQueue.next;
    let newState = current.baseState;
    let newBaseState = null;
    let newBaseQueueFirst = null;
    let newBaseQueueLast = null;
    let update = first;
    do { // <❗❗
      const updateLane = update.lane;
      if (!isSubsetOfLanes(renderLanes, updateLane)) {
        // Priority is insufficient. Skip this update. If this is the first
        // skipped update, the previous update/state is the new base
        // update/state.
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
        // Update the remaining priority in the queue.
        // TODO: Don't need to accumulate this. Instead, we can remove
        // renderLanes from the original lanes.
        currentlyRenderingFiber.lanes = mergeLanes(
          currentlyRenderingFiber.lanes,
          updateLane
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
    } while (update !== null && update !== first); // ❗❗/>
    if (newBaseQueueLast === null) {
      newBaseState = newState;
    } else {
      newBaseQueueLast.next = (newBaseQueueFirst: any);
    }
    // Mark that the fiber performed work, but only if the new state is
    // different from the current state.
    if (!is(newState, hook.memoizedState)) {
      markWorkInProgressReceivedUpdate();
    }
    hook.memoizedState = newState;
    hook.baseState = newBaseState;
    hook.baseQueue = newBaseQueueLast;
    queue.lastRenderedState = newState;
  }
  return [hook.memoizedState, dispatch];
}
```

큰 내용이지만 핵심적인 부분에만 집중해 보겠습니다.

```typescript
do {
  const updateLane = update.lane;
  if (!isSubsetOfLanes(renderLanes, updateLane)) {
    // Priority is insufficient. Skip this update. If this is the first
    // skipped update, the previous update/state is the new base
    // update/state.
    ...
  }
  update = update.next;
} while (update !== null && update !== first);
```

네, update를 통해 루프를 도는데 `isSubsetOfLanes()`으로 레인을 확인하고, `renderLanes`는 [`renderWithHooks()`](https://github.com/facebook/react/blob/5a1e558df21bd3cafbaea01cc418fa69d14a8cab/packages/react-reconciler/src/ReactFiberHooks.old.js#L378)에서 설정되고, 역추적하고, 루트 함수 호출은 `performUnitOfWork()`에서 이뤄집니다.

```typescript
next = beginWork(current, unitOfWork, subtreeRenderLanes);
```

휴, 이야기 끝났습니다. 지금까지 레인이 어떻게 작동하는지 대략적으로 살펴봤습니다.

## 5\. 요약

1. 파이버에서 이벤트가 발생하면 몇 가지 요인에 의해 결정되는 레인 정보로 업데이트가 생성됩니다.
    
2. 조상 파이버들은 `childLanes`와 함께 표시되므로, 모든 파이버에 대해 하위 노드의 레인 정보를 얻을 수 있습니다.
    
3. 루트에서 우선순위가 가장 높은 레인 가져오기 → 스케줄러 우선순위에 매핑하기 → 스케줄러에서 작업을 예약하여 파이버 트리 조정하기
    
4. 조정에서 작업할 우선 순위가 가장 높은 레인(현재 렌더링 레인)을 선택합니다.
    
5. 파이버 트리를 순회하고, 훅의 업데이트를 확인하고, 렌더링 레인에 레인이 포함된 업데이트를 실행합니다.
    

따라서 단일 파이버에서 여러 업데이트를 개별적으로 실행할 수 있습니다.

## 6\. 하지만 레인의 요점은 무엇일까요? 몇 가지 예를 살펴보겠습니다.

데모는 천 개 이상의 단어를 설명합니다.

### 6.1 데모 - 긴 목록 렌더링으로 입력이 차단됨

[첫 번째 데모](https://jser.dev/demos/react/lanes-priority/without-transition.html)를 열고 입력란에 [](https://jser.dev/demos/react/lanes-priority/without-transition.html)무언가를 입력하면 지연이 발생하고 입력란이 응답하지 않는 것을 느낄 수 있습니다.

[![](https://jser.dev/static/lanes-without-transition.gif align="left")](https://jser.dev/static/lanes-without-transition.gif)

각 셀의 렌더링에 딜레이를 적용했기 때문에 렉이 발생합니다.

```typescript
function Cell() {
  const start = Date.now();
  while (Date.now() - start < 1) {} // ❗❗ 딜레이!
  return <span className={`cell ${COLORS[Math.round(Math.random())]}`} />;
}
function _Cells() {
  return (
    <div className="cells">
      {new Array(1000).fill(0).map((_, index) => (
        <Cell key={index} />
      ))}
    </div>
  );
}
const Cells = React.memo(_Cells);
function App() {
  const [text, setText] = useState("");
  return (
    <div className="app">
      <input
        type="text"
        value={text}
        onChange={(e) => {
          setText(e.target.value);
        }}
      />
      <Cells text={text} />
    </div>
  );
}
```

두 번째 데모를 참조하여 `startTransition()`을 사용하여 `<Cells>`을 업데이트하기 위한 레인을 분리하여 사례를 개선할 수 있습니다.

### 6.2 데모 - 무거운 작업을 Transtion lanes로 이동해서 입력이 차단되지 않음

[두 번째 데모](https://jser.dev/demos/react/lanes-priority/with-transition.html)를 열어 사용해 보세요.

[![](https://jser.dev/static/lanes-with-transition.gif align="left")](https://jser.dev/static/lanes-with-transition.gif)

입력은 즉시 반응하는 반면 셀은 나중에 렌더링되는 것을 볼 수 있습니다.

`<Cell />`의 업데이트를 transition lanes로 옮겼기 때문입니다.

```typescript
function App() {
  const [text, setText] = useState("");
  const deferredText = React.useDeferredValue(text); // ❗❗ 
  return (
    <div className="app">
      <input
        type="text"
        value={text}
        onChange={(e) => {
          setText(e.target.value);
        }}
      />
      <Cells text={deferredText} />
    </div>
}
```

여기서 비결은 transition lanes에 업데이트를 넣는 `useDeferredValue()`입니다. 이 기본 제공 API에 대한 자세한 내용은 이 에피소드 - [React.useDeferredValue()는 어떻게 작동하나요?](https://ted-projects.com/react-internals-deep-dive-17) 를 확인하세요.

또한 개발 도구를 열면 이 두 가지의 차이점을 확인할 수 있습니다.

첫 번째:

```typescript
pendingLanes 0000000000000000000000000000001
pendingLanes 0000000000000000000000000000001
performSyncWorkOnRoot()
lanes to work on  0000000000000000000000000000001
workLoopSync
pendingLanes 0000000000000000000000000000000
pendingLanes 0000000000000000000000000000000
```

레인이 하나만 있는 것을 볼 수 있습니다. 즉, 입력과 셀에 대한 업데이트가 동일한 배치로 처리됩니다.

두 번째 데모의 상황은 조금 다릅니다.

```typescript
pendingLanes 0000000000000000000000000000001
pendingLanes 0000000000000000000000000000001
performSyncWorkOnRoot()
lanes to work on  0000000000000000000000000000001
workLoopSync
pendingLanes 0000000000000000000000001000000
pendingLanes 0000000000000000000000001000000
pendingLanes 0000000000000000000000001000000
performConcurrentWorkOnRoot()
pendingLanes 0000000000000000000000001000000
```

두 개의 패스가 있는 것을 볼 수 있는데, 첫 번째 패스는 입력에 대한 SyncLane이지만 Cell에 대한 패스는 `TransitionLane1`입니다.

### 6.3 데모 - 내부 API를 사용하여 예약하기.

[세 번째 데모](https://jser.dev/demos/react/lanes-priority/with-schedule-api.html)도 쉽게 이해할 수 있습니다.

```typescript
function App() {
  const [num, setNum] = React.useState(1);
  const renders = React.useRef([]);
  renders.current.push(num);
  return (
    <div>
      <button
        onClick={() => {
          setCurrentUpdatePriority(4); // ❗❗
          setNum((num) => num + 1);
          setCurrentUpdatePriority(1); // ❗❗
          setNum((num) => num * 10);
        }}
      >
        click me
      </button>
      {renders.current.map((log, i) => (
        <p key={i}>{log}</p>
      ))}
    </div>
  );
}
```

`seState()`를 동시에 두 번 호출했지만 각각 다른 업데이트 우선순위(레인)를 사용하여 첫 번째 호출은 `InputContinuousLane`, 두 번째 호출은 `SyncLane`으로 설정했습니다.

그렇다면 어떤 결과를 기대하시나요? (💬아래 내용 확인을 바로 안하고 예상해보세요!)

(💬의도적 공백)

우선순위를 고려하지 않으면 `1 -> 20`으로 함께 처리한다고 생각할 수 있습니다.

실제 결과는 `1 -> 10 -> 20`입니다.

개발자 도구를 열고 버튼을 클릭하면 무슨 일이 일어나고 있는지 확인할 수 있습니다.

```typescript
pendingLanes 0000000000000000000000000000100
pendingLanes 0000000000000000000000000000101
pendingLanes 0000000000000000000000000000101
performSyncWorkOnRoot()
lanes to work on  0000000000000000000000000000001
workLoopSync
render App() with state  10
pendingLanes 0000000000000000000000000000100
pendingLanes 0000000000000000000000000000100
performConcurrentWorkOnRoot()
pendingLanes 0000000000000000000000000000100
lanes to work on 0000000000000000000000000000100
shouldTimeSlice false
workLoopSync
render App() with state  20
pendingLanes 0000000000000000000000000000000
pendingLanes 0000000000000000000000000000000
```

먼저 SyncLane을 처리했으므로 `1 * 10 = 10`, 나머지 레인을 처리한 다음 일관성을 위해 SyncLane의 훅 업데이트가 여전히 실행되어야 하므로 `(1 + 1) * 10 = 20`이 됩니다.

이번 에피소드는 여기까지입니다. React 내부를 더 잘 이해하는 데 도움이 되었기를 바랍니다.

(원본 게시일: 2022-03-26)