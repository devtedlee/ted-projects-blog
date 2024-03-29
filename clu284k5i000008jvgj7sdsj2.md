---
title: "유닛 테스트 네이밍 규칙에 대해 알아보자"
datePublished: Fri Mar 22 2024 05:31:38 GMT+0000 (Coordinated Universal Time)
cuid: clu284k5i000008jvgj7sdsj2
slug: unit-testing-naming-convention
cover: https://cdn.hashnode.com/res/hashnode/image/stock/unsplash/tpAyLp9Ro50/upload/1af062a6241e6ad4caac03fa65cec6f8.jpeg
tags: unit-testing, namingconvention

---

테스트에 대한 세미나에 참여하며 유닛 테스트를 작성할 때 네이밍 규칙에 따라 많은 것이 달라질 수 있을거라는 생각이 들어서 어떤 네이밍 규칙이 있는지 조사해보고, 그 중 어떤 것을 선택해볼지 고민하고 정리하는 글을 써봤습니다.

워낙 다양한 유닛테스트 네이밍 기법들이 있어서 다 정리하는건 복잡도가 너무 늘어나게 되는 것 같아서 제 나름의 기준으로 네 가지를 대표로 소개해보겠습니다.

[더 많은 네이밍 규칙 참고 블로그](https://it-is-mine.tistory.com/3)

## 네이밍 규칙 5가지 소개

### 1) (메서드명)\_(상태나 조건)\_(예상 결과)

* 이 방식은 테스트하려는 '메서드나 함수의 이름', '테스트 상태 또는 입력값', 그리고 '예상되는 결과'를 네이밍에 사용합니다. 테스트의 목적을 명확하게 전달하고, 어떤 상황에서 어떤 결과가 기대되는지 파악하기가 용이합니다.
    
* 장점: 테스트 의도가 명확하고 직관적, 특정 상황에서의 기대 결과 파악 용이
    
* 단점: 메서드명이나 조건이 복잡하면 네이밍 길어짐, 리팩토링 시 테스트 이름도 변경해야함
    
    ```javascript
    test('add_두양수_양수반환', () => {
      expect(add(10, 5)).toBe(15);
    });
    ```
    

### 2) Should\_(예상행동)\_When\_(시나리오)

* '시나리오'에서 '예상되는 행동'을 강조하여 네이밍 하는 구조를 갖고 있습니다. 테스트의 의도를 자연스러운 언어로 표현하는데 유용합니다.
    
* 장점: 테스트 의도와 시나리오 표현이 명확함, 자연어에 가까운 표현 사용 가능
    
* 단점: 네이밍 길어질 수 있음, 간결함을 잃기 쉬움
    
    ```javascript
    test('Should_결과가Null_When_나누는수가0', () => {
        expect(divide(10, 0).toBeNull();
    });
    ```
    

### 3) (조건)\_(행동)\_(기대 결과)

* '특정 조건' 에서 수행될 '행동'과 그에 따른 '기대 결과'를 구조로 네이밍 합니다. 테스트 상황과 행동, 결과를 모두 포함함으로써 전체적인 테스트 시나리오를 명확하게 설명해줍니다.
    
* 장점: 종합적인 정보(상황, 행동, 결과) 제공 가능. 테스트 케이스를 통해 시나리오를 명확히 이해할 수 있음
    
* 단점: 복잡한 테스트 시나리오가 이름을 길게 함. 포함해야 될 내용이 많아서 간결성이 떨어짐.
    
    ```javascript
    test('나누는수가0_나누기_오류반환', () => {
        expect(()=> divide(10, 0)).toThrow('나누는 수는 0이 될 수 없습니다');
    });
    ```
    

### 4) (테스트 기능)\_(행동에 대한 설명)

* 테스트하려는 '기능'과 그 기능의 '특정 행동에 대한 설명'을 포함하는 네이밍 구조를 사용합니다. 테스트 대상의 기능과 기능에서 수행되는 작업의 세부사항을 명확히 해줍니다.
    
* 장점: 기능 중심의 네이밍으로 테스트 대상 범위가 명확함. 기능의 행동에 대한 설명이 있어서 구현을 이해하기 좋음
    
* 단점: 특정 조건이나 시나리오를 간과하기 쉬움. 행동에 대한 설명이 구체적이지 않을수록 더 헷갈리게 될 수 있음.
    
    ```javascript
    test('로그인_유효한인증정보_사용자인증성공', () => {
        expect(login('사용자명', '비밀번호')).toBe('인증 성공');
    });
    ```
    

### 5) (유저스토리)\_(행동)\_(예상결과)

* 사용자 스토리와 관련된 행동 및 예상 결과를 포함하여 네이밍하는 구조입니다. 유저 스토리 기반의 네이밍이라 특히 애자일 개발 방법론과 잘 호환됩니다.
    
* 장점: 사용자 관점에서 테스트의 목적을 연결하기 좋습니다. 실제 사용 사례와 밀접히 연결되어 사용자 경험(UX)개선에 도움이 됩니다.
    
* 단점: 사용자 스토리가 구체적이거나 복잡한 경우 네이밍이 길어집니다. 모든 유닛 테스트를 사용자 스토리에 매핑하기 어렵습니다.
    
    ```javascript
    test('사용자가로그인할때_유효한인증정보제공_로그인성공메세지표시', () => {
        expect(login('사용자명', '비밀번호').toBe('로그인 성공');
    });
    ```
    

## 어떤 네이밍 규칙을 사용할까?

* 위의 네이밍 규칙들은 개발 커뮤니티 내에서도 어떤 네이밍을 사용하자는 베스트 프렉티스는 정해지지 않았습니다. 일을 하는 개발 팀 내부에서도 일하는 스타일(애자일 등), 팀의 가치관, KPI등에 따라 우선순위가 달라질 수 있기 때문입니다.
    
* 저의 경우 유저 스토리 방식과 BDD(행동 주도 개발, Behavior Driven Development)를 선호하는 편이라 BDD에 잘맞는 GivenWhenThen 방식이나 유저스토리 방식 중에 선택을 고민하게 됐습니다.
    
* 제가 일하는 팀에서 유저 스토리를 다루고 있지도 않고, 두 방식 중 작은 팀에게는 GIven\_When\_Then 방식이 더 적절하다는 판단 하에 이 방식으로 진행해보기로 선택했습니다.
    
* 결국 이러한 선택은 팀의 작업 방식, 프로젝트의 성격, 그 외 CI/CD등 인프라 팀의 협업 등 다양한 환경이 얽히기 때문에 잘 논의해서 선택하는 편이 좋다고 생각합니다.