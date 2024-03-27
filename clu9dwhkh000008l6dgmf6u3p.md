---
title: "프론트엔드 개발자가 알면 좋은 웹 접근성"
datePublished: Wed Mar 27 2024 05:47:42 GMT+0000 (Coordinated Universal Time)
cuid: clu9dwhkh000008l6dgmf6u3p
slug: web-accessibility-for-frontend
cover: https://cdn.hashnode.com/res/hashnode/image/upload/v1711518394787/b7de43af-1898-4258-8164-5d6418b9ef73.webp
tags: frontend-development, web-accessibility

---

웹 접근성이란 웹 사이트, 도구, 기술이 장애를 가진 사용자들이 사용할 수 있도록 설계 및 개발된 것을 말합니다. ([W3C 공식 홈페이지](https://www.w3.org/WAI/fundamentals/accessibility-intro/ko))

여기서 장애의 범위는 간단 합니다. 웹에 접근하는데 영향을 주는 모든 장애를 말합니다.  
여기에는 청각, 인지, 신경, 신체, 언어, 시각이 포함됩니다.

그리고 이러한 개선으로 인해 장애를 갖지 않은 사람에게도 이점을 줍니다.  
작은 화면으로 인해 불편한 사람, 노화로 인해 기능적 능력이 퇴화하여 어려운 사람, 중상 등으로 다쳐서 일시적인 장애를 겪는 사람, 느린 인터넷을 사용하는 사람 등등 이러한 경우에도 웹 접근성으로 인한 혜택을 누릴 수 있습니다.

프론트엔드 개발자 시점에서 웹 접근성을 챙기는 것은 법적인 요구사항의 충족 뿐만 아니라 더 넓은 사용자 베이스를 배려하는 UX를 제공하고, 심지어 테스트를 작성하는데 도움을 받을 수 있습니다.

## 주요 웹 접근성 요소

### 1) 시멘틱 마크업(Semantic Markup)

* HTML5의 시멘틱 태그(`<header>`, `<nav>`, `<footer>` 등)를 적절히 사용하여 문서의 구조를 명확하게 하고, 스크린 리더가 콘텐츠를 쉽게 해석할 수 있게 해야 합니다.
    

```xml
<!-- 좋은 예 -->
<nav> <!-- 네비게이션 메뉴 -->
  <ul>
    <li><a href="#home">Home</a></li>
    <li><a href="#about">About</a></li>
  </ul>
</nav>

<!-- 나쁜 예 -->
<div> <!-- 네비게이션 메뉴 -->
  <ul>
    <li><a href="#home">Home</a></li>
    <li><a href="#about">About</a></li>
  </ul>
</div>
```

### 2) 대체 텍스트(Alt Text)

* 이미지와 같은 비텍스트 콘텐츠에 대체 텍스트를 제공하여, 시각 장애가 있는 사용자가 스크린 리더를 통해 콘텐츠를 이해할 수 있도록 합니다.
    

```xml
<!-- 좋은 예 -->
<img src="dog.jpg" alt="갈색 래브라도 레트리버가 공을 물고 뛰는 모습">

<!-- 나쁜 예 -->
<img src="dog.jpg">
```

### 3) 키보드 접근성 (Keyboard Accessibility)

* 모든 인터랙션은 키보드만으로도 수행될 수 있어야 합니다. 대표적인 예로 탭(Tab) 키를 통한 요소 간 이동, 엔터(Enter) 키를 통한 입력 등이 있습니다.
    

```xml
<!-- 모든 버튼과 링크는 키보드로 접근 가능해야 함 -->
<button>Click Me</button>
<a href="https://example.com">Visit Example</a>
```

### 4) ARIA의 사용 (Accessible Rich Internet Application)

* ARIA 랜드마크 역할, 속성 및 상태를 사용하여 웹 어플리케이션의 접근성을 향상 시킵니다. 특히 JS로 동적으로 생성된 콘텐츠가 있을 때 유용합니다.
    

```xml
<!-- div를 버튼으로 인식 시키기 -->
<div role="button" tabindex="0" onclick="alert('버튼 클릭됨!')">클릭하세요</div>

<!-- 스크린 리더에게 설명 제공하기 -->
<button aria-label="검색하기" onclick="search()">
  <img src="search-icon.png" alt="">
</button>

<!-- 상태 사용하기 -->
<button aria-pressed="false" onclick="toggleButton(this)">
  토글
</button>
```

### 5) 명확한 폼 레이블과 오류 메세지

* `<form>`필드에는 명확한 레이블이 있어야 하며, 입력 오류 시 이해하기 쉬운 오류 메세지를 제공해야 합니다.
    

```xml
<!-- 좋은 예 -->
<label for="email">이메일:</label>
<input type="email" id="email" name="email">
<!-- 이메일 필드가 비어있을 때 -->
<span role="alert">이메일 주소를 입력해주세요.</span>

<!-- 나쁜 예 -->
<input type="email" id="email" name="email">
<!-- 오류 메시지 없음 -->
```

### 6) 색상 대비(Color Contrast)

* 텍스트와 배경 사이에 충분한 대비가 있어야 합니다. 색상에 민감하지 않은 사용자에게도 콘텐츠를 쉽게 읽을 수 있게 해주기 위함입니다.
    

### 7) 반응형 디자인(Responsive Design)

* 다양한 장치와 화면 크기에서 콘텐츠가 잘 보이고 사용하기 쉬워야합니다. 어떤 장치를 사용하든 일관된 접근성을 보장해줍니다.
    

```css
/* CSS 미디어 쿼리를 사용한 반응형 디자인 */
@media (max-width: 600px) {
  nav ul {
    display: block;
  }
}
```

## 테스트 작성에도 도움 되는 웹 접근성

* **명확한 구조와 역할**: HTML을 보다 시멘틱하게 사용하고, ARIA 역할도 적절하게 활용하게 됩니다. 그러다보니 테스트에서도 어떤 요소를 테스트해야 될지 더 명확해 집니다.
    
* **일관된 상호작용 모델**: 키보드 접근성과 같은 일관된 상호작용 모델을 사용하게 되기 때문에, 사용자의 인터랙션을 모방하기 쉬워집니다.
    

좀 더 자세한 예시는 좋은 아티클을 소개해 드리겠습니다.

> [접근성이 이끄는 웹 프론트엔드 테스트 자동화](https://tech.wonderwall.kr/articles/a11ydriventestautomation/)

# 마치며

간단하게 웹 접근성이 무엇인지, 어떤 식으로 구현하면 웹 접근성을 갖출 수 있게 되는지 조사해서 정리해봤습니다. 조사를 하며, 왜 자꾸 IDE 들이 `<img>` 태그에 alt를 추가하라고 경고를 하는지를 새삼 알 수도 있었고, 웹 접근성을 잘 지키는 페이지를 개발하면 장애를 가진 분들 뿐 아니라 일반 유저들에게도 도움될 수 있다는 점을 알 수 있었습니다.