---
title: "React useRef를 사용하는 이유"
datePublished: Mon Mar 18 2024 03:58:32 GMT+0000 (Coordinated Universal Time)
cuid: cltwf1fz4000008la2i0ba0a7
slug: why-we-use-react-useref
cover: https://cdn.hashnode.com/res/hashnode/image/upload/v1710734241797/a40000fc-404a-453b-aa4f-ece29d0bddd8.png
tags: reactjs, useref

---

자꾸만 익숙했던 방식대로 `querySelector`를 사용하려고 하는 스스로를 설득하기 위해 리액트 `useRef` 훅의 내용을 정리합니다.

간단하게 먼저 결론부터 언급하자면, `useRef`를 사용하는 주된 이유는 React의 선언적이고 반응형인 특성과 잘 맞기 때문입니다.

## 선언적 프로그래밍에 잘 맞는다

* React는 선언적 프로그래밍 패러다임을 따릅니다. UI의 현재 상태를 선언하고, 데이터가 변경될 때마다 React가 UI를 자동으로 업데이트 해줍니다. 이 때 `useRef`를 사용하면 React의 랜더링 사이클과 일관성을 유지하면서 DOM 요소에 접근할 수 있습니다.
    
* `document.querySelector`등 Element 요소를 반환하는 방식을 사용하면, 명령적인 코드가 되기 때문에 React의 선언적 특성과 어울리지 않습니다.
    

## 렌더링 사이클과 통합된다

* `useRef`를 사용하면 React의 렌더링 사이클에 통합됩니다. 그래서 컴포넌트가 렌더링될 때 자동으로 참조가 업데이트 됩니다. 결과적으로 DOM요소가 변경될 때마다 안정적으로 참조를 유지할 수 있게 됩니다. 리액트 렌더링 사이클인 마운팅 -&gt; 업데이트 -&gt; 언마운팅의 과정이 있는데, `useRef`를 통해 참조된 객체는 이 과정과 라이프 사이클이 동기화 됩니다.
    
* `document.querySelector`를 사용하면 리액트 라이프사이클과 동기화가 어긋날 수 있어 문제가 생길 수 있습니다. 만약 라이프사이클을 통한 DOM 업데이트 시점과 실제 React 렌더링 라이프사이클을 통해 업데이트 되는 시점이 다를 경우 dom을 찾을 수 없거나, 제대로 업데이트 되지 않은 DOM 요소를 찾을 수도 있기 때문입니다. 이런 경우 컴포넌트가 업데이트(=리렌더링)될 때마다 수동으로 DOM 요소를 다시 찾아야 할 필요가 생깁니다.
    

## 메모리 누수 방지

* 렌더링 사이클 통합과 이어지는 장점으로, React 컴포넌트가 언마운트될 때, `useRef`를 통해 생성된 참조도 자동으로 정리가 되기 때문에 메모리 누수에 대해 안전합니다.
    
* `document.querySelector`로 가져온 참조의 경우 더이상 활용하지 않게된 Element 를 변수에 가지고 있거나 계속 참조하여 가지고 있음으로 인해 메모리 낭비가 커지기 쉽습니다.
    

## 리액트 16.8 부터 적용된 함수 컴포넌트와의 호환

* `useRef`는 React의 훅 중 하나로, 함수 컴포넌트 내에서 상태나 DOM 요소에 대한 참조를 유지하는데 사용됩니다. 클래스 컴포넌트에서는 `this.refs`를 통해서 사용했던 기능이지만, 16.8버전부터 생겨난 함수 컴포넌트에서는 `useRef`가 동일한 역할을 해줍니다.
    

## 예시를 통해 차이를 알아보기

* `document.querySelector` 사용 케이스
    
    ```javascript
    import React, { useEffect } from 'react';
    
    function AutofocusInput() {
      useEffect(() => {
        // 컴포넌트가 마운트된 후 input 요소를 선택하여 포커스를 줌
        const inputElement = document.querySelector('#myInput');
        if (inputElement) {
          inputElement.focus();
        }
      }, []);
    
      return (
        <input id="myInput" type="text" />
      );
    }
    ```
    
    이 예시에서 컴포넌트가 마운트 된 후 `useEffect` 내부에서 `document.querySelector`를 사용해 DOM요소를 찾고 포커스를 주고 있습니다.  
    \- 이 방법은 작동하지만, React의 선언전 특성과 잘 어울리지 않습니다.  
    \- 또한 ID를 기반으로 요소를 찾기 때문에 ID가 중복되거나 변경될 경우 예상 못한 동작이 발생할 수 있습니다.  
    \- 그리고 리액트 렌더링 사이클과 동기화 되지 않기 때문에 만약 리렌더링이 발생되어 요소에 변화가 생기면 참조가 유효하지 않게 될 가능성도 있습니다.
    
* `useRef` 사용 케이스
    
    ```javascript
    import React, { useEffect } from 'react';
    
    function AutofocusInput() {
      useEffect(() => {
        // 컴포넌트가 마운트된 후 input 요소를 선택하여 포커스를 줌
        const inputElement = document.querySelector('#myInput');
        if (inputElement) {
          inputElement.focus();
        }
      }, []);
    
      return (
        <input id="myInput" type="text" />
      );
    }
    ```
    
    이 예시에서 `useRef`를 사용하면 React의 선언적 패러다임을 유지하면서도 렌더링 사이클과의 통합을 보장합니다.  
    또한 컴포넌트의 렌더링 사이클과 자연스럽게 통합되어있기 때문에 컴포넌트의 상태 변화나 리렌더링에 영향을 받지 않고 일관된 참조를 유지할 수 있습니다.
    

## 마치며

왜 `useRef`를 사용해야 하는지 정리하며, 리액트의 프로그래밍 철학, 랜더링 동작원리, 메모리까지 연관 되었으므로 꼭 사용해야 된다는 점을 돌이켜 볼 수 있었습니다. 이 글을 보신 분들에게도 지식을 상기하거나 새롭게 아는데 도움이 되었으면 좋겠습니다.