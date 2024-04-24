---
title: "[번역] React에서 Context는 어떻게 동작하나요?"
datePublished: Wed Apr 24 2024 12:49:37 GMT+0000 (Coordinated Universal Time)
cuid: clvdtaxik00010ali4fze91fb
slug: react-internals-deep-dive-9
cover: https://cdn.hashnode.com/res/hashnode/image/upload/v1713962934193/0bff74ea-091b-4efb-87bd-d17956563415.jpeg
tags: reactjs, context, react-internals

---

> ***영문 블로그 글을 번역했습니다. 허가를 받으면 시리즈를 이어갈 예정입니다.  
> 원문링크:***[https://jser.dev/react/2021/07/28/how-does-context-work](https://jser.dev/react/2021/07/28/how-does-context-work)

---

> ***ℹ️***[***React Internals Deep Dive***](https://jser.dev/series/react-source-code-walkthrough.html)***에피소드 9,***[***유튜브에서 제가 설명하는 것***](https://www.youtube.com/watch?v=nygcudhNuII&list=PLvx8w9g4qv_p-OS-XdbB3Ux_6DMXhAJC3&index=9)***을 시청해주세요.***
> 
> ***⚠***[***React@18.2.0***](https://github.com/facebook/react/releases/tag/v18.2.0)***기준, 최신 버전에서는 구현이 변경되었을 수 있습니다.***
> 
> ***💬 역자 주석: Jser의 코멘트는 ❗❗로 표시 해뒀습니다.  
> 그 외 주석은 리액트 소스 코드 자체의 주석입니다.  
> ... 은 생략된 코드입니다.***

컴포넌트 트리(tree of components)로 React 코드를 작성하면, `props`로 데이터를 전달할 수 있지만 트리가 매우 깊은 경우에는 번거롭습니다. 예를 들어, 모든 컴포넌트에 사용될 수 있는 전역 색을 정의하려면 결국 모든 컴포넌트가 이를 지원하기 위해 새로운 prop이 필요할 수 있습니다.

[Context](https://reactjs.org/docs/context.html)는 이 문제를 해결하기 위해 `props` 없이 하위 트리에 데이터를 전달할 수 있게 해줍니다.

## 1\. React Context 데모

여기에 Context를 사용하는 간단한 리액트 앱이 있습니다. 이건 그저 `JSer`를 렌더링 합니다.

[데모 링크(코드 샌드박스)](https://codesandbox.io/p/sandbox/jolly-mendeleev-rqxscy?file=%2FApp.js&utm_medium=sandpack)

```typescript
import { createContext } from 'react';

const Context = createContext("123");

function Component1() {
  return <Component2 />;
}

function Component2() {
  return <Context.Consumer>
    {(value) => value}
  </Context.Consumer>;
}
```

이 데모는 매우 간단한데, 자 이제 React Context가 내부적으로 어떻게 동작하는지 알아봅시다.

## 2\. React.createContext()

```typescript
import { REACT_PROVIDER_TYPE, REACT_CONTEXT_TYPE } from "shared/ReactSymbols";
import type { ReactContext } from "shared/ReactTypes";
export function createContext<T>(defaultValue: T): ReactContext<T> {
  const context: ReactContext<T> = {
    $$typeof: REACT_CONTEXT_TYPE,
    // As a workaround to support multiple concurrent renderers, we categorize
    // some renderers as primary and others as secondary. We only expect
    // there to be two concurrent renderers at most: React Native (primary) and
    // Fabric (secondary); React DOM (primary) and React ART (secondary).
    // Secondary renderers store their context values on separate fields.
    _currentValue: defaultValue,
    _currentValue2: defaultValue,
    // Used to track how many concurrent renderers this context currently
    // supports within in a single renderer. Such as parallel server rendering.
    _threadCount: 0,
    // These are circular
    Provider: (null: any),
    Consumer: (null: any),
  };
  context.Provider = { // ❗❗ Provider 
    $$typeof: REACT_PROVIDER_TYPE,
    _context: context,
    // ❗❗ ↖ Provider는 부모 ref로서 _context를 갖고 있습니다.
  };
  context.Consumer = context; // ❗❗ Consumer 
  return context;
}
```

즉 `createContext()`는 기본값을 보유한 객체를 반환하고 `Provider`와 `Consumer`를 노출하기만 하면 됩니다.

1. `Provider`는 `REACT_PROVIDER_TYPE`의 특수 엘리먼트 타입으로, 곧 다룰 예정입니다.
    
2. `Consumer`는 흥미로운데, 이건 `context`에 설정됩니다.
    

위의 데모 코드와 마찬가지로 `Provider`와 `Consumer`는 실제로 렌더링에 사용되므로 `Context`는 이들을 페어링하는 방법입니다.

## 3\. Provider

[여기서](https://github.com/facebook/react/blob/848e802d203e531daf2b9b0edb281a1eb6c5415d/packages/react-reconciler/src/ReactFiber.new.js#L527) 엘리먼트 타입 `REACT_PROVIDER_TYPE`은 파이버 태그 `ContextProvider`에 매핑됩니다.

Provider의 유일한 목적은`<Provider value={...}/>` 구문에서 `Consumer`가 사용하는 값을 설정하는 것이며, 아래는 렌더링 중에 Provider가 처리되는 방식입니다.

> ℹ 렌더링 내부에 대한 자세한 내용은 [React가 최초 마운트를 수행하는 방법](https://ted-projects.com/react-internals-deep-dive-2) 과 [React가 리-렌더링하는 방법을](https://ted-projects.com/react-internals-deep-dive-3) 참조하십시오.

```typescript
function beginWork() {
  case ContextProvider:
      return updateContextProvider(current, workInProgress, renderLanes);
}
function updateContextProvider(
  current: Fiber | null,
  workInProgress: Fiber,
  renderLanes: Lanes,
) {
  const providerType: ReactProviderType<any> = workInProgress.type;
  const context: ReactContext<any> = providerType._context;
    // ❗❗                                         ↗
    // ❗❗ 따라서 Privder 타입에서 내부 Context를 쉽게 가져올 수 있습니다.

  const newProps = workInProgress.pendingProps;
  const oldProps = workInProgress.memoizedProps;
  const newValue = newProps.value;
  pushProvider(workInProgress, context, newValue);
  // ❗❗ ↖ 값이 어디론가 '푸시' 됩니다.

  if (enableLazyContextPropagation) {
    // In the lazy propagation implementation, we don't scan for matching
    // consumers until something bails out, because until something bails out
    // we're going to visit those nodes, anyway. The trade-off is that it shifts
    // responsibility to the consumer to track whether something has changed.
  } else {
    if (oldProps !== null) {
      const oldValue = oldProps.value;
      if (is(oldValue, newValue)) { // ❗❗ is(oldValue, newValue)
        // No change. Bailout early if children are the same.
        if (
          oldProps.children === newProps.children && // ❗❗ oldProps.children === newProps.children
          !hasLegacyContextChanged()
        ) {
          return bailoutOnAlreadyFinishedWork(
            current,
            workInProgress,
            renderLanes,
          );
    // ❗❗ 컨텍스트 값과 자식 컴포넌트들에서 업데이트가 없으면, 리액트는 bail out 합니다.
    // ❗❗ 리액트가 서브트리의 렌더링을 건너 뛴다는 의미입니다.
        }
      } else {
        // The context value changed. Search for matching consumers and schedule
        // them to update.
        propagateContextChange(workInProgress, context, renderLanes);
        // ❗❗ ↖ 컨텍스트 값이 바뀌면, 리액트는 페어링된 Consumerd에게 업데이트하라고 알려야만 합니다.
      }
    }
  }
  const newChildren = newProps.children;
  reconcileChildren(current, workInProgress, newChildren, renderLanes);
  return workInProgress.child;
  // ❗❗ ↖ 자식을 리턴함으로서 렌더링을 계속 진행합니다.
  // ❗❗ 자세한 내용은 React가 내부적으로 파이버 트리를 순회하는 방법을 참조하십시오.
}
```

렌더링 중에 Provider가 작동하는 방식을 요약해 보겠습니다.

1. 호출하면 먼저 `pushProvider()`로 새 값을 업데이트합니다.
    
2. 값이 변하지 않으면, [bailout](https://jser.dev/react/2022/01/07/how-does-bailout-work)을 시도합니다
    
3. 그렇지 않으면 `propagateContextChange()`로 Consumer를 업데이트하고 자식들을 렌더링합니다.
    

### 3.1 `pushProvider()`

```typescript
export function pushProvider<T>(
  providerFiber: Fiber,
  context: ReactContext<T>,
  nextValue: T
): void {
  if (isPrimaryRenderer) {
    push(valueCursor, context._currentValue, providerFiber);
    // ❗❗ 값을 파이버 스택에 'push'하여 경로를 따라 컨텍스트 정보를 수집합니다.
    context._currentValue = nextValue;
    // ❗❗ 컨텍스트를 설정하여, Consumer가 쉽게 접근할 수 있도록 해줍니다.
  } else {
    ...
  }
}
```

이렇게 값이 [Fiber 스택](https://github.com/facebook/react/blob/49f8254d6dec7c6de789116e20ffc5a9401aa725/packages/react-reconciler/src/ReactFiberStack.new.js#L59) 구조로 푸시됩니다.

**파이버 스택은 루트에서 현재** [](https://github.com/facebook/react/blob/49f8254d6dec7c6de789116e20ffc5a9401aa725/packages/react-reconciler/src/ReactFiberStack.new.js#L59)**파이버까지의 경로를 따라 정보를 보관한다고** 생각할 수 있습니다. 동일한 Context에 여러 Provider가 있는 경우 스택 구조는 Fiber 노드에 대해 가장 가까운 값을 사용하도록 합니다.

컨텍스트 자체는 최신 값으로 설정되어 있지만 Fiber 스택은 **이전 값**을 저장하고 있는데, 이는 컨텍스트에 기본값이 있기 때문입니다.

### 3.2 `popProvider()`

푸시(push)가 있는 곳에는 팝(pop)이 있습니다. [completeWork()](https://github.com/facebook/react/blob/49f8254d6dec7c6de789116e20ffc5a9401aa725/packages/react-reconciler/src/ReactFiberCompleteWork.new.js#L1259-L1264)에서 `popProvider()`가 호출됩니다.

이는 파이버 트리가 `workInProgress`를 순회 중일 때 컨텍스트가 올바른 값을 반영하고 있는지 확인하기 위한 것입니다. 엘리먼트를 Provider의 형제(sibling)라고 생각하면, React가 이 파이버로 이동하면 Provider의 정보가 *없어야* 하므로, Context 정보를 팝(pop)해야 합니다.

Fiber 스택은 루트에서 현재 파이버 노드까지의 경로를 따라 컨텍스트 정보들을 보유한다는 점을 유의하세요.

```typescript
export function popProvider(
  context: ReactContext<any>,
  providerFiber: Fiber
): void {
  const currentValue = valueCursor.current;
    // ❗❗             ↗ 파이버 스택의 Context 값은 이전(previous) 값입니다.
  pop(valueCursor, providerFiber);
  if (isPrimaryRenderer) {
    if (
      enableServerContext &&
      currentValue === REACT_SERVER_CONTEXT_DEFAULT_VALUE_NOT_LOADED
    ) {
      context._currentValue = context._defaultValue;
    } else {
      context._currentValue = currentValue;
      // ❗❗ 컨텍스트 값을 이전 값으로 설정
    }
  } else {
    ...
  }
}
```

파이버 스택이 이전 값을 저장하기 때문에 `popProvider()`가 하는 일은 간단합니다. 컨텍스트에 설정하고 팝하기만 하면 끝입니다!

## 4\. Consumer

`propagateContextChange()`의 작동 방식을 이해하려면 먼저 `Consumer`의 작동 방식을 이해해야 합니다.

위에서 언급했듯이, `Consumer`는 실제로 컨텍스트 자체이며 `ContextConsumer`파이버 태그가 사용됩니다[(소스)](https://github.com/facebook/react/blob/49f8254d6dec7c6de789116e20ffc5a9401aa725/packages/react-reconciler/src/ReactFiber.new.js#L546-L548).

```typescript
function updateContextConsumer(
  current: Fiber | null,
  workInProgress: Fiber,
  renderLanes: Lanes
) {
  let context: ReactContext<any> = workInProgress.type;
  // ❗❗ ↗ Consumer 타입 자체가 컨텍스트 입니다!
  const newProps = workInProgress.pendingProps;
  const render = newProps.children;
  prepareToReadContext(workInProgress, renderLanes);
  
  const newValue = readContext(context);
  // ❗❗ ↗ Context Value를 읽습니다.
  let newChildren;
  newChildren = render(newValue);
  // ❗❗         ↗ Consumer는 Render Prop 패턴에 사용해야 합니다.
  // ❗❗ <Consumer>{(val) => ...}</Consumer>
  // React DevTools reads this flag.
  workInProgress.flags |= PerformedWork;
  reconcileChildren(current, workInProgress, newChildren, renderLanes);
  return workInProgress.child;
  // ❗❗ 자식들을 계속해서 렌더링합니다.
}
```

좋아요. 이것은 사실 아주 간단합니다.

1. 먼저 `prepareToReadContext()`함수를 실행하고
    
2. 그리고 `readContext()` 로 값을 읽고
    
3. 소비자가 자식의 렌더링 prop을 예상하기 때문에 `newChildren = render(newValue)`로 설정합니다.
    

### 4.1 `prepareToReadContext()`

```typescript
export function prepareToReadContext(
  workInProgress: Fiber,
  renderLanes: Lanes
): void {
  currentlyRenderingFiber = workInProgress;
  lastContextDependency = null;
  lastFullyObservedContext = null;
  const dependencies = workInProgress.dependencies;
    // ❗❗                             ↗ 
    // ❗❗ 컴포넌트는 여러 컨텍스트를 사용할 수 있으며,
    // ❗❗ dependencies는 사용 중인 컨텍스트를 의미합니다
  if (dependencies !== null) {
    if (enableLazyContextPropagation) {
      // Reset the work-in-progress list
      dependencies.firstContext = null;
    } else {
      const firstContext = dependencies.firstContext;
      if (firstContext !== null) {
        if (includesSomeLane(dependencies.lanes, renderLanes)) {
          // Context list has a pending update. Mark that this fiber performed work.
          markWorkInProgressReceivedUpdate();
        }
        // Reset the work-in-progress list
        dependencies.firstContext = null;
        // ❗❗         ↗ dependencies를 리셋합니다!
        // ❗❗ 이것은 컨텍스트 자체에 불과하기 때문에 <Consumer>{..}</Consumer> 에는 큰 의미가 없지만
        // ❗❗ 컴포넌트가 여러번 호출할 가능성이 있는 useContext()에는 더 유용합니다.
      }
    }
  }
}
```

`prepareToReadContext()`가 `updateContextConsumer()`에서만 호출되는 것이 *아니라* 기본적으로 모든 컴포넌트를 렌더링할 때 호출된다는 점을 기억하세요. `dependencies`는 컴포넌트에 사용된 모든 Context를 추적하여, 나중에 컨텍스트 값이 변경될 때, React가 어떤 컴포넌트를 업데이트할지 알 수 있도록 하기 위한 것임을 잊지 마세요.

### 4.2 `readContext()`

```typescript
export function readContext<T>(context: ReactContext<T>): T {
  const value = isPrimaryRenderer
    ? context._currentValue
    : context._currentValue2;
  if (lastFullyObservedContext === context) {
    // Nothing to do. We already observe everything in this context.
  } else {
    const contextItem = { // ❗❗ 
      context: ((context: any): ReactContext<mixed>),
      memoizedValue: value,
      next: null,
    };
    if (lastContextDependency === null) {
      // This is the first dependency for this component. Create a new list.
      lastContextDependency = contextItem;
      currentlyRenderingFiber.dependencies = { // ❗❗ dependencies 
        lanes: NoLanes,
        firstContext: contextItem,
      };
      if (enableLazyContextPropagation) {
        currentlyRenderingFiber.flags |= NeedsPropagation;
      }
    } else {
      // Append a new context item.
      lastContextDependency = lastContextDependency.next = contextItem;
                                              // ❗❗ ↗ dependencies는 링크드 리스트 입니다.
    }ㅋ
  }
  return value;
}
```

파이버의 경우 여러 컨텍스트를 사용할 수 있으므로, dependencies는 실제로 링크드 리스트입니다.

`readContext()`는 컨텍스트에서 값을 읽고 dependencies을 업데이트하기만 하면 됩니다.

### 4.3 `useContext()`는 `readContext()`의 alias 입니다.

```typescript
const HooksDispatcherOnMount: Dispatcher = {
  useCallback: mountCallback,
  useContext: readContext, // ❗❗ 
  useEffect: mountEffect,
  ...
};
const HooksDispatcherOnUpdate: Dispatcher = {
  useCallback: updateCallback,
  useContext: readContext, // ❗❗ 
  useEffect: updateEffect,
  ...
};
```

네, 간단합니다.

## 5\. `propagateContextChange()`

컨텍스트의 값이 변경되면, 모든 소비자에 대한 업데이트를 예약해야 하는데, 이는 업데이트가 어떻게든 건너뛰어지지 않도록 하기 위해서입니다.

**업데이트 스케줄링**은 기본적으로 루트에서 파이버 노드까지 경로의 모든 노드에 대한 `lanes`와 `childLanes`를 업데이트하는 것을 의미합니다. 이에 대한 자세한 내용은 [조정에서 React bailout이 작동하는 방식에서](https://jser.dev/react/2022/01/07/how-does-bailout-work) 확인할 수 있습니다.

[소스 코드](https://github.com/facebook/react/blob/49f8254d6dec7c6de789116e20ffc5a9401aa725/packages/react-reconciler/src/ReactFiberNewContext.new.js#L219)에는 꽤 많은 줄이 있으므로 여기에 붙여넣지 않겠습니다.

아이디어는 꽤 간단한데, 코드를 보면, 이 Provider 아래의 하위 트리가 순회되고, 각 파이버에 대해 `dependencies`가 확인되며, 만약 여기서 컨텍스트가 사용되는 것을 발견되면 `scheduleContextWorkOnParentPath()`를 호출하여 일부 작업을 예약한다는 것을 쉽게 알 수 있습니다.

Provider의 `beginWork()` 단계에서 스캔될 때까지 기다렸다가, 소비자가 아직 트리에 없는 경우면 어떻게 하냐고 질문할 수 있습니다.

좋은 질문입니다. 언뜻 보기에는 `Consumer` 노드가 렌더링된 후에만 종속성이 업데이트되므로 순서가 조금 이상해 보입니다. 하지만 실제로는 렌더링 중에 Consumer 노드가 추가되면 자동으로 최신 값을 사용하므로 한 번만 스캔하면 됩니다.

## 6\. 요약

컨텍스트는 사실 이해하기가 그리 복잡하지 않습니다. 핵심 아이디어는 **경로를 따라 정보를 저장할 수 있는 파이버 스택**입니다.

## 7\. 코딩 챌린지

이제 이해를 돕기 위해 제가 만든 간단한 코딩 챌린지를 소개합니다. 즐겨보세요!

* JSer 블로그 [링크](https://jser.dev/react/2021/07/28/how-does-context-work#7-coding-challenge)
    
* BFE.dev [링크](https://bigfrontend.dev/open_problem/ktqp92jl5iuhojaohzb)
    

(원글 게시일 2021-07-28)