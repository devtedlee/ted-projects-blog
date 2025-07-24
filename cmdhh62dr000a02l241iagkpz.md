---
title: "Nginx 프록시 설정 문제 해결기"
datePublished: Thu Jul 24 2025 14:15:41 GMT+0000 (Coordinated Universal Time)
cuid: cmdhh62dr000a02l241iagkpz
slug: retrospect-front-server-nginx
cover: https://cdn.hashnode.com/res/hashnode/image/upload/v1753365020153/18253f91-0a00-4f82-ad77-042160e14912.jpeg
ogImage: https://cdn.hashnode.com/res/hashnode/image/upload/v1753366531198/d8e297cb-32af-4289-9529-cef647dc5bd0.jpeg
tags: frontend, nginx

---

최근 QA 환경에서 API 호출이 제대로 되지 않는 문제를 겪었습니다. 겉보기에는 단순해 보였던 이 문제는 Nginx 프록시 설정의 여러 복잡한 요소들이 얽혀 있는 흥미로운 케이스였습니다. 문제 해결 과정에서 배운 것들을 정리해 보았습니다.

## 시작: 단순해 보였던 405 에러

QA 환경에서 다음과 같은 API 호출이 실패하고 있었습니다:

```typescript
POST https://qa.leadershipcube.ai/api/assessment/v1/login
→ 405 Method Not Allowed
```

첫 번째 가설은 "백엔드 서버가 POST 메서드를 지원하지 않나?"였습니다. 하지만 곧 더 깊은 문제가 숨어있다는 것을 깨달았습니다.

## 🔍 디버깅의 시작: curl로 추적하기

문제를 정확히 파악하기 위해 `curl`을 사용해 백엔드 서버에 직접 요청을 보내봤습니다:

```bash
# 백엔드 서버 직접 호출
curl -X POST https://gw.qa.hunet.io/assessment/v1/login \
  -H "Content-Type: application/json" \
  -H "x-hunet-franchise-type: 271" \
  -d '{"email": "test@example.com", "employeeCode": "Test"}'
```

**결과: 200 OK ✅**

이상했습니다. 백엔드는 정상적으로 동작하는데, 프록시를 거치면 왜 405 에러가 나는지 알 수 없었죠.

## 첫 번째 발견: Host 헤더 문제

Nginx 프록시 설정을 자세히 살펴보니 문제를 발견했습니다:

```nginx
# 기존 설정
location ~ ^/api/assessment {
  rewrite ^/api/assessment(.*) /assessment$1 break;
  proxy_pass https://api_backend;
  proxy_set_header Host $host;  # 여기가 문제!
}
```

`proxy_set_header Host $host;` 설정 때문에 Nginx가 브라우저의 원래 `Host` 헤더(`qa.xxx.ai`)를 백엔드로 그대로 전달하고 있었습니다.

하지만 백엔드 서버는 자신의 주소(`gw.qa.xxx.io`)가 `Host` 헤더에 있어야만 요청을 처리하도록 설정되어 있었습니다.

### 해결책 1: 올바른 Host 헤더 설정

```nginx
location /api/assessment/ {
  proxy_pass https://api_backend/assessment/;
  proxy_set_header Host gw.qa.xxx.io;  # 백엔드가 기대하는 Host로 명시
}
```

## 두 번째 문제: 301 리다이렉션 지옥

Host 헤더 문제를 해결하고 나니, 이번에는 `/api/auth` 엔드포인트에서 다른 문제가 발생했습니다:

```typescript
PUT https://qa.xxx.ai/api/auth
→ 301 Moved Permanently
→ Location: https://qa.xxx.ai/api/auth/
```

### 원인: 슬래시(/) 불일치

```nginx
# 문제가 되는 설정
location /api/auth/ {  # 슬래시로 끝남
  proxy_pass ...;
}
```

클라이언트는 `/api/auth` (슬래시 없음)으로 요청했는데, Nginx 설정은 `/api/auth/` (슬래시 있음)로 되어 있어 Nginx가 "친절하게" 리다이렉션을 보낸 것이었습니다.

### 해결책 2: 정규식으로 유연하게 처리

```nginx
# 시도 1: 정규식 사용
location ~ ^/api/auth/?$ {
  proxy_pass https://api_backend/auth/;
}

# 시도 2: rewrite 사용
location /api/auth {
  rewrite ^/api/auth(.*)$ /auth$1 break;
  proxy_pass https://api_backend;
}
```

## 로컬 테스트 환경 구축

매번 QA 서버에 배포해서 테스트하는 것은 비효율적이었습니다. Mac북에서 사용할 수 있는 도커 cli 중에 podman을 사용해 로컬 테스트 환경을 구축했습니다.

```bash
podman run --rm -d --name local-nginx-test \
  -p 8080:80 \
  -v ./build/qa/default.conf:/etc/nginx/conf.d/default.conf:ro \
  -v ./public:/usr/share/nginx/html:ro \
  docker.io/nginx:latest
```

### 예상치 못한 문제들

로컬 환경에서 여러 예상치 못한 문제들을 만났습니다:

1. **DNS resolver 충돌**: 로컬 컨테이너가 사설 DNS 서버에 접근하지 못해 발생
    
2. **Nginx 설정 문법 오류**: `set` 지시어를 잘못된 위치에 사용
    
3. **Podman 네트워킹 문제**: 포트 포워딩이 예상대로 동작하지 않음
    

각각을 하나씩 해결해 나가면서, 로컬에서 실제 QA 환경과 동일한 동작을 재현할 수 있게 되었습니다.

## 마지막 발견: HTTP 메서드 문제

모든 설정을 올바르게 고쳤는데도 여전히 405 에러가 발생했습니다. 마지막에 알고 보니

```bash
# 실패하는 요청
curl -X POST /api/auth ...
→ 405 Method Not Allowed

# 성공하는 요청  
curl -X PUT /api/auth ...
→ 200 OK ✅
```

토큰 갱신 API는 `POST`가 아니라 `PUT` 메서드를 사용해야 했습니다. 실수도 연속해서 저지르다보면 인지도 못하는 쉬운 실수가 생기더라구요.

## 최종 해결책: 단순함의 승리

모든 문제를 해결한 후, 복잡했던 설정을 더 간단하고 명확하게 리팩토링했습니다:

```nginx
# 최종 설정 - 간단하고 명확함
server {
  set $backend_host "gw.qa.xxx.io";
  
  location /api/assessment/ {
    proxy_pass https://api_backend/assessment/;
    proxy_set_header Host $backend_host;
    # ... 기타 헤더들
  }
  
  location /api/auth {
    proxy_pass https://api_backend/auth;
    proxy_set_header Host $backend_host;
    # ... 기타 헤더들
  }
}
```

## 배운 점들

### 1\. 체계적인 디버깅의 중요성

* **가설 세우기**: 각 단계에서 명확한 가설을 세우고 검증
    
* **격리된 테스트**: `curl`을 사용해 Nginx를 우회한 직접 테스트
    
* **로컬 재현**: 로컬 환경에서 문제를 재현해 빠른 반복 테스트
    

### 2\. Nginx의 미묘한 동작들

* **Host 헤더의 중요성**: 가상 호스팅 환경에서 Host 헤더가 라우팅에 미치는 영향
    
* **경로 매칭의 복잡성**: 슬래시 유무에 따른 리다이렉션 동작
    
* **upstream vs 변수**: `proxy_pass`에서 upstream과 변수 사용의 차이점
    

### 3\. 도구의 활용

* **curl**: API 테스트의 강력한 도구
    
* **Podman/Docker**: 로컬 테스트 환경 구축
    
* **로그 분석**: 에러 메시지에서 단서 찾기
    

## 결론

겉보기에 단순해 보였던 405 에러 하나가 Host 헤더, 경로 매칭, HTTP 메서드, DNS 설정 등 여러 복잡한 요소들이 얽힌 문제였습니다.

이 과정을 통해 Nginx 프록시의 동작 원리를 이전보다 깊이 이해하게 되었고, 무엇보다 **체계적인 디버깅과 로컬 테스트 환경의 중요성**을 다시 한번 깨달았습니다.

가끔 복잡한 해결책보다는 단순하고 명확한 설정이 더 나은 선택일 수 있다는 것도 좋은 교훈이었습니다.