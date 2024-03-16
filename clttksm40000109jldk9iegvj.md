---
title: "Javascript resize 이벤트 구현 팁"
datePublished: Sat Mar 16 2024 04:16:20 GMT+0000 (Coordinated Universal Time)
cuid: clttksm40000109jldk9iegvj
slug: javascript-resize-event
cover: https://cdn.hashnode.com/res/hashnode/image/stock/unsplash/EB6LSbp8x4w/upload/5911708597731c2f299b635974b6a9ea.jpeg
tags: javascript

---

홈페이지 개발을 하다보면 resize 이벤트를 자주 구현하게 되는데요. resize 이벤트를 구현할 때 아무래도 이벤트의 발생 빈도가 잦다보니 화면의 깜빡임이나, 성능이 느려지는 현상이 가끔 관찰 됩니다. 이런 문제를 좀 더 깔끔하게 해결할 방법을 조사하고 정리해보겠습니다.

resize 이벤트는 윈도우 크기가 변경될 때마다 발생하는데, 이는 사용자가 브라우저 창의 크기를 드래그 형식으로 조정하거나 하면 매우 빈번하게 발생할 수 있습니다. 이러한 이벤트의 발생은 위에서 언급한 것처럼 화면의 사용성을 헤치거나 성능 문제를 일으킬 수 있습니다. 이를 위해 적절한 최적화 방법을 고려해보는 것이 중요합니다.

## requestAnimationFrame(rAF) 사용

* `requestAnimationFrame`은 브라우저가 다음 repaint를 수행하기 전에 지정된 콜백을 실행할 수 있도록 예약해 주는 방식입니다. 이를 활용하면 만약 resize 이벤트가 너무 자주 발생하더라도 실제로 화면을 다시 그리는 작업은 브라우저의 프레임 속도에 맞춰 효율적으로 수행됩니다. 즉, 불필요한 계산을 줄이고, 부드러운 사용자 경험을 유지하는데 도움을 줄 수 있습니다.
    
* **예시**
    
    ```javascript
    let rafId = null;
    
    function onResize() {
        cancelAnimationFrame(rafId);
        rafId = requestAnimationFrame(() => {
            // 이벤트 처리 로직 작성
        });
    }
    
    window.addEventListener('resize', onResize);
    ```
    

## 디바운스 활용

* 이벤트가 발생한 후 일정 시간 동안 추가적인 이벤트 호출이 없을 때만 이벤트 처리 함수를 실행하는 방식입니다. 마지막 이벤트 발생 후 일정 시간이 지나야 실행되므로, 빈번한 이벤트 발생 시 불필요한 처리를 줄일 수 있습니다.
    
* **예시**
    
    ```javascript
    function debounce(func, wait = 500) {
        let timeout;
        
        return function (...args) {
            const later = () => {
                clearTimeout(timeout);
                func(...args);
            };
    
            clearTimeout(timeout);
            timeout = setTimeout(later, wait);
        }
    }
    
    function onResize() {
        // 이벤트 처리 로직
    }
    
    const debouncedResizeHandler = debounce(onResize, 250);
    window.addEventListener('resize', debouncedResizeHandler); 
    ```
    

## 스로틀 활용

* 정해진 시간 간격으로 이벤트 처리 함수를 실행하는 방식입니다. 이벤트가 아무리 빈번하게 발생해도, 이 간격을 기준으로 함수가 실행되어 처리량을 제한할 수 있습니다.
    
* **예시**
    
    ```javascript
    function throttle(func, limit = 500) {
        let inThrottle;
        return function() {
            const args = arguments;
            const context = this;
            if (!inThrottle) {
                func.apply(context, args);
                inThrottle = true;
                setTimeout(() => inThrottle =false, limit);
            }
        }
    }
    
    function onResize() {
        // 이벤트 처리 로직
    }
    
    const throttledResizeHandler = throttle(onResize, 250);
    window.addEventListener('resize', throttledResizeHandler ); 
    ```
    

## 마치며

크게 세가지 방식의 resize 이벤트 최적화 방식을 살펴봤습니다. 상황에 따라 혹은 사용 사례에 따라 최적화 방식을 선택해야 합니다. 예를 들어, 화면의 크기가 변할 때마다 즉시 반응해야 하는 인터렉티브한 요소가 있으면 `raf`를 사용합니다. 사용자가 크기 조정을 멈춘 후에 어떤 동작을 실행해야 한다면 디바운스, 일정 간격으로 반응을 제한하고 싶다면 스로틀을 활용할 수 있습니다.  
제 글을 통해 소개시켜드린 방식이 머리 한켠에 잘 있다가 필요할 때 떠올리실 수 있게 되길 바라며 글을 마칩니다.