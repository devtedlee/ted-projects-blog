---
title: "[번역] React 동시성 모드에서 Suspense는 어떻게 동작하는가 2 - Offscreen 컴포넌트"
datePublished: Wed Apr 17 2024 04:25:33 GMT+0000 (Coordinated Universal Time)
cuid: clv3b7qnq000w08k0g8rfc53r
slug: react-internals-deep-dive-7-2
cover: https://cdn.hashnode.com/res/hashnode/image/upload/v1713327784180/b7c4df86-f43a-49e2-a2c4-645cb3d7ff91.jpeg
tags: react-internals

---

> ***영문 블로그 글을 번역했습니다. 허가를 받으면 시리즈를 이어갈 예정입니다.  
> 원문링크:*** [https://jser.dev/react/2022/04/17/offscreen-component](https://jser.dev/react/2022/04/17/offscreen-component)

---

> ***ℹ️*** [***React Internals Deep Dive***](https://jser.dev/series/react-source-code-walkthrough.html) ***에피소드 23,*** [***유튜브에서 제가 설명하는 것***](https://www.youtube.com/watch?v=o21owCSvJHw&list=PLvx8w9g4qv_p-OS-XdbB3Ux_6DMXhAJC3&index=23)***을 시청해주세요.***
> 
> ***⚠*** [***React@18.2.0***](https://github.com/facebook/react/releases/tag/v18.2.0) ***기준, 최신 버전에서는 구현이 변경되었을 수 있습니다.***

[Suspense에서의 조정](https://ted-projects.com/react-internals-deep-dive-7-1)에 대한 이전 게시물에서 Suspense의 내부에서 Offscreen 컴포넌트가 어떻게 사용되는지 살펴봤습니다.

React 18의 [릴리즈 노트](https://react.dev/blog/2022/03/29/react-v18#what-is-concurrent-react)에서 React 팀은 이 Offscreen 컴포넌트를 출시한 의도를 언급했습니다.

이번 에피소드에서는 Offscreen 컴포넌트가 어떻게 작동하는지 자세히 살펴보겠습니다.

## 데모

Offscreen 컴포넌트를 사용해 보려면 실험적(experimental) 기능이 켜져 있는 빌드를 사용해야 합니다.

[여기에 데모](https://jser.dev/demos/react/offscreen/)를 올려두었으니 직접 사용해 보세요.

```typescript
const Offscreen = React.unstable_Offscreen;
function Component() {
  const [count, setCount] = React.useState(0);
  console.log("render Component: count => ", count);
  React.useLayoutEffect(() => {
    console.log("render Component: layout effect in Component");
  }, []);
  React.useEffect(() => {
    console.log("render Component: effect in Component");
    setCount((_) => _ + 1);
  }, []);
  return <p>{count}</p>;
}
function App() {
  const [hidden, setHidden] = React.useState(true);
  console.log("render App");
  return (
    <div>
      <button onClick={() => setHidden((_) => !_)}>toggle</button>
      <Offscreen mode={hidden ? "hidden" : "visible"}>
        <Component />
      </Offscreen>
    </div>
  );
}
const rootElement = document.getElementById("container");
ReactDOM.createRoot(rootElement).render(<App />);
```

콘솔을 열면 몇 가지 흥미로운 사항을 확인할 수 있습니다.

1. Component가 보이지 않을 때
    
    * 여전히 렌더링 됩니다.
        
    * 패시브 effect는 실행됩니다.
        
    * 레이아웃 effect는 실행되지 않습니다.
        
    * renderRootConcurrent가 두 번 호출됩니다.
        
2. 버튼을 클릭한 후 Component가 표시되면
    
    * 레이아웃 effect가 실행됩니다
        

"render"는 internal React 파이버 트리를 구성하거나 업데이트하는 것을 의미하며, 준비가 완료되면 커밋 단계에서 DOM에 반영된다는 것을 알고 있습니다.

또한 보이지 않는 state를 검사하면 숨겨진 요소(element)가 있지만 숨겨져 있을 뿐이라는 것을 알 수 있습니다.

[![](https://jser.dev/static/offscreen-style-hidden.png align="left")](https://jser.dev/static/offscreen-style-hidden.png)

따라서 위의 동작을 통해 Offscreen이 수행하는 작업은 다음과 같습니다.

**어떻게든 보이지 않는 콘텐츠의 렌더링을 지연(defer)시키고 CSS를 사용하여 숨깁니다.**

## OffscreenComponent의 데이터 타입

```typescript
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
export type OffscreenProps = {|
  // TODO: Pick an API before exposing the Offscreen type. I've chosen an enum
  // for now, since we might have multiple variants. For example, hiding the
  // content without changing the layout.
  //
  // Default mode is visible. Kind of a weird default for a component
  // called "Offscreen." Possible alt: <Visibility />?
  mode?: OffscreenMode | null | void,
  children?: ReactNodeList,
|};
// We use the existence of the state object as an indicator that the component
// is hidden.
export type OffscreenState = {|
  // TODO: This doesn't do anything, yet. It's always NoLanes. But eventually it
  // will represent the pending work that must be included in the render in
  // order to unhide the component.
  baseLanes: Lanes,
  cachePool: SpawnedCachePool | null,
|};
export type OffscreenInstance = {};
export type OffscreenMode =
  | "hidden"
  | "unstable-defer-without-hiding"
  | "visible";
```

1. `REACT_OFFSCREEN_TYPE`은 Offscreen element의 타입으로, `hidden` `visible` 또는 `unstable-defer-without-hiding`을 모드로 설정합니다.
    
2. `OffscreenState`는 null이 아닌 경우, Offscreen이 보이지 않음을 의미하는 중요한 값입니다.
    

다음은 Suspense에서 `createFiberFromOffscreen()` 이 어떻게 사용되는지 보여주는 기본 예시입니다.

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

여기서 우리는 'visible' Offscreen 파이버를 만들어 Suspense 자식을 자식으로 감싸고 있습니다.

## Offscreen Component 조정하기(Reconciling)

Offscreen은 스위치처럼 작동하기 때문에, 실제 호스트 노드에 대한 참조를 보유하지 않고 업데이트 조정 메서드만 가지고 있습니다. ([소스](https://github.com/facebook/react/blob/8e2f9b086e7abc7a92951d264a6a5d048defd914/packages/react-reconciler/src/ReactFiberBeginWork.new.js#L636))

```typescript
function updateOffscreenComponent(
  current: Fiber | null,
  workInProgress: Fiber,
  renderLanes: Lanes
) {
  const nextProps: OffscreenProps = workInProgress.pendingProps;
  const nextChildren = nextProps.children;
  const prevState: OffscreenState | null =
    current !== null ? current.memoizedState : null;
  if (nextProps.mode === "hidden" || enableLegacyHidden) {
    // Rendering a hidden tree.
    if ((workInProgress.mode & ConcurrentMode) === NoMode) {
      // legacy mode
      ...
    } else if (!includesSomeLane(renderLanes, OffscreenLane)) {
      // prepare to render hidden component in OffscreenLane
      ...
    } else {
      // render hidden component in OffscreenLane
      ...
    }
  } else {
    // Rendering a visible tree.
    ...
  }
  // go to children
  {
    reconcileChildren(current, workInProgress, nextChildren, renderLanes);
    return workInProgress.child;
  }
}
```

코드가 꽤 많으므로 먼저 가장 바깥쪽에 있는 if 브랜치를 살펴봅시다. 위의 주석을 통해 다음과 같은 것을 알 수 있습니다.

1. 비록 OffscreenLane 아래에서 지만 "hidden" 상태에서도 렌더링은 계속 진행된다
    
2. OffscreenLane은 즉석에서 추가되고, 두 단계가 있는데, 하나는 준비 단계이고 다른 하나는 렌더링 단계이므로 프로세스가 지연(defer)됩니다.
    

```typescript
export const IdleLane: Lanes = /*                       */ 0b0100000000000000000000000000000;
export const OffscreenLane: Lane = /*                   */ 0b1000000000000000000000000000000;
```

OffscreenLane은 가장 낮은 우선순위를 보였으며, 심지어 IdleLane보다도 낮았는데, 이는 합리적인데 왜냐하면 만약 숨겨져 있으면, 그것은 사용자들에게 보이지 않는다는 것이고, 우리가 결국은 처리해야만 한다는 의미이기 때문입니다.

### Offscreen 렌더링은 어떻게 예약될까요?

```typescript
if (!includesSomeLane(renderLanes, (OffscreenLane: Lane))) {
      let spawnedCachePool: SpawnedCachePool | null = null;
```

앞서 언급했듯이 숨겨진 컴포넌트의 렌더링은 OffscreenLane에서 예약되므로, OffscreenLane이 현재 renderLanes에 없으면, 지금 예약해야 한다는 뜻입니다.

`spawnedCachePool`은 Cache 컴포넌트에 관한 것이므로 일단 건너뛰겠습니다.

```typescript
// We're hidden, and we're not rendering at Offscreen. We will bail out
// and resume this tree later.
let nextBaseLanes;
if (prevState !== null) {
  const prevBaseLanes = prevState.baseLanes;
  nextBaseLanes = mergeLanes(prevBaseLanes, renderLanes);
} else {
  nextBaseLanes = renderLanes;
}
```

이전 baseLanes와 현재 rendeLanes를 병합하여 baseLanes를 준비하는데, 이는 약간 까다로운 작업입니다.

아래의 경우를 가정해 보겠습니다.

1. SyncLane에서 렌더링 중이며, 타깃 파이버는 Offscreen 컴포넌트 아래에 있습니다.
    
2. Offscreen 컴포넌트에서 벗어났으며, 이는 파이버를 업데이트할 수 없음을 의미합니다.
    
3. OffscreenLane에서 보이지 않는 렌더링을 계속할 때, 우리는 SyncLane을 포함해야 할 필요가 있습니다.
    

따라서 `OffscreenState.baseLanes`는 이전에 건너뛴 작업 레인을 저장하는 방법입니다. 이를 보여주기 위해 [다른 데모](https://jser.dev/demos/react/offscreen//work-left/)를 만들었습니다.

```typescript
function Component({ onClick }) {
  const [count, setCount] = React.useState(0);
  console.log("visible, render Component:", count);
  return (
    <div>
      <button
        onClick={() => {
          setCount((_) => _ + 1);
          onClick();
        }}
      >
        schedule work and hide the offscreen component
      </button>
      <p>{count}</p>
    </div>
  );
}
function App() {
  const [hidden, setHidden] = React.useState(false);
  console.log("render App");
  return (
    <div>
      <button onClick={() => setHidden((_) => !_)}>
        toggle offscreen component
      </button>
      <Offscreen mode={hidden ? "hidden" : "visible"}>
        <Component
          onClick={() => {
            setHidden(true);
          }}
        />
      </Offscreen>
    </div>
  );
}
```

데모와 콘솔을 열고, 두 번째 버튼을 클릭해보면,

1. `subtreeRenderLanes is set to 00000000000000000000000000000001` =&gt; 이벤트 액션이 동기화 레인에 있음.
    
2. `pushrenderLanes 1` =&gt; Offscreen이 bail out 됨
    
3. `2nd render set to NoLanes 1000000000000000000000000000000` =&gt; OffscreenLane에서 다시 렌더링합니다.
    
4. `subtreeRenderLanes is set to 1000000000000000000000000000001` =&gt; 건너 뛰었던 첫 번째 SyncLane을 결합합니다.
    
5. `enough priority` =&gt; `<Component>`를 렌더링할 때, SyncLane에 예약되어 있기 때문에 충분한 우선순위를 갖습니다.
    

휴, 너무 많네요. OffscreenLane이 어떻게 예약되는지 다시 살펴보겠습니다.

```typescript
// Schedule this fiber to re-render at offscreen priority. Then bailout.
workInProgress.lanes = workInProgress.childLanes = laneToLanes(OffscreenLane);
```

이 줄은 이 Offscreen 컴포넌트를 리-렌더링해야 한다는 것을 표시하기 위해 `lanes`를 설정하는 중요한 줄입니다. [조정중에 React bailout이 작동하는 방식](https://jser.dev/react/2022/01/07/how-does-bailout-work)에서 이 문제를 다룬 적이 있습니다.

`childLanes`를 루트로 바로 업데이트하는 데 도움이 되는 `markUpdateLaneFromFiberToRoot()`가 있지만, 여기에는 그런 호출이 없고 다른 로직으로 처리됩니다. 계속해 보겠습니다.

```typescript
const nextState: OffscreenState = {
  baseLanes: nextBaseLanes,
  cachePool: spawnedCachePool,
};
workInProgress.memoizedState = nextState;
workInProgress.updateQueue = null;
// We're about to bail out, but we need to push this to the stack anyway
// to avoid a push/pop misalignment.
pushRenderLanes(workInProgress, nextBaseLanes);
return null;
```

`OffscreenState`를 생성하고 이 Offscreen에 설정합니다. `OffscreenState`는 보이지 않음을 나타내는 것임을 기억하세요.

우리는 `return null`에 익숙해져야 하며, 이는 조정자(reconciler)에게 bailout이 발생하고, 더 깊이 진행하지 말고 완료를 시작해야 한다는 것을 알리는 것입니다.

`completeWork()` 내부. ([소스](https://github.com/facebook/react/blob/8e2f9b086e7abc7a92951d264a6a5d048defd914/packages/react-reconciler/src/ReactFiberCompleteWork.new.js#L832))

```typescript
case OffscreenComponent:
case LegacyHiddenComponent: {
  popRenderLanes(workInProgress);
  var _nextState = workInProgress.memoizedState;
  var nextIsHidden = _nextState !== null;
  if (current !== null) {
    var _prevState2 = current.memoizedState;
    var prevIsHidden = _prevState2 !== null;
    if (
      prevIsHidden !== nextIsHidden && // LegacyHidden doesn't do any hiding — it only pre-renders.
      !enableLegacyHidden
    ) {
      workInProgress.flags |= Visibility;
    }
  }
  if (
    !nextIsHidden ||
    (workInProgress.mode & ConcurrentMode) === NoMode
  ) {
    bubbleProperties(workInProgress);
  } else {
    // Don't bubble properties for hidden children unless we're rendering
    // at offscreen priority.
    if (includesSomeLane(subtreeRenderLanes, OffscreenLane)) {
      bubbleProperties(workInProgress);
      {
        // Check if there was an insertion or update in the hidden subtree.
        // If so, we need to hide those nodes in the commit phase, so
        // schedule a visibility effect.
        if (workInProgress.subtreeFlags & (Placement | Update)) {
          workInProgress.flags |= Visibility;
        }
      }
    }
  }
  popTransition(workInProgress, current);
  return null;
}
```

1. `prevIsHidden !== nextIsHidden` 또는 자식에 삽입 등이 있는지 확인하여 업데이트가 필요함을 나타내는 `Visibility` 플래그를 설정합니다.
    
2. `bubbleProperties()`가 호출됩니다.
    

그리고 실제로 `lanes`는 이 함수에서 수집되어 부모 파이버의 `childLanes`로 설정됩니다.

```typescript
function bubbleProperties(completedWork) {
  var didBailout =
    completedWork.alternate !== null &&
    completedWork.alternate.child === completedWork.child;
  var newChildLanes = NoLanes;
  var subtreeFlags = NoFlags;
  if (!didBailout) {
    // Bubble up the earliest expiration time.
    if ((completedWork.mode & ProfileMode) !== NoMode) {
      // In profiling mode, resetChildExpirationTime is also used to reset
      // profiler durations.
      var actualDuration = completedWork.actualDuration;
      var treeBaseDuration = completedWork.selfBaseDuration;
      var child = completedWork.child;
      while (child !== null) {
        newChildLanes = mergeLanes(
          newChildLanes,
          mergeLanes(child.lanes, child.childLanes)
        );
        subtreeFlags |= child.subtreeFlags;
        subtreeFlags |= child.flags; // When a fiber is cloned, its actualDuration is reset to 0. This value will
        // only be updated if work is done on the fiber (i.e. it doesn't bailout).
        // When work is done, it should bubble to the parent's actualDuration. If
        // the fiber has not been cloned though, (meaning no work was done), then
        // this value will reflect the amount of time spent working on a previous
        // render. In that case it should not bubble. We determine whether it was
        // cloned by comparing the child pointer.
        actualDuration += child.actualDuration;
        treeBaseDuration += child.treeBaseDuration;
        child = child.sibling;
      }
      completedWork.actualDuration = actualDuration;
      completedWork.treeBaseDuration = treeBaseDuration;
    } else {
      var _child = completedWork.child;
      while (_child !== null) {
        newChildLanes = mergeLanes(
          newChildLanes,
          mergeLanes(_child.lanes, _child.childLanes)
        );
        subtreeFlags |= _child.subtreeFlags;
        subtreeFlags |= _child.flags; // Update the return pointer so the tree is consistent. This is a code
        // smell because it assumes the commit phase is never concurrent with
        // the render phase. Will address during refactor to alternate model.
        _child.return = completedWork;
        _child = _child.sibling;
      }
    }
    completedWork.subtreeFlags |= subtreeFlags;
  } else {
    // Bubble up the earliest expiration time.
    if ((completedWork.mode & ProfileMode) !== NoMode) {
      // In profiling mode, resetChildExpirationTime is also used to reset
      // profiler durations.
      var _treeBaseDuration = completedWork.selfBaseDuration;
      var _child2 = completedWork.child;
      while (_child2 !== null) {
        newChildLanes = mergeLanes(
          newChildLanes,
          mergeLanes(_child2.lanes, _child2.childLanes)
        ); // "Static" flags share the lifetime of the fiber/hook they belong to,
        // so we should bubble those up even during a bailout. All the other
        // flags have a lifetime only of a single render + commit, so we should
        // ignore them.
        subtreeFlags |= _child2.subtreeFlags & StaticMask;
        subtreeFlags |= _child2.flags & StaticMask;
        _treeBaseDuration += _child2.treeBaseDuration;
        _child2 = _child2.sibling;
      }
      completedWork.treeBaseDuration = _treeBaseDuration;
    } else {
      var _child3 = completedWork.child;
      while (_child3 !== null) {
        newChildLanes = mergeLanes(
          newChildLanes,
          mergeLanes(_child3.lanes, _child3.childLanes)
        ); // "Static" flags share the lifetime of the fiber/hook they belong to,
        // so we should bubble those up even during a bailout. All the other
        // flags have a lifetime only of a single render + commit, so we should
        // ignore them.
        subtreeFlags |= _child3.subtreeFlags & StaticMask;
        subtreeFlags |= _child3.flags & StaticMask; // Update the return pointer so the tree is consistent. This is a code
        // smell because it assumes the commit phase is never concurrent with
        // the render phase. Will address during refactor to alternate model.
        _child3.return = completedWork;
        _child3 = _child3.sibling;
      }
    }
    completedWork.subtreeFlags |= subtreeFlags;
  }
  completedWork.childLanes = newChildLanes;
  return didBailout;
}
```

이것 또한 복잡하지만, 형제(siblings) 플래그와 lanes를 모으는 while 루프가 있고, 마지막 줄은 부모 파이버에 반영하는 것을 볼 수 있습니다.

`completeWork()`에서 하는 이유는 합리적인데, 어차피 여기서 조상(ancester) 파이버를 순회할 것이므로 여기서 하는 것입니다.

또한 한 번의 렌더링이 완료되면 항상 `ensureRootIsScheduled()`가 호출되므로, 첫 번째 패스(pass)가 끝나면, React는 할 일이 더 있는지 확인하고 두 번째 패스에서 OffscreenLane을 찾아 렌더링할 수 있게 됩니다.

### Offscreen 렌더링은 어떻게 이루어지나요?

```typescript
// This is the second render. The surrounding visible content has already
// committed. Now we resume rendering the hidden tree.
// Rendering at offscreen, so we can clear the base lanes.
const nextState: OffscreenState = {
  baseLanes: NoLanes,
  cachePool: null,
};
workInProgress.memoizedState = nextState;
// Push the lanes that were skipped when we bailed out.
const subtreeRenderLanes =
  prevState !== null ? prevState.baseLanes : renderLanes;
pushRenderLanes(workInProgress, subtreeRenderLanes);
```

여기에 화려한 것은 없습니다, 한 가지 차이점은 `return null`이 없기 때문에 조정자(reconciler)가 자식에게 내려가서 렌더링 된 `<Component/>`가 보인다는 점입니다.

### 어떻게 렌더링 됐는데 DOM에서는 숨겨질 수 있는건가요?

DOM 조작이 커밋 단계에 있다는 것을 알았으니, 답을 찾아보겠습니다.

하지만 본격적으로 들어가기 전에 `completeWork()`에서 내재적(intrinsic) DOM 요소를 처리하는 방법을 기억해 봅시다.

```typescript
case HostComponent: {
  popHostContext(workInProgress);
  const rootContainerInstance = getRootHostContainer();
  const type = workInProgress.type;
  ...
  const instance = createInstance(
    type,
    newProps,
    rootContainerInstance,
    currentHostContext,
    workInProgress,
  );
  appendAllChildren(instance, workInProgress, false, false);
  workInProgress.stateNode = instance;
  ...
```

많은 줄을 생략했지만, DOM 구성이 `completeWork()`에서 이루어지고 있음은 알 수 있습니다. 하지만 Offscreen 컴포넌트에는 처리할 DOM 노드가 없습니다.

하지만 `appendAllChildren()`은 실제로 Offscreen 컴포넌트를 건너뛰고 그 자식들에서 DOM 노드를 수집하겠죠?

좋은 질문입니다, 보이지 않는 콘텐츠의 렌더링이 지연되고 completeWork가 두 번 호출된다는 점을 기억하세요. 첫 번째 패스(pass)에서, 아직 생성된 DOM 요소가 없으므로 아무것도 추가되지 않으며, 두 번째 패스에서, 모든 props가 변경되지 않으므로 여기서 아무 작업도 수행하지 않습니다.

이 DOM(Offscreen에 있는 숨겨진 DOM)은 커밋 단계에서 연결되는데, 제 추측으로는 이것을 동기식으로 숨겨야 하기에 중단(interupt)이 없는 커밋 단계가 유일한 선택입니다.

> 이것은 'invisible state'에 대해서만 해당되며, 보이는 경우에는 정상적으로 모든 자식을 추가합니다.

### 어떻게 Offscreen 컴포넌트가 hidden =&gt; visible로 처리되나요?

`updateOffscreenComponent()` 에서는

```typescript
// Rendering a visible tree.
let subtreeRenderLanes;
if (prevState !== null) {
  // We're going from hidden -> visible.
  subtreeRenderLanes = mergeLanes(prevState.baseLanes, renderLanes);
  let prevCachePool = null;
  if (enableCache) {
    // If the render that spawned this one accessed the cache pool, resume
    // using the same cache. Unless the parent changed, since that means
    // there was a refresh.
    prevCachePool = prevState.cachePool;
  }
  pushTransition(workInProgress, prevCachePool, null);
  // Since we're not hidden anymore, reset the state
  workInProgress.memoizedState = null;
} else {
  // We weren't previously hidden, and we still aren't, so there's nothing
  // special to do. Need to push to the stack regardless, though, to avoid
  // a push/pop misalignment.
  subtreeRenderLanes = renderLanes;
  if (enableCache) {
    // If the render that spawned this one accessed the cache pool, resume
    // using the same cache. Unless the parent changed, since that means
    // there was a refresh.
    if (current !== null) {
      pushTransition(workInProgress, null, null);
    }
  }
}
pushRenderLanes(workInProgress, subtreeRenderLanes);
```

여기에는 멋진 기능은 없으며, 단지 `memoizedState`를 지워 visible을 나타냅니다, 마법은 여기에 있지 않습니다.

## Offscreen 컴포넌트는 hide/unhide를 커밋 단계에서 정합니다.

사실상 마법은 커밋 단계입니다, `commitMutationEffectsOnFiber()`. ([소스](https://github.com/facebook/react/blob/8e2f9b086e7abc7a92951d264a6a5d048defd914/packages/react-reconciler/src/ReactFiberCommitWork.new.js#L1986))

```typescript
case OffscreenComponent: {
  var _wasHidden = current !== null && current.memoizedState !== null;
  // Before committing the children, track on the stack whether this
  // offscreen subtree was already hidden, so that we don't unmount the
  // effects again.
  var prevOffscreenSubtreeWasHidden = offscreenSubtreeWasHidden;
  offscreenSubtreeWasHidden = prevOffscreenSubtreeWasHidden || _wasHidden;
  recursivelyTraverseMutationEffects(root, finishedWork);
  offscreenSubtreeWasHidden = prevOffscreenSubtreeWasHidden;
  commitReconciliationEffects(finishedWork);
  if (flags & Visibility) {
    var _newState = finishedWork.memoizedState;
    var _isHidden = _newState !== null;
    var offscreenBoundary = finishedWork;
    {
      // TODO: This needs to run whenever there's an insertion or update
      // inside a hidden Offscreen tree.
      hideOrUnhideAllChildren(offscreenBoundary, _isHidden);
    }
    {
      if (_isHidden) {
        if (!_wasHidden) {
          if ((offscreenBoundary.mode & ConcurrentMode) !== NoMode) {
            nextEffect = offscreenBoundary;
            var offscreenChild = offscreenBoundary.child;
            while (offscreenChild !== null) {
              nextEffect = offscreenChild;
              disappearLayoutEffects_begin(offscreenChild);
              offscreenChild = offscreenChild.sibling;
            }
          }
        }
      }
    }
  }
  return;
}
```

두 가지 작업이 수행되었음을 알 수 있습니다.

1. 가시성(visivility)이 변경되면 `hideOrUnhideAllChildren()`을 호출 합니다.
    
2. visible이 되면 레이아웃 effect를 트리거합니다.
    

```typescript
function hideOrUnhideAllChildren(finishedWork, isHidden) {
  // Only hide or unhide the top-most host nodes.
  let hostSubtreeRoot = null;
  if (supportsMutation) {
    // We only have the top Fiber that was inserted but we need to recurse down its
    // children to find all the terminal nodes.
    let node: Fiber = finishedWork;
    while (true) {
      if (node.tag === HostComponent) {
        if (hostSubtreeRoot === null) {
          hostSubtreeRoot = node;
          try {
            const instance = node.stateNode;
            if (isHidden) {
              hideInstance(instance);
            } else {
              unhideInstance(node.stateNode, node.memoizedProps);
            }
          } catch (error) {
            captureCommitPhaseError(finishedWork, finishedWork.return, error);
          }
        }
      } else if (node.tag === HostText) {
        if (hostSubtreeRoot === null) {
          try {
            const instance = node.stateNode;
            if (isHidden) {
              hideTextInstance(instance);
            } else {
              unhideTextInstance(instance, node.memoizedProps);
            }
          } catch (error) {
            captureCommitPhaseError(finishedWork, finishedWork.return, error);
          }
        }
      } else if (
        (node.tag === OffscreenComponent ||
          node.tag === LegacyHiddenComponent) &&
        (node.memoizedState: OffscreenState) !== null &&
        node !== finishedWork
      ) {
        // Found a nested Offscreen component that is hidden.
        // Don't search any deeper. This tree should remain hidden.
      } else if (node.child !== null) {
        node.child.return = node;
        node = node.child;
        continue;
      }
      if (node === finishedWork) {
        return;
      }
      while (node.sibling === null) {
        if (node.return === null || node.return === finishedWork) {
          return;
        }
        if (hostSubtreeRoot === node) {
          hostSubtreeRoot = null;
        }
        node = node.return;
      }
      if (hostSubtreeRoot === node) {
        hostSubtreeRoot = null;
      }
      node.sibling.return = node.return;
      node = node.sibling;
    }
  }
}
```

기본적으로 `memoizedState` 때문에 숨겨져 있음을 알 수 있으며, null이 되면, visible이라는 것을 의미합니다.

visible이 되면, 자식으로 내려가서 첫 번째 HostComponent 또는 HostText를 찾을 때까지 계속 시도합니다.

그런 다음 Host 컴포넌트가 visible인 경우 `hideInstance()` 및 `unhideInstance()`를 통해 숨김으로 설정합니다. while 루프는 첫 번째 수준까지만 내려가는걸 유의해주세요.

```typescript
export function hideInstance(instance: Instance): void {
  // TODO: Does this work for all element types? What about MathML? Should we
  // pass host context to this method?
  instance = ((instance: any): HTMLElement);
  const style = instance.style;
  if (typeof style.setProperty === "function") {
    style.setProperty("display", "none", "important");
  } else {
    style.display = "none";
  }
}
export function hideTextInstance(textInstance: TextInstance): void {
  textInstance.nodeValue = "";
}
export function unhideInstance(instance: Instance, props: Props): void {
  instance = ((instance: any): HTMLElement);
  const styleProp = props[STYLE];
  const display =
    styleProp !== undefined &&
    styleProp !== null &&
    styleProp.hasOwnProperty("display")
      ? styleProp.display
      : null;
  instance.style.display = dangerousStyleValue("display", display);
}
export function unhideTextInstance(
  textInstance: TextInstance,
  text: string
): void {
  textInstance.nodeValue = text;
}
```

그리고 그것은 CSS `display:none`으로 이루어집니다. 맙소사, 더 멋진 줄 알았는데요. 따라서 디버거를 설정하면 아래와 같이 DOM 변경 후 스타일이 설정됩니다.

[![](https://jser.dev/static/offscreen-visiblity-style.gif align="left")](https://jser.dev/static/offscreen-visiblity-style.gif)

잠깐만요, DOM은 언제 연결되나요?

`commitReconciliationEffects(finishedWork)`에 있습니다. [소스](https://github.com/facebook/react/blob/8e2f9b086e7abc7a92951d264a6a5d048defd914/packages/react-reconciler/src/ReactFiberCommitWork.new.js#L2316).

```typescript
function commitReconciliationEffects(finishedWork: Fiber) {
  // Placement effects (insertions, reorders) can be scheduled on any fiber
  // type. They needs to happen after the children effects have fired, but
  // before the effects on this fiber have fired.
  const flags = finishedWork.flags;
  if (flags & Placement) {
    try {
      commitPlacement(finishedWork);
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
```

여기에 DOM 노드를 삽입하고, 즉시 숨김으로 설정합니다. 동기식이기 때문에, 브라우저는 깜박이는 state를 렌더링할 기회가 없습니다.

## 요약

Offscreen 컴포넌트는 다음과 같이 동작합니다.

1. `visible`또는 `hidden` state를 가지고 있습니다.
    
2. `hidden`이면, 첫 번째 패스(pass)에서 OffscreenLane에 의한 bailout으로 조정(reconcile)을 지연(defer)시킵니다.
    
3. `visible`이면, 정상적으로 조정됩니다.
    
4. `completeWork`에서
    
    * state(visible/hidden)가 변경될 경우 `Visibility` 플래그가 설정됩니다.
        
    * 보이는(visible) DOM이 여기에 삽입됩니다.
        
5. 커밋 단계에서
    
    * 숨겨진(hidden) DOM이 여기에 삽입됩니다.
        
    * `Visibility` 플래그가 있는 경우 React는 DOM 노드를 hides / unhides 합니다.
        

왜 이렇게 복잡하게 만들었을지 궁금한가요? 그냥 스타일을 이용한 CSS 트릭을 사용하면 될텐데요. 네, 이 프로세스의 전반적인 목적은 **숨겨진 항목의 렌더링 우선순위를 낮추는** 것인데, 이것이 바로 동시성 모드의 백미(💬gold 의역)입니다.

(원본 게시일: 2022-04-17)