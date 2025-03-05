---
title: "any와 unknown 타입에 대해 알아보자 - Typescript"
datePublished: Sat Mar 09 2024 06:35:26 GMT+0000 (Coordinated Universal Time)
cuid: cltjpojqu000108l870dwddwc
slug: any-unknown-typescript
cover: https://cdn.hashnode.com/res/hashnode/image/upload/v1710132659535/3798c955-f8bd-43ed-969b-51ae0ae83400.png
tags: typescript, any-unknown

---

### ✅ any와 unknown 타입은 무엇인가요?

* **any 타입**: Typescript에서 `any` 타입은 ‘모든 종류의 값을 할당할 수 있음’을 나타내며, 이는 TypeScript 컴파일러에 의한 타입 체크를 회피할 수 있게 해줍니다. 이는 기본적으로 타입 시스템을 무시하고 JavaScript처럼 동적으로 코드를 작성하고 싶을 때 사용됩니다.
    
* **unknown 타입**: Typescript 3.0에서 도입된 `unknown` 타입은 `any` 타입과 유사하게 모든 종류의 값을 할당할 수 있지만, `unknown` 타입의 값을 사용하기 전에 해당 값의 타입을 좀 더 구체적으로 검사해야 합니다. 이는 더 안전한 타입 사용을 유도합니다.
    

### ✅ any와 unknown은 뭐가 다른가요?

* **타입 안정성**: `any` 타입은 타입 체크를 회피하여 타입 안정성을 감소시키는 반면, `unknown` 타입은 값을 사용하기 전에 타입 검사를 수행함으로써 더 안전한 타입 사용을 가능하게 합니다.
    
* **제어권**: `any` 타입을 사용하면 개발자가 Typescript의 타입 시스템 제어를 완전히 포기하는 것과 같지만, `unknown` 타입은 개발자에게 타입을 확인하고 처리하는 책임을 줍니다.
    
* **any의 장점**: 다만 `any`는 안전하지 않고, 제어권에 대해 좀 더 포기한 만큼 자유로운 코드 작성이 가능합니다.
    

### 🖥 any가 위험한 케이스

```typescript
function dangerous(param: any) {
    console.log(param.toUpperCase());
}

// 런타임 오류 발생 가능: 'toUpperCase'는 숫자에서 사용할 수 없음.
dangerous(10);
```

### 🖥 unknown 이 안전한 케이스

```typescript
function safe(param: unknown) {
    if (typeof param === 'string') {
        console.log(param.toUpperCase()); // 안전하게 사용
    } else {
        console.log('Not a string');
    }
}

safe('hello'); // "HELLO"
safe(10); // "Not a string"
```

## 🔎 any보다 안전한 unknown 타입 탄생 비화

위에서 잠시 언급 했듯 `unknown` 타입은 Typescript 3.0 때 생겼습니다. 이는 필요에 의한 것이었는데요. `unknown`타입이 있기 전 `any`는 Typescript에서 가장 많이 사용되는 타입이었습니다. 하지만 단점이 명확했죠. 타입 중에 "이 값은 어떤 값일 수 있으므로 사용하기 전에 어떤 유형의 검사를 수행해야 합니다" 라는 신호를 보내줄 수 있는 타입이 필요 했습니다.([RC 발표](https://devblogs.microsoft.com/typescript/announcing-typescript-3-0-rc-2/#the-unknown-type) 참고) 그리고 바로 그 역할을 수행해 줄 수 있는 `unknown` 타입이 생기게 됐습니다.

### ✨ 간단 정리

`any` 타입은 Typescript의 타입 체크를 회피하여 모든 값을 받을 수 있지만, `unknown` 타입은 타입 안전성을 보장하기 위해 값을 사용하기 전에 타입을 검사해야 합니다. `any` 타입의 사용은 타입 체크를 우회 함으로써 런타임 오류의 위험을 증가 시킬 수 있으며, `unknown` 타입은 타입 검사를 통해 보다 안전한 코드 작성을 가능하게 합니다.

각자가 처한 상황에 맞게 팀에서 논의해서 빠른 기능 구현에 도움이 되는 `any` 타입을 사용할지, 유지 보수가 용이하고 안전한 `unknown` 타입을 사용할지 정하면 좋겠습니다. (개인적으로 any는 왠만하면 지양하길 추천합니다)