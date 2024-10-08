---
title: "리액트 useSyncExternalStore 톺아보기"
datePublished: Tue Apr 02 2024 05:41:21 GMT+0000 (Coordinated Universal Time)
cuid: cluhybfw2000509jwax9240wg
slug: react-use-sync-external-store
cover: https://cdn.hashnode.com/res/hashnode/image/upload/v1712036410396/65d1b619-d01f-4b57-9d00-85802d3f0792.webp
tags: reacthooks, react-18

---

`useSyncExternalStore`는 React 18에서 도입된 훅입니다. 외부(External) 상태 저장소를 React 컴포넌트와 동기화 하기 위해서 사용됩니다. React 18에서 Concurrent Mode가 도입되며 외부 상태 관리 패키지를 사용할 때 [티어링](https://github.com/reactwg/react-18/discussions/69) 이슈가 발생할 수 있게 되어 이를 보완하고자 만들어진 방식입니다. 간단하게 기능을 설명하자면 이 훅을 통해 컴포넌트가 외부 상태의 변화를 구독하고, 해당 상태가 변경될 때마다 컴포넌트를 리렌더링 해줍니다.

> ﹗티어링 이슈: 동시성을 지원하는 다양한 분야에서 발생할 수 있는 문제. 리액트에서는 18버전부터 UI의 동시적 업데이트를 지원하게 됐는데, 이 과정에서 동일한 데이터 소스에 따라 렌더링이 이뤄지는 두 가지 UI가 있을 때, 기존의 동기적인(async) 업데이트일 경우 순서대로 업데이트가 보장되기 때문에 문제가 없었습니다. 하지만 동시(concurrent)모드가 적용됨에 따라 각 스레드에서 각자가 두개의 UI를 업데이트하다가 하나가 실패할 경우가 발생하거나, 중간에 사용자 입력등으로 인해 업데이트가 중단되는 동작이 생기게 되고, 그런 케이스에 대해 같은 데이터 소스의 UI의 상태가 서로 다른 것을 티어링이 나뉜다고 표현합니다.

## 기본 사용법

```javascript
import { useSyncExternalStore } from 'react';

function useCustomHook(store) {
  const state = useSyncExternalStore(
    store.subscribe,
    store.getSnapshot,
    store.getServerSnapshot // 이 인자는 서버 사이드 렌더링에만 필요합니다.
  );

  return state;
}
```

* 세 가지 인자를 받아서 사용합니다
    
    * `subscribe`: 외부 저장소의 변화를 구독하는 함수입니다. 이 함수는 리스너 함수를 인자로 받아 상태가 변할 때마다 리스너를 호출합니다.
        
    * `getSnapshot`: 현재 상태의 스냅샷을 반환하는 함수입니다. 컴포넌트가 상태를 필요로 할 때 호출됩니다.
        
    * `getServerSnapshot` (옵션): 서버 사이드 렌더링 환경에서 사용되는 스냅샷을 반환하는 함수입니다. 클라이언트 사이드와 다른 스냅샷을 제공해줄 수 있습니다.
        

## 어디에 사용 하면 되나?

* 주로 리액트 어플리케이션 내에서 Redux, Zustand, MobX 등과 같은 외부 상태 관리 라이브러리와 함께 사용됩니다. 특히 컴포넌트가 외부 상태의 특정 부분만을 구독하고 싶을 때 유용합니다.
    

## 외부 상태 관리 라이브러리 구현 예시 - 주의사항 첨가

* 잘못 구현하면 불필요한 리렌더링을 일으킬 수 있습니다. zustand와 연동한 예제를 구현하겠습니다.
    

### 잘못된 케이스

```javascript
// store.js
import create from 'zustand';

const useStore = create(set => ({
    count: 0,
    increase: () => set(state => ({ count: state.count + 1})),
}));

// Component.js
import React from 'react';
import { useSyncExternalStore } from 'react';
import useStore from './store';

const subscribe = (callback) => useStore.subscribe(callback);
const getSnapshot = () => useStore.getState();

export default function Component() {
    const { count, increase } = useSyncExternalStore(subscribe, getSnapshot);
    
    return (
        <div>
            <p>카운트: {count}</p>
            <button onClick={increase}>증가</button>
        </div>
    );
}
```

* 위 예제는 `useSyncExternalStore`를 사용하여 zustand 스토어의 모든 상태 변화를 구독중입니다. 만약 count 외 다른 상태가 변경되도 리렌더링이 일어날 위험성이 있습니다.
    

### 올바른 케이스

```javascript
// store.js
import create from 'zustand';
// zustand에서 지원하는 미들웨어를 사용하여 선택적 구독을 할 수 있도록 구현
import { subscribeWithSelector } from 'zustand/middleware'

const useStore = create(subscribeWithSelector(set => ({
  count: 0,
  increase: () => set(state => ({ count: state.count + 1 })),
})));

// Component.js
import React from 'react';
import { useSyncExternalStore } from 'react';
import useStore from './store';

const subscribe = (callback) => useStore.subscribe(callback, state => state.count);
const getSnapshot = () => useStore.getState().count;

export default function Component() {
  const count = useSyncExternalStore(subscribe, getSnapshot);

  return (
    <div>
      <p>카운트: {count}</p>
      <button onClick={() => useStore.getState().increase()}>증가</button>
    </div>
  );
}
```

* 위 예제에서는 `subscribe` 함수를 통해 `count`상태만 구독했습니다. 이렇게 하면 컴포넌트의 다른 상태가 업데이트 되더라도 불필요한 리렌더링이 발생하지 않습니다.
    
* 그리고 `increase` 함수를 호출 할 때도 zustand가 제공하는 API를 직접 사용하여 상태 변경을 트리거합니다.
    

## useSyncExternalStore가 제공하는 이점

### 외부 상태 라이브러리 사용 시 이점

* 괜히 단계만 하나 추가되는게 아닐까 싶지만, 나름대로의 이점들이 있습니다.
    
* [리액트 동시성 모드](https://medium.com/swlh/what-is-react-concurrent-mode-46989b5f15da)(Concurrent Mode)와의 통합을 지원해줍니다. 이를 통해 애플리케이션의 반응성과 사용자 경험을 개선할 수 있습니다.
    
* 상태 관리 코드의 일관성 유지를 돕습니다. 모든 외부 상태 접근을 `useSyncExternalStore`를 통해 처리함으로써 코드의 패턴이 일관되게 유지됩니다. 즉, 상태 관리 라이브러리 교체 시 좀 더 결합도가 낮아짐으로써 변경에 유연해집니다.
    

### 외부 상태 라이브러리 없이 간단하게 사용 가능

* 외부 상태 라이브러리가 굳이 필요하지 않은 규모의 프로젝트 일 경우 `useSyncExternalStore`를 통해 간단한 상태를 제공해줄 수 있습니다.
    
* 상태 관리 로직을 완전히 제어할 수 있어서 나름의 이점이 있습니다.
    
* 다만 상태 관리 라이브러리들이 제공하는 다양한 기능(상태 업데이트, 구독 관리, 미들웨어, 디버깅 등)이 필요하거나, 다량의 상태 업데이트가 필요할 경우 별도 성능 최적화 작업 등 라이브러리들의 기능이 필요할 수 있습니다. 프로젝트의 상황에 따라 결정하는 것이 좋습니다.
    

## 마치며

* 간단하게 `useSyncExternalStore` 훅의 기본 사용법, 왜 사용하는지, 구현 예시, 적용했을 때의 장점 등에 대해 알아 봤습니다.
    
* 이 훅이 왜 생겨났는지 리액트 팀에서 언급한 내용을 간단하게 정리하자면, React 애플리케이션 내에서 외부 데이터 소스를 보다 원활하고 예측 가능하며 고성능으로 통합할 수 있도록 해주기 위해서 입니다. 좀 더 알고 싶으시면 아래 링크들을 참고해주세요.
    
    * [동시 렌더링 때문에 적용된 useSyncExternalStore 깃허브 링크](https://github.com/reactwg/react-18/discussions/86)
        
    * [동시 렌더링 중 티어링 이슈 해결을 위한 것이라는 블로그 글](https://blog.saeloun.com/2021/12/30/react-18-useSyncExternalStore-api/)