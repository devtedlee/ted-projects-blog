---
title: "react-router와 UI 컴포넌트의 조용한 충돌"
datePublished: Wed Jul 30 2025 11:00:45 GMT+0000 (Coordinated Universal Time)
cuid: cmdpuuhd4002802jlalqw5ztk
slug: react-router-ui-race-condition
cover: https://cdn.hashnode.com/res/hashnode/image/upload/v1753781924411/966c2964-25d5-47e7-a8e7-084a64753276.png
tags: react-router, reactjs, race-condition

---

개발을 하다 보면 분명 코드는 멀쩡해 보이는데, 원하는 대로 동작하지 않아 몇 시간을 헤매는 경험, 다들 한 번쯤 있으실거 같습니다. 최근 저와 제 AI 페어 프로그래밍 파트너(AKA. cursor)도 `react-router-dom`의 `navigate` 함수가 아무런 오류 메시지 없이 먹통이 되는 현상을 겪었습니다.

오늘은 이 "조용한 버그"의 원인을 파헤치고 해결하는 과정을 공유해보겠습니다. 특히 `shadcn/ui`와 같은 헤드리스(Headless) UI 라이브러리를 사용하신다면 좀 더 흥미롭게 읽으실 수 있으실겁니다.

### 현상: `console.log`는 찍히는데, 페이지 이동은 안 된다?

상황은 단순했습니다. 사용자가 드롭다운 메뉴에서 '로그아웃' 버튼을 클릭하면 로그아웃 페이지로 이동시키는 기능이었습니다. 코드는 다음과 같았습니다.

```typescript
// Profile.tsx
import { useNavigate } from 'react-router-dom';
import { DropdownMenuItem } from '@/components/ui/DropdownMenu';

// ...

const navigate = useNavigate();

const handleLogout = () => {
  console.log("로그아웃 시도");
  navigate("/logout"); // 페이지 이동 실행
  console.log("navigate 함수 호출됨");
};

// ...

<DropdownMenuItem onSelect={handleLogout}>
  로그아웃
</DropdownMenuItem>
```

간단하고 명백해 보이는 코드였습니다. 하지만 실제 동작은 우리를 미궁에 빠뜨렸습니다. '로그아웃' 버튼을 클릭하면 브라우저 콘솔에는 "로그아웃 시도", "navigate 함수 호출됨"이 모두 찍혔지만, 정작 URL은 바뀌지 않고 페이지는 미동도 하지 않았습니다. react-router에 설정해 둔 라우트 로더(`loader`)도 전혀 호출되지 않았죠. 마치 `navigate` 함수가 유령처럼 실행만 되고 사라지는 것처럼 느껴졌습니다.

### 1차 삽질: `useNavigate`의 잘못된 사용?

처음에는 `react-router-dom`의 버전업에 따른 사용법 변경을 의심했습니다. 혹시 `replace` 함수를 직접 `import`해서 써야 하나? 아니면 `Link` 컴포넌트만 써야 하나? 여러 가설을 세우고 시도했지만 모두 실패였습니다. 코드는 `useNavigate`의 표준적인 사용법을 정확히 따르고 있었습니다.

### 2차 삽질: `useEffect`와의 충돌?

다음 가설은 "상태 변경과의 충돌"이었습니다. 혹시 `navigate`가 호출된 직후 다른 상태(`state`)가 변경되면서 컴포넌트가 리렌더링되고, 이 때문에 페이지 이동이 중단되는 것은 아닐까?

이 가설은 상당히 그럴듯했습니다. 실제로 `navigate`와 상태 변경 함수를 동시에 호출하면 종종 이런 문제가 발생하거든요. 그래서 `setTimeout`으로 `navigate` 호출을 살짝 지연시켜 리렌더링이 끝난 후에 실행되도록 하는 꼼수를 써봤습니다.

```typescript
// 임시 해결책 (하지만 좋은 방법은 아닙니다!)
const handleLogout = () => {
  setTimeout(() => navigate("/logout"), 50);
};
```

놀랍게도, 이 방법은 통했습니다! 페이지가 드디어 이동하기 시작했습니다. 하지만 이건 근본적인 해결책아니고 결국 Hack에 불과했습니다. 왜 이런 일이 발생하는지 정확한 원인을 알고 싶었습니다.

### 진짜 원인: UI 컴포넌트의 "기본 동작"과의 경합 조건 (Race Condition)

문제의 핵심은 `DropdownMenuItem` 컴포넌트의 내부 동작에 있었습니다.

`shadcn/ui`의 `DropdownMenu`는 내부적으로 `Radix UI`를 사용합니다. 이 라이브러리의 `DropdownMenuItem`은 `onSelect` 이벤트가 발생하면 (즉, 사용자가 메뉴를 클릭하면) **두 가지 일**을 하도록 설계되어 있습니다.

1. 우리가 `onSelect`에 전달한 함수(`handleLogout`)를 실행한다.
    
2. **메뉴를 닫는 "기본 동작"을 실행한다.** (내부적으로 `open` 상태를 `false`로 바꾼다.)
    

이 두 가지 동작이 거의 동시에 실행되면서 "경합 조건(Race Condition)"이 발생한 것입니다.

1. `handleLogout`이 호출되고, `navigate` 함수가 페이지 이동 프로세스를 **"시작"** 합니다.
    
2. **하지만 페이지 이동이 완료되기 전에,** `DropdownMenu`가 닫히면서 내부 상태가 변경되고, 이로 인해 `Profile` 컴포넌트가 리렌더링됩니다.
    
3. 이 갑작스러운 리렌더링이, 이제 막 출발하려던 페이지 이동 프로세스를 **"취소"** 시켜버린 것입니다.
    

이것이 바로 콘솔은 찍히지만 페이지 이동은 안 되는, 유령 버그의 정체였습니다.

### 최종 해결책: 라이브러리가 의도한 방법, `asChild`

`setTimeout` 같은 임시방편 대신, 라이브러리가 이런 경우를 위해 마련해 둔 우아한 해결책이 있었습니다. 바로 `asChild` 속성입니다.

```typescript
// 최종 해결책
import { Link } from 'react-router-dom';
import { DropdownMenuItem } from '@/components/ui/DropdownMenu';

// ...

<DropdownMenuItem asChild>
  <Link to="/logout" replace>
    로그아웃
  </Link>
</DropdownMenuItem>
```

`asChild` 속성은 `DropdownMenuItem`에게 "너 자신을 렌더링하지 말고, 네 자식 컴포넌트(`Link`)에게 너의 모든 스타일과 동작을 물려줘"라고 지시합니다.

그 결과, 우리는 `react-router-dom`의 `Link` 컴포넌트를 사용하지만, 겉보기에는 완벽한 `DropdownMenuItem`처럼 보이는 컴포넌트를 얻게 됩니다.

이 방법이 왜 완벽한 해결책일까요?

* **진짜 링크(**`<a>` **태그)로 동작:** 이제 메뉴 아이템은 단순한 버튼이 아니라, 페이지 이동을 위한 시맨틱한 링크가 됩니다. 브라우저는 링크 클릭 시 페이지 이동을 다른 UI 상태 변경보다 우선시하므로 충돌 자체가 발생하지 않습니다.
    
* **불필요한 핸들러 제거:** 더 이상 `navigate`나 `handleLogout` 함수가 필요 없어 코드가 훨씬 간결해집니다.
    
* `replace` **속성 활용:** `Link` 컴포넌트의 `replace` 속성을 사용하면, 로그아웃 페이지가 브라우저 히스토리에 쌓이지 않게 하여 뒤로가기 문제를 깔끔하게 방지할 수 있습니다.
    

### 마무리 회고

이번 삽질을 통해 몇 가지 중요한 교훈을 얻었습니다.

1. `navigate`가 동작하지 않으면, 리렌더링과의 충돌을 가장 먼저 의심하라. 특히 인터랙티브한 UI 컴포넌트 내부에서 `navigate`를 호출할 때는 더욱 그렇습니다.
    
2. **사용하는 라이브러리의 문서를 다시 한번 살펴보자.** `setTimeout` 같은 꼼수가 통하더라도, 분명 더 우아하고 안정적인 "공식적인" 해결책이 존재할 가능성이 높습니다. `asChild`가 바로 그런 케이스였습니다.
    
3. **Headless UI 라이브러리는 편리하지만, 내부 동작 원리를 이해하는 것이 중요하다.** 내부적으로 어떤 상태를 관리하고, 어떤 기본 동작이 실행되는지 알면 이런 디버깅 시간을 크게 줄일 수 있습니다.
    

혹시 저와 비슷한 문제를 겪고 계신 분이 있다면, 이 글이 문제 해결의 실마리가 되기를 바랍니다. 읽어주셔서 감사합니다.