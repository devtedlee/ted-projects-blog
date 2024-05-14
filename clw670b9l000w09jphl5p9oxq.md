---
title: "[번역] React.useDeferredValue() 는 어떻게 동작하나요?"
datePublished: Tue May 14 2024 09:30:49 GMT+0000 (Coordinated Universal Time)
cuid: clw670b9l000w09jphl5p9oxq
slug: react-internals-deep-dive-17
cover: https://cdn.hashnode.com/res/hashnode/image/upload/v1715673532156/73189a1f-7082-4abe-b009-efcd700732c4.jpeg
tags: reactjs, react-internals, usedeferredvalue

---

> ***영문 블로그 글을 번역했습니다. 허가를 받으면 시리즈를 이어갈 예정입니다.  
> 원문링크:***[https://jser.dev/react/2022/01/26/how-does-react-usedeferredvalue-work](https://jser.dev/react/2022/01/26/how-does-react-usedeferredvalue-work)

---

> ***ℹ️***[***React Internals Deep Dive***](https://jser.dev/series/react-source-code-walkthrough.html) ***에피소드 17,*** [***유튜브에서 제가 설명하는 것***](https://www.youtube.com/watch?v=db31-3xw_3U&list=PLvx8w9g4qv_p-OS-XdbB3Ux_6DMXhAJC3&index=17)***을 시청해주세요.***
> 
> ***⚠***[***React@18.2.0***](https://github.com/facebook/react/releases/tag/v18.2.0)***기준, 최신 버전에서는 구현이 변경되었을 수 있습니다.***
> 
> ***💬 역자 주석: JSer의 코멘트는 ❗❗로 표시 해뒀습니다.  
> 그 외 주석은 리액트 소스 코드 자체의 주석입니다.  
> ... 은 생략된 코드입니다.***

React 동시 모드에서는 `useTransition()`과 `Suspense` 외에 `useDeferredValue()`라는 API가 하나 더 있는데, 이것이 무엇이고 어떻게 작동하는지 알아보겠습니다.

## `useDeferredValue()` 는 어떤 문제를 해결하려고 하나요?

[React 홈페이지에 자세한 답변](https://reactjs.org/docs/concurrent-mode-reference.html#usedeferredvalue)이 있지만 데모가 깨져 있습니다.

여기서 간단히 시연해 보겠습니다.

## useDeferredValue()가 없는 경우

이 [데모](https://jser.dev/demos/react/usedeferredvalue/no-defer.html)는 useDeferredValue()가 없는 경우입니다.

Next 버튼을 클릭하면, 제목과 게시물에 대한 두 개의 API 목(Mock)이 실행되며, 제목 API는 더 빠르고(300ms) 게시물 API는 느리게(1000ms) 실행됩니다. 다음 버튼을 클릭하면 약 1000ms 후에 제목과 글이 모두 전환되는 것을 확인할 수 있습니다.

[![](https://jser.dev/static/usedeferredvalue-1.gif align="left")](https://jser.dev/static/usedeferredvalue-1.gif)

이는 버튼 클릭 핸들러가 `useTransition()`을 사용하고 있기 때문인데, 게시글 API가 데이터를 반환할 때까지 에러가 발생해서, 제목과 글이 모두 지연되기 때문입니다.

이것은 좋지 않은데, 제목이 더 빨리 표시되지 않는 이유는 무엇인가요?

## useDeferredValue()가 있는 경우

이 [데모](https://jser.dev/demos/react/usedeferredvalue/no-defer.html)는 useDeferredValue()가 있는 경우입니다.

Next 버튼을 다시 한번 클릭하면, 제목이 먼저 보여지고, 게시글이 따라오는 것을 볼 수 있습니다.

[![](https://jser.dev/static/usedeferredvalue-2.gif align="left")](https://jser.dev/static/usedeferredvalue-2.gif)

이게 훨씬 낫습니다.

## useDeferredValue()를 직접 생성해보자

[useTransition()의 작동 방식](https://ted-projects.com/react-internals-deep-dive-8)은 이미 다루었습니다. 간단히 말해서 에러가 발생하면 커밋을 중지하는 것입니다.

여기서 우리의 문제는, 제목 API가 데이터를 반환하면 React가 리렌더링을 시도하지만 게시물이 렌더링될 때 데이터가 준비되지 않았기 때문에 에러를 던져서 발생하는 문제입니다.

```typescript
function ProfilePage({ resource }) {
  return (
    <React.Suspense fallback={<h1>Loading profile...</h1>}>
      <ProfileDetails resource={resource} />
      <React.Suspense fallback={<h1>Loading posts...</h1>}>
        <ProfileTimeline resource={resource} />
      </React.Suspense>
    </React.Suspense>
  );
}
```

이 문제를 해결하기 위해 게시물 섹션의 `리소스`를 state별로 캐시하고 `useEffect()`에서 업데이트할 수 있습니다.

```typescript
function useDeferredValue(value) {
  const [state, setState] = React.useState(value);
  React.useEffect(() => {
    // since value might be promise which causes suspension
    // we should wrap it with startTransition
    React.startTransition(() => {
      setState(value);
    });
  }, [value]);
  return state;
}
```

됐습니다. 이제 코드를 우리가 구현한 곳으로 변경합니다.

```typescript
function ProfilePage({ resource }) {
  const deferredResource = useDeferredValue(resource);
  return (
    <React.Suspense fallback={<h1>Loading profile...</h1>}>
      <ProfileDetails resource={resource} />
      <React.Suspense fallback={<h1>Loading posts...</h1>}>
        <ProfileTimeline resource={deferredResource} />
      </React.Suspense>
    </React.Suspense>
  );
}
```

열려있는 [데모에서 우리가 만든 useDeferredValue() 함수를 사용](https://jser.dev/demos/react/usedeferredvalue/my-defer.html)하면 `React.useDeferredValue()`와 동일하게 작동합니다.

## React.useDeferredValue()는 어떻게 동작 하나요?

debugger를 설정하면, 이전에 해왔던 것처럼 마운트용과 업데이트용 소스 코드를 찾을 수 있습니다.

```typescript
function mountDeferredValue<T>(value: T): T {
  const [prevValue, setValue] = mountState(value);
  mountEffect(() => {
    const prevTransition = ReactCurrentBatchConfig.transition;
    ReactCurrentBatchConfig.transition = 1;
    try {
      setValue(value);
    } finally {
      ReactCurrentBatchConfig.transition = prevTransition;
    }
  }, [value]);
  return prevValue;
}

function updateDeferredValue<T>(value: T): T {
  const [prevValue, setValue] = updateState(value);
  updateEffect(() => {
    const prevTransition = ReactCurrentBatchConfig.transition;
    ReactCurrentBatchConfig.transition = 1;
    try {
      setValue(value);
    } finally {
      ReactCurrentBatchConfig.transition = prevTransition;
    }
  }, [value]);
  return prevValue;
}
```

잠깐만요, 기본적으로 우리가 작성한 것과 동일합니다! `mountState`와 `updateState`는 `useState`에 대한 구현일 뿐입니다.

도대체 `ReactCurrentBatchConfig.transition이` 무엇인지 궁금하실 텐데요, `startTransition()`의 소스를 보겠습니다. [(소스](https://github.com/facebook/react/blob/e62a8d754548a490c2a3bcff3b420e5eedaf11c0/packages/react/src/ReactStartTransition.js))

```typescript
export function startTransition(scope: () => void) {
  const prevTransition = ReactCurrentBatchConfig.transition;
  ReactCurrentBatchConfig.transition = 1;
  try {
    scope();
  } finally {
    ReactCurrentBatchConfig.transition = prevTransition;
  }
}
```

똑같습니다!!! 하지만 `ReactCurrentBatchConfig.transition`은 도대체 무엇을 하는 것일까요?

이 함수 내부의 업데이트는 트랜지션 lanes에서 예약되어야 하며, 일시 중단 시 커밋되지 않도록 React에 알려주는 내부 구현입니다.

자세한 내용은 [제 동영상](https://www.youtube.com/watch?v=G0sHIjjiyJ0&t=2140s)에서 확인할 수 있습니다.

(원본 게시일: 2022-01-26)