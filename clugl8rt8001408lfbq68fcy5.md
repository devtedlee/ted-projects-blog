---
title: "리액트 애니메이션 적용 주의사항"
datePublished: Mon Apr 01 2024 06:47:36 GMT+0000 (Coordinated Universal Time)
cuid: clugl8rt8001408lfbq68fcy5
slug: react-animation-troubleshooting
cover: https://cdn.hashnode.com/res/hashnode/image/upload/v1711953983584/9986dc8f-b1c1-430c-b8c9-ca8cd484cfaf.webp
tags: animation, reactjs

---

리액트에서 애니메이션을 적용하는 것은 웹 애플리케이션에 생동감을 더하는 훌륭한 방법입니다. 하지만 CSS만을 사용해서 애니메이션을 적용하려 할 때, 리액트의 동작 방식과 충돌하여 원하는 대로 애니메이션이 동작하지 않는 경우가 있습니다. 그 중 제가 마주한 두 가지 케이스에 대해 문제 상황을 간단한 수도 코드로 재현해보고 어떻게 해결했는지 정리해 봤습니다.

## 문제 상황 2가지

### case 1) **리액트의 리렌더링과 호환되지 않는 애니메이션 동작**

```css
/* Animation.css */
.fadeIn {
  animation: fadeInEffect 1s;
}

@keyframes fadeInEffect {
  from { opacity: 0; }
  to { opacity: 1; }
}
```

```javascript
import './Animation.css'; // CSS 파일 임포트

function MyComponent({ isVisible }) {
  return <div className={isVisible ? "fadeIn" : ""}>안녕하세요!</div>;
}
```

* 위 예시에서 `isVisible` 상태에 따라 `fadeIn` 클래스를 적용하려고 시도했으나 `isVisible`이 true로 설정된 후 컴포넌트가 리렌더링 될 때마다가 애니메이션이 재실행 되지 않습니다.
    
* 리액트는 상태 데이터가 변경 될 때 컴포넌트를 리렌더링 합니다. CSS 애니메이션은 주로 요소가 처음 페이지에 로드될 때 동작하는데, 리액트에서 상태 변경으로 인한 리렌더링이 일어나면 애니메이션이 예상대로 동작하지 않을 수 있습니다.
    

### case 2) **HTML 요소 중** `key`**가 일치하지 않으면 동작하지 않는 애니메이션**

```javascript
function MyList({ items }) {
  return (
    <ul>
      {items.map((item) => (
        <li key={item.id}>{item.text}</li> // key 값이 변경되지 않음
      ))}
    </ul>
  );
}
```

* 동일한 `key`값을 가진 요소에 애니메이션을 적용하려고 할 때, key 값이 변경되지 않으면 애니메이션이 재시작되지 않습니다.
    
* 이 `key`가 변경되지 않는 경우 리액트는 컴포넌트를 재사용합니다. 따라서, 새 요소에 대한 애니메이션을 적용하려 해도 `key`가 바뀌지 않으면 애니메이션이 시작되지 않습니다.
    

## 해결 방안

### case 1) 리렌더링과 호환되지 않는 경우

```javascript
import React, { useEffect, useState } from 'react';
import './Animation.css'; // CSS 파일 임포트

function MyComponent() {
  const [isVisible, setIsVisible] = useState(true);

  useEffect(() => {
    const timer = setTimeout(() => {
      setIsVisible(false); // 0.5초 후에 isVisible 상태를 false로 변경
    }, 500); // 애니메이션 지속 시간과 일치시키기

    return () => clearTimeout(timer); // 컴포넌트 언마운트 시 타이머 정리
  }, []);

  return (
    <div className={isVisible ? 'fadeIn' : ''}>안녕하세요!</div>
  );
}

export default MyComponent;
```

* 상태 데이터와 useEffect 훅을 연결하여 리렌더링이 발생할 때 애니메이션이 매번 새로 실행되도록 개선 했습니다.
    

### case 2) `key`가 일치하지 않으면 동작하지 않는 경우

```javascript
function MyList({ items }) {
  return (
    <ul>
      {items.map((item) => (
        // 각 항목에 고유한 key 값을 할당하여 리액트가 새 요소로 인식하도록 합니다.
        <li key={item.id + Date.now()}>{item.text}</li>
      ))}
    </ul>
  );
}
```

* `key`의 값을 동적으로 변경하여 리액트가 새로운 요소로 인식하게 만들어줬습니다.
    
* 다만 `Date.now()`는 그냥 예시로 설정한것이므로, 다른 고유성을 반영할 수 있는 키를 활용하시는걸 권장합니다.
    

## 기타 주의사항

* 위에서 알아본 두 가지 케이스는 어디까지나 제가 마주한 문제 케이스에서의 해결책이었습니다.
    
* 복잡한 애니메이션을 구현할 때는 전용 애니메이션 라이브러리를 도입하시는 것을 추천드립니다. 대표적으로 [Framer Motion](https://www.framer.com/motion/), [React Spring](https://www.react-spring.dev/) 등이 있습니다.