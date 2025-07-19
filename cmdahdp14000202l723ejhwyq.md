---
title: "Node.js 버전으로 인한 키 쌍 조회 실패 디버깅 회고"
seoTitle: "postmortem-fedify-nodejs"
datePublished: Sat Jul 19 2025 16:47:14 GMT+0000 (Coordinated Universal Time)
cuid: cmdahdp14000202l723ejhwyq
slug: postmortem-fedify-nodejs
cover: https://cdn.hashnode.com/res/hashnode/image/upload/v1752944378297/2ddf731e-9bde-4429-b8be-8ada30ec0644.jpeg
ogImage: https://cdn.hashnode.com/res/hashnode/image/upload/v1752944391396/eab6198d-c8f8-4fa2-b3c2-516ba02d57a1.jpeg
tags: nodejs, postmortem, fedify

---

최근 Fedify 라이브러리를 사용해 [ActivityPub 기반 마이크로 블로그를 만드는 튜토리얼](https://hackers.pub/@hongminhee/2025/fedify-tutorial-ko)을 따라하다가, 예상치 못한 버그에 부딪혔어요. federation.ts 파일에서 키 쌍이 제대로 생성되는데도 조회가 실패하는 문제였죠. 이 글은 그 디버깅 과정을 회고하며, 비슷한 함정을 피할 수 있도록 내용을 정리해봅니다. (튜토리얼 링크: [Fedify 튜토리얼 - 테스트 2](https://hackers.pub/@hongminhee/2025/fedify-tutorial-ko#0197b1ac-005b-774b-b75f-03f54699312b--%ED%85%8C%EC%8A%A4%ED%8A%B8-2))

## 문제: 키 쌍이 DB에 있는데 왜 조회가 안 될까?

프로젝트를 진행하던 중, 사용자(johndoe) 조회(lookup) 요청에서 publicKey 필드가 비어 나오는 이슈가 발생했습니다. 결과는 이랬어요:

```json
Person {
  id: URL "http://localhost:8000/users/johndoe",
  name: "johndoe",
  url: URL "http://localhost:8000/users/johndoe",
  preferredUsername: "johndoe",
  inbox: URL "http://localhost:8000/users/johndoe/inbox",
  endpoints: Endpoints { sharedInbox: URL "http://localhost:8000/inbox" }
}
```

SQLite DB의 keys 테이블에는 RSASSA-PKCS1-v1\_5와 Ed25519 키 쌍이 잘 저장되어 있었는데, ctx.getActorKeyPairs(identifier)가 빈 배열을 반환하니 publicKey가 누락됐어요. 처음엔 DB 쿼리나 키 생성 로직이 잘못된 줄 알았습니다. (유저명: johndoe, 로컬 서버: [http://localhost:8000](http://localhost:8000))

## 디버깅 과정: 30분의 추적

fedify 프로젝트 메인테이너이신 [hongminhee](https://hackers.pub/@hongminhee)님과 채팅하며 실시간으로 디버깅했어요. 타임라인은 다음과 같아요:

* **오전 12:38**: 이슈 보고. setKeyPairsDispatcher 구현 공유 (DB 조회 → 키 생성 → pairs 반환).
    
* **오전 12:42**: 로그 추가 조언. return pairs 전에 console.debug({pairs}) 찍어보니 pairs가 제대로 나옴:
    
* ```json
      pairs [
        { privateKey: CryptoKey {...}, publicKey: CryptoKey {...} },  // RSASSA-PKCS1-v1_5
        { privateKey: CryptoKey {...}, publicKey: CryptoKey {...} }   // Ed25519
      ]
    ```
    
* 하지만 요청은 실패. 에러: TypeError: t.buffer.transfer is not a function.
    
* **오전 12:47**: DB 초기화 후 재시도. 로그는 찍히지만 실패. 스택 트레이스 확인
    
* ```bash
      TypeError: t.buffer.transfer is not a function
      at y (file:///.../chunk-JF3YMNGM.js:1:140)
      // ... (byte-encodings와 Fedify 내부 함수 관련)
    ```
    
* **오전 12:59**: Node.js 버전 확인. 내 버전: 20.19.2.
    
* **오전 1:01**: Fedify가 Node 22+를 요구한다는 사실 발견. 업그레이드 시도 (nvm use 22.17.0).
    
* **오전 1:04~1:07**: 초기 업그레이드 실패 (터미널 세션 문제). node -v로 확인 후 재시도.
    
* **오전 1:12**: 성공! 조회 정상 작동.
    

## 근본 원인: 버전 호환성과 가이드 불일치

* **주요 원인**: Node.js 20.19.2에서 ArrayBuffer.prototype.transfer 메서드가 없어 Fedify의 exportSpki 함수가 런타임 에러를 일으킴. Fedify는 Node 22+를 요구하지만, 내 환경은 20이었다.
    
* **기여 요인**:
    
    * 튜토리얼 가이드가 "Node.js 20 버전 이상"으로 20.19.2로 설치되있던 npm버전을 그대로 사용함. 하지만 실제 Fedify 요구사항은 22+라 불일치.
        
    * nvm으로 버전 변경 시 터미널 세션 간 동기화 미스 (변경 후 node -v 확인 안 함).
        
    * 에러가 키 내보내기 단계에서 발생해 초기 증상이 DB 문제로 오해.
        

## 해결: 업그레이드와 재테스트

* Node.js를 22.17.0으로 업그레이드 (nvm install/use 후 터미널 재시작).
    
* npm install 재실행.
    
* DB 초기화 후 조회 재시도 → 성공.
    

## 교훈: 환경 체크부터, 가이드 blind follow 피하기

* **가장 근본 원인: node 버전을 의심하지도 못하고 stack trace 메세지를 제대로 읽지 않아 도움을 요청했던 것**
    
* **버전 관리**: 라이브러리 문서의 **최소 요구 버전**을 공식 사이트에서 확인. 튜토리얼 가이드만 믿지 말고, GitHub README나 package.json의 engines 필드를 살펴보기.
    
* **디버깅 팁**: 로그부터 찍고, 스택 트레이스 키워드로 검색 (e.g., "ArrayBuffer transfer Node.js"). nvm 사용 시 항상 node -v 검증.
    
* **예방**: 프로젝트 시작 전에 체크리스트 (Node 버전, deps 설치). .nvmrc 파일 추가로 자동화.