---
title: "Redux-Saga 동시성 액션 문제 해결하기"
datePublished: Thu Mar 14 2024 04:38:39 GMT+0000 (Coordinated Universal Time)
cuid: cltqqpm75000008jm8jf63ley
slug: redux-saga-concurrency-troubleshooting
cover: https://cdn.hashnode.com/res/hashnode/image/stock/unsplash/HgoKvtKpyHA/upload/9d9dc4a1b3e384a7422c0eecf024f225.jpeg
tags: concurrency, reactjs, redux-saga

---

리액트로 페이지를 운영 하다 보니 동시 입력이 불가능한 시간에 동시에 API 요청이 되있는 오류 케이스를 발견하게 됐습니다. 디버깅을 통해 문제를 찾던 중 리덕스 사가의 `takeLatest` 헬퍼 함수가 원인이라는 것을 알게 됐습니다.

리덕스 사가를 단순히 레퍼런스를 보고 사용만 했지 `takeLatest`, `takeLeading`, `throttle` 등 헬퍼 함수들에 대해 너무 모르고 사용했다는 점을 반성하고 정리해보려고 합니다.

Redux-Saga의 `takeLatest`는 거의 동시에 발생하는 여러 액션 중 마지막 액션만 처리해주는 헬퍼 함수입니다.

하지만 두 개의 액션이 거의 동시에 발생하면 두 번째 액션이 첫 번째 액션을 취소하기도 전에 둘 다 실행될 수 있는 위험성이 있습니다. 이런 케이스는 드물긴 하지만, 액션이 비동기 로직을 처리하고 상태를 업데이트하는 데 시간이 조금이라도 걸릴 때 나타날 수 있습니다.

**동시성 문제 - 코드 예시**

```javascript
import { call, put, takeLatest } from 'redux-saga/effects';

function* insertSaga(action) {
    const result = yield call(api.insert, action.data);
    yield put(insertSuccess(result);
}

function* watchInsert() {
  yield takeLatest('insert_request', insertSaga);
}
```

이런 상황은 Redux-Saga의 내부 동작 방식과 JS 이벤트 루프의 특성 때문에 생기게 됩니다. 이에 대해 한번 좀 더 자세히 살펴보겠습니다.

## Redux-Saga의 내부 동작 방식

Redux-Saga는 Generator 함수를 사용해 비동기 작업을 관리하는 미들웨어입니다. Saga는 이벤트(액션)를 감시하다가 특정 액션이 디스패치되면, 그에 대응하는 작업(사이드 이펙트)을 실행하는 방식으로 동작합니다. `takeLatest`는 이러한 사이드 이펙트 중 하나로, 동일한 액션이 여러 번 발생할 때 가장 마지막에 발생한 액션에 대해서만 응답을 처리하고 이전에 실행된 작업들을 자동으로 취소합니다.

내부적으로 `takeLatest`는 두 가지 중요한 Saga 이펙트를 사용합니다.  
1\. `take` 이펙트: 특정 액션 타입을 기다리고, 그 액션이 발생하면 제너레이터를 다시 실행시킵니다.  
2\. `fork` 이펙트: 비동기 작업을 수행하는 새로운 task를 생성합니다. `takeLatest`에서는 이전에 시작된 task가 있으면 자동으로 취소(`cancel`이펙트 사용)하고, 새로운 액션에 대해 `fork`를 다시 호출하여 새로운 비동기 작업을 시작합니다.

`takeLatest`는 내부적으로 `fork`된 태스크에 대한 참조를 유지하고 있어서, 새로운 액션이 발생하면 이전 task를 취소하고 새로운 task를 시작하는 방식으로 작동합니다.

## JS 이벤트 루프의 특성

JS는 단일 스레드 기반 언어로, 이벤트 루프를 사용해 비동기 작업을 처리합니다. 이벤트 루프는 Call Stack, Task Queue, Microtask Queue와 같은 여러 구성 요소로 이루어져 있습니다. JS의 비동기 작업은 Call Stack이 비어있을 때 Task Queue 나 Microtask Queue에서 순차적으로 실행 됩니다.

## 문제 상황과의 연관성

Redux-Saga의 `takeLatest`가 처리하는 동안 JS 이벤트 루프도 동시에 동작하고 잇습니다. 그래서 거의 동시에 발생한 여러 액션을 `takeLatest`가 감지하고 처리하려 할 때, JS의 비동기 처리 방식 때문에 예상치 못한 타이밍 문제가 발생할 수 있습니다. 특히, 이전 작업이 취소되고 새 작업이 시작되는 과정에서 비동기 작업이 얽히면, 두 작업이 거의 동시에 실행되어 상태 업데이트가 겹치는 현상이 발생할 수 있습니다. 앞으로는 이런 형태의 동시 입력의 가능성이 있는 액션은 `takeLatest`를 사용하지 않거나 추가적인 대응이 필요하다는 점을 알 수 있었습니다.

## 4가지 대응 방안

1. **throttle 사용 고려**: 일정 시간 내에 발생하는 액션 중 최초 액션만 인정하고 다른 액션은 무시하는 방식입니다.
    
    ```javascript
    import { call, put, throttle } from 'redux-saga/effects';
    
    function* insertSaga(action) {
        const result = yield call(api.insert, action.data);
        yield put(insertSuccess(result));
    }
    
    function* watchInsert() {
      yield throttle(500, 'insert_request', insertSaga);
    }
    ```
    
2. **서버 측 중복 요청 처리**: 서버 측에서도 유니크 토큰 등을 활용하여 중복 요청 여부를 판별하여 동시 입력을 예방할 수 있습니다.
    
3. **로딩 상태 관리**: 사용자 인터페이스에서 요청이 처리되는 동안 '로딩 중' 상태를 사용자에게 알리는 형태의 문제 해결도 가능합니다.
    
    ```javascript
    // 액션 생성 함수
    export const setLoading = (loading) => ({ type: SET_LOADING, loading });
    
    // 사가
    function* insertSaga(action) {
        // 로딩 상태 시작
        yield put(setLoading(true));
        const result = yield call(api.insert, action.data);
        yield put(insertSuccess(result.data));
        // 로딩 상태 종료
        yield put(setLoading(false));
    }
    ```
    
4. **디바운스(debounce) 사용 고려**: throttle 방식과 유사하지만 다른 구현 방법입니다. 지정된 시간 동안 추가 액션이 발생하지 않을 때까지 기다렸다가 마지막 액션을 실행하는 방식입니다.
    
    ```javascript
    import { call, put, debounce } from 'redux-saga/effects';
    
    function* insertSaga(action) {
        const result = yield call(api.insert, action.data);
        yield put(insertSuccess(result));
    }
    
    function* watchInsert() {
      yield debounce(500, 'insert_request', insertSaga);
    }
    ```
    

## 마치며

`takeLatest` 를 활용하다 마주한 동시성 이슈를 통해 이런 종류의 문제 가능성이 있는 경우 어떻게 대응하면 좋은지 알아봤습니다. 저는 이번 케이스의 경우 리덕스 사가에서 제공하는 `throttle` 헬퍼 함수를 활용하여 문제를 해결했습니다. 프론트엔드에서 문제를 해결하고 싶었기 때문입니다.  
어떤 대응 방안을 선택할지는 각자가 개발하는 소프트웨어의 특성과 요구사항에 맞게 잘 적용하면 문제를 예방 혹은 해결하실 수 있을겁니다.