---
title: "클로저로 구현하는 디자인 패턴"
datePublished: Sun Mar 17 2024 10:28:36 GMT+0000 (Coordinated Universal Time)
cuid: cltvdj7x3000009ld0vog63cn
slug: closure-and-design-pattern
cover: https://cdn.hashnode.com/res/hashnode/image/stock/unsplash/WwCTFNpZx8g/upload/a3b087441cfb57057370c8538009317a.jpeg
tags: design-patterns, closure, javascript

---

클로저는 주변 상태(어휘적 환경)에 대한 참조와 함께 묶인(포함된) 함수의 조합입니다([mdn 참고](https://developer.mozilla.org/ko/docs/Web/JavaScript/Closures)). 클로저를 통한 설계를 연습도 할 겸, 클로저와 호환이 잘 되는 네 개의 디자인 패턴을 소개하고, 예시 코드를 작성해보겠습니다. 예시 코드들은 간단한 사용자 설정 관리 시스템을 구현하는 코드들로 작성 해보겠습니다.

## 모듈 패턴(Module Pattern)

* 모듈 패턴은 특정 기능을 캡슐화하고, 공개 인터페이스만을 외부에 노출하는 패턴입니다. 여기서 캡슐화는 객체의 상태를 나타내는 프로퍼티와 프로퍼티를 참조하고 조작할 수 있는 동작인 메서드를 하나로 묶는 것을 말합니다.
    
* 모듈 패턴은 클로저를 사용하기 적합한 디자인 패턴 중 하나입니다. 클로저 자체가 상태를 안전하게 변경하고 유지하기 위해 활용되기 때문입니다.
    
    ```javascript
    const userSettingsModule = (function () {
        let settings = {
            theme: 'dark',
            language: 'ko'
        };
    
        return {
            getSetting: function (key) {
                return settings[key];
            },
            setSetting: function (key, value) {
                settings[key] = value;
            }
        }
    })();
    
    userSettingsModule.getSetting('theme'); // 'dark'
    userSettingsModule.setSetting('theme', 'light');
    userSettingsModule.getSetting('theme'); // 'light'
    ```
    
    이 예시에서 `settings` 객체는 모듈 내부에서만 접근 가능하고, `getSetting`과 `setSetting` 메서드를 통해서만 조작할 수 있습니다. 클로저를 통해 `settings`의 상태가 안전하게 캡슐화됩니다.
    

## 팩토리 패턴(Factory Pattern)

* 팩토리 패턴은 객체를 생성하는 인터페이스를 정의하고, 인스턴스화는 서브클래스에서 수행하게 하는 패턴입니다.
    
* 클로저를 사용하여 특정 인터페이스를 만족하는 객체를 동적으로 생성하고 반환할 수 있습니다. 클로저를 통해 생성된 객체 각각이 자신만의 독립적인 렉시컬 환경(=정적 환경)을 가질 수 있어서 객체마다 다른 상태를 유지할 수 있습니다.
    
    * 참고: 렉시컬 환경은 실행할 스코프 범위 안에 있는 변수와 함수를 프로퍼티로 저장하는 객체입니다.
        
* ```javascript
    function settingsFactory() {
        return {
            theme: 'dark',
            language: 'ko',
            changeSetting: function (key, value) {
                this[key] = value;
            }
        };
    }
    
    const userSettings = settingsFactory();
    userSettings.theme; // 'dark'
    userSettings.changeSetting('theme', 'light');
    userSettings.theme; // 'light'
    ```
    
    이 예시에서 `settingsFactory` 함수는 각 사용자 설정 객체를 생성합니다. 생성된 객체는 독립적인 설정 값을 유지하며, 클로저를 통해 각 객체의 상태를 관리합니다.
    

## 전략 패턴(Strategy Pattern)

* 전략 패턴은 여러 알고리즘을 정의하고, 런타임에 알고리즘을 선택할 수 있게 해주는 패턴입니다.
    
* 각 전략을 클로저로 구현하여 특정 컨텍스트에서 다양한 알고리즘을 쉽게 교체할 수 있습니다.
    
* ```javascript
    function themeStrategy(theme) {
        const themes = {
            dark: function () {
                console.log('apply dark theme');
            },
            light: function () {
                console.log('apply light theme');
            }
        };
    
        return themes[theme] || themes.dark;
    }
    
    const currentTheme = themeStrategy('light');
    currentTheme(); // apply light theme
    ```
    
    전략 패턴에서는 `themeStrategy` 함수가 여러 테마 적용 전략 중 하나를 선택해 반환 합니다. 반환된 함수(전략)는 클로저를 통해 자신이 호출될 때 필요한 정보(여기서는 콘솔 로그 메시지)를 "기억"합니다.
    

## 명령 패턴(Command Pattern)

* 명령 패턴은 요청을 객체로 캡슐화하여, 사용자가 보낸 요청을 다양한 요청 종류와 파라미터와 함께 저장, 실행, 취소할 수 있게 해주는 패턴입니다.
    
* 명령을 클로저로 캡슐화하여, 실행할 명령과 함께 모든 필요한 정보를 보관할 수 있습니다. 이를 통해 명령의 실행을 지연시키거나, 취소, 재시도 하는 등의 작업을 유연하게 처리할 수 있습니다.
    
* ```javascript
    function createChangeSettingCommand(settings, key, value) {
        const previous = settings[key];
        return {
            execute: function() {
                settings[key] = value;
            },
            undo: function() {
                settings[key] = previous;
            }
        };
    }
    
    const settings = {
        theme: 'light',
        language: 'ko'
    };
    
    const changeThemeCommand = createChangeSettingCommand(settings, 'theme', 'dark');
    changeThemeCommand.execute();
    settings.theme; // 'dark'
    changeThemeCommand.undo();
    settings.theme; // 'light'
    ```
    
    명령 패턴에서 `createChangeSettingCommand` 함수는 설정을 변경하는 명령 객체를 생성합니다. 이 객체는 `execute`와 `undo` 메서드를 통해 설정 변경 작업을 수행하고 되돌릴 수 있습니다. 클로저를 통해 이전 상태(`previous`)를 기억하고, 필요할 때 원상태로 되돌릴 수 있습니다.
    

## 마치며

이처럼 클로저는 각 패턴의 중요한 구성 요소로서, 상태를 안전하게 캡슐화 하고, 필요한 데이터와 함수를 "기억"하는 데 도움을 줍니다. 이 네 가지 패턴은 클로저와의 호환성이 좋아서 프론트엔드 개발에서 널리 사용되며, 특히 자바스크립트 같은 언어의 기능을 활용할 때 유용합니다. 프론트엔드 코드를 설계 하시거나, 리팩터링 할 때 이 블로그 글이 도움이 됐으면 좋겠습니다.