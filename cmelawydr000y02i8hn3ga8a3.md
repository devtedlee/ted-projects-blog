---
title: "Flexbox와 Truncate의 배신"
datePublished: Thu Aug 21 2025 11:11:26 GMT+0000 (Coordinated Universal Time)
cuid: cmelawydr000y02i8hn3ga8a3
slug: css-flexbox-truncate
cover: https://cdn.hashnode.com/res/hashnode/image/upload/v1755755844288/9c63d4c7-7b22-4067-9607-a9c99f50fdee.png
ogImage: https://cdn.hashnode.com/res/hashnode/image/upload/v1755755888481/5d880c8a-92bc-4015-a20f-67bf5f9a1e9e.png
tags: flexbox, css, tailwind-css

---

프론트엔드 개발에서 텍스트를 다루는 것은 기본입니다. 하지만 `flex` 레이아웃과 `truncate` 같은 유틸리티가 복합적으로 얽히기 시작하면, 가장 단순해 보이는 텍스트조차 예상과 다르게 동작해서 스타일 버그를 만들게 됩니다.

최근 겪었던 실제 사례를 통해 이 문제를 깊이 파헤쳐 보겠습니다. 상황은 이랬습니다. 헤더에 표시되는 회사 이름 중, `이더리움그룹`처럼 긴 이름은 정상 출력되는데, `(주)비트`처럼 훨씬 짧은 이름이 말줄임표(...) 처리되는 현상이 발생했습니다. 직관에 반하는 이 현상을 마주하고 혼란스러웠고 문제를 해결해도 개운하게 이해가 안됐었습니다. 결론부터 말하면, 범인은 `truncate`가 아니라 `flex-shrink`였습니다.

### **Case Study: 왜 짧은 텍스트가 잘리는가?**

문제의 컴포넌트는 다음과 같은 구조를 가졌습니다.(의사 코드입니다)

```xml
// Header Layout
<header className="flex justify-between items-center">
  {/* Left Section */}
  <div className="flex items-center gap-3">
    <img src="/logo.svg" alt="logo" />
    <CompanyName companyName={name} />
  </div>

  {/* Right Section */}
  <div className="flex items-center gap-4">
    <span>{user.name}</span>
    <Avatar />
  </div>
</header>
```

`CompanyName` 컴포넌트는 반응형 분기에 따라 `md` 이상 스크린에서 `truncate`가 적용되도록 설정되어 있었습니다. 초기 분석 시에 `truncate`와 텍스트 줄바꿈(`word-break`)의 문제에 집중했었죠.

### **1차 분석:** `word-break`와 `truncate`의 상호작용

최초 코드에는 한글 단어가 깨지지 않도록 `break-keep` (`word-break: keep-all`)이 적용해둔 상태였습니다. 여기서 첫 번째 단서를 발견했습니다.

* `이더리움그룹`: `break-keep`에 의해 통짜 단어로 인식된다.
    
* `(주)비트`: 괄호 `()`는 CSS에서 단어 분리 문자로 취급될 수 있습니다. 이로 인해 브라우저는 `(주)`와 `비트`를 별개의 어휘 단위로 판단, 그 사이에서 줄바꿈이 발생할 수 있습니다.
    

이것이 초기 "깨짐" 현상의 원인이었습니다. 하지만 `truncate`는 `white-space: nowrap`을 포함하므로 줄바꿈 자체가 일어나면 안 됩니다. 그런데 왜 `(주)비트`는 여전히 두 줄로 "깨지면서" 동시에 `truncate`가 되려고 했을까요?

범인은 `md:whitespace-normal` 같은 반응형 오버라이드였습니다. 특정 분기점에서 `nowrap`을 해제하면서 `break-keep`의 동작과 충돌을 일으킨 것입니다.

이 분석에 따라 `break-all`을 적용하고 반응형 클래스를 정리했습니다. 줄바꿈 문제는 해결됐지만, 근본적인 `truncate` 현상은 사라지지 않았습니다.

### **2차 분석: 진짜 범인,** `flex-shrink`

문제의 본질은 `CompanyName` 컴포넌트 자체에 있지 않았습니다. 헤더의 `flex` 컨테이너가 원인이었습니다.

1. **Flexbox의 공간 분배**: `justify-between`은 자식 요소들을 양 끝으로 밀어냅니다. 왼쪽엔 `[로고, 회사이름]`, 오른쪽엔 `[사용자이름, 아바타]`가 위치합니다.
    
2. `flex-shrink`의 기본 동작: `flex` 아이템은 기본적으로 `flex-shrink: 1` 속성을 가집니다. 이는 컨테이너에 공간이 부족할 경우, 할당된 공간을 양보하며 스스로 "수축(shrink)"할 수 있음을 의미합니다.
    
3. **시나리오**: 오른쪽의 `사용자이름`이 길어지면, 오른쪽 섹션은 더 많은 공간을 요구합니다. `flex` 컨테이너의 전체 너비는 고정되어 있으므로, 왼쪽 섹션은 공간을 양보해야 합니다. 이때 `CompanyName`이 `flex-shrink: 1` 속성에 따라 수축하게됩니다.
    

결국 `(주)비트`라는 텍스트 길이는 문제가 아니었습니다. 다른 `flex` 아이템 때문에 `CompanyName` 컴포넌트 자체가 **자신의 콘텐츠보다 좁은 너비를 할당받았기 때문에** `truncate`가 발동한 것입니다.

### **해결책:** `flex-shrink: 0`

해결책은 간단했습니다. `CompanyName` 컴포넌트가 수축하지 않도록 명시하는 것입니다.

```typescript
function CompanyName({ className, companyName }) {
  return (
    /** 'shrink-0'가 핵심 */
    <div
      className={cn(
        "shrink-0 px-3 py-1 font-medium rounded-md border md:truncate ...",
        className
      )}
    >
      {companyName}
    </div>
  );
}
```

Tailwind 유틸리티 `shrink-0` (`flex-shrink: 0`)를 추가함으로써, 이 컴포넌트는 더 이상 형제 요소에게 공간을 양보하지 않고 자신의 고유 콘텐츠 너비를 유지하게 됩니다. 이로써 짧은 텍스트가 잘리는 현상은 해결됐습니다.

### **CSS 텍스트 처리 속성 요약**

이 사례에 등장한 핵심 속성들을 표로 정리하면 다음과 같습니다. 이 표 하나로 대부분의 텍스트 처리 이슈를 진단할 수 있습니다. 텍스트 처리 이슈를 마주할때마다 제가 다시 보기 위해 정리 해뒀습니다.

| 목표 | CSS | Tailwind 클래스 | 설명 |
| --- | --- | --- | --- |
| **자연스러운 줄바꿈** | `white-space: normal;` | `whitespace-normal` | 띄어쓰기 단위로 줄바꿈 (기본값) |
| **강제 한 줄 유지** | `white-space: nowrap;` | `whitespace-nowrap` | 절대 줄바꿈 안함. 컨테이너를 뚫고 나감 |
| **한글 단어 유지** | `word-break: keep-all;` | `break-keep` | 한글 등에서 단어 단위 줄바꿈 시도 |
| **강제 글자 단위 분리** | `word-break: break-all;` | `break-all` | 단어 무시하고 아무데서나 줄바꿈 |
| **말줄임표(...) 처리** | `nowrap` + `hidden` + `ellipsis` | `truncate` | 3가지 속성을 한 번에 적용 |
| **Flex 아이템 수축 방지** | `flex-shrink: 0;` | `shrink-0` | Flex 공간이 부족해도 아이템 크기 유지 |

#### **키 포인트**

* `truncate`는 텍스트 길이가 아닌, 할당된 공간의 크기에 따라 동작한다.
    
* `flex` 레이아웃에서 아이템의 크기는 형제 요소에 의해 동적으로 변할 수 있다. `flex-shrink` 속성을 항상 염두에 두어야 한다.
    
* `break-keep`은 [CJK 텍스트](https://ko.wikipedia.org/wiki/CJK) 처리 시 유용하지만, 특수 문자에 의해 의도치 않은 단어 분리가 발생할 수 있다.
    
* 문제가 발생하면 해당 요소만 보지 말고, 부모의 레이아웃 시스템(Flex, Grid 등)과 형제 요소와의 상호작용까지 반드시 확인해야 한다.
    

단순한 UI 버그처럼 보였던 이슈를 통해 CSS의 핵심 원리인 레이아웃, 컨텍스트, 그리고 속성 간의 상호작용을 돌아볼 수 있었습니다. 정리를 안해두면 왠지 계속 비슷한 씨름을 하게 될 것 같아 글로 정리해봤습니다.  

감사합니다.