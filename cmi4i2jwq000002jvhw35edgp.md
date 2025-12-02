---
title: "[기술회고] NavLink 스타일링 버그 수정기"
datePublished: Tue Nov 18 2025 11:38:29 GMT+0000 (Coordinated Universal Time)
cuid: cmi4i2jwq000002jvhw35edgp
slug: ai-navlink-styling
cover: https://cdn.hashnode.com/res/hashnode/image/upload/v1762334499649/85008179-539c-400f-8862-c1648097fd07.png
ogImage: https://cdn.hashnode.com/res/hashnode/image/upload/v1762334503125/c3046da0-c00f-47ee-b7ef-e4bcbe72459b.png
tags: ai, react-router

---

## TL;DR

2개의 depth를 가진 네비게이션의 활성화(`active`) 상태를 표현하는 스타일이 사라져버린 버그를 수정하는 과정에서, AI가 디자인 가이드를 잘못 해석해 3번 잘못된 수정을 제안 했습니다. 결국 제가 판단한 추가적인 피드백으로 문제를 해결할 수 있었습니다. 그에 대한 사례를 간단히 정리 해봤습니다.

---

## 문제의 시작: 사라진 활성 상태

헤더의 2depth 메뉴에서 현재 페이지를 나타내는 활성 상태 스타일이 작동하지 않는다는 이슈를 할당 받았습니다.

```typescript
// 문제가 된 코드
<NavLink
  to={item.href}
  className={cn(
    "block p-0 w-full text-center ...",
    activeFirstDepthMenu?.label === menu.label
      ? "text-normal"
      : "text-placeholder",
  )}
>
```

NavLink의 `className`이 문자열로 고정되면서 `isActive` prop을 활용하지 못하고 있었습니다.

---

## 1차 시도: 과도한 수정

AI는 문제를 파악하고 NavLink의 `className`을 콜백 함수로 변경했습니다.

```typescript
<NavLink
  className={({ isActive }) =>
    cn(
      "block p-0 w-full text-center ... hover:text-primary-normal",
      isActive
        ? "text-primary-normal font-medium"  // 👈 파란색으로 강조
        : activeFirstDepthMenu?.label === menu.label
          ? "text-normal"
          : "text-placeholder",
    )
  }
>
```

**AI의 판단**: "활성 상태는 primary 색상으로 강조해야지!"

**실제 가이드**: 활성 상태도 text-normal + font-medium으로 표시

---

## 2차 시도: 규칙을 오해하다

개발자가 직접 `text-primary-normal`을 `text-normal`로 수정했습니다.

그런데 AI는 이렇게 이해했습니다:

> "아! 2depth 메뉴는 **선택되지 않은 1depth의 하위는 회색**, **마우스 오버만 파란색**이구나!"

```typescript
className={({ isActive }) =>
  cn(
    "... hover:text-primary-normal",
    isActive
      ? "text-normal font-medium"
      : "text-placeholder",  // 👈 모두 placeholder
  )
}
```

**AI의 판단**: "1depth 상관없이 활성 항목만 normal, 나머지는 전부 placeholder!"

**문제**: 같은 카테고리 내 다른 메뉴까지 흐리게 표시되어 가독성 저하

---

## 3차 시도: 다시 복잡하게

개발자로부터 피드백:

> "Line 198의 className 콜백이 isActive가 false일 때 항상 placeholder를 반환하면서, 현재 1depth 아래의 다른 2depth 링크들도 전부 placeholder로 표시됩니다."

AI는 3단계 분기가 필요하다고 판단했습니다

```typescript
className={({ isActive }) =>
  cn(
    "... hover:text-primary-normal",
    isActive
      ? "text-normal font-medium"
      : activeFirstDepthMenu?.label === menu.label  // 👈 다시 추가
        ? "text-normal"
        : "text-placeholder",
  )
}
```

**AI의 판단**: "같은 1depth 내에서는 읽기 쉽게 normal 색상을 유지해야 해!"

**실제 가이드**: 그런 규칙 없음. 활성 항목 외에는 모두 placeholder

---

## 마침내: 사람의 개입

개발자의 강력한 피드백:

> "하 진짜 가이드에 맞는 구현이라니까 왜 자꾸 activeFirstDepthMenu 들이대냐고 아니라고!!!"

**최종 정답**

```typescript
className={({ isActive }) =>
  cn(
    "block p-0 w-full text-center whitespace-nowrap rounded transition-all text-body-14 hover:text-primary-normal",
    isActive
      ? "text-normal font-medium"
      : "text-placeholder",
  )
}
```

**실제 디자인 가이드**:

* 활성화된 2depth 항목: `text-normal font-medium`
    
* 나머지 모든 2depth 항목: `text-placeholder`
    
* 마우스 오버: `hover:text-primary-normal`
    

단순하고 명확한 규칙이었습니다.

---

## AI는 왜 계속 틀렸을까?

### 1\. **암묵적인 디자인 가이드를 추론하려 했다**

AI는 명시되지 않은 "일반적인 UX 패턴"을 적용하려 했습니다

* "활성 항목은 primary 색상으로 강조한다"
    
* "같은 카테고리 내에서는 가독성을 유지한다"
    

하지만 이 프로젝트에는 **고유한 디자인 시스템**이 있었고, 그것은 일반적인 패턴과 달랐습니다.

### 2\. **컨텍스트를 과도하게 해석했다**

개발자의 피드백

> "선택되지 않은 1Depth의 하위 메뉴는 회색 비활성 상태로 노출됩니다."

AI의 해석

> "아! 그럼 선택된 1Depth의 하위는 활성 상태로 봐야겠네!"

**실제 의미**: 모든 2depth는 기본적으로 회색. 오직 현재 페이지만 예외.

### 3\. **점진적 수정의 함정**

각 피드백에 대해 "기존 코드에 뭔가 추가"하는 방식으로 접근했습니다

1. `isActive` 추가
    
2. `activeFirstDepthMenu` 체크 제거
    
3. `activeFirstDepthMenu` 체크 다시 추가 ← 엉뚱한 방향
    

처음부터 디자인 가이드를 정확히 이해했다면 2번에서 끝났을 일입니다.

---

## 교훈: AI와 협업할 때

### ✅ DO

1. **명확한 디자인 스펙 제공**
    
    * "활성 항목만 X 스타일, 나머지는 Y 스타일"처럼 명확하게
        
    * 예외 케이스를 구체적으로 명시
        
2. **잘못된 방향은 즉시 차단**
    
    * "아니야" 보다는 "이건 필요 없어, 실제로는 A와 B 두 가지만 구분하면 돼"
        
3. **실제 가이드 문서 공유**
    
    * 디자인 시스템 문서나 Figma 링크 제공
        

### ❌ DON'T

1. **AI가 "일반적인 패턴"을 적용할 거라 기대**
    
    * 프로젝트마다 고유한 규칙이 있음
        
2. **모호한 피드백**
    
    * "가독성이 떨어진다" → 무엇을 기준으로?
        
    * "활성 상태가 표시되지 않는다" → 무엇이 활성 상태인지?
        
3. **AI의 "논리적 추론"을 과신**
    
    * AI는 컨텍스트를 유추하려 하지만, 틀릴 수 있음
        

---

## 결론

이번 사례는 **AI가 문법은 완벽하게 처리하지만, 도메인 지식(디자인 가이드)은 여전히 사람의 명확한 전달이 필요**하다는 것을 보여줍니다.

최종 코드는 단 3줄이었지만, 그 3줄에 도달하기까지 AI와 사람 사이의 4번의 대화가 필요했습니다.

AI 도구는 강력하지만, **"무엇이 옳은지"는 아직 사람이 판단해줘야** 했습니다. 특히 프로젝트 고유의 규칙과 컨텍스트가 중요한 영역에서는 더욱 그러리고 예상됩니다.

사실 이슈 자체는 크지 않았고, 문제도 간단히 해결할 수 있었지만, 이러한 사례들을 정리해서 모아둘 수 있다면 개인적으로 좋은 자료가 될 수 있을거 같아 글로 정리해봤습니다. 읽어주셔서 감사합니다.