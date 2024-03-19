---
title: "웹소켓에 대해 알아보자"
datePublished: Tue Mar 19 2024 06:20:23 GMT+0000 (Coordinated Universal Time)
cuid: cltxzjpqu000g09l6el6t1t00
slug: websocket-in-react
cover: https://cdn.hashnode.com/res/hashnode/image/stock/unsplash/67FG6zD8WPQ/upload/0f3d82257a47a5fb3b8292e77da29ff3.jpeg
tags: reactjs, websocket

---

웹소켓(WebSocket)은 클라이언트와 서버 간에 실시간, 양방향, 지속적인 데이터 통신을 가능하게 하는 기술입니다. HTTP 프로토콜과 달리, 웹소켓은 한 번 연결이 맺어지면 그 연결을 유지하면서 데이터를 빠르게 주고 받을 수 있습니다.

## 장점 and 단점

### **장점**

1. **양방향 통신**: 클라이언트와 서버의 연결이 유지되는 동안 언제든 서로에게 데이터를 주고 받을 수 있습니다.
    
2. **낮은 지연 시간**: 연결이 되어있는 상태이기 때문에 데이터 교환 시 서로를 검증하는 시간이 없어서 HTTP 요청보다 지연 시간이 낮습니다.
    
3. **효율적인 자원 사용**: 한번의 연결 설정만으로 데이터를 주고 받을 수 있기 때문에 HTTP와 다르게 연결을 설정할 때 마다 사용되는 컴퓨팅 자원들을 아낄 수 있습니다.
    

### 단점

1. **호환성 문제**: 구형 브라우저나 일부 환경에서는 웹소켓을 지원하지 않을 수 있습니다.
    
2. **스케일링 이슈:** 많은 수의 연결을 관리해야 하는 경우, 서버의 부하 및 아키텍처 설계가 복잡해질 수 있습니다.
    
    * 예를 들어 실시간 채팅 어플리케이션을 구현할 경우 자동 재연결을 보장하거나, 메세지 순서를 보장하거나, 누락된 메세지를 재전송하도록 처리해야하는 등 고려해야 할 사항들이 많아집니다.
        

## 리액트에서 웹소켓의 기본 사용법

* 보통 `useEffect` 훅 내에서 웹소켓 연결을 설정하고, 연결을 종료하는 로직을 작성합니다.
    
* ```javascript
    import React, { useEffect } from 'react';
    
    function ChatComponent() {
        useEffect(() => {
            const ws = new WebSocket('wss://sample.com/chat');
            ws.onopen = () => {
                console.log('Connected');
            }
            ws.onmessage = (event) => {
                console.log('Message from sever ', event.data);
            }
    
            // 언마운트 시 연결 종료 처리
            return () => {
                ws.close();
            }
        }, []); // 마운트 시 1회만 실행
    
        return <div> Chat </div>;
    }
    ```
    

## 리액트에서 웹소켓 사용 시 주의사항

1. 재연결 로직: 네트워크 불안정, 모바일 브라우저의 사용성 등으로 인해 연결이 끊길 수 있습니다. 이를 위해 자동 재연결 로직을 구현하는 것이 좋습니다.
    
2. 메모리 누수 방지: 웹소켓 이벤트 리스너 등록시에는 컴포넌트 언마운트 시 이를 제거해야 메모리 누수를 방지할 수 있습니다.
    
3. 보안 프로토콜 사용: `wss://` 프로토콜을 사용하여 데이터 전송 시 암호화 되도록 처리해주는 편이 좋습니다.
    

## 주의사항을 반영한 예시

```javascript
import React, { useEffect, useRef } from 'react';

function ImprovedChatComponent() {
    const wsRef = useRef(null);

    useEffect(() => {
        const setWebSocket = () => {
            const ws = new WebSocket('wss://sample.com/chat');
            ws.onopen = () => {
                console.log('Connected');
            };
            ws.onclose = () => {
                console.log('Disconnected. Attempting to reconnect...');
                setTimeout(setWebSocket, 3000); // 연결 끊기면 3초뒤 재시도
            };

            // useRef를 사용하여 웹소켓 인스턴스 저장
            wsRef.current = ws;
        }
        
        setWebSocket();
        
        // 언마운트 시 웹소켓 연결 종료
        return () => {
            if (wsRef.current) {
                wsRef.current.close();
            }
        }
    }, []);

    return <div> Chat </div>;
}
```

`onclose` 이벤트 핸들러 내에서 연결이 끊어졌을 때 자동으로 재연결을 시도하는 로직을 구현했습니다.  
`useRef`를 사용하여 웹소켓 인스턴스를 저장하고 있는데, 이렇게 함으로서 컴포넌트 생명주기에 맞춰 적절하게 웹소켓 연결과 종료를 할 수 있어서 자원 낭비를 막을 수 있습니다.

## 마치며

운영하는 서비스에서 웹소켓 관련 코드를 수정하다보니 웹소켓에 대해 잘 모르고 있다는 점을 반성하며 내용을 정리하는 글을 써봤습니다. 웹소켓의 가장 어려운 부분은 여러가지 연결에 대한 설계일 것이라고 판단되지만, 클라이언트 단에서도 리액트 컴포넌트의 생명주기에 맞춰서 세심하게 설정해줘야 한다는 점을 알 수 있었습니다.