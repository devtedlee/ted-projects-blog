---
title: "명령형/선언형 프로그래밍을 이해해보자"
datePublished: Thu Mar 07 2024 04:00:29 GMT+0000 (Coordinated Universal Time)
cuid: cltgp9kh4000108l0h5it1ejh
slug: understand-imperative-declarative-programming
cover: https://cdn.hashnode.com/res/hashnode/image/stock/unsplash/42uZfLm9uUQ/upload/3d91256cb3188eeff85ea8d1feb25290.jpeg
tags: declarative-vs-imperative-programming

---

프로그래밍 세계에서 '선언적(Declarative)'이랑 '명령형(Imperative)' 프로그래밍은 마치 요리를 할 때 레시피를 따라하는 것과 직접 재료를 손질하고 조리하는 차이라고 할 수 있습니다.

### 명령형 프로그래밍(Imperative Programming)

명령형 프로그래밍은 '어떻게(How)'에 초점을 맞춥니다. 즉, 목표를 달성하기 위한 단계별 지시사항을 컴퓨터에게 제공합니다. 마치 요리할 때 "당근을 썰고, 물을 끓이고, 당근을 물에 넣어라" 같이 말입니다.

**예시:**

```javascript
let numbers = [1, 2, 3, 4, 5];
let doubledNumbers = [];

for (let i = 0; i < numbers.length; i++) {
    doubledNumbers.push(numbers[i] * 2);
}
```

위 코드에서는 배열의 각 요소를 순회하며, 각 요소를 2배로 만들어 새 배열에 넣는 과정을 직접 구현했습니다.

### 선언적 프로그래밍(Declarative Programming)

반면 선언적 프로그래밍은 '무엇을(What)'에 초점을 맞춥니다. 즉, 당신이 원하는 결과를 선언하고, 그 과정은 신경 쓰지 않는 겁니다. 컴퓨터가 알아서 최선의 방법을 찾아냅니다.(보통 우리가 많이 작성하는 코드에서는 함수나 클래스가 알아서 하는 겁니다) 마치 "당근 스프 만들어"라고 말하는 것과 비슷해. 방법은 신경 쓰지 않고 결과만 명시하는 겁니다.

**예시:**

```javascript
let numbers = [1, 2, 3, 4, 5];
let doubledNumbers = numbers.map(number => number * 2);
```

위 코드에서는 `.map`이라는 함수를 사용해서 배열의 각 요소를 2배로 만드는 작업을 설명합니다. '어떻게' 하는지는 쓰지 않았지만 결과적으로 같은 작업을 수행합니다.

선언적 프로그래밍은 뭘 해야 하는지 말해주고, 명령형은 어떻게 해야 하는지 까지 설명해주는 겁니다.

## SOLID 원칙과 함께 생각해보기

### 단일 책임 원칙(Single Responsibility Principle)

단일 책임 원칙은 클래스나 모듈이 하나의 책임만 가져야 한다고 말합니다. 선언적 프로그래밍에서는 이 원칙을 더 잘 따를 수 있어요, 왜냐하면 데이터 변환 또는 처리를 담당하는 함수들을 작성할 때 각 함수가 하나의 작업만 수행하도록 만들기 쉽기 때문입니다.

**예시:**

```javascript
// 선언적 방식
const getUserFullName = user => `${user.firstName} ${user.lastName}`;
const getUserAge = user => new Date().getFullYear() - user.birthYear;

// 이 예시에서 각 함수는 하나의 책임만 지니며, 
// 이를 조합해 더 복잡한 로직을 구성할 수 있습니다.
```

### 의존성 역전 원칙(Dependency Inversion Principle)

의존성 역전 원칙은 고수준 모듈이 저수준 모듈의 구현 세부 사항에 의존해서는 안 되며, 둘 다 추상화에 의존해야 한다고 말합니다. 선언적 프로그래밍은 종종 함수형 프로그래밍 패러다임과 밀접하게 관련되어 있으며, 이는 높은 수준의 추상화를 제공합니다.

**예시:**

```javascript
// 선언적 방식
const users = [
  { name: "John Doe", age: 30 },
  { name: "Jane Doe", age: 25 }
];

// 고수준의 추상화 함수
const findUsersOverAge = (users, age) => users.filter(user => user.age > age);

// 이 예시에서 `findUsersOverAge` 함수는 저수준의 데이터 구조(여기서는 users 배열)에 의존하지 않으며, 
// 대신 데이터를 처리하는 데 필요한 추상화된 방식에 의존합니다.
```

이렇게 선언적 프로그래밍은 각 기능을 명확히 분리된 작은 단위로 나눔으로써 코드의 재사용성과 테스트 용이성을 높이며, 디자인 원칙을 준수하는 데 도움을 줍니다.