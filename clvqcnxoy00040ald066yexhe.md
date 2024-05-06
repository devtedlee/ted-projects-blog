---
title: "[번역] React useImperativeHandle() 은 어떻게 동작하나요?"
datePublished: Fri May 03 2024 07:24:51 GMT+0000 (Coordinated Universal Time)
cuid: clvqcnxoy00040ald066yexhe
slug: react-internals-deep-dive-12
cover: https://cdn.hashnode.com/res/hashnode/image/upload/v1714721050713/6f070904-cfec-4f69-8e3f-74371f50ff57.jpeg
tags: reactjs, react-internals, useimperativehandle

---

> ***영문 블로그 글을 번역했습니다. 허가를 받으면 시리즈를 이어갈 예정입니다.  
> 원문링크:*** [https://jser.dev/react/2021/12/25/how-does-useImperativeHandle-work](https://jser.dev/react/2021/12/25/how-does-useImperativeHandle-work)

---

> ***ℹ️***[***React Internals Deep Dive***](https://jser.dev/series/react-source-code-walkthrough.html) ***에피소드 12,*** [***유튜브에서 제가 설명하는 것***](https://www.youtube.com/watch?v=iuKpQAhunac&list=PLvx8w9g4qv_p-OS-XdbB3Ux_6DMXhAJC3&index=12)***을 시청해주세요.***
> 
> ***⚠***[***React@18.2.0***](https://github.com/facebook/react/releases/tag/v18.2.0)***기준, 최신 버전에서는 구현이 변경되었을 수 있습니다.***
> 
> ***💬 역자 주석: Jser의 코멘트는 ❗❗로 표시 해뒀습니다.  
> 그 외 주석은 리액트 소스 코드 자체의 주석입니다.  
> ... 은 생략된 코드입니다.***

[useImperativeHandle()](https://reactjs.org/docs/hooks-reference.html#useimperativehandle)을 사용해본 적이 있나요? 내부적으로 어떻게 동작하는지 한번 알아보죠.

## 사용법

다음은 [공식 사용 예시](https://reactjs.org/docs/hooks-reference.html#useimperativehandle)입니다.

```typescript
function FancyInput(props, ref) {
  const inputRef = useRef();
  useImperativeHandle(ref, () => ({
    focus: () => {
      inputRef.current.focus();
    },
  }));
  return <input ref={inputRef} />;
}
FancyInput = forwardRef(FancyInput);
```

위의 코드를 통해 이제 `FancyInput`에 ref를 첨부할 수 있습니다.

```typescript
function App() {
  const ref = useRef();
  const focus = useCallback(() => {
    ref.current?.focus();
  }, []);
  return (
    <div>
      <FancyInput ref={inputRef} />
      <button onClick={focus} />
    </div>
  );
}
```

간단해 보이지만, 왜 이렇게 할까요?

## ref.current만 업데이트하면 어떨까요?

`ImperativeHandle()`을 사용하는 대신 아래와 같이 `ref.current`만 업데이트하면 어떨까요?

```typescript
function FancyInput(props, ref) {
  const inputRef = useRef();
  ref.current = () => ({
    focus: () => {
      inputRef.current.focus();
    },
  });
  return <input ref={inputRef} />;
}
```

사실 그냥 작동하지만 문제가 있습니다. `FancyInput`은 정리가 아닌 수락된 ref의 `current`만 설정합니다.

[React 내부 심층 분석 11 - useRef()는 어떻게 작동할까요?](https://ted-projects.com/react-internals-deep-dive-11) 에서 설명한 것처럼, React는 요소에 연결된 ref를 자동으로 정리하지만 이제는 그렇지 않습니다.

렌더링 도중 `ref`가 변경되면 어떻게 하나요? 그러면 이전 ref가 여전히 ref를 보유하게 되므로 `<FancyInput ref={inputRef} />`를 사용하려면 정리해야 합니다.

이 문제를 어떻게 해결할 수 있을까요? 정리하는 데 도움이 될 수 있는 `useEffect()`가 있으므로 다음과 같은 방법을 시도해 볼 수 있습니다.

```typescript
function FancyInput(props, ref) {
  const inputRef = useRef();
  useEffect(() => {
    ref.current = () => ({
      focus: () => {
        inputRef.current.focus();
      },
    });
    return () => {
      ref.current = null;
    };
  }, [ref]);
  return <input ref={inputRef} />;
}
```

하지만 잠깐만요, `ref`가 함수 ref가 아닌 RefObject인지 어떻게 알 수 있을까요? 그럼 확인해보겠습니다.

```typescript
function FancyInput(props, ref) {
  const inputRef = useRef();
  useEffect(() => {
    if (typeof ref === "function") {
      ref({
        focus: () => {
          inputRef.current.focus();
        },
      });
    } else {
      ref.current = () => ({
        focus: () => {
          inputRef.current.focus();
        },
      });
    }
    return () => {
      if (typeof ref === "function") {
        ref(null);
      } else {
        ref.current = null;
      }
    };
  }, [ref]);
  return <input ref={inputRef} />;
}
```

이것은 실제로 `useImperativeHandle()`의 작동 방식과 매우 유사합니다. `useImperativeHandle()`이 레이아웃 이펙트라는 점을 제외하면, ref 설정은 `useEffect()`보다 빠른 `useLayoutEffect()`와 동일한 단계에서 이루어집니다.

> ℹ [유튜브에서 useLayoutEffect 에 대해 설명](https://www.youtube.com/watch?v=6HLvyiYv7HI) 하는 모습 보기

## 이제 소스 코드를 살펴봅시다

이펙트의 경우 마운트 및 업데이트가 있으며, `useImperativeHandle()`이 호출되는 시점에 따라 달라집니다.

이것은 `mountImperativeHandle()`,[(원본 코드](https://github.com/facebook/react/blob/4729ff6d1f191902897927ff4ecd3d1f390177fa/packages/react-reconciler/src/ReactFiberHooks.new.js#L1799-L1835))의 단순화된 버전입니다.

```typescript
function mountImperativeHandle<T>(
  ref: {|current: T | null|} | ((inst: T | null) => mixed) | null | void,
  create: () => T,
  deps: Array<mixed> | void | null,
): void {
  return mountEffectImpl(
    fiberFlags,
    HookLayout,
    imperativeHandleEffect.bind(null, create, ref),
    effectDeps,
  );
}
```

또한 업데이트의 경우, [원본 코드](https://github.com/facebook/react/blob/4729ff6d1f191902897927ff4ecd3d1f390177fa/packages/react-reconciler/src/ReactFiberHooks.new.js#L1837-L1862)

```typescript
function updateImperativeHandle<T>(
  ref: {| current: T | null |} | ((inst: T | null) => mixed) | null | void,
  create: () => T,
  deps: Array<mixed> | void | null
): void {
  // TODO: If deps are provided, should we skip comparing the ref itself?
  const effectDeps =
    deps !== null && deps !== undefined ? deps.concat([ref]) : null;
  return updateEffectImpl(
    UpdateEffect,
    HookLayout,
    imperativeHandleEffect.bind(null, create, ref),
    effectDeps
  );
}
```

다음 사항에 유의하세요.

1. 내부적으로는 `mountEffectImpl`과 `updateEffectImpl`이 사용됩니다. `useEffect()` 및 `useLayoutEffect()`는 [여기](https://github.com/facebook/react/blob/4729ff6d1f191902897927ff4ecd3d1f390177fa/packages/react-reconciler/src/ReactFiberHooks.new.js#L1698)와 [여기](https://github.com/facebook/react/blob/4729ff6d1f191902897927ff4ecd3d1f390177fa/packages/react-reconciler/src/ReactFiberHooks.new.js#L1744-L1759)에서 동일한 작업을 수행합니다.
    
2. 두 번째 인수는 `HookLayout`으로 레이아웃 이펙트를 의미합니다.
    

퍼즐의 마지막 조각, `imperativeHandleEffect()`의 작동 방식입니다.[(코드](https://github.com/facebook/react/blob/4729ff6d1f191902897927ff4ecd3d1f390177fa/packages/react-reconciler/src/ReactFiberHooks.new.js#L1769-L1797))

```typescript
function imperativeHandleEffect<T>(
  create: () => T,
  ref: {| current: T | null |} | ((inst: T | null) => mixed) | null | void
) {
  if (typeof ref === "function") {
    const refCallback = ref;
    const inst = create();
    refCallback(inst);
    return () => {
      refCallback(null);
    };
  } else if (ref !== null && ref !== undefined) {
    const refObject = ref;
    const inst = create();
    refObject.current = inst;
    return () => {
      refObject.current = null;
    };
  }
}
```

완벽함을 위한 디테일을 제쳐두면, 실제로는 우리가 쓴 것과 매우 비슷해 보이죠?

## 마무리

`useImperativeHandle()` 은 마법이 아니며, 단지 ref 설정과 정리를 래핑할 뿐이며, 내부적으로는 `useLayoutEffect()`와 같은 단계에 있으므로 `useEffect()`보다 조금 더 빠릅니다.

(원본 게시일: 2021-12-25)