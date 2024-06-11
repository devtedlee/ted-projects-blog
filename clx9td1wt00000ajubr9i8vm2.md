---
title: "[번역] React에서 SuspenseList는 어떻게 동작하나요?"
datePublished: Tue Jun 11 2024 02:59:36 GMT+0000 (Coordinated Universal Time)
cuid: clx9td1wt00000ajubr9i8vm2
slug: react-internals-deep-dive-25
cover: https://cdn.hashnode.com/res/hashnode/image/upload/v1718073884608/3a4ef371-2425-4d79-aff1-67426c4dd44c.jpeg
tags: react-internals, react-suspense

---

> ***영문 블로그 글을 번역했습니다. 허가를 받으면 시리즈를 이어갈 예정입니다.  
> 원문링크:***[https://jser.dev/react/2022/06/19/how-does-suspense-list-work](https://jser.dev/react/2022/06/19/how-does-suspense-list-work)

---

> ***ℹ️***[***React Internals Deep Dive***](https://jser.dev/series/react-source-code-walkthrough.html)***에피소드 25,***[***유튜브에서 제가 설명하는 것***](https://www.youtube.com/watch?v=ByeeMsIElFE&list=PLvx8w9g4qv_p-OS-XdbB3Ux_6DMXhAJC3&index=25)***을 시청해주세요.***
> 
> ***⚠***[***React@18.2.0***](https://github.com/facebook/react/releases/tag/v18.2.0)***기준, 최신 버전에서는 구현이 변경되었을 수 있습니다.***
> 
> ***💬 역자 주석: JSer의 코멘트는 ❗❗로 표시 해뒀습니다.  
> 그 외 주석은 리액트 소스 코드 자체의 주석입니다.  
> ... 은 생략된 코드입니다.***

## 1\. 데모 - SuspenseList란 무엇인가요?

서스펜스 자체는 준비가 되지 않았을 때 폴백(fallback)을 보여주고 promise가 해결(resolve)되면 내용을 드러내는데, 문제는 서스펜스 구성 요소가 여러 개일 경우 순서가 보장되지 않아 깜박거림이 발생할 수 있기 때문에 일종의 조정(coordinating)이 필요하다는 점입니다.

[SuspenseList](https://17.reactjs.org/docs/concurrent-mode-reference.html#suspenselist)는 바로 이런 용도로 사용됩니다.

먼저 [SuspenseList를 사용하지 않고 여러 개의 Suspense 데모](https://jser.dev/demos/react/suspense/multiple-suspense.html)를 시도해 보겠습니다.

```svelte
<div>Hi</div>
<React.Suspense fallback={<p>loading...</p>}>
  <Child resource={resource1} />
</React.Suspense>
<React.Suspense fallback={<p>loading...</p>}>
  <Child resource={resource2} />
</React.Suspense>
<React.Suspense fallback={<p>loading...</p>}>
  <Child resource={resource3} />
</React.Suspense>
```

[![](https://jser.dev/static/multiple-suspense.gif align="left")](https://jser.dev/static/multiple-suspense.gif)

두 번째 프로미스가 더 빨리 이행되는 것을 볼 수 있는데, 이는 멋진 경험은 아닙니다.

서스펜스 하나에 모든 `<Child/>`를 담으면 어떨까요? 서스펜스를 따로 사용하면 더 나은 점진적 경험을 만들 수 있고, 충족되는 동안 최대한 많은 것을 보여줄 수 있다는 장점이 있습니다.

프로미스에 대한 해결 순서가 무엇이든 위에서 아래로 내용을 공개하는 것도 좋은 경험이 될 수 있습니다.

여기에서 [SuspenseList를 사용한 다른 데모](https://jser.dev/demos/react/suspense/multiple-suspense-with-suspenselist-foward.html)를 시도해 보겠습니다.

> SuspenseList를 사용해보기 위해 실험용 빌드를 사용합니다.

```svelte
<div>Hi</div>
<React.SuspenseList revealOrder="forwards">
  <React.Suspense fallback={<p>loading...</p>}>
    <Child resource={resource1} />
  </React.Suspense>
  <React.Suspense fallback={<p>loading...</p>}>
    <Child resource={resource2} />
  </React.Suspense>
  <React.Suspense fallback={<p>loading...</p>}>
    <Child resource={resource3} />
  </React.Suspense>
</React.SuspenseList>
```

[![](https://jser.dev/static/suspenselist.gif align="left")](https://jser.dev/static/suspenselist.gif)

두 번째 프로미스가 더 빨리 fullfilled(💬[resolve or reject 된 상태](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Promise#description)를 말함) 되었음에도 불구하고, 공개 순서가 위에서 아래로 유지되는 것을 볼 수 있습니다.

## 2\. SuspenseList는 어떻게 동작하나요?

### 2.1 siblings의 정보를 어떻게 확인하고 전달하나요?

상당히 복잡하기 때문에 먼저 이런 기능을 어떻게 구현할지 생각해 보겠습니다.

핵심 정보는 \*\*프로미스 이행 순서(promise fullfilling order)\*\*에 대한 정보로, 서스펜스가 자신의 내용을 공개하려고 할 때, 다른 형제들의 프로미스 상태와 자신의 순서를 포함한 형제들의 정보가 필요하기 때문에 기본적으로 공개 여부를 결정하기 위해 추가 정보가 필요합니다.

파이버의 트리 구조 때문에 조상을 통해서만 형제자매에게 일부 정보를 공유할 수 있기 때문에, 기본적으로 더 많은 제어를 위해 컨텍스트가 필요합니다.

Suspense에 대한 [이전 게시물](https://ted-projects.com/react-internals-deep-dive-7-1)에서 Suspense 렌더링에 다음과 같은 코드가 있습니다.

```typescript
function updateSuspenseComponent(current, workInProgress, renderLanes) {
  const nextProps = workInProgress.pendingProps;
  let suspenseContext: SuspenseContext = suspenseStackCursor.current;
  let showFallback = false;
  const didSuspend = (workInProgress.flags & DidCapture) !== NoFlags;
  if (
    didSuspend ||
    shouldRemainOnFallback(suspenseContext, current, workInProgress, renderLanes)
  ) {
    // Something in this boundary's subtree already suspended. Switch to
    // rendering the fallback children.
    showFallback = true;
    workInProgress.flags &= ~DidCapture;
  }
```

`showFallback`은 Suspense 자체의 `didSuspend` 뿐만 아니라 `shouldRemainOnFallback()`을 확인하여 결정되며, 이것이 우리가 이야기하고 있는 컨텍스트인 것 같습니다.

### 2.2 shouldRemainOnFallback()

```typescript
// TODO: Probably should inline this back
function shouldRemainOnFallback(
  suspenseContext: SuspenseContext,
  current: null | Fiber,
  workInProgress: Fiber,
  renderLanes: Lanes
) {
  // If we're already showing a fallback, there are cases where we need to
  // remain on that fallback regardless of whether the content has resolved.
  // For example, SuspenseList coordinates when nested content appears.
  if (current !== null) {
    const suspenseState: SuspenseState = current.memoizedState;
    if (suspenseState === null) {
      // Currently showing content. Don't hide it, even if ForceSuspenseFallback
      // is true. More precise name might be "ForceRemainSuspenseFallback".
      // Note: This is a factoring smell. Can't remain on a fallback if there's
      // no fallback to remain on.
      return false;
    }
  }
  // Not currently showing content. Consult the Suspense context.
  return hasSuspenseContext(
    suspenseContext,
    (ForceSuspenseFallback: SuspenseContext)
  );
}
```

주석을 보면 SuspenseList가 명시적으로 언급되어 있음을 알 수 있습니다. 첫 번째 브랜치는 기본적으로 이미 공개된 경우 콘텐츠를 계속 표시합니다.

**ForceSuspenseFallback이 suspenseContext에 있으면, 프로미스가 이행되더라도 여전히 폴백이 표시되어야함**을 알 수 있습니다.

### 2.3 SuspenseContext와 ReactFiberStack

SuspenseContext는 ReactFiberStack을 기반으로 하며, 동일한 구현을 가진 몇 가지 다른 컨텍스트가 있습니다.

[소스 코드](https://github.com/facebook/react/blob/e61fd91f5c523adb63a3b97375ac95ac657dc07f/packages/react-reconciler/src/ReactFiberSuspenseContext.new.js#L102)에서 SuspenseContext는 조정하는 동안 경로를 따라 Suspense의 정보를 추적하는 것이고, `ForceSuspenseFallback`은 숫자의 플래그일 뿐입니다.

```typescript
// ForceSuspenseFallback can be used by SuspenseList to force newly added
// items into their fallback state during one of the render passes.
export const ForceSuspenseFallback: ShallowSuspenseContext = 0b10;
export function addSubtreeSuspenseContext(
  parentContext: SuspenseContext,
  subtreeContext: SubtreeSuspenseContext
): SuspenseContext {
  return parentContext | subtreeContext;
}
export function pushSuspenseContext(
  fiber: Fiber,
  newContext: SuspenseContext
): void {
  push(suspenseStackCursor, newContext, fiber);
}
export function popSuspenseContext(fiber: Fiber): void {
  pop(suspenseStackCursor, fiber);
}
```

`suspenseStackCursor`를 살펴보겠습니다.

```typescript
export const suspenseStackCursor: StackCursor<SuspenseContext> = createCursor(
  DefaultSuspenseContext
);
```

비밀은 `ReactFiberStack`에 있습니다. ([코드](https://github.com/facebook/react/blob/e61fd91f5c523adb63a3b97375ac95ac657dc07f/packages/react-reconciler/src/ReactFiberStack.new.js#L59))

```typescript
const valueStack: Array<any> = [];
let index = -1;
function createCursor<T>(defaultValue: T): StackCursor<T> {
  return {
    current: defaultValue,
  };
}
function isEmpty(): boolean {
  return index === -1;
}
function pop<T>(cursor: StackCursor<T>, fiber: Fiber): void {
  cursor.current = valueStack[index];
  valueStack[index] = null;
  index--;
}
function push<T>(cursor: StackCursor<T>, value: T, fiber: Fiber): void {
  index++;
  valueStack[index] = cursor.current;
  cursor.current = value;
}
export { createCursor, isEmpty, pop, push };
```

`.current`는 최신 값을 가리키고, `valueStack`은 이전 값을 모두 보유하므로 `.current`를 `pop()`에서 설정할 수 있습니다. `valueStack`은 하나뿐이어서 모든 종류의 커서가 동일한 값 스택을 사용하므로 값의 불일치를 피하려면 `push()`와 `pop()`을 정확히 일치시켜야 합니다.

따라서 논리는 다음과 같습니다.

1. 파이버에서 `beginWork()`를 수행할 때 `push()`를 호출합니다.
    
2. 파이버의 `completeWork()`를 호출할 때 `pop()`을 호출합니다.
    

이 두 함수가 언제 호출되는지 살펴보겠습니다.

### 2.4 pushSuspenseContext()가 호출될 때

`pushSuspenseContext()`가 호출되는 건

1. `updateSuspenseComponent()` ([코드](https://github.com/facebook/react/blob/e61fd91f5c523adb63a3b97375ac95ac657dc07f/packages/react-reconciler/src/ReactFiberBeginWork.new.js#L2003))
    
2. `updateSuspenseListComponent()` ([코드](https://github.com/facebook/react/blob/e61fd91f5c523adb63a3b97375ac95ac657dc07f/packages/react-reconciler/src/ReactFiberBeginWork.new.js#L3046))
    
3. `attemptEarlyBailoutIfNoScheduledUpdate()`
    

앞의 두 가지는 매우 간단하고, 세 번째는 내부 개선 사항이므로 지금은 건너뛰겠습니다.

`popSuspenseContext()`가 호출되는 건

1. `completeWork()` ([코드](https://github.com/facebook/react/blob/e61fd91f5c523adb63a3b97375ac95ac657dc07f/packages/react-reconciler/src/ReactFiberCompleteWork.new.js#L846))
    
2. `unwindWork()`
    
3. `unwindInterruptedWork()`
    

다시 말하지만, 첫 번째는 매우 간단합니다.

위의 타이밍에 대해 자세히 알아보겠습니다.

### 2.5 updateSuspenseComponent()에서의 SuspenseContext

```typescript
let suspenseContext: SuspenseContext = suspenseStackCursor.current;
suspenseContext = setDefaultShallowSuspenseContext(suspenseContext);
pushSuspenseContext(workInProgress, suspenseContext);
```

```typescript
// The Suspense Context is split into two parts. The lower bits is
// inherited deeply down the subtree. The upper bits only affect
// this immediate suspense boundary and gets reset each new
// boundary or suspense list.
const SubtreeSuspenseContextMask: SuspenseContext = 0b01;
// ForceSuspenseFallback can be used by SuspenseList to force newly added
// items into their fallback state during one of the render passes.
export const ForceSuspenseFallback: ShallowSuspenseContext = 0b10;
export function setDefaultShallowSuspenseContext(
  parentContext: SuspenseContext
): SuspenseContext {
  return parentContext & SubtreeSuspenseContextMask;
}
```

하위 트리를 위한 더 낮은 비트만 유지한다는 것을 알 수 있습니다. `ForceSuspenseFallback`은 더 높은 비트이므로 현재 파이버 내부에서만 작동합니다.

### 2.6 두 번의 패스는 어떻게 하나요?

여기서 몇 가지 배경 지식을 소개하겠습니다. [서스펜스에서의 조정](https://ted-projects.com/react-internals-deep-dive-7-1)에 대해 이야기할 때 서스펜스가 예외를 처리하는 방식에서 **두 번의 패스** 렌더링 기술을 언급했습니다.

1. 렌더링 서스펜스, 아무 문제 없음, 콘텐츠로 이동합니다.
    
2. 예외를 포착하고, 서스펜스 경계로 돌아가서 일부 플래그를 업데이트합니다.
    
3. 다시 서스펜스를 렌더링하고, 플래그로 인해 폴백으로 전환됩니다.
    

이것은 트리 내부에서 분기 로직을 수행하는 방법의 예시이며, 비슷한 작업을 수행하려는 경우 여기에서 접근 방식을 일반화할 수 있습니다.

1. 특수 상태를 유지하는 일부 분기 로직에 대한 특수 컴포넌트를 생성합니다.
    
2. 이 컴포넌트는 가지고 있는 다른 상태에 따라 다른 작업을 수행하며, 예를 들어 다음과 같은 작업을 수행할 수 있습니다.
    
    * 컨텍스트 값 업데이트
        
    * 조정 프로세스 중단(interrupt)
        
    * (기본적으로 게이트웨이처럼 작동하기 때문에 무엇이든 가능합니다).
        

### 2.7 순회 알고리즘을 다시 살펴보겠습니다

[리액트의 파이버 트리 순회는 어떻게 동작하나요?](https://ted-projects.com/react-internals-deep-dive-15)에서 코드 조각을 붙여넣었습니다.

```typescript
let nextNode = root;
function begin() {
  while (nextNode) {
    console.log("begin ", nextNode.val);
    if (nextNode.child) {
      nextNode = nextNode.child;
    } else {
      complete();
    }
  }
}
function complete() {
  while (nextNode) {
    console.log("complete ", nextNode.val);
    if (nextNode.sibling) {
      nextNode = nextNode.sibling;
      // go to sibling and begin new
      return;
    }
    nextNode = nextNode.return;
  }
}
begin();
```

기본적으로 다음을 의미합니다.

1. 각 노드에 대해 캡처 단계와 버블링 단계가 있는 DOM 이벤트와 마찬가지로 진입(begin)과 종료(complete)의 두 단계가 있습니다.
    
2. `begin`동안에, null을 반환하면, 더이상 작업이 없으므로 `complete`를 시작한다는 의미입니다.
    
3. `complete`동안에, 형제자매가 있는 경우, 형제자매에서 `begin`됩니다.
    
4. 또한 글로벌 `workInProgress`는 끝없이 조정됩니다.
    

위의 논리를 바탕으로 다음과 같은 질문에 답할 수 있습니다.

**컴포넌트를 계속 렌더링하는 방법은 무엇인가요?**

completeWork에서 부모 `.return`으로 이동하지 말고 workInProgress를 그 자체로 설정하세요.

**컴포넌트를 n번 렌더링하는 방법은 무엇인가요?**

state를 사용하여 컴포넌트의 렌더링 횟수를 유지할 수 있습니다. 그리고 `completeWork()`에서 카운트를 확인하고 최대값을 초과하지 않으면 이전 질문에 대한 답을 반복합니다.

**children로부터 정보를 수집하여 다른 children에게 전달하는 방법은 무엇인가요?**

1. 먼저 모든 children을 렌더링하고 필요한 정보를 파이버에 노출(expose)시킬 수 있습니다.
    
2. 컨트롤이 컴포넌트로 돌아갈 수 있도록 렌더링을 중단하는 방법도 필요합니다. 그렇지 않으면 렌더링이 그냥 끝나고 DOM이 커밋됩니다.
    
3. 중단된 후, 이제 다시 자식들을 순회하여 정보를 수집할 수 있습니다. 이것은 렌더링이 아닌 수집을 위한 순회일 뿐입니다.
    
4. 필요한 정보로 컨텍스트를 업데이트하고 하위 파이버를 다시 조정합니다.
    

SuspenseList의 경우, 주문 정보 때문에 더 복잡합니다. 위의 지식을 숙지하고 계속 읽어주세요.

### 2.8 `updateSuspenseListComponent()`에서의 SuspenseContext

```typescript
let suspenseContext: SuspenseContext = suspenseStackCursor.current;
const shouldForceFallback = hasSuspenseContext(
  suspenseContext,
  (ForceSuspenseFallback: SuspenseContext)
);
if (shouldForceFallback) {
  suspenseContext = setShallowSuspenseContext(
    suspenseContext,
    ForceSuspenseFallback
  );
  workInProgress.flags |= DidCapture;
} else {
  const didSuspendBefore =
    current !== null && (current.flags & DidCapture) !== NoFlags;
  if (didSuspendBefore) {
    // If we previously forced a fallback, we need to schedule work
    // on any nested boundaries to let them know to try to render
    // again. This is the same as context updating.
    propagateSuspenseContextChange(
      workInProgress,
      workInProgress.child,
      renderLanes
    );
  }
  suspenseContext = setDefaultShallowSuspenseContext(suspenseContext);
}
pushSuspenseContext(workInProgress, suspenseContext);
```

코드에서 SuspenseList가 부모 컨텍스트에서 `ForceSuspenseFallback`을 설정하는 것을 볼 수 있습니다.

하지만 잠시만요? 모든 협력 로직을 초기화하는 곳은 SuspenseList가 되어야 하지 않을까요? 그 자손을 기반으로 `ForceSuspenseFallback`을 추가하는 진정한 로직은 어디에 있을까요?

사실 이 함수의 내부를 조금 더 자세히 읽어보면 이 질문에 대한 답이 나옵니다.

```typescript
if ((workInProgress.mode & ConcurrentMode) === NoMode) {
  // In legacy mode, SuspenseList doesn't work so we just
  // use make it a noop by treating it as the default revealOrder.
  workInProgress.memoizedState = null;
} else {
  switch (revealOrder) {
    case "forwards": {
      ...
      break;
    }
    case "backwards": {
      ...
      break;
    }
    case "together": {
      ...
      break;
    }
    default: {
      // The default reveal order is the same as not having
      // a boundary.
      workInProgress.memoizedState = null;
    }
  }
}
return workInProgress.child;
```

자, 데모에서 사용하는 `reveal="fowards"`에 집중해 보겠습니다.

```typescript
case "forwards": {
  const lastContentRow = findLastContentRow(workInProgress.child);
  let tail;
  if (lastContentRow === null) {
    // The whole list is part of the tail.
    // TODO: We could fast path by just rendering the tail now.
    tail = workInProgress.child;
    workInProgress.child = null;
  } else {
    // Disconnect the tail rows after the content row.
    // We're going to render them separately later.
    tail = lastContentRow.sibling;
    lastContentRow.sibling = null;
  }
  initSuspenseListRenderState(
    workInProgress,
    false, // isBackwards
    tail,
    lastContentRow,
    tailMode
  );
  break;
}
```

먼저 하위 목록에서 검색하고 `findFirstSuspended()`를 사용하여 이미 콘텐츠를 표시하는 마지막 행을 찾습니다. Suspense는 트리 깊숙이 있을 수 있으므로 `findFirstSuspended()`는 재귀적으로 Suspense 또는 SuspenseList가 있는지 찾아서 지연된 것을 찾습니다.

```typescript
function findLastContentRow(firstChild: null | Fiber): null | Fiber {
  // This is going to find the last row among these children that is already
  // showing content on the screen, as opposed to being in fallback state or
  // new. If a row has multiple Suspense boundaries, any of them being in the
  // fallback state, counts as the whole row being in a fallback state.
  // Note that the "rows" will be workInProgress, but any nested children
  // will still be current since we haven't rendered them yet. The mounted
  // order may not be the same as the new order. We use the new order.
  let row = firstChild;
  let lastContentRow: null | Fiber = null;
  while (row !== null) {
    const currentRow = row.alternate;
    // New rows can't be content rows.
    if (currentRow !== null && findFirstSuspended(currentRow) === null) {
      lastContentRow = row;
    }
    row = row.sibling;
  }
  return lastContentRow;
}
```

그렇다면 `lastContentRow`를 찾는 이유는 무엇일까요? 다음 코드가 중요합니다.

```typescript
if (lastContentRow === null) {
  // The whole list is part of the tail.
  // TODO: We could fast path by just rendering the tail now.
  tail = workInProgress.child;
  workInProgress.child = null;
} else {
  // Disconnect the tail rows after the content row.
  // We're going to render them separately later.
  tail = lastContentRow.sibling;
  lastContentRow.sibling = null;
}
```

`tail`은 폴백 목록을 의미하며, 더 정확하게는 폴백의 시작점이어야 합니다.

`lastContentRow === null`은 모두 폴백이므로 꼬리가 첫 번째 자식으로 설정되고, 그렇지 않으면 꼬리가 다음 형제자매로 설정된다는 의미입니다.

따라서 기본적으로 `SuspenseList`는 자식을 두 개의 목록으로 나누려고 하는데, 하나는 이미 렌더링된 콘텐츠이고 다른 하나는 폴백입니다. 첫 번째 폴백만 검색하므로 꼬리에 있는 콘텐츠는 여전히 꼬리입니다.

```typescript
Content Content Content Fallback Fallback Content Fallback Content
                        //❗❗ ↖ 이게 꼬리
```

더 흥미로운 점은 모두 폴백인 경우 `workInProgress.child가` null로 설정되고, 이전 섹션에서 언급한 알고리즘을 상기하면 `null은` 더 이상 자식으로 이동하지 않음을 의미하며, `completeWork()` 가 SuspenseList에서 바로 실행된다는 점입니다.

콘텐츠 행이 있는 경우, `lastContentRow.sibling = null;` 을 의미합니다.

1. 콘텐츠 목록과 폴백 목록의 두 가지 목록으로 나뉩니다.
    
2. 콘텐츠 목록이 완료되면 기본적으로 React는 폴백 목록이 되어야 하는 형제 목록으로 이동해야 하지만 연결이 끊어진 상태이므로 SuspenseList에서 `completeWork(` )가 실행됩니다.
    

여기에서 **아동의 일시 중단된 서스펜스 목록은 서스펜스 리스트를 완료한 후에만 조정되는** 것을 볼 수 있습니다 -&gt; 이것은 매우 중요합니다.

계속해서 아래에서 이러한 상태가 SuspenseList에 저장되어 있는 것을 볼 수 있습니다.

```typescript
initSuspenseListRenderState(
  workInProgress,
  false, // isBackwards
  tail,
  lastContentRow,
  tailMode
);
```

```typescript
function initSuspenseListRenderState(
  workInProgress: Fiber,
  isBackwards: boolean,
  tail: null | Fiber,
  lastContentRow: null | Fiber,
  tailMode: SuspenseListTailMode
): void {
  const renderState: null | SuspenseListRenderState =
    workInProgress.memoizedState;
  if (renderState === null) {
    workInProgress.memoizedState = ({
      isBackwards: isBackwards,
      rendering: null,
      renderingStartTime: 0,
      last: lastContentRow,
      tail: tail,
      tailMode: tailMode,
    }: SuspenseListRenderState);
  } else {
    // We can reuse the existing object from previous renders.
    renderState.isBackwards = isBackwards;
    renderState.rendering = null;
    renderState.renderingStartTime = 0;
    renderState.last = lastContentRow;
    renderState.tail = tail;
    renderState.tailMode = tailMode;
  }
}
```

`memoizedState`는 렌더링 방법에 대한 설정(configuration)을 담고 있습니다. `rendering`은 다음에 공개해야 할 대상 행(row)인 것 같습니다.

다른 프로퍼티는 변형에 불과하므로 지금은 잊어버리겠습니다. `forwards`가 어떻게 작동하는지 알면 나머지는 모두 알 수 있습니다.

### 2.9 마법은 `completeWork()`에 있습니다

[코드](https://github.com/facebook/react/blob/e61fd91f5c523adb63a3b97375ac95ac657dc07f/packages/react-reconciler/src/ReactFiberCompleteWork.new.js#L1266)

```typescript
case SuspenseListComponent: {
  popSuspenseContext(workInProgress);
  const renderState: null | SuspenseListRenderState =
    workInProgress.memoizedState;
  if (renderState === null) {
    // We're running in the default, "independent" mode.
    // We don't do anything in this mode.
    bubbleProperties(workInProgress);
    return null;
  }
  let didSuspendAlready = (workInProgress.flags & DidCapture) !== NoFlags;
  const renderedTail = renderState.rendering;
  if (renderedTail === null) {
    ...
    // Next we're going to render the tail.
  } else {
    // Append the rendered row to the child list.
    ...
  }
  if (renderState.tail !== null) {
    // We still have tail rows to render.
    // Pop a row.
    const next = renderState.tail;
    renderState.rendering = next;
    renderState.tail = next.sibling;
    renderState.renderingStartTime = now();
    next.sibling = null;
    // Restore the context.
    // TODO: We can probably just avoid popping it instead and only
    // setting it the first time we go from not suspended to suspended.
    let suspenseContext = suspenseStackCursor.current;
    if (didSuspendAlready) {
      console.log("push ForceSuspenseFallback");
      suspenseContext = setShallowSuspenseContext(
        suspenseContext,
        ForceSuspenseFallback
      );
    } else {
      suspenseContext = setDefaultShallowSuspenseContext(suspenseContext);
    }
    pushSuspenseContext(workInProgress, suspenseContext);
    // Do a pass over the next row.
    // Don't bubble properties in this case.
    return next;
  }
  bubbleProperties(workInProgress);
  return null;
}
```

이것은 코드의 일부가 생략된 거대한 코드 덩어리입니다. 저는 이 코드를 이해하려고 꽤 많은 시간을 보냈습니다. 좋은 소식은 여기서 `ForceSuspenseFallback`이 어떻게 설정되어 있는지 마침내 알 수 있다는 것입니다.

잠시만 기다려주시고 이제 시작합니다.

```typescript
popSuspenseContext(workInProgress);
const renderState: null | SuspenseListRenderState = workInProgress.memoizedState;
 // ❗❗ ↗ renderState는 설정입니다,
 // ❗❗ 만약 여기에 아무것도 없으면, SuspenseList 는 그냥 작동하지 않는(no-op) 컴포넌트 입니다.

if (renderState === null) {
  // We're running in the default, "independent" mode.
  // We don't do anything in this mode.
  bubbleProperties(workInProgress);
  return null;
}
let didSuspendAlready = (workInProgress.flags & DidCapture) !== NoFlags;
// ❗❗ ↗ didSuspendAlready 는 SuspenseList 첫 Suspended Suspense를 찾으라고 지시하는 로컬 플래그입니다.
```

SuspenseList에는 `DidCapture` 플래그도 있는데, `forwards`에서는 공개 순서가 필요하므로 SuspenseList가 첫 번째 Suspended Suspense를 찾으면 동일한 프로미스를 사용하여 트리거및 업데이트합니다. `didSuspendAlready`를 사용하여 예정된 프로미스를 사용하지 않도록 할 수 있습니다.

```typescript
const renderedTail = renderState.rendering;
if (renderedTail === null) {
  // We just rendered the head.
  ....
  // Next we're going to render the tail.
} else {
  ...
}
if (renderState.tail !== null) {
    // We still have tail rows to render.
    // Pop a row.
    const next = renderState.tail;
    renderState.rendering = next;
    renderState.tail = next.sibling;
    renderState.renderingStartTime = now();
    next.sibling = null;
    // Restore the context.
    // TODO: We can probably just avoid popping it instead and only
    // setting it the first time we go from not suspended to suspended.
    let suspenseContext = suspenseStackCursor.current;
    if (didSuspendAlready) {
      suspenseContext = setShallowSuspenseContext(
        suspenseContext,
        ForceSuspenseFallback,
      );
    } else {
      suspenseContext = setDefaultShallowSuspenseContext(suspenseContext);
    }
    pushSuspenseContext(workInProgress, suspenseContext);
    // Do a pass over the next row.
    // Don't bubble properties in this case.
    return next;
  }
  bubbleProperties(workInProgress);
  return null;
```

이 마지막 코드 조각은 매우 중요합니다.

1. `tail`가 하나씩 앞으로 굴러가는 것을 볼 수 있습니다. `renderState.tail = next.sibling`
    
2. 이전 tail은 `next.sibling = null`로 분리되어 있으므로, 조정될 때 `completeWork()` 는 이전 꼬리로 이동하지 않고 SuspenseList로 다시 이동합니다.
    
3. 이전 `tail`이 반환되며, 이는 `beginWork()`가 그 `tail`에서 시작됨을 의미합니다.
    

생략된 코드 내부.

```typescript
if (renderedTail === null) {
  // We just rendered the head.
  if (!didSuspendAlready) {
    // This is the first pass. We need to figure out if anything is still
    // suspended in the rendered set.
    // If new content unsuspended, but there's still some content that
    // didn't. Then we need to do a second pass that forces everything
    // to keep showing their fallbacks.
    // We might be suspended if something in this render pass suspended, or
    // something in the previous committed pass suspended. Otherwise,
    // there's no chance so we can skip the expensive call to
    // findFirstSuspended.
    const cannotBeSuspended =
      renderHasNotSuspendedYet() &&
      (current === null || (current.flags & DidCapture) === NoFlags);
    if (!cannotBeSuspended) {
      let row = workInProgress.child;
      while (row !== null) {
        const suspended = findFirstSuspended(row);
        if (suspended !== null) {
          didSuspendAlready = true;
          workInProgress.flags |= DidCapture;
          cutOffTailIfNeeded(renderState, false);
          // If this is a newly suspended tree, it might not get committed as
          // part of the second pass. In that case nothing will subscribe to
          // its thenables. Instead, we'll transfer its thenables to the
          // SuspenseList so that it can retry if they resolve.
          // There might be multiple of these in the list but since we're
          // going to wait for all of them anyway, it doesn't really matter
          // which ones gets to ping. In theory we could get clever and keep
          // track of how many dependencies remain but it gets tricky because
          // in the meantime, we can add/remove/change items and dependencies.
          // We might bail out of the loop before finding any but that
          // doesn't matter since that means that the other boundaries that
          // we did find already has their listeners attached.
          const newThenables = suspended.updateQueue;
          if (newThenables !== null) {
            workInProgress.updateQueue = newThenables;
            workInProgress.flags |= Update;
          }
          // Rerender the whole list, but this time, we'll force fallbacks
          // to stay in place.
          // Reset the effect flags before doing the second pass since that's now invalid.
          // Reset the child fibers to their original state.
          workInProgress.subtreeFlags = NoFlags;
          resetChildFibers(workInProgress, renderLanes);
          // Set up the Suspense Context to force suspense and immediately
          // rerender the children.
          pushSuspenseContext(
            workInProgress,
            setShallowSuspenseContext(
              suspenseStackCursor.current,
              ForceSuspenseFallback
            )
          );
          // Don't bubble properties in this case.
          return workInProgress.child;
        }
        row = row.sibling;
      }
    }
    if (renderState.tail !== null && now() > getRenderTargetTime()) {
      // We have already passed our CPU deadline but we still have rows
      // left in the tail. We'll just give up further attempts to render
      // the main content and only render fallbacks.
      workInProgress.flags |= DidCapture;
      didSuspendAlready = true;
      cutOffTailIfNeeded(renderState, false);
      // Since nothing actually suspended, there will nothing to ping this
      // to get it started back up to attempt the next item. While in terms
      // of priority this work has the same priority as this current render,
      // it's not part of the same transition once the transition has
      // committed. If it's sync, we still want to yield so that it can be
      // painted. Conceptually, this is really the same as pinging.
      // We can use any RetryLane even if it's the one currently rendering
      // since we're leaving it behind on this node.
      workInProgress.lanes = SomeRetryLane;
    }
  } else {
    cutOffTailIfNeeded(renderState, false);
  }
  // Next we're going to render the tail.
}
```

콘텐츠 목록과 폴백 목록을 분할한 직후에 분기에 대해 대략적으로 수행하는 작업은 다음과 같습니다.

1. 첫 번째 중단된 서스펜스를 찾아 프로미스를 연결합니다.
    
2. SuspenseContext에서 `ForceSuspenseFallback`을 설정합니다.
    
3. 호출하면 분할을 되돌릴(revert) 수 있는 `resetChildFibers()` 이후, 전체 목록을 리렌더링합니다.
    

왜 전체 목록을 리렌더링하나요? 조정 중이므로 다른 프로미스는 이미 이행(fullfilled)되었을 수 있습니다. 처음부터 모든 항목이 폴백을 렌더링하도록 강제하지 않으면 실제로 초기 상태의 순서가 깨지게 됩니다. 따라서 이 리렌더링을 통해 SuspenseList는 깨끗한 상태에서 작업할 수 있습니다.

두 번째 패스의 다른 브랜치의 경우 리렌더링할 필요가 없습니다.

```typescript
} else {
  // Append the rendered row to the child list.
  if (!didSuspendAlready) {
    const suspended = findFirstSuspended(renderedTail);
    if (suspended !== null) {
      workInProgress.flags |= DidCapture;
      didSuspendAlready = true;
      // Ensure we transfer the update queue to the parent so that it doesn't
      // get lost if this row ends up dropped during a second pass.
      const newThenables = suspended.updateQueue;
      if (newThenables !== null) {
        workInProgress.updateQueue = newThenables;
        workInProgress.flags |= Update;
      }
      cutOffTailIfNeeded(renderState, true);
      // This might have been modified.
      if (
        renderState.tail === null &&
        renderState.tailMode === 'hidden' &&
        !renderedTail.alternate &&
        !getIsHydrating() // We don't cut it if we're hydrating.
      ) {
        // We're done.
        bubbleProperties(workInProgress);
        return null;
      }
    } else if (
      // The time it took to render last row is greater than the remaining
      // time we have to render. So rendering one more row would likely
      // exceed it.
      now() * 2 - renderState.renderingStartTime >
        getRenderTargetTime() &&
      renderLanes !== OffscreenLane
    ) {
      // We have now passed our CPU deadline and we'll just give up further
      // attempts to render the main content and only render fallbacks.
      // The assumption is that this is usually faster.
      workInProgress.flags |= DidCapture;
      didSuspendAlready = true;
      cutOffTailIfNeeded(renderState, false);
      // Since nothing actually suspended, there will nothing to ping this
      // to get it started back up to attempt the next item. While in terms
      // of priority this work has the same priority as this current render,
      // it's not part of the same transition once the transition has
      // committed. If it's sync, we still want to yield so that it can be
      // painted. Conceptually, this is really the same as pinging.
      // We can use any RetryLane even if it's the one currently rendering
      // since we're leaving it behind on this node.
      workInProgress.lanes = SomeRetryLane;
    }
  }
  if (renderState.isBackwards) {
    // The effect list of the backwards tail will have been added
    // to the end. This breaks the guarantee that life-cycles fire in
    // sibling order but that isn't a strong guarantee promised by React.
    // Especially since these might also just pop in during future commits.
    // Append to the beginning of the list.
    renderedTail.sibling = workInProgress.child;
    workInProgress.child = renderedTail;
  } else {
    const previousSibling = renderState.last;
    if (previousSibling !== null) {
      previousSibling.sibling = renderedTail;
    } else {
      workInProgress.child = renderedTail;
    }
    renderState.last = renderedTail;
  }
}
```

꽤 자세한 내용이지만 지금은 건너 뛰겠습니다.

### 2.10 요약

SuspenseList에서 무슨 일이 일어나고 있는지 요약해 보겠습니다:

1. SuspenseList를 업데이트할 때 먼저 마지막 콘텐츠 행(서스펜스가 아닌)을 검색하여 자식을 목록, `head` 및 `tail`로 분할합니다.
    
2. `head`는 정상적으로 렌더링됩니다.
    
3. SuspenseList는 꼬리를 하나씩 렌더링하여 `completeWork()`
    
    * 각각을 형제에서 분리합니다. 이렇게 하면 자식이 완료될 때마다 SuspenseList의 `completeWork()`가 호출됩니다.
        
    * 첫 번째 Suspended Suspense가 프로미스에서 재시도 리스너를 설정했는지 확인하고 컨텍스트에서 `ForceSuspsensFallback`을 설정하여 나중에 오는 서스펜스가 일시 중단되지 않은 경우에도 폴백을 렌더링합니다.
        
    * 렌더링할 tail이 없는 경우 전체 목록을 리렌더링하여 렌더링된 head가 다시 일시 중단되는지 확인하는 몇 가지 검사가 있습니다.
        

두 번째 단계에서 tail을 하나씩 팝업하는 이유는 무엇인가요?

잘 모르겠습니다. 서스펜스 목록에는 의심스러운 목록이 많기 때문에 모든 children을 렌더링하는 데 시간이 많이 걸릴 수 있기 때문에 점차적으로 tail에 있는 것들을 옮기는 것 같습니다. 첫 번째 패스의 경우 모두 폴백이 될 것이므로 괜찮습니다. 그러나 공개 단계에서는 다른 이야기입니다. 조정이 중단되었다가 나중에 다시 재개된다고 가정할 때 SuspenseList 내부에서 마지막으로 검사한 위치를 추적해야 합니다. 이 정보는 SuspenseList의 completeWork() 함수에서 수행해야 합니다.

### 2.11 일러스트레이션

네, 위의 내용은 이해하려면 머리를 많이 써야 합니다. 설명하기 위해 다이어그램을 준비했습니다.

[![](https://jser.dev/static/suspenselist-1.png align="left")](https://jser.dev/static/suspenselist-1.png)

먼저 SuspenseList를 조정해 보겠습니다.

[![](https://jser.dev/static/suspenselist-2.png align="left")](https://jser.dev/static/suspenselist-2.png)

초기 단계에서는 콘텐츠 행이 발견되지 않으므로(새 파이버는 계산되지 않음) `tail`이 `div`로 설정되고 SuspenseList에서 `child`가 null로 설정되어 모든 행이 `tail`이 됩니다.

[![](https://jser.dev/static/suspenselist-3.png align="left")](https://jser.dev/static/suspenselist-3.png)

`child가` null이므로 더 이상 할 작업이 없으므로 `completeWork()`가 시작됩니다.

[![](https://jser.dev/static/suspenselist-4.png align="left")](https://jser.dev/static/suspenselist-4.png)

이것은 매우 초기 렌더링이므로 전체 목록을 렌더링하는 단계는 없지만 SuspenseList가 tail을 하나씩 렌더링하기 시작합니다. 렌더링되는 파이버의 `sibling`이 제거되는 것을 볼 수 있습니다.

[![](https://jser.dev/static/suspenselist-5.png align="left")](https://jser.dev/static/suspenselist-5.png)

`div`에 더 이상 아무것도 없으므로, 완료되었습니다.

(💬이미지 누락 생략)

그리고 SuspenseList에서 `completeWork()`를 다시 호출합니다.

[![](https://jser.dev/static/suspenselist-7.png align="left")](https://jser.dev/static/suspenselist-7.png)

결국 `tail`이 `null`이 되고 루프가 멈춥니다. 초기 렌더링에는 일시 중단된 서스펜스가 없기 때문에 프로세스가 종료됩니다.

#### 버튼이 클릭된 후

[![](https://jser.dev/static/suspenselist-8.png align="left")](https://jser.dev/static/suspenselist-8.png)

마지막 Suspense는 `lastContentRow`이므로 `tail`은 여전히 `null`인 형제에게 설정되고 콘텐츠 행의 조정이 계속됩니다.

[![](https://jser.dev/static/suspenselist-9.png align="left")](https://jser.dev/static/suspenselist-9.png)

`div`가 렌더링됩니다.

[![](https://jser.dev/static/suspenselist-10.png align="left")](https://jser.dev/static/suspenselist-10.png)

그런 다음 첫 번째 서스펜스

[![](https://jser.dev/static/suspenselist-11.png align="left")](https://jser.dev/static/suspenselist-11.png)

실제 구조는 Offscreen 컴포넌트로 더 복잡하므로 점선을 사용하여 폴백이 렌더링되었음을 표시하겠습니다.

[![](https://jser.dev/static/suspenselist-12.png align="left")](https://jser.dev/static/suspenselist-12.png)

결국 모든 서스펜스 폴백이 렌더링되고 `completeWork()`가 `SuspenseList`에서 다시 작동합니다.

[![](https://jser.dev/static/suspenselist-13.png align="left")](https://jser.dev/static/suspenselist-13.png)

이제 SuspenseList는 **일시 중단이 가능하므로,** 그 자식을 검색하여 일시 중단된 서스펜스를 발견하면 `DidCapture`가 설정되고 `ForceSuspenseFallback`이 `SuspenseContext`로 설정되며 전체 목록도 리렌더링됩니다.

[![](https://jser.dev/static/suspenselist-14.png align="left")](https://jser.dev/static/suspenselist-14.png)

다음 서스펜스로 넘어가는데, ForceSuspenseFallback이 있기 때문에 모든 서스펜스는 더 깊은 확인 없이 폴백을 렌더링합니다.

[![](https://jser.dev/static/suspenselist-15.png align="left")](https://jser.dev/static/suspenselist-15.png)

결국 SuspenseList에서 `completeWork()` 가 다시 호출되지만 렌더링할 `tail`이 남아 있지 않으므로 완료됩니다.

#### 두 번째 프로미스가 이행 될(fulfilled) 때.

[![](https://jser.dev/static/suspenselist-16.png align="left")](https://jser.dev/static/suspenselist-16.png)

프로세스는 이전과 유사하며, 먼저 플래그와 컨텍스트 플래그가 재설정되고, `tail`이 첫 번째 일시 중단된 서스펜스로 설정되며, `head`가 `tail`에서 연결됩니다.

[![](https://jser.dev/static/suspenselist-17.png align="left")](https://jser.dev/static/suspenselist-17.png)

`div`가 작업됩니다.

[![](https://jser.dev/static/suspenselist-18.png align="left")](https://jser.dev/static/suspenselist-18.png)

tail 렌더링 준비

[![](https://jser.dev/static/suspenselist-19.png align="left")](https://jser.dev/static/suspenselist-19.png)

첫 번째 서스펜스로 이동

[![](https://jser.dev/static/suspenselist-20.png align="left")](https://jser.dev/static/suspenselist-20.png)

이번에는 렌더링 대기 중인 `tail`이 있기 때문에 SuspenseList에서 `completeWork()`가 시작되고, SuspenseList는 그 자식을 검색하여 첫 번째 일시 중단된 Suspense를 찾아내어 플래그를 다시 설정합니다.

[![](https://jser.dev/static/suspenselist-21.png align="left")](https://jser.dev/static/suspenselist-21.png)

이제 tail이 2번째 서스펜스로 이동하고 프로미스가 이행되더라도 서스펜스 컨텍스트에서 ForceSuspenseFallback 플래그로 인해 여전히 폴백을 렌더링합니다.

공개 순서(reveal order)가 유지되는 방식은 이렇습니다.

나머지는 여기서 건너뛰겠습니다.

#### 첫 번째 프로미스가 해결되었을 때

기본적으로 흐름은 동일하며 처음 두 가지 프로미스들은 예외없이 이행되므로 내용을 공개합니다.

[![](https://jser.dev/static/suspenselist-22.png align="left")](https://jser.dev/static/suspenselist-22.png)

(원본 게시일: 2022-06-19)