---
title: "[번역] React에서 빈 값(empty values)은 어떻게 다뤄지나요?"
datePublished: Fri May 17 2024 02:59:34 GMT+0000 (Coordinated Universal Time)
cuid: clwa3cpsv000309l32eqkd28d
slug: react-internals-deep-dive-18
cover: https://cdn.hashnode.com/res/hashnode/image/upload/v1715909409386/10040f3f-5690-4eaf-9979-85bb7142d850.jpeg
tags: reactjs, react-internals

---

> ***영문 블로그 글을 번역했습니다. 허가를 받으면 시리즈를 이어갈 예정입니다.  
> ***[***원문링크:***](https://jser.dev/react/2022/01/26/how-does-react-usedeferredvalue-work) [https://jser.dev/react/2022/02/04/how-React-handles-empty-values](https://jser.dev/react/2022/02/04/how-React-handles-empty-values)

---

> [***ℹ️React Internals Deep Dive***](https://jser.dev/series/react-source-code-walkthrough.html) ***에피소드 18,*** [***유튜브에서 제가 설명하는***](https://www.youtube.com/watch?v=0Xr852RL_HA&list=PLvx8w9g4qv_p-OS-XdbB3Ux_6DMXhAJC3&index=18)***것을 시청해주세요.***
> 
> ***⚠React@18.2.0기준, 최신 버전에서는 구현이 변경되었을 수 있습니다.***
> 
> ***💬 역자 주석: JSer의 코멘트는 ❗❗로 표시 해뒀습니다.  
> 그 외 주석은 리액트 소스 코드 자체의 주석입니다.  
> ... 은 생략된 코드입니다.***

[Booleans, Null, Undefined는 무시된다](https://reactjs.org/docs/jsx-in-depth.html#booleans-null-and-undefined-are-ignored)는 것을 모두 알고 계시겠지만, 아래와 같은 예제는 모두 동일하게 렌더링합니다.

```svelte
<div />
<div></div>
<div>{false}</div>
<div>{null}</div>
<div>{undefined}</div>
<div>{true}</div>
```

하지만 이러한 값은 React 내부에서 정확히 어떻게 처리될까요? 알아봅시다.

이러한 값은 컴포넌트가 아니어서 일부 컴포넌트의 자식으로만 존재하므로 [reconcileChildren()](https://github.com/facebook/react/blob/e62a8d754548a490c2a3bcff3b420e5eedaf11c0/packages/react-reconciler/src/ReactFiberBeginWork.new.js#L287)을 사용해 보겠습니다.

> ℹ 이 포스팅은 저도 배우고 있는 시리즈의 일환이기도 한데, 당황스러운 부분이 있으시다면 먼저 제 과거 포스팅이나 영상을 보시면 될 것 같습니다. 위와 같이 이미 알고 있는 함수이기 때문에 그냥 넘어가는데, 그 이유를 설명하기는 어렵습니다.

[reconcileChildren()](https://github.com/facebook/react/blob/e62a8d754548a490c2a3bcff3b420e5eedaf11c0/packages/react-reconciler/src/ReactFiberBeginWork.new.js#L287)에서 첫 번째 렌더링에는 `mountChildFibers()`가, 이후 렌더링에는 `reconcileChildFibers()`가 사용됩니다.

하지만 실제로 이 두 기능은 부작용[을 추적할지 여부만 다를 뿐 동일](https://github.com/facebook/react/blob/e62a8d754548a490c2a3bcff3b420e5eedaf11c0/packages/react-reconciler/src/ReactFiberBeginWork.new.js#L287)합니다.

([소스](https://github.com/facebook/react/blob/e62a8d754548a490c2a3bcff3b420e5eedaf11c0/packages/react-reconciler/src/ReactChildFiber.old.js#L1366-L1367))

```typescript
export const reconcideChildFibers  = ChildReconciler(true);
export const mountChildFibers  = ChildReconciler(false);
```

[ChildReconciler()](https://github.com/facebook/react/blob/e62a8d754548a490c2a3bcff3b420e5eedaf11c0/packages/react-reconciler/src/ReactChildFiber.old.js#L268)에서 **side effect**는 '삭제'를 의미한다는 것을 알 수 있습니다.

`ChildReconciler()`는 위의 플래그 아래 몇 가지 클로저 함수로 구성되며, 실제 [reconcileChildFibers()](https://github.com/facebook/react/blob/e62a8d754548a490c2a3bcff3b420e5eedaf11c0/packages/react-reconciler/src/ReactChildFiber.old.js#L1260)를 내보냅니다(export).

```typescript
function reconcileChildFibers(
  returnFiber: Fiber,
  currentFirstChild: Fiber | null,
  newChild: any,
  lanes: Lanes
): Fiber | null {
  if (typeof newChild === "object" && newChild !== null) {
    switch (newChild.$$typeof) {
      case REACT_ELEMENT_TYPE:
        return placeSingleChild(
          reconcileSingleElement(
            returnFiber,
            currentFirstChild,
            newChild,
            lanes
          )
        );
      case REACT_PORTAL_TYPE:
        return placeSingleChild(
          reconcileSinglePortal(returnFiber, currentFirstChild, newChild, lanes)
        );
      case REACT_LAZY_TYPE:
        if (enableLazyElements) {
          const payload = newChild._payload;
          const init = newChild._init;
          // TODO: This function is supposed to be non-recursive.
          return reconcileChildFibers(
            returnFiber,
            currentFirstChild,
            init(payload),
            lanes
          );
        }
    }
    if (isArray(newChild)) {
      return reconcileChildrenArray(
        returnFiber,
        currentFirstChild,
        newChild,
        lanes
      );
    }
    if (getIteratorFn(newChild)) {
      return reconcileChildrenIterator(
        returnFiber,
        currentFirstChild,
        newChild,
        lanes
      );
    }
    throwOnInvalidObjectType(returnFiber, newChild);
  }
  if (
    (typeof newChild === "string" && newChild !== "") ||
    typeof newChild === "number"
  ) {
    return placeSingleChild(
      reconcileSingleTextNode(
        returnFiber,
        currentFirstChild,
        "" + newChild,
        lanes
      )
    );
  }
  // Remaining cases are all treated as empty.
  return deleteRemainingChildren(returnFiber, currentFirstChild);
```

이 것은 4단계로 구성되어 있습니다.

1. `$$typeof`를 기반으로 단일 엘레먼트 타입을 처리합니다.
    
2. 배열 또는 이터레이터를 처리합니다.
    
3. 비어 있지 않은 문자열 및 숫자
    
4. 나머지는 비어있는 것(empty)으로 처리되어 이전 파이버가 있으면 삭제됩니다.
    

따라서 파이버를 만들 때 **null, undefined 및 boolean은 단순히 무시된다는** 것을 알 수 있습니다.

배열에 있는 경우는 어떨까요, [reconcideChildrenArray()](https://github.com/facebook/react/blob/848e802d203e531daf2b9b0edb281a1eb6c5415d/packages/react-reconciler/src/ReactChildFiber.old.js#L750)를 살펴보겠습니다.

`reconcideChildrenArray()` 에는 나중에 다루고 싶은 몇 가지 알고리즘이 있습니다.

코드를 살펴보면 새로운 파이버가 생성될 수 있는 두 곳을 확인할 수 있습니다.

```typescript
const newFiber = updateSlot(returnFiber, oldFiber, newChildren[newIdx], lanes);
const newFiber = updateFromMap(
  existingChildren,
  returnFiber,
  newIdx,
  newChildren[newIdx],
  lanes
);
```

`reconcileChildrenArray()`에서는 배열 항목을 반복하여 새로운 링크드 파이버 리스트를 구성합니다.

newFiber가 `null`이면 단순히 무시되고, 파이버 트리에 추가되지 않습니다.

[updateSlot()](https://github.com/facebook/react/blob/848e802d203e531daf2b9b0edb281a1eb6c5415d/packages/react-reconciler/src/ReactChildFiber.old.js#L564) 및 [updateFromMap()](https://github.com/facebook/react/blob/848e802d203e531daf2b9b0edb281a1eb6c5415d/packages/react-reconciler/src/ReactChildFiber.old.js#L564)에서 빈 값이 무시되고 `null`이 반환되는 유사한 패턴을 발견할 수 있습니다.

```typescript
function updateSlot(
  returnFiber: Fiber,
  oldFiber: Fiber | null,
  newChild: any,
  lanes: Lanes,
): Fiber | null {
  if (
    (typeof newChild === 'string' && newChild !== '') ||
    typeof newChild === 'number'
  ) {
    ...
  }
  if (typeof newChild === 'object' && newChild !== null) {
    ...
  }
  return null;
}
```

그게 다입니다. 이제 React에서 빈 값이 어떻게 처리되는지 알게 되었습니다. - **이것들은 간단하게 무시됩니다.**

한 가지 작은 문제는 실제로 `reconcileChildrenArray()`의 조정(reconciling) 알고리즘에 영향을 미친다는 것인데, 이에 대해서는 곧 포스팅을 작성할 예정이니 계속 지켜봐 주시기 바랍니다.

(원본 게시일: 2022-02-04)