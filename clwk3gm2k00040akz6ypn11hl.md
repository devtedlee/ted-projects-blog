---
title: "[번역] React 스케쥴러는 어떻게 동작하나요?"
datePublished: Fri May 24 2024 03:00:18 GMT+0000 (Coordinated Universal Time)
cuid: clwk3gm2k00040akz6ypn11hl
slug: react-internals-deep-dive-20
cover: https://cdn.hashnode.com/res/hashnode/image/upload/v1716453470843/611d1397-4b9a-4760-9255-0d4742c4570f.jpeg
tags: reactjs, scheduler, react-internals

---

> ***영문 블로그 글을 번역했습니다. 허가를 받으면 시리즈를 이어갈 예정입니다.  
> 원문링크:***[https://jser.dev/react/2022/03/16/how-react-scheduler-works](https://jser.dev/react/2022/03/16/how-react-scheduler-works)

---

> ***ℹ️***[***React Internals Deep Dive***](https://jser.dev/series/react-source-code-walkthrough.html)***에피소드 20,***[***유튜브에서 제가 설명하는 것***](https://www.youtube.com/watch?v=FLrzXQ0_u6Y&list=PLvx8w9g4qv_p-OS-XdbB3Ux_6DMXhAJC3&index=20)***을 시청해주세요.***
> 
> ***⚠***[***React@18.2.0***](https://github.com/facebook/react/releases/tag/v18.2.0)***기준, 최신 버전에서는 구현이 변경되었을 수 있습니다.***
> 
> ***💬 역자 주석: JSer의 코멘트는 ❗❗로 표시 해뒀습니다.  
> 그 외 주석은 리액트 소스 코드 자체의 주석입니다.  
> ... 은 생략된 코드입니다.***

## 1\. 리액트 스케쥴러가 필요한 이유

[이 시리즈의 첫 번째 에피소드](https://www.youtube.com/watch?v=OcB3rTln-fI&list=PLvx8w9g4qv_p-OS-XdbB3Ux_6DMXhAJC3&index=1)에서 이미 다룬 바 있는 다음 코드([소스](https://github.com/facebook/react/blob/555ece0cd14779abd5a1fc50f71625f9ada42bef/packages/react-reconciler/src/ReactFiberWorkLoop.js#L2119))부터 시작하겠습니다.

```typescript
function workLoopSync() {
  while (workInProgress !== null) {
    performUnitOfWork(workInProgress);
  }
}
```

한마디로 React는 내부적으로 파이버 트리의 각 파이버에서 작동하며, `workInProgress`는 현재 위치를 추적하는 것이고, 순회 알고리즘은 [이전 포스트](https://ted-projects.com/react-internals-deep-dive-15)에서 이미 설명했습니다.

`workLoopSync()`는 동기식이기 때문에 작업을 중단할 수 없으므로 React는 잠시 동안 루프 내에서 계속 작업하기만 하면 됩니다.

동시 모드에서는 상황이 달라집니다([소스](https://github.com/facebook/react/blob/555ece0cd14779abd5a1fc50f71625f9ada42bef/packages/react-reconciler/src/ReactFiberWorkLoop.js#L2296)).

```typescript
function workLoopConcurrent() {
  while (workInProgress !== null && !shouldYield()) {
    performUnitOfWork(workInProgress);
  }
}
```

동시 모드에서는, 우선순위가 높은 작업이 우선순위가 낮은 작업을 중단할 수 있으므로 작업을 중단하고 다시 시작할 수 있는 방법이 필요하며, 이를 위해 `shouldYield()` 가 트릭을 수행하지만, 분명히 그 이상의 기능이 있습니다.

## 2\. 먼저 몇 가지 배경지식에서부터 시작하겠습니다

### 2.1 이벤트 루프

솔직히 설명이 잘 안 되니 [javascript.info에서 설명](https://javascript.info/event-loop)을 읽어보시거나 [Jake Archibald의 멋진 동영상](https://twitter.com/jaffathecake/status/961980260194684928)을 시청하시기 바랍니다.

간단히 말해, 자바스크립트 엔진은 다음과 같은 작업을 수행합니다.

1. 태스크 큐(Task Queue)에서 작업(매크로 작업)을 가져와 실행합니다.
    
2. 예약된 마이크로 태스크가 있으면, 실행합니다.
    
3. 렌더링이 필요한지 확인하고 수행합니다.
    
4. 작업이 더 있으면 1을 반복하거나 더 많은 작업을 기다립니다.
    

실제로 일종의 루프가 있기 때문에 `loop`라는 용어는 매우 명확합니다.

### 2.2 렌더링을 차단하지 않고 새 작업을 예약하기 위해 settImmediate()를 사용

렌더링을 차단하지 않고 일부 작업을 예약하기 위해(위의 3번째 단계), 우리는 이미 `setTimeout(callback, 0)`의 트릭에 익숙해져 있으며, 이는 새로운 매크로 작업을 예약합니다.

이벤트 개선 API인 [setImmediate()](https://developer.mozilla.org/en-US/docs/Web/API/Window/setImmediate)가 있지만 IE와 node.js에서만 사용할 수 있습니다.

`setTimeout()`은 [중첩 호출에서 실제로 최소 약 4ms의 지연](https://javascript.info/settimeout-setinterval)이 있는 반면, `setImmediate()`는 지연이 없으므로 더 좋습니다.

이제 React Scheduler([소스](https://github.com/facebook/react/blob/e62a8d754548a490c2a3bcff3b420e5eedaf11c0/packages/scheduler/src/forks/Scheduler.js#L550))의 첫 번째 코드를 만질 준비가 되었습니다.

```typescript
let schedulePerformWorkUntilDeadline;
if (typeof localSetImmediate === "function") { // ❗❗ 
  // Node.js and old IE.
  // There's a few reasons for why we prefer setImmediate.
  //
  // Unlike MessageChannel, it doesn't prevent a Node.js process from exiting.
  // (Even though this is a DOM fork of the Scheduler, you could get here
  // with a mix of Node.js 15+, which has a MessageChannel, and jsdom.)
  // https://github.com/facebook/react/issues/20756
  //
  // But also, it runs earlier which is the semantic we want.
  // If other browsers ever implement it, it's better to use it.
  // Although both of these would be inferior to native scheduling.
  schedulePerformWorkUntilDeadline = () => {
    localSetImmediate(performWorkUntilDeadline);
  };
} else if (typeof MessageChannel !== "undefined") { // ❗❗ 
  // DOM and Worker environments.
  // We prefer MessageChannel because of the 4ms setTimeout clamping.
  const channel = new MessageChannel();
  const port = channel.port2;
  channel.port1.onmessage = performWorkUntilDeadline;
  schedulePerformWorkUntilDeadline = () => {
    port.postMessage(null);
  };
} else { // ❗❗ 
  // We should only fallback here in non-browser environments.
  schedulePerformWorkUntilDeadline = () => {
    localSetTimeout(performWorkUntilDeadline, 0);
  };
}
```

여기에서는 두 가지 다른 `setImmediate()`의 폴백(fallback)을 볼 수 있는데, MessageChannel과 setTimeout이 있습니다.

### 2.3 Priority Queue

[우선순위 큐](https://en.wikipedia.org/wiki/Priority_queue)는 스케줄링을 위한 일반적인 데이터 구조입니다. [직접 자바스크립트로 우선순위 큐를 직접 만들어](https://bigfrontend.dev/problem/create-a-priority-queue-in-JavaScript) 보시기 바랍니다.

이는 React의 요구사항에 완벽하게 부합합니다. 우선순위가 다른 이벤트가 들어오기 때문에 처리할 우선순위가 가장 높은 이벤트를 빠르게 찾아야 합니다.

React는 min-heap으로 우선순위 큐를 구현하며, 소스 코드는 [여기](https://github.com/facebook/react/blob/e62a8d754548a490c2a3bcff3b420e5eedaf11c0/packages/scheduler/src/SchedulerMinHeap.js)에서 확인할 수 있습니다.

## 3\. workLoopConcurrent의 콜 스택

이제, `workLoopConcurrent`가 어떻게 호출되는지 한번 보도록 하죠.

[![](https://jser.dev/static/scheduler-1.png align="left")](https://jser.dev/static/scheduler-1.png)

모든 코드는 [ReactFiberWorkLoop.js](https://github.com/facebook/react/blob/e62a8d754548a490c2a3bcff3b420e5eedaf11c0/packages/react-reconciler/src/ReactFiberWorkLoop.old.js)에 있으며, 이를 분석해 보겠습니다.

우리는 `ensureRootIsScheduled()`를 여러 번 만났고, 꽤 많은 곳에서 사용하고 있습니다. 이름에서 알 수 있듯이 `ensureRootIsScheduled()`는 업데이트가 있는 경우 React가 작업을 수행하도록 예약합니다.

`performConcurrentWorkOnRoot()`를 직접 호출하지 않고 `scheduleCallback(priority, callback)`을 통해 콜백으로 처리한다는 점에 유의하세요. `scheduleCallback()`은 [스케줄러](https://github.com/facebook/react/blob/e62a8d754548a490c2a3bcff3b420e5eedaf11c0/packages/scheduler/src/forks/Scheduler.js)의 API입니다.

곧 스케줄러에 대해 자세히 살펴보겠지만, 지금은 스케줄러가 적절한 시간에 작업을 실행한다는 점만 기억하세요.

### 3.1 performConcurrentWorkOnRoot()는 중단된 경우 자체 클로저를 반환합니다.

진행 상황에 따라 performConcurrentWorkOnRoot()가 다르게 반환 되는 것을 보셨나요?

1. `shouldYield()`가 참이면 workLoopConcurrent가 중단되어 불완전한 `update(RootInComplete)`가 발생하고, `performConcurrentWorkOnRoot()`는 `performConcurrentWorkOnRoot.bind(null, root)`를 반환합니다. [(코드](https://github.com/facebook/react/blob/555ece0cd14779abd5a1fc50f71625f9ada42bef/packages/react-reconciler/src/ReactFiberWorkLoop.js#L1167))
    
2. 완료되면, null 을 반환합니다.
    

작업이 `shouldYield()`에 의해 중단된 경우 어떻게 다시 시작될 수 있는지 궁금할 수 있습니다. 네, 이것이 답변입니다. **스케줄러는 작업 콜백의 반환값을 보고 작업이 계속되는지 확인하며**, 반환 값은 일종의 리스케쥴링입니다. 이에 대해서는 곧 다룰 예정입니다.

## 4\. Scheduler

마지막으로, 스케줄러의 영역에 들어섰습니다. 처음에는 겁이 났지만 곧 불필요하다는 것을 깨달았으니 부담스러워하지 마세요.

메시지 큐(Message Queue)는 제어권을 전달하는 방법이고, 스케줄러는 정확히 이와 같은 역할을 합니다.

위에서 언급한 `scheduleCallback()`은 스케줄러 세계에서 [unstable\_scheduleCallback](https://github.com/facebook/react/blob/e62a8d754548a490c2a3bcff3b420e5eedaf11c0/packages/scheduler/src/forks/Scheduler.js#L308)입니다.

### 4.1 scheduleCallback() - 스케쥴러는 expirationTime으로 작업들을 예약합니다.

스케줄러가 작업을 예약하려면, 먼저 우선순위와 함께 작업을 저장해야 합니다. 이는 이미 배경 지식으로 다룬 우선순위 큐를 통해 이루어집니다.

`expirationTime`을 사용하여 우선 순위를 나타냅니다. **만료 시간이 빠르면 빠를수록 더 빨리 처리해야** 하므로 공정합니다. 다음은 작업이 생성되는 `scheduleCallback()` 내부의 코드입니다.

```typescript
var currentTime = getCurrentTime();
var startTime;
if (typeof options === "object" && options !== null) {
  var delay = options.delay;
  if (typeof delay === "number" && delay > 0) {
    startTime = currentTime + delay;
  } else {
    startTime = currentTime;
  }
} else {
  startTime = currentTime;
}
var timeout;
switch (priorityLevel) {
  case ImmediatePriority:
    timeout = IMMEDIATE_PRIORITY_TIMEOUT;
    break;
  case UserBlockingPriority:
    timeout = USER_BLOCKING_PRIORITY_TIMEOUT;
    break;
  case IdlePriority:
    timeout = IDLE_PRIORITY_TIMEOUT;
    break;
  case LowPriority:
    timeout = LOW_PRIORITY_TIMEOUT;
    break;
  case NormalPriority:
  default:
    timeout = NORMAL_PRIORITY_TIMEOUT;
    break;
}
var expirationTime = startTime + timeout;
var newTask = { // ❗❗ 
  id: taskIdCounter++,
  callback,
  priorityLevel,
  startTime,
  expirationTime,
  sortIndex: -1,
};
// ❗❗ task는 스케줄러가 처리하는 작업의 단위입니다.
```

코드는 매우 간단하며, 각 우선순위에 따라 다른 시간 제한이 있으며 [여기](https://github.com/facebook/react/blob/e62a8d754548a490c2a3bcff3b420e5eedaf11c0/packages/scheduler/src/forks/Scheduler.js#L63)에 정의되어 있습니다.

```typescript
// Times out immediately
var IMMEDIATE_PRIORITY_TIMEOUT = -1;
// Eventually times out
var USER_BLOCKING_PRIORITY_TIMEOUT = 250;
var NORMAL_PRIORITY_TIMEOUT = 5000;
    // ❗❗                     ↗ 기본값은 5초 타임아웃 입니다.
var LOW_PRIORITY_TIMEOUT = 10000;
// Never times out
var IDLE_PRIORITY_TIMEOUT = maxSigned31BitInt;
```

따라서 기본적으로 5초의 시간 제한이 설정되어 있으며 사용자 차단에 대해서는 250ms가 설정되어 있습니다. 곧 이러한 우선순위에 대한 몇 가지 예를 살펴보겠습니다.

작업이 생성되었으니 이제 우선순위 큐에 넣을 차례입니다.

```typescript
if (startTime > currentTime) {
  // This is a delayed task.
  newTask.sortIndex = startTime;
  push(timerQueue, newTask); // ❗❗
  if (peek(taskQueue) === null && newTask === peek(timerQueue)) {
    // All tasks are delayed, and this is the task with the earliest delay.
    if (isHostTimeoutScheduled) {
      // Cancel an existing timeout.
      cancelHostTimeout();
    } else {
      isHostTimeoutScheduled = true;
    }
    // Schedule a timeout.
    requestHostTimeout(handleTimeout, startTime - currentTime);
  }
} else {
  newTask.sortIndex = expirationTime;
  push(taskQueue, newTask); // ❗❗
  // Schedule a host callback, if needed. If we're already performing work,
  // wait until the next time we yield.
  if (!isHostCallbackScheduled && !isPerformingWork) {
    isHostCallbackScheduled = true;
    requestHostCallback(flushWork); // ❗❗
  }
}
```

아 맞다, 작업을 예약할 때 `setTimeout()` 과 같은 지연 옵션이 있을 수 있습니다. 이 부분은 따로 보관해 두었다가 나중에 다시 살펴보겠습니다.

`else` 브랜치에만 집중하세요. 두 가지 중요한 호출을 볼 수 있습니다.

1. `push(taskQueue, newTask)` - 큐에 작업을 추가합니다. 이것은 우선순위 큐 API일 뿐이므로 그냥 건너뛰겠습니다.
    
2. `requestHostcallback(flushWork)` - 작업들을 처리합니다!
    

`requestHostCallback(flushWork)`는 필수인데, 왜냐하면 스케줄러는 호스트에 구애받지 않고 모든 호스트에서 실행될 수 있는 독립적인 블랙박스에 불과하므로, *요청해야* 합니다.

### 4.2 `requestHostCallback()`

```typescript
function requestHostCallback(callback) {
  scheduledHostCallback = callback;
  if (!isMessageLoopRunning) {
    isMessageLoopRunning = true;
    schedulePerformWorkUntilDeadline(); // ❗❗
  }
}
const performWorkUntilDeadline = () => {
  if (scheduledHostCallback !== null) {
    const currentTime = getCurrentTime();
    // Keep track of the start time so we can measure how long the main thread
    // has been blocked.
    startTime = currentTime;
    const hasTimeRemaining = true;
    // If a scheduler task throws, exit the current browser task so the
    // error can be observed.
    //
    // Intentionally not using a try-catch, since that makes some debugging
    // techniques harder. Instead, if `scheduledHostCallback` errors, then
    // `hasMoreWork` will remain true, and we'll continue the work loop.
    let hasMoreWork = true; // <❗❗
    try {
      hasMoreWork = scheduledHostCallback(hasTimeRemaining, currentTime);
    } finally {
      if (hasMoreWork) {
        // If there's more work, schedule the next message event at the end
        // of the preceding one.
        schedulePerformWorkUntilDeadline(); // ❗❗/>
        // ❗❗ ↗ 스케줄러가 큐에 있는 작업들을 계속해서 처리하는 것을 볼 수 있습니다.
        // ❗❗ 여기가 브라우저에서 페인트(paint)를 할 수 있는 기회를 제공하는 곳입니다.
      } else { // <❗❗
        isMessageLoopRunning = false;
        scheduledHostCallback = null;
      }
    } // ❗❗/>
  } else {
    isMessageLoopRunning = false;
  }
  // Yielding to the browser will give it a chance to paint, so we can
  // reset this.
  needsPaint = false;
};
```

2.2에서 언급했듯이 `schedulePerformWorkUntilDeadline()`은 `performWorkUntilDeadline()`의 래퍼일 뿐입니다.

`scheduledHostCallback`은 `requestHostCallback()`에서 설정되고 `performWorkUntilDeadline()`에서 바로 호출되는데, 이는 비동기 특성 때문에 메인 스레드가 렌더링할 기회를 주기 위한 것입니다.

몇 가지 세부 사항은 무시하고 가장 중요한 대목을 소개합니다.

```typescript
hasMoreWork = scheduledHostCallback(hasTimeRemaining, currentTime)
```

즉, `flushWork()`는 `(true, currentTime`)과 함께 호출됩니다.

> 왜 여기에 true로 하드코딩되어 있는지 모르겠습니다. 아마도 리팩토링 실수 때문일 수 있습니다.

### 4.3 flushWork()

```typescript
try {
  // No catch in prod code path.
  return workLoop(hasTimeRemaining, initialTime);
} finally {
  //
}
```

flushWork 는 `workLoop()`를 감쌌을 뿐입니다.

### 4.4 workLoop() - 스케쥴러의 핵심

조정의 `workLoopConcurrent()` 와 마찬가지로 스케줄러의 핵심은 `workLoop()`입니다. 프로세스가 비슷하기 때문에 이름이 비슷합니다.

```typescript
if (
  currentTask.expirationTime > currentTime &&
  //                    (               )
  (!hasTimeRemaining || shouldYieldToHost())
) {
  // This currentTask hasn't expired, and we've reached the deadline.
  break;
}
```

`workLoopConcurrent()`와 마찬가지로, 여기서도 `shouldYieldToHost()`를 확인합니다. 이 부분은 나중에 다루겠습니다.

```typescript
const callback = currentTask.callback;
if (typeof callback === "function") {
  currentTask.callback = null;
  currentPriorityLevel = currentTask.priorityLevel;
  const didUserCallbackTimeout = currentTask.expirationTime <= currentTime;
  const continuationCallback = callback(didUserCallbackTimeout); // ❗❗ 
  currentTime = getCurrentTime();
  if (typeof continuationCallback === "function") { // ❗❗ 
    // ❗❗ ↗ 작업의 반환값이 중요한 이유는 다음과 같습니다.
    // ❗❗ 유의하세요, 이 브랜치에서는 작업이 팝업되지 않습니다!
    currentTask.callback = continuationCallback;
  } else {
    if (currentTask === peek(taskQueue)) {
      pop(taskQueue);
    }
  }
  advanceTimers(currentTime);
} else {
  pop(taskQueue);
}
```

자세히 살펴보겠습니다.

`currentTask.callback`, 이 경우 실제로는 `performConcurrentWorkOnRoot()`입니다.

```typescript
const didUserCallbackTimeout  = currentTask.expirationTime <= currentTime;
const continuationCallback  = callback(didUserCallbackTimeout);
```

만료 여부를 나타내는 플래그와 함께 호출됩니다.

타임아웃이 발생하면 `performConcurrentWorkOnRoot()` 가 동기화 모드로 돌아갑니다. ([코드](https://github.com/facebook/react/blob/555ece0cd14779abd5a1fc50f71625f9ada42bef/packages/react-reconciler/src/ReactFiberWorkLoop.js#L1053-1059))즉, 이제부터는 어떤 중단도 없어야 합니다.

```typescript
const shouldTimeSlice =
  !includesBlockingLane(root, lanes) &&
  !includesExpiredLane(root, lanes) &&
  (disableSchedulerTimeoutInWorkLoop || !didTimeout);
let exitStatus = shouldTimeSlice
  ? renderRootConcurrent(root, lanes)
  : renderRootSync(root, lanes);
```

좋습니다, 이제 `workLoop()`로 돌아가죠

```typescript
if (typeof continuationCallback === "function") {
  currentTask.callback = continuationCallback;
} else {
  if (currentTask === peek(taskQueue)) {
    pop(taskQueue);
  }
}
```

여기서 중요한 점은 **콜백의 반환값이 함수가 아닐 때만 태스크가 팝업된다는 것**입니다. 함수인 경우 태스크의 콜백이 팝업되지 않으므로 다음 번에 workLoop()를 호출하면 동일한 태스크가 다시 발생합니다.

즉, **이 콜백의 반환값이 함수인 경우 이 작업이 완료되지 않았으므로 다시 작업해야** 합니다.

```typescript
advanceTimers(currentTime)
```

이것은 지연된 작업인데, 나중에 다시 돌아와서 보겠습니다.

### 4.5 `shouldYield()`는 어떻게 동작하나요?

### [소스](https://github.com/facebook/react/blob/555ece0cd14779abd5a1fc50f71625f9ada42bef/packages/scheduler/src/forks/Scheduler.js#L487)

```typescript
function shouldYieldToHost() {
  const timeElapsed = getCurrentTime() - startTime;
  if (timeElapsed < frameInterval) {
    // The main thread has only been blocked for a really short amount of time;
    // smaller than a single frame. Don't yield yet.
    return false;
  }
  // The main thread has been blocked for a non-negligible amount of time. We
  // may want to yield control of the main thread, so the browser can perform
  // high priority tasks. The main ones are painting and user input. If there's
  // a pending paint or a pending input, then we should yield. But if there's
  // neither, then we can yield less often while remaining responsive. We'll
  // eventually yield regardless, since there could be a pending paint that
  // wasn't accompanied by a call to `requestPaint`, or other main thread tasks
  // like network events.
  /** 💬 주석 번역
메인 스레드가 무시할 수 없는(non-neligible) 시간 동안 차단되었습니다.
브라우저가 우선순위가 높은 작업을 수행할 수 있도록 메인 스레드에 대한 제어권을 
양도(yield)할 수 있습니다. 주요 작업은 페인팅과 사용자 입력입니다. 
보류 중인 페인트나 보류 중인 입력이 있으면 양도해야 합니다. 
하지만 둘 다 없다면 응답성을 유지하면서 양도하는 빈도를 줄일 수 있습니다. 
요청 페인트 호출이 수반되지 않은 보류 중인 페인트나 
네트워크 이벤트와 같은 다른 메인 스레드 작업이 있을 수 있기 때문에 
결국에는 양도할 것입니다.
  */
  if (enableIsInputPending) {
    if (needsPaint) {
      // There's a pending paint (signaled by `requestPaint`). Yield now.
      // 💬 보류 중인 페인트가 있습니다('requestPain'로 신호). 지금 양도하세요.
      return true;
    }
    if (timeElapsed < continuousInputInterval) {
      // We haven't blocked the thread for that long. Only yield if there's a
      // pending discrete input (e.g. click). It's OK if there's pending
      // continuous input (e.g. mouseover).
/** 💬 주석 번역
    그렇게 오랫동안 스레드를 차단한 적은 없습니다. 
    보류 중인 개별 입력(예: 클릭)이 있는 경우에만 양도하세요.
    기 중인 연속 입력이 있어도 괜찮습니다.(예: 마우스오버)
*/
      if (isInputPending !== null) {
        return isInputPending();
      }
    } else if (timeElapsed < maxInterval) {
      // Yield if there's either a pending discrete or continuous input.
      // 💬 보류 중인 불연속형 또는 연속형 입력이 있는 경우 양도합니다.
      if (isInputPending !== null) {
        return isInputPending(continuousOptions);
      }
    } else {
      // We've blocked the thread for a long time. Even if there's no pending
      // input, there may be some other scheduled work that we don't know about,
      // like a network event. Yield now.
/** 💬 주석 번역
  오랫동안 스레드를 차단했습니다. 보류 중인 입력이 없더라도 
  네트워크 이벤트와 같이 저희가 모르는 다른 예정된 작업이 있을 수 있습니다. 
  지금 양도하세요.
*/
      return true;
    }
  }
  // `isInputPending` isn't available. Yield now.
  // 💬 `isInputPending`을 사용할 수 없습니다. 지금 양도하세요.
  return true;
}
```

사실 복잡하지 않고 주석에 모든 것이 설명되어 있습니다. 가장 기본적인 라인은 다음과 같습니다.

```typescript
const timeElapsed = getCurrentTime() - startTime;
if (timeElapsed < frameInterval) {
  // The main thread has only been blocked for a really short amount of time;
  // smaller than a single frame. Don't yield yet.
  return false;
}
return true;
```

따라서 각 작업에는 5ms(`frameInterval`)가 주어지며, 시간이 다 되면 양도해야 합니다.

이것은 스케줄러에서 `task`를 실행하기 위한 것이지 각 `performUnitOfWork()`에 대한 것이 아니라는 점에 유의하세요. `startTime`은 `performWorkUntilDeadline()`에서만 설정되므로 각 `flushWork()`에 대해 재설정되며, **여러 작업이** `flushWork()`**에서 처리될 수 있는 경우 그 사이에는 양도가 없다는 것**을 알 수 있습니다.

이것은 아래의 리액트 퀴즈를 이해하는 데 도움이 될 것입니다.

💬 퀴즈는 직접 JSer의 블로그 글에서 풀어보세요! [링크](https://jser.dev/react/2022/03/16/how-react-scheduler-works#45-how-shouldyield-work)

## 5\. 요약

휴, 이건 많았네요. 전체 다이어그램을 그려 보겠습니다.

[![](https://jser.dev/static/scheduler-2.png align="left")](https://jser.dev/static/scheduler-2.png)

아직 몇 가지 누락된 부분이 있지만 큰 진전이 있었습니다. React 내부를 더 잘 이해하는 데 도움이 되었기를 바랍니다. 이미 소화하기에는 너무 큰 다이어그램이니, 다른 내용은 다음 에피소드에서 다루도록 하겠습니다.

(원본 게시일: 2022-03-16)