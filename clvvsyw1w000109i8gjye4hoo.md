---
title: "[번역] React.memo()는 어떻게 동작하나요?"
datePublished: Tue May 07 2024 03:00:06 GMT+0000 (Coordinated Universal Time)
cuid: clvvsyw1w000109i8gjye4hoo
slug: react-internals-deep-dive-14
cover: https://cdn.hashnode.com/res/hashnode/image/upload/v1714983445317/1b53e8d5-3a2c-4e48-9ea6-a58cfe049225.jpeg
tags: reactjs, react-internals, reactmemo

---

> ***영문 블로그 글을 번역했습니다. 허가를 받으면 시리즈를 이어갈 예정입니다.  
> 원문링크:*** [https://jser.dev/react/2022/01/11/how-react-memo-works](https://jser.dev/react/2022/01/11/how-react-memo-works)

---

> ***ℹ️***[***React Internals Deep Dive***](https://jser.dev/series/react-source-code-walkthrough.html) ***에피소드 14,*** [***유튜브에서 제가 설명하는 것***](https://www.youtube.com/watch?v=0jbV6apamhs&list=PLvx8w9g4qv_p-OS-XdbB3Ux_6DMXhAJC3&index=14)***을 시청해주세요.***
> 
> ***⚠***[***React@18.2.0***](https://github.com/facebook/react/releases/tag/v18.2.0)***기준, 최신 버전에서는 구현이 변경되었을 수 있습니다.***
> 
> ***💬 역자 주석: Jser의 코멘트는 ❗❗로 표시 해뒀습니다.  
> 그 외 주석은 리액트 소스 코드 자체의 주석입니다.  
> ... 은 생략된 코드입니다.***

[조정에서 React bailout은 어떻게 작동하는가](https://ted-projects.com/react-internals-deep-dive-13)에서 하위 트리에 변경된 것이 없다고 판단될 때 React가 어떻게 조정을 중지하는지 이해했습니다.

잠깐, `React.memo()`가 하는 일과 똑같지 않나요, 맞죠?

## 1\. 데모 타임

다시 [이전 데모](https://jser.dev/demos/react/how-bailout-works/index.html)를 열어 보겠습니다. 버튼을 클릭하면 C와 D가 모두 리렌더링됩니다.

이제 코드를 조금 변경하여 `React.memo()`를 D에 적용해 보겠습니다.

다음은 [메모가 포함된 새 데모](https://jser.dev/demos/react/how-bailout-works/memo.html)이며, 이 과정을 반복하면 예상했던 대로 D가 리렌더링되지 않는 것을 확인할 수 있습니다.

[![](https://jser.dev/static/memo-1.png align="left")](https://jser.dev/static/memo-1.png)

## 2\. `React.memo()`는 새로운 엘리먼트를 만듭니다: REACT\_MEMO\_TYPE

먼저 `React.memo()` 가 사용되는 위치에 디버거를 설정하면 [memo()의 소스 코드](https://github.com/facebook/react/blob/e62a8d754548a490c2a3bcff3b420e5eedaf11c0/packages/react/src/ReactMemo.js)로 연결됩니다.

이것은 매우 간단합니다.

```typescript
export function memo<Props>(
  type: React$ElementType,
  compare?: (oldProps: Props, newProps: Props) => boolean
) {
  const elementType = {
    $$typeof: REACT_MEMO_TYPE,
    type,
    compare: compare === undefined ? null : compare,
  };
  return elementType;
}
```

코드에서 알 수 있듯이 `memo(` )는 다음과 같은 요소를 생성합니다.

1. `$$typeof`를 `REACT_MEMO_TYPE`으로 설정하고
    
2. 전달된 함수에 설정된 `type`은 `D()`입니다.
    
3. `compare` 함수를 제공합니다.
    

> ℹ *솔직히 말하면*, 저는 `React.memo()`가 두 번째 인수를 받는다는 사실을 몰랐습니다.

이제 `React.memo()` 가 모든 것을 묶는(wrap) 새로운 파이버 노드를 생성한다는 것을 알았습니다. 간단해 보이는데, `D()`에서 리렌더링을 피하기 위해 추가 로직을 추가하려면, 해당 로직으로 묶을 무언가가 필요합니다.

## 3\. MemoComponent의 조정

이제 다시 지난 포스트에서 설명한 대로 파이버 노드를 확인하고 업데이트하는 [beginWork()](https://github.com/facebook/react/blob/ceee524a8f45b97c5fa9861aec3f36161495d2e1/packages/react-reconciler/src/ReactFiberBeginWork.old.js#L3649)로 이동해 보겠습니다.

`REACT_MEMO_TYPE`이 사용되는 위치를 쉽게 찾을 수 있습니다. [소스](https://github.com/facebook/react/blob/ceee524a8f45b97c5fa9861aec3f36161495d2e1/packages/react-reconciler/src/ReactFiberBeginWork.old.js#L3827-L3862)

```typescript
case MemoComponent: {
  const type = workInProgress.type;
  const unresolvedProps = workInProgress.pendingProps;
  // Resolve outer props first, then resolve inner props.
  let resolvedProps = resolveDefaultProps(type, unresolvedProps);
  resolvedProps = resolveDefaultProps(type.type, resolvedProps);
  return updateMemoComponent(
    current,
    workInProgress,
    type,
    resolvedProps,
    renderLanes,
  );
}
case SimpleMemoComponent: {
  return updateSimpleMemoComponent(
    current,
    workInProgress,
    workInProgress.type,
    workInProgress.pendingProps,
    renderLanes,
  );
}
```

실제로는 두 가지 컴포넌트가 있는데, 하나는 `MemoComponent`이고 다른 하나는 `SimpleMemoComponent`인데, 그 이유는 곧 설명하겠습니다.

왜 `REACT_MEMO_TYPE`이 아니라 `MemoComponent`인가요? `REACT_MEMO_TYPE`은 엘리먼트에 사용되는 `$$typeof`이고, `MemoComponent`는 함수입니다.

엘리먼트에서 파이버가 생성되면 태그는 [여기에서](https://github.com/facebook/react/blob/a724a3b578dce77d427bef313102a4d0e978d9b4/packages/react-reconciler/src/ReactFiber.old.js#L540-L542) `MemoComponent`로 설정됩니다.

## 4\. `updateMemoComponent()`

`updateMemoComponent()`는 `React.memo()`가 작동하는 방식의 핵심입니다. [소스](https://github.com/facebook/react/blob/f2a59df48bec2352a4dd3b5415282a4ea240a4f8/packages/react-reconciler/src/ReactFiberBeginWork.old.js)

```typescript
function updateMemoComponent(
  current: Fiber | null,
  workInProgress: Fiber,
  Component: any,
  nextProps: any,
  renderLanes: Lanes
): null | Fiber {
  if (current === null) {
  // ❗❗ ↖ 만약 최초 만운트인 경우
    const type = Component.type;
    if (
      isSimpleFunctionComponent(type) &&
      Component.compare === null &&
      // SimpleMemoComponent codepath doesn't resolve outer props either.
      Component.defaultProps === undefined
    ) {
      let resolvedType = type;
      // If this is a plain function component without default props,
      // and with only the default shallow comparison, we upgrade it
      // to a SimpleMemoComponent to allow fast path updates.
      workInProgress.tag = SimpleMemoComponent;
      workInProgress.type = resolvedType;
      return updateSimpleMemoComponent( // ❗❗
        current,
        workInProgress,
        resolvedType,
        nextProps,
        renderLanes
      ); // ❗❗
      // ❗❗ ↖ 최적화된 지점(branch)으로 이동
    }
    const child = createFiberFromTypeAndProps(
      Component.type,
      null,
      nextProps,
      workInProgress,
      workInProgress.mode,
      renderLanes
    );
    child.ref = workInProgress.ref;
    child.return = workInProgress;
    workInProgress.child = child;
    return child;
  }
  // ❗❗ ↖ 이제 아래는 리렌더링 지점입니다
  const currentChild = ((current.child: any): Fiber); // This is always exactly one child
  const hasScheduledUpdateOrContext = checkScheduledUpdateOrContext(
    current,
    renderLanes
  );
  if (!hasScheduledUpdateOrContext) {
    // This will be the props with resolved defaultProps,
    // unlike current.memoizedProps which will be the unresolved ones.
    const prevProps = currentChild.memoizedProps;
    // Default to shallow comparison
    let compare = Component.compare;
    compare = compare !== null ? compare : shallowEqual;
    if (compare(prevProps, nextProps) && current.ref === workInProgress.ref) {
      return bailoutOnAlreadyFinishedWork(current, workInProgress, renderLanes);
    }
    // ❗❗ 만약 props가 변경되지 않으면 빠른 bailout을 시도합니다
  }
  // React DevTools reads this flag.
  workInProgress.flags |= PerformedWork;
  const newChild = createWorkInProgress(currentChild, nextProps);
  newChild.ref = workInProgress.ref;
  newChild.return = workInProgress;
  workInProgress.child = newChild;
  return newChild;
}
```

세부 사항에도 불구하고 기본 논리는 다음과 같습니다.

```plaintext
if (first time) {
  if (is simple) {
    update fiber tag to SimpleMemoComponent, so next time we go directly to updateSimpleMemoComponent()
    updateSimpleMemoComponent()
  } else {
    create child fibers
  }
} else {
  if (we can bailout) {
    try bailout
  } else {
    reconcile children
  }
}
```

## 5\. SimpleMemoComponent는 내부적으로 최적화됩니다

```typescript
function updateSimpleMemoComponent(
  current: Fiber | null,
  workInProgress: Fiber,
  Component: any,
  nextProps: any,
  renderLanes: Lanes
): null | Fiber {
  if (current !== null) {
    const prevProps = current.memoizedProps;
    if (
      shallowEqual(prevProps, nextProps) &&
      current.ref === workInProgress.ref &&
      // Prevent bailout if the implementation changed due to hot reload.
      (__DEV__ ? workInProgress.type === current.type : true)
    ) {
      didReceiveUpdate = false;
      if (!checkScheduledUpdateOrContext(current, renderLanes)) {
        workInProgress.lanes = current.lanes;
        return bailoutOnAlreadyFinishedWork(
          current,
          workInProgress,
          renderLanes
        );
      } else if ((current.flags & ForceUpdateForLegacySuspense) !== NoFlags) {
        // This is a special case that only exists for legacy mode.
        // See https://github.com/facebook/react/pull/19216.
        didReceiveUpdate = true;
      }
    }
  }
  return updateFunctionComponent(
    current,
    workInProgress,
    Component,
    nextProps,
    renderLanes
  );
}
```

`updateSimpleMemoComponent()`가 훨씬 더 간단한데:

1. props를 얕게 비교하고 bailout을 시도합니다.
    
2. 새 파이버가 생성되지 않으므로, 파이버 트리에 `D()`가 없습니다.
    

콘솔에서 파이버 트리를 볼 수 있습니다. (방법을 보려면 [여기에서 동영상](https://youtu.be/0jbV6apamhs?t=758)을 보세요.)

[![](https://jser.dev/static/memo-2.png align="left")](https://jser.dev/static/memo-2.png)

예, 메모 엘리먼트의 type은 `D()`이지만 그 자식은 `div`로 바로 이동합니다.

compare 함수에 전달을 설정하면, React는 더 이상 단순하게 처리할 수 없습니다.

```typescript
const  D  = React.memo(_D, (a, b) => true);
```

이제 파이버 트리를 살펴보면 상황이 달라집니다.

[![](https://jser.dev/static/memo-3.png align="left")](https://jser.dev/static/memo-3.png)

1. tag는 14, 지금은 단순하지 않습니다.
    
2. type이 더 이상 `D`가 아닙니다.
    
3. 자식은 `div`이지만 `D` 입니다.
    

예, 간단히 말해, **SimpleMemoComponent는 함수 컴포넌트에 대한 내부 최적화로, 메모 파이버와 래핑된 파이버를 하나로 합칩(merge)니다**.

[리액트 네이티브의 View Flattening](https://reactnative.dev/docs/view-flattening)과 비슷해 보입니다.

이 최적화에 대한 원래 홍보 자료는 [여기에서](https://github.com/facebook/react/pull/13903) 확인할 수 있습니다.

## 6\. `checkScheduledUpdateOrContext()`는 항상 호출됩니다

`updateMemoComponent()`와 `updateSimpleMemoComponent()`에서는 props를 비교하지만, 일부 이벤트나 컨텍스트와 같은 다른 함수에 의해 래핑된 컴포넌트가 업데이트 예약을 받았을 수 있으므로 항상 `checkScheduledUpdateOrContext()`가 실행된다는 것을 알아두세요.

따라서 props는 동일하지만 bailout이 발생하지 않는 경우가 있습니다.

즉, [React 홈페이지](https://reactjs.org/docs/react-api.html#reactmemo)에 명시된 것처럼 `React.memo()` 는 성능 최적화를 위한 것이지 렌더링을 '방지'하기 위해 사용하지 마세요.

여기까지 `React.memo()`에 대해 알아봤습니다. 도움이 되길 바랍니다.

(원본 게시일: 2022-01-11)