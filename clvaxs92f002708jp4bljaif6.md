---
title: "[번역] 리액트에서 useTransition()은 어떻게 동작하나요?"
datePublished: Mon Apr 22 2024 12:31:45 GMT+0000 (Coordinated Universal Time)
cuid: clvaxs92f002708jp4bljaif6
slug: react-internals-deep-dive-8
cover: https://cdn.hashnode.com/res/hashnode/image/upload/v1713789041776/b1384840-df7b-4a0a-ad16-118ed759a216.jpeg
tags: react-internals

---

> ***영문 블로그 글을 번역했습니다. 허가를 받으면 시리즈를 이어갈 예정입니다.  
> 원문링크:***[https://jser.dev/2023-05-19-how-does-usetransition-work](https://jser.dev/2023-05-19-how-does-usetransition-work)

---

> ***ℹ️***[***React Internals Deep Dive***](https://jser.dev/series/react-source-code-walkthrough.html)***에피소드 8,***[***유튜브에서 제가 설명하는 것***](https://www.youtube.com/watch?v=G0sHIjjiyJ0&list=PLvx8w9g4qv_p-OS-XdbB3Ux_6DMXhAJC3&index=8)***을 시청해주세요.***
> 
> ***⚠***[***React@18.2.0***](https://github.com/facebook/react/releases/tag/v18.2.0)***기준, 최신 버전에서는 구현이 변경되었을 수 있습니다.***
> 
> ***💬 역자 주석: Jser의 코멘트는 ❗❗로 표시 해뒀습니다.  
> 그 외 주석은 리액트 소스 코드 자체의 주석입니다.  
> ... 은 생략된 코드입니다.***

## 1\. useTransition()은 무엇을 하나요?

이 기능이 무엇인지 알고 싶으시다면 [react.dev의 공식 문서](https://react.dev/reference/react/useTransition)에서 가장 잘 설명되어 있습니다.

사용 방법은 다음과 같습니다.

```typescript
function TabContainer() {
  const [isPending, startTransition] = useTransition();
  const [tab, setTab] = useState('about');
  function selectTab(nextTab) {
    startTransition(() => {
      setTab(nextTab);
    });
  }
}
```

`startTransition()` 안에 `setState()` 호출을 넣으면 업데이트가 전환(transition)으로 표시되므로 우선순위가 낮고 두 가지 주요 의미를 갖게 됩니다.

1. 이제 업데이트를 중단할 수 있습니다 [(데모)](https://react.dev/reference/react/useTransition#marking-a-state-update-as-a-non-blocking-transition).
    
2. Suspense 폴백(fallback)의 깜박임 없음 [(데모](https://react.dev/reference/react/useTransition#preventing-unwanted-loading-indicators))
    

React 소스 코드를 살펴봄으로써 어떻게 작동하는지 알아봅시다.

## 2\. useTransition()은 어떻게 동작하나요?

`useTransition()` 은 훅이며, 이 시리즈의 수많은 에피소드를 통해 어디서 찾을 수 있는지 잘 알고 있습니다. 네, `ReactFiberHooks.js에` 있습니다.

### 2.1 mountTransition()

`mountTransition()`은 최초 렌더링 때 사용됩니다.

```typescript
function mountTransition() {
  const stateHook = mountStateImpl((false: Thenable<boolean> | boolean));
        // ❗❗       ↗ 이것은 mountState() 내부와 동일합니다,
        // ❗❗ 즉 우리는 이것을 useState()의 내부 호출이 있는것과 동일하게 여길 수 있습니다. 

  // The `start` method never changes.
  const start = startTransition.bind(
    null,
    currentlyRenderingFiber,
    stateHook.queue,
    true,
    false,
  );
  const hook = mountWorkInProgressHook();
   // ❗❗ ↗ startTransition()을 보유하는 기본 훅

  hook.memoizedState = start;
  return [false, start];
  // ❗❗ 최초 isPending은 false 입니다.
}
```

위 코드는 `const [isPending, startTransition] = useTransition()`의 구문을 설명합니다.

또한 `mountTransition()`이 내부에 2개의 훅을 생성하는 것을 볼 수 있습니다:

1. `isPending:boolean`을 보유하는 state 훅
    
2. `startTransition()`을 보유하는 또 다른 훅
    

이는 다음 섹션에서 중요합니다.

### 2.2 startTransition()은 2개의 state 업데이트 (하나의 normal 와 하나의 under transition)를 트리거합니다.

```typescript
function startTransition<S>(
  fiber: Fiber,
  queue: UpdateQueue<S | Thenable<S>, BasicStateAction<S | Thenable<S>>>,
  pendingState: S,
    // ❗❗ ↖ true, mountTransition() 에서 바인딩된 값
  finishedState: S,
    // ❗❗ ↖ false, mountTransition() 에서 바인딩된 값
  callback: () => mixed,
    // ❗❗ ↖ 전달 받은 callback
  options?: StartTransitionOptions,
): void {
  const previousPriority = getCurrentUpdatePriority();
  setCurrentUpdatePriority(
    higherEventPriority(previousPriority, ContinuousEventPriority),
  );
  const prevTransition = ReactCurrentBatchConfig.transition;
  if (enableAsyncActions) {
    // We don't really need to use an optimistic update here, because we
    // schedule a second "revert" update below (which we use to suspend the
    // transition until the async action scope has finished). But we'll use an
    // optimistic update anyway to make it less likely the behavior accidentally
    // diverges; for example, both an optimistic update and this one should
    // share the same lane.
    dispatchOptimisticSetState(fiber, false, queue, pendingState);
    // ❗❗ ↖ 낙관적 업데이트는 다른 주제이므로 지금은 건너뛰겠습니다.
  } else {
    ReactCurrentBatchConfig.transition = null;
    // ❗❗ ↖ 여기서 ReactCurrentBatchConfig.transition이 null로 설정되어 있음을 주목하세요.
    dispatchSetState(fiber, queue, pendingState);
    // ❗❗ ↖ 이건 기본적으로 setStat(true)와 같습니다.
  }
  const currentTransition = (ReactCurrentBatchConfig.transition =
    ({}: BatchConfigTransition));
    // ❗❗ 이제부터 ReactCurrentBatchConfig.transition이 Non-null로 설정되어 있음을 확인하십시오.

  ...
  try {
    if (enableAsyncActions) {
      ...
    } else {
      // Async actions are not enabled.
      dispatchSetState(fiber, queue, finishedState);
    // ❗❗ 이것이 false면 isPending이 false로 되돌아갑니다.
    // ❗❗ setState(true) 은 이미 호출됐습니다, 그러니 여기 있는것 두 번째 setState() 호출입니다.
    // ❗❗ 하지만 이번 호출은 좀 다른데, ReactCurrentBatchConfig.transition이 null이 아니라는것 때문입니다.
      callback();
    // ❗❗ ↖ 콜백은 null이 아닌 ReactCurrentBatchConfig.transition 아래에서 실행된다는 점에 유의하세요.
    }
  } catch (error) {
    ...
  }
}
```

여기서 까다로운 부분은, `setState()`를 두 번 호출하는 것인데, `setState()`가 파이버에 다시 실행해야 한다는 플래그를 설정하고, 그리고 다음으로 루트에서 리-렌더링이 예약 되는 것을 알기 때문입니다. 이는 동기식이 *아니므로* 두 번의 `setState()` 호출이 모두 처리됩니다.

그러나 이 두 호출은 내부 데이터 구조에서 서로 다른 우선순위를 가지며, 서로 다른 Lanes을 가지고 있습니다. 리-렌더링할 때마다 가장 높은 Lane의 업데이트가 선택됩니다. (이에 대한 이해는 [React 소스 코드에서 레인이란 무엇인가요?](https://jser.dev/react/2022/03/26/lanes-in-react) 를 참조하세요.

이 두 호출의 유일한 차이점은 `ReactCurrentBatchConfig.transition`뿐이므로 Lane 설정을 변경하려면 어딘가에서 이 호출을 사용해야 합니다.

### 2.3 requestUpdateLane()은 `ReactCurrentBatchConfig.transition이` 설정된 경우 Transition Lanes를 반환합니다.

`setState()` 내에서 `requestUpdateLane()` 이 호출됩니다.

```typescript
function dispatchSetState<S, A>(
    // ❗❗ ↗ 이것은 setState()의 내부 구현입니다.
  fiber: Fiber,
  queue: UpdateQueue<S, A>,
  action: A,
): void {
  
  const lane = requestUpdateLane(fiber);
    // ❗❗      ↗ 업데이트의 레인(우선순위)은 고정되지 않고 동적으로 결정됩니다.

  const update: Update<S, A> = {
    lane, // ❗❗ lane
    revertLane: NoLane,
    action,
    hasEagerState: false,
    eagerState: null,
    next: (null: any),
  };
  ...
}
```

이제 `requestUpdateLane()` 자체를 살펴보겠습니다.

```typescript
export function requestCurrentTransition(): Transition | null {
  return ReactCurrentBatchConfig.transition;
    // ❗❗ ReactCurrentBatchConfig.transition은 전역 플래그처럼 작동하여,
    // ❗❗ 전환(transition)을 나타냅니다.
}
export function requestUpdateLane(fiber: Fiber): Lane {
  ...
  const isTransition = requestCurrentTransition() !== NoTransition;
  if (isTransition) {
    const actionScopeLane = peekEntangledActionLane();
    return actionScopeLane !== NoLane
      ? // We're inside an async action scope. Reuse the same lane.
        actionScopeLane
      : // We may or may not be inside an async action scope. If we are, this
        // is the first update in that scope. Either way, we need to get a
        // fresh transition lane.
        requestTransitionLane();
        // ❗❗ ↖ 만약 isTransition이면, Transition Lane을 반환합니다.
  }
  ...
}
```

### 2.4 Transition Lanes는 낮은 우선순위의 레인입니다.

```typescript
export function requestTransitionLane(): Lane {
  // The algorithm for assigning an update to a lane should be stable for all
  // updates at the same priority within the same event. To do this, the
  // inputs to the algorithm must be the same.
  //
  // The trick we use is to cache the first of each of these inputs within an
  // event. Then reset the cached values once we can be sure the event is
  // over. Our heuristic for that is whenever we enter a concurrent work loop.
  if (currentEventTransitionLane === NoLane) {
    // All transitions within the same event are assigned the same lane.
    currentEventTransitionLane = claimNextTransitionLane();
  }
  return currentEventTransitionLane;
}
export function claimNextTransitionLane(): Lane {
  // Cycle through the lanes, assigning each new transition to the next lane.
  // In most cases, this means every transition gets its own lane, until we
  // run out of lanes and cycle back to the beginning.
  const lane = nextTransitionLane;
  nextTransitionLane <<= 1;
  // ❗❗ ↖ 왼쪽 shifting(<<)으로 다음 transition lane으로 비트 이동합니다. 
  if ((nextTransitionLane & TransitionLanes) === NoLanes) {
    nextTransitionLane = TransitionLane1;
    // ❗❗ ↖ 만약 더이상의 transition lane이 없으면, 제일 처음 trasition lane을 사용합니다.
  }
  return lane;
}
export const NoLanes: Lanes = /*                        */ 0b0000000000000000000000000000000;
export const NoLane: Lane = /*                          */ 0b0000000000000000000000000000000;
export const SyncHydrationLane: Lane = /*               */ 0b0000000000000000000000000000001;
export const SyncLane: Lane = /*                        */ 0b0000000000000000000000000000010;
export const InputContinuousHydrationLane: Lane = /*    */ 0b0000000000000000000000000000100;
export const InputContinuousLane: Lane = /*             */ 0b0000000000000000000000000001000;
export const DefaultHydrationLane: Lane = /*            */ 0b0000000000000000000000000010000;
export const DefaultLane: Lane = /*                     */ 0b0000000000000000000000000100000;
export const SyncUpdateLanes: Lane = /*                 */ 0b0000000000000000000000000101010;
const TransitionLanes: Lanes = /*                       */ 0b0000000011111111111111110000000;
    // ❗❗                                             ↗ 총 16개의 transition lane이 있으며,
    // ❗❗                                        이 레인들은 SyncLane 등 보다 우선순위가 낮습니다.
const TransitionLane1: Lane = /*                        */ 0b0000000000000000000000010000000;
const TransitionLane2: Lane = /*                        */ 0b0000000000000000000000100000000;
const TransitionLane3: Lane = /*                        */ 0b0000000000000000000001000000000;
const TransitionLane4: Lane = /*                        */ 0b0000000000000000000010000000000;
const TransitionLane5: Lane = /*                        */ 0b0000000000000000000100000000000;
const TransitionLane6: Lane = /*                        */ 0b0000000000000000001000000000000;
const TransitionLane7: Lane = /*                        */ 0b0000000000000000010000000000000;
const TransitionLane8: Lane = /*                        */ 0b0000000000000000100000000000000;
const TransitionLane9: Lane = /*                        */ 0b0000000000000001000000000000000;
const TransitionLane10: Lane = /*                       */ 0b0000000000000010000000000000000;
const TransitionLane11: Lane = /*                       */ 0b0000000000000100000000000000000;
const TransitionLane12: Lane = /*                       */ 0b0000000000001000000000000000000;
const TransitionLane13: Lane = /*                       */ 0b0000000000010000000000000000000;
const TransitionLane14: Lane = /*                       */ 0b0000000000100000000000000000000;
const TransitionLane15: Lane = /*                       */ 0b0000000001000000000000000000000;
const TransitionLane16: Lane = /*                       */ 0b0000000010000000000000000000000;
```

Transition 레인은 우선순위가 낮기 때문에 중단될 수 있습니다. 이는 동시성 모드의 핵심(💬gold 의역)이며, 어떻게 중단될 수 있는지 알아보려면 [스케줄러 작동 방식](https://jser.dev/react/2022/03/16/how-react-scheduler-works)을 참조하세요.

### 2.5 `updateTransition()`

이는 최초 마운트 후 `useTransition()`이 호출되는 경우입니다.

```typescript
function updateTransition(): [
  boolean,
  (callback: () => void, options?: StartTransitionOptions) => void,
] {
  const [booleanOrThenable] = updateState(false);
  const hook = updateWorkInProgressHook();
  const start = hook.memoizedState;
  const isPending =
    typeof booleanOrThenable === 'boolean'
      ? booleanOrThenable
      : // This will suspend until the async action scope has finished.
        useThenable(booleanOrThenable);
  return [isPending, start];
}
```

따라서 코드를 보면, 기본적으로 훅에서 데이터를 반환하는 것을 알 수 있습니다.

### 2.6 `startTransition()`

`startTransition()`은 컴포넌트 외부에서 사용할 수 있는 전역 명령형 API입니다.

```typescript
export function startTransition(
  scope: () => void,
  options?: StartTransitionOptions,
) {
  const prevTransition = ReactCurrentBatchConfig.transition;
  ReactCurrentBatchConfig.transition = ({}: BatchConfigTransition);
    // ❗❗                ↗ transition을 위한 글로벌 플래그만을 설정합니다.

  const currentTransition = ReactCurrentBatchConfig.transition;
  try {
    scope();
  } finally {
    ReactCurrentBatchConfig.transition = prevTransition;
  }
}
```

이건 꽤 간단합니다, 마치 `isPending`이 없는 `useTransition()` 함수의 일부 같습니다.

## 3\. 학습 내용을 통해 데모를 더 잘 이해하기

공식 문서에 나열된 사용 사례를 살펴보고 어떻게 이게 가능한지 알아보겠습니다.

### 3.1 사용 사례 1 - state 업데이트를 논-블록 transition으로 표시하기

[데모](https://react.dev/reference/react/useTransition#examples)

`useTransition()`을 사용하지 않으면 다음과 같은 일이 발생합니다.

* 'Posts(slow)'를 클릭합니다:
    
    * SyncLane(클릭 이벤트이므로 `DiscreteEventPriority`에서 매핑됨)의 업데이트가 파이버에 설정됩니다.
        
    * 루트에서 리-렌더링이 예약됩니다.
        
* 루트에서 리-렌더링합니다:
    
    * SyncLane이기 때문에 동시성 모드가 *아니며*, React는 모든 것을 한 번에 렌더링하려고 시도합니다.
        
    * PostList 렌더링이 차단되어, 다른 버튼 클릭이 작동하지 않습니다.
        

위의 설명을 이해하기 위해 리-렌더링 예약이 어떻게 작동하는지 보여주는 다음 코드를 읽어보겠습니다.

> ℹ [React 스케줄러는 어떻게 작동하는가?](https://jser.dev/react/2022/03/16/how-react-scheduler-works/#1-to-begin-with) 에서도 관련 주제를 설명합니다.

```typescript
// Use this function to schedule a task for a root. There's only one task per
// root; if a task was already scheduled, we'll check to make sure the priority
// of the existing task is the same as the priority of the next level that the
// root has work on. This function is called on every update, and right before
// exiting a task.
function ensureRootIsScheduled(root: FiberRoot, currentTime: number) {
  const existingCallbackNode = root.callbackNode;
  // Check if any lanes are being starved by other work. If so, mark them as
  // expired so we know to work on those next.
  markStarvedLanesAsExpired(root, currentTime);
  // Determine the next lanes to work on, and their priority.
  const nextLanes = getNextLanes(
    // ❗❗          ↗ getNextLanes() 은 최우선순위 레인을 반환 합니다.
    // ❗❗             만약 SyncLane과 Transition Lanes들이 같이 있으면, SyncLane이 선택될 것입니다.
    root === workInProgressRoot ? workInProgressRootRenderLanes : NoLanes,
  );
  if (nextLanes === NoLanes) {
    // Special case: There's nothing to work on.
    if (existingCallbackNode !== null) {
      cancelCallback(existingCallbackNode);
    }
    root.callbackNode = null;
    root.callbackPriority = NoLane;
    return;
  }
  ...
  if (existingCallbackNode != null) {
    // Cancel the existing callback. We'll schedule a new one below.
    cancelCallback(existingCallbackNode);
    // ❗❗ ↖ 이게 중요합니다!!
    // ❗❗ 만약 리-렌더링이 끝나지 않으면, 우리는 새로운 것을 예약하고,
    // ❗❗ 오래된 것은 취소되게 됩니다.
    // ❗❗ 이게 바로 중단(interruption)이 발생하는 방식입니다.
  // Schedule a new callback.
  let newCallbackNode;
  if (newCallbackPriority === SyncLane) {
    // Special case: Sync React callbacks are scheduled on a special
    // internal queue
    if (root.tag === LegacyRoot) {
      scheduleLegacySyncCallback(performSyncWorkOnRoot.bind(null, root));
    } else {
      scheduleSyncCallback(performSyncWorkOnRoot.bind(null, root));
    // ❗❗                 ↗ SyncLane의 경우, 조정(reconciliation)은 동시성 모드가 아닌 동기화 작업입니다. 
    // ❗❗                 즉, 메인 스레드에 양보(yield)하지 않으며 잠재적으로 차단이 될 수 있습니다.
    }
    if (supportsMicrotasks) {
      scheduleMicrotask(() => {
        // In Safari, appending an iframe forces microtasks to run.
        // https://github.com/facebook/react/issues/22459
        // We don't support running callbacks in the middle of render
        // or commit so we need to check against that.
        if (
          (executionContext & (RenderContext | CommitContext)) ===
          NoContext
        ) {
          // Note that this would still prematurely flush the callbacks
          // if this happens outside render or commit phase (e.g. in an event).
          flushSyncCallbacks();
        }
      });
    } else {
      // Flush the queue in an Immediate task.
      scheduleCallback(ImmediateSchedulerPriority, flushSyncCallbacks);
    }
    newCallbackNode = null;
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
    // ❗❗ 만약 SyncLane이 아니면, 동시성 모드가 사용되고,
    // ❗❗ 조정은 때떄로 메인 스레드에 양보(yield)합니다.
    // ❗❗ UI가 익터렉티브 해지고 이전 리-렌더링을 취소할 수 있습니다.
    );
  }
  root.callbackPriority = newCallbackPriority;
  root.callbackNode = newCallbackNode;
}
```

위의 지식이 있으면, `useTransition()`을 사용할 때 데모가 다르게 동작하는 이유를 쉽게 이해할 수 있습니다.

* "Posts(slow)"를 클릭합니다.
    
    * TransitionLane(SyncLane 아님)의 업데이트가 파이버에 설정됩니다.
        
    * 루트에서 리-렌더링이 스케줄링됩니다.
        
* 루트에서 리-렌더링합니다:
    
    * SyncLane이 아니기 때문에 동시 모드가 사용되며, PostList를 계속 렌더링합니다.
        
    * 비록 PostList가 무겁지만, 동시 모드는 버튼을 클릭할 수 있도록 때때로 메인 스레드에 양보(yield)합니다.
        
* "Contact"를 클릭합니다.
    
    * TransitionLane의 업데이트가 파이버에 다시 설정됩니다.
        
    * 루트에서 리-렌더링이 예약되었지만, 기존 리-렌더링이 아직 완료되지 않았으므로 취소합니다.
        
    * 이전 리렌더링이 중단되고, 새로고침 리-렌더링이 발생하여, 최신 state가 렌더링됩니다.
        

### 3.2 사용 사례 2 - transition에서 부모 컴포넌트 업데이트하기

[데모](https://react.dev/reference/react/useTransition#updating-the-parent-component-in-a-transition)

사용 사례 1에서 트랜지션은 스케줄러의 작업에 대한 용어이므로 특정 파이버가 아닌 전체 리-렌더링에 관한 것이므로 부모에 대한 `useTransition()`도 작동합니다.

실제로 글로벌 `startTransition()` API를 사용하면 어디서나 전환(transition)을 트리거할 수 있습니다.

### 3.3 사용 사례 3 - transition 중에 보류 중인 시각적 state 표시

[데모](https://react.dev/reference/react/useTransition#displaying-a-pending-visual-state-during-the-transition)

[2.2절](https://jser.dev/2023-05-19-how-does-usetransition-work#22-starttransition-triggers-2-state-updates---one-normal-and-one-under-transition)에서 `useTransition()` 내부에는 의미상 두 번의 `setState()` 호출이 있는데, 첫 번째 호출은 트랜지션이 아니므로 SyncLane이고 `isPending`을 유지하는 state가 성공적으로 설정된다는 점을 기억하세요.

이것이 바로 indicator(표시기)를 볼 수 있는 이유입니다.

### 3.4 사용 사례 4 - 원치 않는 로딩 표시기 방지하기

이것은 더 흥미로운데, 위의 사용 사례들과는 다른 의미를 가지고 있습니다. React.dev에 [더 자세한 설명](https://react.dev/reference/react/Suspense#preventing-already-revealed-content-from-hiding)이 있습니다.

Suspense 콘텐츠를 폴백으로 전환해야 하는 시나리오에 대한 내용이므로, Suspense와 관련된 코드를 읽어보겠습니다.

> ℹ 서스펜스 자체에 대한 자세한 내용은 [동시 모드에서 서스펜스가 내부적으로 작동하는 방식을](https://ted-projects.com/react-internals-deep-dive-7-1) 참조하세요.

```typescript
function completeWork(
  current: Fiber | null,
  workInProgress: Fiber,
  renderLanes: Lanes,
): Fiber | null {
  switch (workInProgress.tag) {
    ...
    case SuspenseComponent: {
      popSuspenseContext(workInProgress);
      const nextState: null | SuspenseState = workInProgress.memoizedState;
      }
      const nextDidTimeout = nextState !== null;
      const prevDidTimeout =
        current !== null &&
        (current.memoizedState: null | SuspenseState) !== null;
      ...
      // If the suspended state of the boundary changes, we need to schedule
      // a passive effect, which is when we process the transitions
      if (nextDidTimeout !== prevDidTimeout) {
        if (enableTransitionTracing) {
          const offscreenFiber: Fiber = (workInProgress.child: any);
          offscreenFiber.flags |= Passive;
        }
        // If the suspended state of the boundary changes, we need to schedule
        // an effect to toggle the subtree's visibility. When we switch from
        // fallback -> primary, the inner Offscreen fiber schedules this effect
        // as part of its normal complete phase. But when we switch from
        // primary -> fallback, the inner Offscreen fiber does not have a complete
        // phase. So we need to schedule its effect here.
        //
        // We also use this flag to connect/disconnect the effects, but the same
        // logic applies: when re-connecting, the Offscreen fiber's complete
        // phase will handle scheduling the effect. It's only when the fallback
        // is active that we have to do anything special.
        if (nextDidTimeout) {
          const offscreenFiber: Fiber = (workInProgress.child: any);
          offscreenFiber.flags |= Visibility;
          // TODO: This will still suspend a synchronous tree if anything
          // in the concurrent tree already suspended during this render.
          // This is a known bug.
          if ((workInProgress.mode & ConcurrentMode) !== NoMode) {
            // TODO: Move this back to throwException because this is too late
            // if this is a large tree which is common for initial loads. We
            // don't know if we should restart a render or not until we get
            // this marker, and this is too late.
            // If this render already had a ping or lower pri updates,
            // and this is the first time we know we're going to suspend we
            // should be able to immediately restart from within throwException.
            const hasInvisibleChildContext =
              current === null &&
              (workInProgress.memoizedProps.unstable_avoidThisFallback !==
                true ||
                !enableSuspenseAvoidThisFallback);
            if ( // ❗❗ 여기서 부터 아래 ❗❗ 까지에 대한 설명
              hasInvisibleChildContext ||
              hasSuspenseContext(
                suspenseStackCursor.current,
                (InvisibleParentSuspenseContext: SuspenseContext),
              )
            ) {
              // If this was in an invisible tree or a new render, then showing
              // this boundary is ok.
              renderDidSuspend();
            } else {
              // Otherwise, we're going to have to hide content so we should
              // suspend for longer if possible.
              renderDidSuspendDelayIfPossible();
              // ❗❗ ↖ 이름에서 알 수 있듯이 React는 폴백으로 돌아가는 것을 피하려고 시도 합니다.
            }
          }
        }
      }
    }
  }
}
export function renderDidSuspendDelayIfPossible(): void {
  if (
    workInProgressRootExitStatus === RootInProgress ||
    workInProgressRootExitStatus === RootSuspended ||
    workInProgressRootExitStatus === RootErrored
  ) {
    workInProgressRootExitStatus = RootSuspendedWithDelay;
    // ❗❗ ↖ 이것은 전체 리-렌더링 결과를 저장하는 전역 변수입니다.
  }
  ...
}
```

따라서 React는 콘텐츠가 이미 공개된 후에 Suspense 폴백이 렌더링되는 것을 좋아하지 않으며, 이는 매우 합리적입니다.

`RootSuspendedWithDelay`는 React가 DOM에 변경 사항을 커밋하기 직전에 검사하는 상태입니다.

```typescript
function finishConcurrentRender(
  root: FiberRoot,
  exitStatus: RootExitStatus,
  finishedWork: Fiber,
  lanes: Lanes,
) {
  // TODO: The fact that most of these branches are identical suggests that some
  // of the exit statuses are not best modeled as exit statuses and should be
  // tracked orthogonally.
  switch (exitStatus) {
    case RootInProgress:
    case RootFatalErrored: {
      throw new Error('Root did not complete. This is a bug in React.');
    }
    case RootSuspendedWithDelay: {
      if (includesOnlyTransitions(lanes)) {
        // This is a transition, so we should exit without committing a
        // placeholder and without scheduling a timeout. Delay indefinitely
        // until we receive more data.
        markRootSuspended(root, lanes);
        return;
    // ❗❗ ↗ 여기서 모든 레인들이 RootSuspendedWithDelay 아래의 트랜지션인 경우 반환되며,
    // ❗❗ 의미상으로는 Suspense 폴백으로 콘텐츠를 다시 깜박이는 것을 좋아하지 않으며,
    // ❗❗ 우리는 이미 낮은 우선순위(트랜지션)인 상태라는 것을 알고 있습니다.
      }
      // Commit the placeholder.
      break;
    }
    case RootErrored:
    case RootSuspended:
    case RootCompleted: {
      break;
    }
    default: {
      throw new Error('Unknown root exit status.');
    }
  }
  ...
  commitRootWhenReady(
    root,
    finishedWork,
    workInProgressRootRecoverableErrors,
    workInProgressTransitions,
    lanes,
  );
}
```

여기서 마법을 볼 수 있습니다. 트랜지션 중에 React가 Suspense 콘텐츠를 폴백으로 되돌려야 하는 상황이 발생하면, React는 이를 무시하고 변경 사항을 DOM에 커밋하지 않습니다. 그리고 Suspense가 작동하는 방식 때문에, 던져진(throw) thenable이 해결(resolve)되면 올바른 콘텐츠를 렌더링하는 리-렌더링이 예약됩니다.

## 4\. 요약

`useTransition()`은 특정 업데이트의 우선순위를 낮출 수 있는 강력한 도구입니다. 클릭과 같은 이벤트는 실제로 동시성 모드에서 업데이트를 트리거하기 때문에 `useTransition()`은 동시성 모드의 명시적 opt-in 역할을 합니다.

업데이트 차단을 피하는 것보다 더 나은 사용 사례는 **불필요한 Suspense 폴백을 피하는것**이므로 가능한 한 많이 사용해야 한다고 생각합니다.