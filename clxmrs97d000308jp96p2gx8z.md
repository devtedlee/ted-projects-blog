---
title: "[번역] React의 Portal은 어떻게 동작하나요?"
datePublished: Thu Jun 20 2024 04:36:26 GMT+0000 (Coordinated Universal Time)
cuid: clxmrs97d000308jp96p2gx8z
slug: react-internals-deep-dive-26
cover: https://cdn.hashnode.com/res/hashnode/image/upload/v1718858147716/a213dd1f-fef4-4374-bc0f-33743f7ed42b.jpeg
tags: react-internals, react-portal

---

> ***영문 블로그 글을 번역했습니다. 허가를 받으면 시리즈를 이어갈 예정입니다.  
> 원문링크:*** [https://jser.dev/react/2022/09/24/how-does-react-portal-work](https://jser.dev/react/2022/09/24/how-does-react-portal-work)

---

> ***ℹ️***[***React Internals Deep Dive***](https://jser.dev/series/react-source-code-walkthrough.html)***에피소드 26,*** [***유튜브에서 제가 설명하는 것***](https://www.youtube.com/watch?v=GVefHHDVltQ&list=PLvx8w9g4qv_p-OS-XdbB3Ux_6DMXhAJC3&index=26)***을 시청해주세요.***
> 
> ***⚠***[***React@18.2.0***](https://github.com/facebook/react/releases/tag/v18.2.0)***기준, 최신 버전에서는 구현이 변경되었을 수 있습니다.***
> 
> ***💬 역자 주석: JSer의 코멘트는 ❗❗로 표시 해뒀습니다.  
> 그 외 주석은 리액트 소스 코드 자체의 주석입니다.  
> ... 은 생략된 코드입니다.***

## React Portal 데모

[React Portal](https://reactjs.org/docs/portals.html)은 모달을 다룰 때 매우 유용합니다. 모달 DOM을 다른 레이어에 배치하는 동시에 모달 자체는 React 파이버 트리에 그대로 두고 이벤트 전파를 유지할 수 있습니다.

[간단한 데모](https://jser.dev/demos/react/portal/)로 이동해 보겠습니다.

[![](https://jser.dev/static/portal-1.gif align="left")](https://jser.dev/static/portal-1.gif)

모달은 App 내부에서 렌더링되지만 DOM 자체는 루트 컨테이너 외부에 있습니다.

```typescript
function App() {
  const [showModal, setShowModal] = useState(false);
  return (
    <div>
      <button onClick={() => setShowModal(true)}>show modal</button>
      {showModal && (
        <Modal>
          <div>
            <p>Hello Modal</p>
            <button onClick={() => setShowModal(false)}>hide modal</button>
          </div>
        </Modal>
      )}
    </div>
  );
}
```

[![](https://jser.dev/static/portal-2.png align="left")](https://jser.dev/static/portal-2.png)

## Portal을 직접 만들 수 있을 것 같나요?

네, 기본적으로 element를 다른 곳에 렌더링하는 것이 맞죠? 아래와 같이 해보겠습니다.

```typescript
function Portal({ children, container }) {
  // 다른 돔에서 자식을 렌더링합니다.
  useLayoutEffect(() => {
    const root = ReactDOM.createRoot(container);
    root.render(children);
    return () => {
      root.unmount(container);
    };
  }, [children]);
  return null;
}
function Modal({ children }) {
  const el = document.createElement("div");
  ...
  return (
    <Portal container={el}>
      <div className="modal-inner">{children}</div>
    </Portal>
  );
}
```

[이 데모](https://jser.dev/demos/react/portal/by-ourself.html)에서 작동합니다.

문제는 이 접근 방식은 모달이 새로운 React 루트로 렌더링되기 때문에 컨텍스트 정보를 상속할 수 없다는 것입니다. 또한 성능 문제가 있는 새로운 React 루트를 생성합니다.

예를 들어, 빌트-인 포털을 사용하면 예상대로 컨텍스트를 얻을 수 있으며, [데모](https://jser.dev/demos/react/portal/context.html)는 다음과 같습니다.

[![](https://jser.dev/static/portal-3.png align="left")](https://jser.dev/static/portal-3.png)

하지만 우리가 구축한 포털에서는 작동하지 않습니다. [데모](https://jser.dev/demos/react/portal/context-by-ourself.html)는 다음과 같습니다.

[![](https://jser.dev/static/portal-4.png align="left")](https://jser.dev/static/portal-4.png)

## Portal은 실제로 내부적으로 어떻게 작동하나요?

[최초 마운트, 어떻게 작동하나요?](https://www.youtube.com/watch?v=EakHciGG3SM) 에서 리액트 파이버에서 실제 DOM으로의 동기화는 대략 다음과 같다는 것을 기억하세요.

1. 조정(reconcile) -&gt; 변경된 파이버가 있는지 감지하고, 변경된 경우 추가/제거 등과 같은 플래그로 표시합니다.
    
    * 1.1 완료(complete) -&gt; 파이버에 대한 DOM 요소를 생성하거나 이미 존재하는 경우 재사용합니다.
        
2. 커밋(commit) -&gt; 해당 플래그가 있는 각 파이버에 대해 그에 따라 DOM을 업데이트합니다.
    

파이버 노드에서 중요한 속성 중 하나는 실제 DOM에 대한 참조(내재적 요소의 경우)를 보유하는 `stateNode`입니다.

Portal의 특별한 점은 아래 구조에 표시된 것처럼 DOM이 있는 위치만 다르다는 것입니다.

```bash
fiber:         Parent > div > Modal > div
DOM(stateNode):     . > div >    .  > div
```

이 stateNode가 DOM 세계에서 실제 구조를 가지고 있음을 알 수 있지만, Portal에서는 상황이 달라져야 합니다.

```bash
fiber:        Parent > div > Modal > Portal > div
DOM(stateNode):   .  > div >   .   >
                                            > div
```

Portal이 내부적으로 실제로 하는 일은 포털이 대상 컨테이너 요소의 stateNode를 스스로 보유하도록 하는 것입니다.

```bash
fiber:        Parent > div > Modal > Portal > div
DOM(stateNode):   . > div >   .    >
                                  container? > div
```

조정의 특성 상 DOM 구조는 React 런타임에 불투명하기 때문에, Portal의 경우 커밋 단계에서 컨테이너를 관리하는 방법에만 집중하면 모든 것이 똑같이 작동할 수 있습니다.

### 1\. createPortal()은 특별한 엘리먼트를 반환합니다.

```typescript
export function createPortal(
  children: ReactNodeList,
  containerInfo: any,
  implementation: any,
  key: ?string = null
): ReactPortal {
  return {
    // This tag allow us to uniquely identify this as a React Portal
    $$typeof: REACT_PORTAL_TYPE,
    key: key == null ? null : "" + key,
    children,
    containerInfo,
    implementation,
  };
}
```

특별할것은 없습니다.

### 2\. createChild()는 Portal을 다르게 다룹니다.

```typescript
function createChild(
  returnFiber: Fiber,
  newChild: any,
  lanes: Lanes
): Fiber | null {
  if (
    (typeof newChild === "string" && newChild !== "") ||
    typeof newChild === "number"
  ) {
    // Text nodes don't have keys. If the previous node is implicitly keyed
    // we can continue to replace it without aborting even if it is not a text
    // node.
    const created = createFiberFromText("" + newChild, returnFiber.mode, lanes);
    created.return = returnFiber;
    return created;
  }
  if (typeof newChild === "object" && newChild !== null) {
    switch (newChild.$$typeof) {
      case REACT_ELEMENT_TYPE: {
        const created = createFiberFromElement(
          newChild,
          returnFiber.mode,
          lanes
        );
        created.ref = coerceRef(returnFiber, null, newChild);
        created.return = returnFiber;
        return created;
      }
      case REACT_PORTAL_TYPE: {
        const created = createFiberFromPortal(
          newChild,
          returnFiber.mode,
          lanes
        );
        created.return = returnFiber;
        return created;
      }
      ...
  }
  return null;
}
```

`createFiberFromPortal()`이 Portal을 위해 사용되는 것을 볼 수 있습니다.

```typescript
export function createFiberFromPortal(
  portal: ReactPortal,
  mode: TypeOfMode,
  lanes: Lanes
): Fiber {
  const pendingProps = portal.children !== null ? portal.children : [];
  const fiber = createFiber(HostPortal, pendingProps, portal.key, mode);
  fiber.lanes = lanes;
  fiber.stateNode = {
    containerInfo: portal.containerInfo,
    pendingChildren: null, // Used by persistent updates
    implementation: portal.implementation,
  };
  return fiber;
}
```

Portal에 대한 `stateNode`는 `containerInfo`를 보유한 객체이며, 파이버 타입도 `HostPortal`임을 알 수 있습니다.

```typescript
export function createFiberFromElement(
  element: ReactElement,
  mode: TypeOfMode,
  lanes: Lanes
): Fiber {
  let owner = null;
  const type = element.type;
  const key = element.key;
  const pendingProps = element.props;
  const fiber = createFiberFromTypeAndProps(
    type,
    key,
    pendingProps,
    owner,
    mode,
    lanes
  );
  return fiber;
}
```

계층 구조 내에서 DOM을 생성해야 하므로 `stateNode`가 설정되지 않는 `createFiberFromElement()`와는 다릅니다. 따라서 커밋 단계까지 설정되지 않습니다. 하지만 Portal의 경우 이미 루트가 어디에 있는지 알고 있습니다.

### 3\. commitPlacement() 에 매직이 있습니다.

`commitPlacement()` 는 실제 DOM 조작에 대해 앞서 언급한 것입니다. `Placement`는 조정 중 새 DOM을 삽입해야 하는 플래그입니다.

```typescript
function commitPlacement(finishedWork: Fiber): void {
  // Recursively insert all host nodes into the parent.
  const parentFiber = getHostParentFiber(finishedWork);
  // Note: these two variables *must* always be updated together.
  switch (parentFiber.tag) {
    case HostComponent: {
      const parent: Instance = parentFiber.stateNode;
      if (parentFiber.flags & ContentReset) {
        // Reset the text content of the parent before doing any insertions
        resetTextContent(parent);
        // Clear ContentReset from the effect tag
        parentFiber.flags &= ~ContentReset;
      }
      const before = getHostSibling(finishedWork);
      // We only have the top Fiber that was inserted but we need to recurse down its
      // children to find all the terminal nodes.
      insertOrAppendPlacementNode(finishedWork, before, parent);
      break;
    }
    case HostRoot:
    case HostPortal: {
      const parent: Container = parentFiber.stateNode.containerInfo;
      const before = getHostSibling(finishedWork);
      insertOrAppendPlacementNodeIntoContainer(finishedWork, before, parent);
      break;
    }
    // eslint-disable-next-line-no-fallthrough
    default:
      throw new Error(
        "Invalid host parent fiber. This error is likely caused by a bug " +
          "in React. Please file an issue."
      );
  }
}
```

내재 요소인 HostComponent의 경우 DOM 요소를 부모에 추가하기만 하면 됩니다. DOM 요소의 생성은 [completeWork()](https://github.com/facebook/react/blob/e61fd91f5c523adb63a3b97375ac95ac657dc07f/packages/react-reconciler/src/ReactFiberCompleteWork.new.js#L1007)에 있습니다.

Portal의 경우 - 대상 컨테이너에 DOM 요소를 추가합니다. 간단합니다.

이제 Portal이 내부적으로 어떻게 작동하는지 알게 되었으며, React 런타임의 깔끔한 아키텍처 덕분에 실제로는 매우 간단합니다.

(원본 게시일: 2022-09-24)