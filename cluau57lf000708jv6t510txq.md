---
title: "개발 모드에 대해 알아야 할 점"
datePublished: Thu Mar 28 2024 06:10:09 GMT+0000 (Coordinated Universal Time)
cuid: cluau57lf000708jv6t510txq
slug: javascript-development-mode
cover: https://cdn.hashnode.com/res/hashnode/image/upload/v1711606142191/00ab5c0a-bc3c-4016-8991-9aead79e763e.png
tags: javascript, bundler, development-environments

---

> NHN 토스트에서 번역한 [댄 아브라모브의 글 - 개발 모드는 어떻게 작동할까?](https://ui.toast.com/posts/ko_20191212) 를 보고 기억할 내역을 정리 및 요약한 글입니다.

개발 모드를 위한 환경 변수의 사용과 배포 환경과의 연관 관계에 대해 실질적으로 도움될 만한 부분이 있다고 생각하여 관련 글을 정리해봤습니다.

## 환경 변수의 역사

* 초기에 JS진영 환경 변수는 `EXPRESS_ENV`로 시작됐습니다. 서버 사이드 어플레이케이션에서 설정을 구분하는 간단한 방법이었습니다.
    
* 2010년에 `NODE_ENV`가 등장하면서 노드 기반의 백엔드 개발에서 널리 채택 되기 시작했습니다.
    
* 이후에는 2013년 부터 웹팩(Webpack)등의 모던 프론트엔드 빌드 툴과 함께 프론트엔드 개발에도 폭넓게 확산 되었습니다.
    

## 환경 변수의 동작 방식

* 환경 변수는 빌드 타임에 문자열로 치환 됩니다.
    

```javascript
// .env
NODE_ENV=production

// app.js
if (process.env.NODE_ENV !== "production") {
  doSomethingDev();
} else {
  doSomethingProd();
}
```

* 위와 같이 써져 있으면 사실상 이것과 똑같이 동작합니다.
    

```javascript
// .env
NODE_ENV=production

// app.js
if ("production" !== "production") {
  doSomethingDev();
} else {
  doSomethingProd();
}
```

## 죽은 코드 제거(Dead-code elimination)와 환경 변수

* 모던 자바스크립트 번들러는 배포 모드에서 죽은 코드를 제거하는 최적화 작업을 수행합니다. 환경 변수는 이 최적화 과정에서 중요한 역할을 합니다.
    

```javascript
// 죽은 코드 제거가 적용된 위 예시
// .env
NODE_ENV=production

// app.js
doSomethingProd();
```

### 죽은 코드 제거를 활용할 때 주의 사항

* `import` 문과 같은 모듈 시스템에서는 환경 변수를 바탕으로 한 조건부 코드 분기가 예상대로 동작하지 않을 수 있습니다. 모듈 시스템은 정적 구조를 가지기 떄문에 런타임에 조건을 바꿀 수 없기 때문입니다.
    

```javascript
// .env
NODE_ENV=production

// app.js
// 절대 사용하지 않겠지만 제거된다는 보장은 없다.
import { someFunc } from "some-module"; 

if (process.env.NODE_ENV !== 'production') {
  someFunc();
}

// 번들러가 모듈 전체를 하나의 단위로 처리하기 때문에, 
// 특정 함수가 사용되지 않더라도 해당 모듈 전체를 최종 번들에 포함시킬 가능성이 높음
```

* 환경 변수를 직접 JS 변수에 할당하는 경우(let, const, var), 번들러가 해당 변수를 코드에서 제거하지 않을 수 있습니다.
    

```javascript
// 환경 변수 값을 로컬 변수에 할당
const nodeEnv = process.env.NODE_ENV;

if (nodeEnv === 'production') { // 죽은 코드 제거 안됨
  console.log('프로덕션 모드에서 실행 중');
} else {
  console.log('개발 모드에서 실행 중');
}
```

## 개발 모드는 필수다

* 어플리케이션 개발 시 개발 모드는 디버깅과 오류 추적을 위해 필수적입니다. 개발 모드의 특성 상 성능이 느려지고, 일견 불필요해 보이는 경고 등이 많이 노출되겠지만 그럼에도 불구하고 치명적인 실수를 막아주기 때문입니다.
    

* **불필요해 보이지만 치명적인 예시**: React key 경고가 없을 때의 위험성. 잘못된 상품을 구매하거나 잘못된 사람에게 메세지가 전송 되거나 이러한 인덱스로 인한 잠재적인 문제를 방지해 줍니다.
    

---

원 글이 훌륭하지만, 나름대로 재정리하고 관련 예시 코드도 작성해보았습니다.  
끝까지 읽어주신 분들 감사합니다.