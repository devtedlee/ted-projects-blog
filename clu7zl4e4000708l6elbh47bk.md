---
title: "요즘 리덕스 - 리덕스 툴킷에 대해 알아보자"
datePublished: Tue Mar 26 2024 06:19:11 GMT+0000 (Coordinated Universal Time)
cuid: clu7zl4e4000708l6elbh47bk
slug: about-redux-toolkit
cover: https://cdn.hashnode.com/res/hashnode/image/upload/v1711433904341/2d67fd74-53c1-4f79-952b-da994cf87b3c.png
tags: redux, redux-saga, redux-toolkit

---

리덕스 툴킷은 2019년 리덕스 공식 팀에 의해 개발 및 출시 되었습니다.  
[공식 문서](https://ko.redux.js.org/redux-toolkit/overview/)에 의하면 리덕스 툴킷은 효율적인 Redux 개발을 위한 리덕스 공식 팀의 견해를 반영한 standalone으로 작동하는 도구 모음입니다. 약어로 RTK로 부르기도 합니다.

Redux는 매우 유연하고 강력한 상태 관리 라이브러리이지만 설정과 사용법이 복잡하고 특유의 보일러플레이트 코드가 많다는 단점이 있었습니다. 이러한 문제점을 해결하고, Redux 개발 경험을 좀 더 쉽고 효율적으로 만들기 위해 탄생했습니다. 리덕스 팀은 이 기술을 Redux 사용을 위한 새로운 표준이 되길 바라며 만들었습니다.

## RTK의 장점

* **보일러플레이트 코드 감소**: 액션 생성자, 리듀서, 상태 초기화 같은 작업을 간소화합니다.
    
* **설정 간소화**: `configureStore` 함수를 통해 스토어 설정을 간단하게 해줍니다. 미들웨어 설정, devTool 확장 통합등을 자동으로 처리합니다.
    
* **간편해진 불변성 관리**: 내부에서 `immer` 라이브러리를 사용하여 상태 업데이트 시 불변성을 유지하는 과정이 간소화됩니다. 불변성 관리 로직으로 인한 코드 결합이 완화됩니다.
    
* **비동기 로직의 통합**: `createAsyncThunk` 그리고 RTK Query가 비동기 로직을 쉽게 관리할 수 있는 기능을 제공합니다. 기존의 Saga의 역할을 대신해줍니다.
    

## 기존 Redux + Saga 예시

* 사용자 정보를 비동기로 불러와 업데이트하는 동작입니다.
    

```javascript
// actions.js
export const loadUserProfileRequest = () => ({ type: 'LOAD_USER_PROFILE_REQUEST'});
export const loadUserProfileSuccess = () => ({ type: 'LOAD_USER_PROFILE_SUCCESS'});
export const loadUserProfileFailure = () => ({ type: 'LOAD_USER_PROFILE_FAILURE'});

// reducer.js
import produce from 'immer';

const initialState = { profile: null, loadting: false, error: null };

const userProfileReducer = (state = initialState, action) =>
    produce(state, draft => {
        switch (action.type) {
            case 'LOAD_USER_PROFILE_REQUEST':
                draft.loading = true;
                draft.error = null;
                break;
            case 'LOAD_USER_PROFILE_SUCCESS':
                draft.loading = false;
                draft.profile = action.payload;
                break;
            case 'LOAD_USER_PROFILE_FAILURE':
                draft.loading = false;
                draft.error = action.payload;
                break;
            default:
                break;
        }
    });

// sagas.js
import { call, put, takeLatest } frm 'redux-sage/effects';
import { locadUserProfileSuccess, loadUserProfileFailure } from './actions';
import { fetchUserProfile } from '../api' // api sudo 코드

function* loadUserProfileSaga() {
    try {
        const profile = yield call(fetchUserProfile);
        yield put(loadUserProfileSuccess(profile));
    } catch (error) {
        yield put(loadUserProfileFailure(error));
    }
}

export function* watchUserProfileSaga() {
    yield takeLatest('LOAD_USER_PROFILE_REQUEST', loadUserProfileSaga);
}
```

## RTK 적용 예시

* Redux + Saga 예시와 로직은 같습니다.
    

```javascript
// features/userProfileSlice.js
import { createSlice, createAsyncThunk } from '@reduxjs/toolkit';
import { fetchUserProfile } from '../api' // sudo 코드 api 호출

export const loadUserProfile = createAsyncThunk(
    'userProfile/loadUserProfile',
    async (_, { rejectWithValue }) => {
        try {
            const profile = await fetchUserProfile();
            return profile;
        } catch (error) {
            return rejectWithValue(error.toString());
        }
    }
);

const userProfileSlice = createSlice({
    name: 'userProfile',
    initialState: {
        profile: null,
        loading: false,
        error: null,
    },
    reducers: [],
    extraReducers: (builder) => {
        builder
            .addCase(loadUserProfile.pending, (state) => {
                state.loading = true;
            })
            .addCase(loadUserProfile.fullfiled, (state, action) => {
                state.loading = false;
                state.profile = action.payload;
            })
            .addCase(loadUserProfile.pending, (state, action) => {
                state.loading = false;
                state.error = action.payload;
            })
    }
});

export default userProfileSlice.reducer;
```

## 코드 비교

* 리덕스 사가 + Immer 활용한 코드의 경우 코드가 조금 더 길고, 설정이 복잡한 경향이 있습니다.
    
* 반면, RTK의 경우 Immer 코드가 내장됨으로 인해 사라졌고, 각 설정들도 파일을 넘나들지 않고 한곳에 위치시킬 수 있습니다.
    

## RTK의 단점 및 고려사항

* Redux에 익숙한 경우 학습을 별도로 해야 합니다.
    
* 과도한 추상화: 기능들이 Redux에 비해 숨겨져 있다보니 추상화된 부분으로 인해 문제를 진단하고 디버깅 하는데 어려움이 있을 수 있습니다.
    
* 리덕스 사가를 사용하는 것보다는 직접적인 제어 흐름 관리가 미흡할 수 있습니다.
    

## 마치며

리덕스 툴킷에 대한 소개, 장점과 단점, 리덕스 saga와 RTK의 예시 코드 비교에 대해 간단하게 알아봤습니다. 조사하면 할수록 리덕스를 사용하는 것보다 새로운 표준으로 팀에서도 밀고 있고 이미 5년차가 된 리덕스 툴킷을 사용하는게 좋겠다는 생각이 많이 드는 시간이었습니다.