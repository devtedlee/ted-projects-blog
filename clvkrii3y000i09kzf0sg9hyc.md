---
title: "[번역] React useRef()는 어떻게 동작하나요?"
datePublished: Mon Apr 29 2024 09:33:54 GMT+0000 (Coordinated Universal Time)
cuid: clvkrii3y000i09kzf0sg9hyc
slug: react-internals-deep-dive-11
cover: https://cdn.hashnode.com/res/hashnode/image/upload/v1714383034290/11124e43-c881-4ad8-88a8-e723230078dc.jpeg
tags: reactjs, react-internals, useref

---

> ***영문 블로그 글을 번역했습니다. 허가를 받으면 시리즈를 이어갈 예정입니다.  
> 원문링크:***[https://jser.dev/react/2021/12/05/how-does-useRef-work](https://jser.dev/react/2021/12/05/how-does-useRef-work)

---

> ***ℹ️***[***React Internals Deep Dive***](https://jser.dev/series/react-source-code-walkthrough.html) ***에피소드 11,*** [***유튜브에서 제가 설명하는 것***](https://www.youtube.com/watch?v=q-B5XalyNpI&list=PLvx8w9g4qv_p-OS-XdbB3Ux_6DMXhAJC3&index=11)***을 시청해주세요.***
> 
> ***⚠***[***React@18.2.0***](https://github.com/facebook/react/releases/tag/v18.2.0)***기준, 최신 버전에서는 구현이 변경되었을 수 있습니다.***
> 
> ***💬 역자 주석: Jser의 코멘트는 ❗❗로 표시 해뒀습니다.  
> 그 외 주석은 리액트 소스 코드 자체의 주석입니다.  
> ... 은 생략된 코드입니다.***

전 당신이 `useRef()`를 꽤 많이 사용하셨을거라고 생각하는데요, 그 내부를 알아봅시다.

## **1\. useRef() 소개**

`useRef()`로 ref를 생성한 다음 프로그래밍 방식으로 `.current` 속성을 설정하거나 DOM 요소에서 `ref` 프로퍼티로 사용합니다.

```typescript
function Component() {
  const ref = useRef(null);
  return <div ref={ref} />;
}
```

이제 여기서 두 가지 퍼즐을 풀어보겠습니다.

1. 최초 렌더링과 리렌더링에서 `useRef()`는 어떻게 작동할까요?
    
2. `ref={ref}`는 어떻게 작동할까요?
    

## 2\. `useRef()`는 어떻게 동작할까요?

앞에서 다룬 것처럼 최초 렌더링과 리렌더링에 사용되는 `mountRef()` 및 `updateRef()`를 직접 조사해 봅시다.

### 2.1 mountRef()

```typescript
function mountRef<T>(initialValue: T): {| current: T |} {
  const hook = mountWorkInProgressHook();
  const ref = { current: initialValue };
  hook.memoizedState = ref;
  return ref;
}
```

이것은 매우 간단합니다

1. `current` 프로퍼티를 가진 ref 객체를 만들고
    
2. `mountWorkInProgressHook()`으로 새로운 훅을 만들고
    
3. ref를 위해 `memoizedState`에 ref를 설정해줍니다.
    

`memoizedState`의 네이밍은 무시해도 됩니다. 이는 state 따위를 보관하기 위한 훅의 내부 이름일 뿐입니다.

`mountRef()`는 매우 간단합니다. `updateRef()`의 경우 다시 `ref` 객체를 반환하면 되겠죠? 계속 진행하겠습니다.

### 2.2 updateRef()

```typescript
function updateRef<T>(initialValue: T): {| current: T |} {
  const hook = updateWorkInProgressHook();
  return hook.memoizedState;
}
```

맞습니다, 너무 간단하죠.

이전 동영상들에서 언급했듯이 `updateWorkInProgressHook()` 은 내부 커서를 사용하여 파이버의 훅 목록을 살펴보는 것입니다. 최초 렌더링의 경우 목록이 비어 있으므로 매번 새로운 훅을 생성합니다. 리렌더링의 경우 이미 훅이 있으므로 해당 훅을 사용합니다.

## 3\. `ref={ref}`는 어떻게 동작하나요?

`useRef()` 는 매우 간단한데, 프로그래밍 방식으로 `current`를 설정하는 것은 해당 프로퍼티의 값만 변경하는 것이므로, `useState()` 와는 달리 업데이트를 트리거하지 않습니다.

더 흥미로운 질문은 DOM 요소로 설정하는 방법인데, 여기에는 두 가지 하위 질문이 포함되어 있습니다.

1. 어떻게 `ref`가 연결(attached)되는가
    
2. 어떻게 `ref`가 분리(detached)되는가
    

### 3.1 어떻게 `ref`가 연결(attached)되나요?

제가 어떻게 알아냈는지에 대한 자세한 내용은 [제 동영상](https://www.youtube.com/watch?v=q-B5XalyNpI)을 참조하실 수 있으며, 여기서는 중요한 부분만 나열했습니다.

커밋 단계에서, `commitAttachRef()`는 `commitLayoutEffectOnFiber()` 내부에서 호출됩니다.

```typescript
function commitAttachRef(finishedWork: Fiber) {
  const ref = finishedWork.ref;
  if (ref !== null) {
    const instance = finishedWork.stateNode;
    let instanceToUse;
    switch (finishedWork.tag) {
      case HostComponent:
        instanceToUse = getPublicInstance(instance);
        break;
      default:
        instanceToUse = instance;
    }
    // Moved outside to ensure DCE works with this flag
    if (enableScopeAPI && finishedWork.tag === ScopeComponent) {
      instanceToUse = instance;
    }
    if (typeof ref === "function") {
      let retVal;
      retVal = ref(instanceToUse);
    } else {
      ref.current = instanceToUse;
    }
  }
}
```

HostComponent인 DOM 요소의 경우 여기에서 DOM 노드가 설정되어 있는 것을 볼 수 있습니다. 또한 콜백 ref 또는 ref 객체를 허용합니다.

이는 ref의 연결(attaching)이 레이아웃 이펙트와 같은 단계에서 발생한다는 것을 의미하며, 여기에 콜백 ref가 사용된다면 `useEffect`보다 더 빨리 연결될 수 있으며, 이는 [React 고급 패턴 - Ref 를 통한 재사용 가능한 동작 훅](https://jser.dev/react/2022/02/18/reusable-behavior-hooks-through-ref)을 이해하는 데 도움이 될 수 있습니다.

### 3.2 어떻게 `ref`가 분리(detached) 되나요?

이 함수 아래에는 분리 함수가 있습니다.

```typescript
function commitDetachRef(current: Fiber) {
  const currentRef = current.ref;
  if (currentRef !== null) {
    if (typeof currentRef === "function") {
      currentRef(null);
    } else {
      currentRef.current = null;
    }
  }
}
```

콜백 ref 또는 ref 객체도 지원하는 것을 볼 수 있습니다. `commitDetachRef()`를 검색해보면, 일관성을 유지하기 위해 먼저 분리해야 하므로 `commitLayoutEffectOnFiber()`보다 훨씬 빠른 `commitMutationEffectsOnFiber()`에서 발생하는 것을 볼 수 있습니다.

### 3.3 React는 `flags`를 통해 연결 또는 분리의 필요 여부를 알 수 있습니다.

위의 함수들이 호출되기 전에 몇 가지 확인할 사항들이 있습니다.

```typescript
if (finishedWork.flags & Ref) {
  commitAttachRef(finishedWork);
}
if (flags & Ref) {
  const current = finishedWork.alternate;
  if (current !== null) {
    commitDetachRef(current);
  }
}
```

이들은 동일한 플래그 `Ref`를 사용하며, `Ref` 플래그가 설정되어 있으면 ref가 변경되었음을 의미하며, 분리를 위해 `current`를 확인하고 null이 아닌 경우 이전 파이버가 더 이상 사용되지 않으므로 이전 파이버의 ref를 분리합니다.

`Ref`는 언제 설정되나요? `markRef()`[(소스](https://github.com/facebook/react/blob/848e802d203e531daf2b9b0edb281a1eb6c5415d/packages/react-reconciler/src/ReactFiberBeginWork.old.js#L943))에서 수행됩니다.

```typescript
function markRef(current: Fiber | null, workInProgress: Fiber) {
  const ref = workInProgress.ref;
  if (
    (current === null && ref !== null) ||
    (current !== null && current.ref !== ref)
  ) {
    // Schedule a Ref effect
    workInProgress.flags |= Ref;
    if (enableSuspenseLayoutEffectSemantics) {
      workInProgress.flags |= RefStatic;
    }
  }
}
```

ref 생성 및 ref 변경을 확인하는 것을 볼 수 있습니다.

`markRef()`는 조정(reconciliation) 내부에 있는 `updateHostComponent()`에서 호출됩니다.[(소스](https://github.com/facebook/react/blob/848e802d203e531daf2b9b0edb281a1eb6c5415d/packages/react-reconciler/src/ReactFiberBeginWork.old.js#L1378))

```typescript
function updateHostComponent(
  current: Fiber | null,
  workInProgress: Fiber,
  renderLanes: Lanes
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
  markRef(current, workInProgress); // ❗❗
  reconcileChildren(current, workInProgress, nextChildren, renderLanes);
  return workInProgress.child;
}
```

좋습니다, 이제 refs에서 무슨 일이 일어나는지 알 수 있습니다.

## 4\. 요약

1. 조정하는 동안, ref 변경/생성은 `flags`의 파이버에 표시(mark)됩니다.
    
2. 커밋하는 동안, 리액트는 `flags`를 확인하여 ref를 분리/연결 합니다.
    
3. `useRef()` 는 ref 객체만 보유하는 간단한 훅입니다.