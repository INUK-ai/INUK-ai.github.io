---
title: 운영체제 #1 - 프로세스와 스레드
date: 2025-04-18 +09:00
categories: [AIBE, CS]
tags: [os, process, thread, 운영체제]
---

### 프로세스(Process)

> 실행 중인 프로그램을 의미

#### 프로세스의 메모리 구조

- **Code 영역** : 실행할 코드가 저장되는 곳
- **Data 영역** : 전역 변수, 정적 변수가 저장되는 곳
- **Stack 영역** : 지역 변수, 함수 호출 정보가 저장되는 곳
- **Heap 영역** : 동적 할당된 메모리가 저장되는 곳

#### 특징

- 각 프로세스는 독립적인 메모리 공간을 가짐
- 한 프로세스가 다른 프로세스의 데이터에 직접 접근 불가능
- 프로세스 간 통신을 위해서는 IPC(Inter-Process Communication) 필요

```bash
# 크롬 브라우저 예시
Google Chrome
├── 메인 프로세스
├── 탭1 프로세스  
├── 탭2 프로세스
└── 확장프로그램 프로세스
```

### 스레드(Thread)

> 프로세스 내에서 실행되는 작업 단위

#### 스레드의 메모리 구조

- **Code, Data, Heap 영역** : 같은 프로세스 내 스레드들과 공유
- **Stack 영역** : 각 스레드마다 독립적으로 할당

#### Node.js 예시

```javascript
// 메인 스레드
console.log('서버 시작');

// 비동기 파일 읽기 (별도 스레드에서 처리)
fs.readFile('large-file.txt', (err, data) => {
    console.log('파일 읽기 완료');
});

// 메인 스레드는 계속 다른 작업 수행
console.log('다른 작업 진행');
```

### 프로세스 vs 스레드 비교

| 구분 | 프로세스 | 스레드 |
|------|----------|--------|
| **메모리** | 독립적 공간 | 공유 |
| **통신** | IPC 필요 | 직접 접근 |
| **생성 비용** | 높음 | 낮음 |
| **안정성** | 높음 | 낮음 |
| **전환 비용** | 높음 | 낮음 |

### 멀티프로세싱 vs 멀티스레딩

**멀티프로세싱이 유리한 경우**
- CPU 집약적 작업
- 안정성이 중요한 경우

```python
from multiprocessing import Pool

def cpu_intensive_task(n):
    return sum(i * i for i in range(n))

if __name__ == '__main__':
    with Pool() as pool:
        results = pool.map(cpu_intensive_task, [1000000, 1000000])
```

**멀티스레딩이 유리한 경우**
- I/O 집약적 작업
- 효율성이 중요한 경우

```python
import threading
import requests

def fetch_url(url):
    response = requests.get(url)
    print(f'{url}: {response.status_code}')

threads = []
urls = ['http://example.com', 'http://google.com']

for url in urls:
    thread = threading.Thread(target=fetch_url, args=(url,))
    threads.append(thread)
    thread.start()

for thread in threads:
    thread.join()
```

### 주의사항

#### Race Condition

```javascript
// 문제가 있는 코드
let counter = 0;

function incrementCounter() {
    counter = counter + 1; // 여러 스레드가 동시 접근 시 문제
}
```

#### 해결 방법

```javascript
// Worker Threads 사용
const { Worker, isMainThread, parentPort } = require('worker_threads');

if (isMainThread) {
    const worker = new Worker(__filename);
    worker.postMessage('increment');
} else {
    parentPort.on('message', (msg) => {
        if (msg === 'increment') {
            parentPort.postMessage(counter++);
        }
    });
}
```