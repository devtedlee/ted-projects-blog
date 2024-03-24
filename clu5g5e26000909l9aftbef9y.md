---
title: "리덕스에 대해 알아보자"
datePublished: Sun Mar 24 2024 11:39:32 GMT+0000 (Coordinated Universal Time)
cuid: clu5g5e26000909l9aftbef9y
slug: about-redux
cover: https://cdn.hashnode.com/res/hashnode/image/upload/v1711280349158/28a4cfda-126e-4ab7-915e-388dd3737df6.webp
tags: reactjs, redux

---

회사 제품에 리덕스를 사용중인데, 이 리덕스가 무엇인지에 대해 너무 모르고 있고, 어떤 기술인지 정리하면 좋겠다는 생각이 들어 글을 작성하게 됐습니다.

리덕스 공식홈페이지에 따르면 Redux는 "자바스크립트 앱을 위한 예측 가능한 상태 컨테이너"라고 합니다. 이름이 리액트 처럼 Re로 시작하다보니 리액트와 결합되어 연상되긴 하지만 알고보면 리액트에 종속되어 있는 라이브러리가 아닙니다. 물론 주로 React와 함께 사용되기는 합니다.

## 등장 배경

리덕스는 크게 두 가지 문제를 해결하기 위해 등장 했습니다.

1. 복잡한 어플리케이션에서 상태 관리의 복잡성을 줄이기
    
2. 다양한 상태 변화를 예측 가능하게 관리
    

이전에는 각 컴포넌트나 모듈에서 상태를 독립적으로 관리했기 때문에, 어플리케이션의 규모가 커질수록 상태를 추적하고 관리하는 것이 점점 어려워졌기 때문이죠.

이런 문제를 해결하기 위해 2015년에 댄 아브라모드(Dan Abramov), 앤드류 클라크(Andrew Clark)가 Redux를 개발했습니다. 리덕스는 React 유럽 컨퍼런스에서 처음으로 소개되었고, 단일 상태 트리를 사용하는 것의 간결함과 예측 가능성 덕에 빠르게 인기를 얻었습니다.

## Flux 패턴이란

리덕스는 플럭스(Flux) 아키텍쳐 패턴의 개념을 기반으로 하고 있습니다.  
액션 -&gt; 디스패처 -&gt; 스토어 -&gt; 뷰의 단방향 데이터 흐름을 가짐으로써 데이터의 저장과 변경을 정해진 구조로만 변경할 수 있도록 구조화 한 것입니다.

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1711277565366/59c82e90-820c-4c24-a585-457c982e515d.png align="center")

액션을 통해서만 상태가 업데이트 될 수 있고, 모든 데이터는 store에 담겨있는 그런 형태입니다. 이런 방식을 통해 컴포넌트 간의 결합도를 낮출 수 있었습니다.

**예제) 비동기 데이터 로딩. 리덕스 구현 Redux Thunk 미들웨어 활용**

```javascript
// actions.js
export const fetchDataSuccess = data => ({
  type: 'FETCH_DATA_SUCCESS',
  payload: data
});

export const fetchData = () => {
  return dispatch => {
    fetch('https://api.example.com/data')
      .then(response => response.json())
      .then(data => dispatch(fetchDataSuccess(data)))
      .catch(error => console.error('Fetching data failed', error));
  };
};

// reducers.js
const initialState = {
  data: [],
};

const dataReducer = (state = initialState, action) => {
  switch (action.type) {
    case 'FETCH_DATA_SUCCESS':
      return {
        ...state,
        data: action.payload,
      };
    default:
      return state;
  }
};

export default dataReducer;
```

## 리덕스 사가의 등장

점차 어플리케이션의 규모가 커지다보면 리덕스를 사용하는 경우 복잡한 비동기 로직을 관리하는게 어려워집니다. 이 과정에서 비동기 로직들을 처리하기 위한 사이드 이펙트 관리를 위한 라이브러리인 Redux Saga가 만들어지게 됐습니다. 리덕스 사가는 2015년경 Yassin Elouafi에 의해 개발되었습니다.

기본적으로 제너레이터 함수를 사용해서 비동기 작업을 더 선언적이고 쉽게 관리할 수 있게 해줍니다. 복잡한 비동기 흐름을 동기 코드처럼 작성할 수 있게 해줍니다.

**예제) 비동기 데이터 로딩. 위의 리덕스 예제를 개선 적용**

```javascript
// actions.js
export const fetchDataSuccess = data => ({
  type: 'FETCH_DATA_SUCCESS',
  payload: data
});

export const fetchDataRequest = () => ({
  type: 'FETCH_DATA_REQUEST',
});

// sagas.js
import { call, put, takeEvery } from 'redux-saga/effects';
// call, put, takeEvery는 사가 이펙트라고 명명됩니다.
// call은 함수를 호출하고 그 결과를 기다리는 역할을 합니다.
// put은 스토어에 액션을 디스패치하여 상태를 업데이트하고자 할 때 사용됩니다.
// takeEvery는 지정한 액션이 디스패치될 때마다 사가를 실행하도록 합니다.

function* fetchDataSaga() {
  try {
    const data = yield call(() => fetch('https://api.example.com/data').then(res => res.json()));
    yield put(fetchDataSuccess(data));
  } catch (error) {
    console.error('Fetching data failed', error);
  }
}

function* watchFetchDataSaga() {
  yield takeEvery('FETCH_DATA_REQUEST', fetchDataSaga);
}

export default watchFetchDataSaga;

// reducers.js는 위의 Redux Thunk 예제와 동일
```

* Redux Thunk를 사용하는 경우, 비동기 로직을 액션 크리에이터 내에서 처리해야합니다.
    
* Redux Saga를 사용하면 사이드 이펙트를 분리된 '사가'에서 관리할 수 있습니다. 이는 비동기 로직을 더 선언적으로 표현할 수 있게 해줍니다.
    

## 리덕스의 현재

여전히 많은 회사나 프로젝트에서 사용되고 있지만, 새로 시작하는 프로젝트에서는 채택되고 있지 않는 경향이 있습니다. React의 Context API나 Hooks등의 기능이 강화되면서 상태관리를 위해 리덕스를 사용하지 않게 된 것입니다. 만약 어플리케이션이 복잡해져서 상태 관리가 필요하다고 해도 Zustand, Jotai 등 보일러플레이트(=구현을 위해 필요한 추가 코드) 코드가 적은 대안 라이브러리들이 생기기도 했습니다.

**예시) Context API나 Hooks를 통한 상태 관리 비동기 처리**

* 상태 관리를 위한 Contet와 Reducer 설정
    

```javascript
// DataContext.js
import React, { createContext, useContext, useReducer, useEffect } from 'react';

// 초기 상태 정의
const initialState = {
  loading: false,
  data: null,
  error: null,
};

// 리듀서 함수
function dataReducer(state, action) {
  switch (action.type) {
    case 'FETCH_START':
      return { ...state, loading: true, error: null };
    case 'FETCH_SUCCESS':
      return { ...state, loading: false, data: action.payload, error: null };
    case 'FETCH_ERROR':
      return { ...state, loading: false, error: action.payload };
    default:
      return state;
  }
}

// Context 생성
const DataContext = createContext();

// 커스텀 훅: 컴포넌트에서 DataContext 사용하기 쉽게 만들어 줍니다.
export function useData() {
  return useContext(DataContext);
}

// Provider 컴포넌트
export function DataProvider({ children }) {
  const [state, dispatch] = useReducer(dataReducer, initialState);

  // 비동기 데이터 가져오기 함수
  const fetchData = async () => {
    dispatch({ type: 'FETCH_START' });
    try {
      const response = await fetch('https://example.com/data');
      const data = await response.json();
      dispatch({ type: 'FETCH_SUCCESS', payload: data });
    } catch (error) {
      dispatch({ type: 'FETCH_ERROR', payload: error });
    }
  };

  // value 객체에 상태, 액션 디스패치 함수, 데이터 가져오기 함수 포함
  const value = { state, dispatch, fetchData };

  return <DataContext.Provider value={value}>{children}</DataContext.Provider>;
}
```

* 위에서 생성한 Context 사용하는 컴포넌트 코드
    

```javascript
// App.js
import React, { useEffect } from 'react';
import { DataProvider, useData } from './DataContext';

function DataComponent() {
  const { state, fetchData } = useData();

  useEffect(() => {
    fetchData(); // 컴포넌트 마운트 시 데이터 가져오기
  }, [fetchData]);

  if (state.loading) return <p>Loading...</p>;
  if (state.error) return <p>Error: {state.error.message}</p>;
  return <div>{state.data && <p>{JSON.stringify(state.data)}</p>}</div>;
}

function App() {
  return (
    <DataProvider>
      <DataComponent />
    </DataProvider>
  );
}

export default App;
```

* `useEffect`를 사용해 컴포넌트가 마운트될 때 데이터를 비동기적으로 가져오는 로직을 구현했습니다.
    

## 마치며

간단하게 리덕스가 무엇인지, 왜 생겨났는지, 리덕스의 핵심인 Flux 패턴에 대해 알아보기도 하고, 리덕스 사가에 대해서도 알아보고, 마지막으로 리덕스가 요즘 왜 자주 선택되지 않는지 이유도 정리해봤습니다. 다음에는 리덕스를 더 쉽게 사용할 수 있게 개발된 Redux Toolkit에 대해 알아보겠습니다.