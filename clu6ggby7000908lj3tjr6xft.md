---
title: "리덕스에서 immer는 왜 사용될까?"
datePublished: Mon Mar 25 2024 04:35:48 GMT+0000 (Coordinated Universal Time)
cuid: clu6ggby7000908lj3tjr6xft
slug: why-use-redux-immer
cover: https://cdn.hashnode.com/res/hashnode/image/upload/v1711341295112/5e72c0e8-d4cc-4c47-a596-b949fdb19e3a.webp
tags: immutable, redux

---

불변성(Immutability) 이란 변하지 않는 것을 말합니다. 그리고 리덕스에서는 상태를 보관할 때 불변성을 부여합니다. 이런 이유로 인해 기존에 존재하는 상태 값을 직접 수정할 수 없습니다. 하지만 action 을 통해 상태를 업데이트 할 때 기존의 불변성도 유지가 되면서, 상태 값을 업데이트 해줘야 합니다.

이 때 많이 사용되는 기술이 `immer`입니다. `immer`는 불변성을 유지하면서도 상태 값을 변경하는데 도움을 줍니다.  
어떤 도움을 주는지는 코드를 보며 이해하면 좋습니다. 일단 depth가 꽤 깊은 상태 데이터를 불변성을 유지하며 수정하는 코드를 먼저 살펴보겠습니다.

## **예시) 직접 불변성 관리하기**

```javascript
const state = {
  user: {
    name: "Tedpool",
    address: {
      city: "Seoul",
      country: "Korea"
    }
  }
};

// 사용자의 주소 정보를 업데이트하는 함수
function updateUserAddress(state, newCity) {
  return {
    ...state,
    user: {
      ...state.user,
      address: {
        ...state.user.address,
        city: newCity
      }
    }
  };
}

// 상태 업데이트 적용
const updatedState = updateUserAddress(state, "Busan");
```

* updateUserAddress로 책임을 분리하고, 재사용성을 높였습니다.
    

## 직접 불변성 관리의 이점

* **예측 가능한 상태 변화**: 자바스크립트 범용 문법만을 사용하기 때문에 예측 가능하고 일관된 방식으로 상태 변경이 이뤄집니다.
    
* **코드의 결합도 완화**: 책임을 명확하게 분리할 수 있고, 재사용 가능한 코드를 작성하기 좋습니다.
    
* **테스트 용이성**: 결합도가 낮아지다보니 함수 단위로 테스트하기가 더 용이합니다.
    

## **예시) immer로 불변성 관리하기**

```javascript
import produce from "immer";

const state = {
  user: {
    name: "Tedpool",
    address: {
      city: "Seoul",
      country: "Korea"
    }
  }
};

// 상태 업데이트 적용
const updatedState = produce(state, draft => {
  draft.user.address.city = "Busan";
});
```

* `produce`와 함께 제공되는 `draft`상태를 직접 수정하는 것처럼 코드를 작성할 수 있습니다. 이 과정에서 실제 상태는 불변성이 유지되며 업데이트 됩니다.
    

## immer로 불변성 관리의 이점

* **코드의 간결성**: 상태의 depth가 깊어질 경우 스프레드 연산자(`...`), `concat`, `map`, `filter`의 구현을 확연하게 줄여줍니다.
    
* **실수로 인한 오류 방지**: 실수로 불변성 처리를 깨트리는 코드를 작성할 가능성을 차단해줍니다.
    
* **성능 최적화**: immer 내부의 동작으로 인해 필요한 부분만 최소한으로 복사하고 업데이트 하기 때문에 성능 최적화가 이루어집니다.
    

## immer의 성능 최적화는 어떻게 이뤄지나?

Immer는 성능 최적화를 위해 프록시(Proxy) 기반의 변경 감지 방식을 활용합니다. 이 방식을 통해 Immer는 애플리케이션 상태 내에서 실제로 변경이 필요한 부분만을 세밀하게 관찰하고, 오직 변경이 필요한 객체에 대해서만 새로운 객체를 생성합니다. 나머지 부분은 원본 객체를 재사용함으로써, 메모리 사용을 효율적으로 관리하고, 불필요한 객체 생성과 구조적 변화를 최소화합니다. 이 과정에서 발생할 수 있는 메모리 낭비와 처리 횟수 증가를 방지하여, 전체적인 애플리케이션의 성능을 최적화 해줍니다.

## 마치며

리덕스 immer의 필요성에 대해 의문을 표하며 없는 편이 낫다는 의견을 봤습니다. 그 의견에 대해 정말인지 살펴보고 싶은 호기심이 생겨 자료를 조사해보고 정리해봤습니다. 조사 결과 불변성을 관리하는 두 가지 방식은 상황에 따라 선택하기 나름이라고 결론을 내릴 수 있었습니다.