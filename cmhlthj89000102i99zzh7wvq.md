---
title: "[기술 회고] Radix UI의 함정"
datePublished: Wed Nov 05 2025 09:50:26 GMT+0000 (Coordinated Universal Time)
cuid: cmhlthj89000102i99zzh7wvq
slug: radix-ui
cover: https://cdn.hashnode.com/res/hashnode/image/upload/v1762333140249/c5e84f6f-4825-4745-85ff-0d67a89eab2c.png
ogImage: https://cdn.hashnode.com/res/hashnode/image/upload/v1762333150025/e739dca3-8aaf-4870-ad1d-df43db2ad622.png
tags: reactjs, radixui

---

### 글을 시작하며: 끝나지 않는 삽질의 시작

React와 Radix UI로 복잡한 UI를 개발하다 보면, Dialog(Modal) 내부에 Select 컴포넌트를 넣는 경우는 매우 흔합니다. 저희 팀도 그랬죠. 모든 것이 완벽하게 작동하는 것처럼 보였습니다. 사용자가 Select를 열고, 옵션을 선택하고, 닫는 모든 과정이 매끄러웠습니다.

*단 한 가지,* `ESC` *키를 누르기 전까지는요.*

Select 드롭다운이 열린 상태에서 `ESC` 키를 누르자, 닫혀야 할 것은 Select 드롭다운 하나인데 뜬금없이 부모인 Modal 전체가 닫혀버렸습니다. 게다가 페이지 전체가 먹통이 되는 치명적인 버그까지 발생했습니다.

이 글은 이 미스터리한 버그를 잡기 위한 삽질 기록이자, Radix UI를 사용하는 개발자라면 누구나 빠질 수 있는 '보이지 않는 함정'과 문제 해결 과정에서의 '인지적 편향'에 대한 이야기입니다.

### 시도했던 모든 오답들: 문제의 본질을 꿰뚫지 못했던 이유

저와 제 AI 페어 프로그래머(AKA. 커서 with Sonnet 4.5)는 이 문제를 해결하기 위해 다양한 방법을 시도했습니다.

1. **표면적인 해결책: 이벤트 제어**
    
    * **시도**: `event.stopPropagation()`, `event.preventDefault()`
        
    * **가설**: "부모에게 이벤트가 전파되는 것을 막으면 해결될 것이다."
        
    * **실패 원인**: Radix UI는 `Portal`을 통해 Dialog와 Select를 DOM 트리의 다른 위치에 렌더링합니다. 이 때문에 단순한 이벤트 버블링 차단으로는 둘의 상호작용을 제어할 수 없었습니다. **현상에만 집중**한 나머지, 근본적인 렌더링 구조를 간과했습니다.
        
2. **구조적인 해결책: 상태 관리**
    
    * **시도**: `Select`의 열림 상태(`open`)를 Modal에서 `useState`로 직접 제어.
        
    * **가설**: "React의 데이터 흐름에 따라 상태를 명시적으로 제어하면 완벽하게 동작할 것이다."
        
    * **실패 원인**: Radix UI 내부의 이벤트 처리 로직이 React의 State 업데이트보다 먼저, 그리고 독립적으로 동작하는 것처럼 보였습니다. "버그는 작성한 코드 안에 있다"는 **강력한 전제에 갇혀 있었습니다.**
        
3. **극단적인 해결책: 전역 리스너**
    
    * **시도**: `document` 레벨에서 `keydown` 이벤트를 `capture` 단계에서 가로채기.
        
    * **가설**: "가장 먼저 이벤트를 가로채서 막아버리면 다른 누구도 이벤트를 받지 못할 것이다."
        
    * **실패 원인**: 이 방법은 Radix UI의 내부 포커스 및 이벤트 관리 시스템과 정면으로 충돌하며 예상치 못한 부작용만 낳았습니다.
        

### 결정적 단서의 등장 (그리고 그것을 놓친 이유)

수많은 시도가 실패로 돌아가던 중, 문제의 핵심에 가장 가까운 단서가 등장했습니다. `Modal` 컴포넌트의 타입을 확장하는 과정에서 `onPointerDownOutside` 이벤트의 타입(`PointerDownOutsideEvent`)을 찾지 못하는 에러가 발생했습니다.

이때, AI 어시스턴트는 다음과 같은 제안을 했습니다.

> "이 타입은 `@radix-ui/react-dismissable-layer` 패키지에 있을 수 있습니다. 이 패키지를 직접 설치해서 타입을 가져옵시다."

이게 결정적 단서였습니다. 하지만 저는 이 단서를 완전히 잘못 해석했습니다.

* **AI의 의도**: "타입 에러를 해결하기 위해 의존성을 추가하자."
    
* **저의 반응**: "불필요한 의존성이다. `Dialog`가 이미 타입을 가지고 있을 것이다."
    

**저와 AI 둘 다 이 패키지를 '버전 충돌의 잠재적 용의자'로 보지 못했습니다.** AI는 '타입 소스'로, 저는 '불필요한 의존성'으로만 본 것입니다. `dismissable-layer`라는 이름이 수면 위로 떠 올랐을 때, "잠깐, Dialog와 Select가 이걸 각자 다른 버전으로 쓰고 있는 거 아닐까?"라는 질문을 던졌어야 했습니다. 하지만 저는 이미 '이벤트 핸들링'이라는 프레임에 갇혀 있었고, 이 중요한 단서를 그대로 흘려보내고 말았습니다.

### 진짜 원인: 보이지 않는 의존성 버전 충돌

결국, Radix UI의 GitHub 이슈 트래커에서 진짜 원인을 찾았습니다.

* [Pressing 'esc' while Select is open inside Dialog closes entire Dialog · Issue #1951](https://github.com/radix-ui/primitives/issues/1951)
    
* [\[DismissableLayer\] Layering breaks with different component versions · Issue #1088](https://github.com/radix-ui/primitives/issues/1088)
    

**원인은 코드 로직이 아닌, 패키지 의존성 버전의 미세한 불일치였습니다.**

Radix UI의 `Dialog`, `Select` 등의 컴포넌트들은 내부적으로 `@radix-ui/react-dismissable-layer` 라는 패키지를 공통으로 사용합니다. 이 패키지는 화면에 열린 "레이어"들의 스택을 관리하며, ESC 키나 외부 클릭 시 가장 위의 레이어부터 순서대로 닫는 역할을 합니다.

문제는, 패키지 매니저가 의존성을 설치할 때, `@radix-ui/react-dialog`가 의존하는 `dismissable-layer` 버전과 `@radix-ui/react-select`가 의존하는 `dismissable-layer` 버전이 미세하게 다를 수 있다는 점입니다.

이렇게 되면, 애플리케이션 내에 **두 개의 독립적인** `DismissableLayer` 인스턴스가 공존하게 됩니다. `Dialog`는 자신만의 레이어 스택을, `Select`는 또 다른 자신만의 스택을 갖게 되는 것이죠. `Dialog`의 레이어 관리자는 `Select`가 열렸다는 사실을 전혀 알지 못합니다. 이 상태에서 `ESC` 키를 누르면, `Dialog`는 자신이 최상위 레이어라고 착각하고 스스로를 닫아버리는 것입니다.

### 해결책: 의존성 버전 강제 통일

원인을 알았으니 해결은 간단합니다. 프로젝트 전체에서 `@radix-ui/react-dismissable-layer`가 단 하나의 버전만 사용되도록 강제하면 됩니다.

`package.json` 파일에 `overrides` (pnpm, npm) 또는 `resolutions` (yarn) 설정을 추가합니다.

`package.json` (pnpm 사용 시)

```json
{
  "pnpm": {
    "overrides": {
      "@radix-ui/react-dismissable-layer": "1.0.5" 
    }
  }
}
```

> **팁**: 버전은 `pnpm list @radix-ui/react-dismissable-layer` 명령어로 프로젝트 내에 설치된 버전을 확인하고 가장 최신 버전으로 통일하는 것이 좋습니다.

설정을 추가한 후, 기존 `node_modules`와 lock 파일을 삭제하고 다시 설치합니다.

```bash
rm -rf node_modules pnpm-lock.yaml
pnpm install
```

### 결론 및 교훈

이번 경험은 저에게 몇 가지 중요한 교훈을 남겼습니다.

1. **가장 깊은 곳을 의심하라**: 설명하기 어려운 UI 버그, 특히 라이브러리 간 상호작용에서 문제가 발생하면 코드 로직뿐만 아니라 **의존성 트리를 먼저 의심**해야 합니다.
    
2. **단서를 놓치지 마라**: 문제 해결 과정에서 등장하는 예상 밖의 에러나 키워드(이번 경우엔 `dismissable-layer`)는 때로 문제의 본질을 가리키는 나침반이 될 수 있습니다.
    
3. **인지적 편향을 경계하라**: "버그는 내 코드에 있다"는 생각, 혹은 하나의 해결책에만 매몰되는 '확증 편향'은 문제의 본질을 보지 못하게 만듭니다. 때로는 한 걸음 물러나 전혀 다른 각도에서 문제를 바라볼 필요가 있습니다.
    

이 글이 Radix UI의 복잡한 상호작용으로 고통받는 다른 개발자들에게 작은 도움이 되기를 바랍니다.