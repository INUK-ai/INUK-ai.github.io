---
title: 운영체제 #4 - 동기화와 교착상태
date: 2025-04-21 +09:00
categories: [AIBE, CS]
tags: [os, synchronization, deadlock, 운영체제, 동기화]
---

### 동기화란?

> 여러 프로세스나 스레드가 공유 자원에 안전하게 접근하도록 순서를 조정하는 것

#### 동기화가 필요한 이유

- **공유 자원** : 여러 프로세스가 동일한 자원 사용
- **경쟁 상태** : 동시 접근 시 예상치 못한 결과 발생
- **데이터 일관성** : 정확한 결과 보장 필요

### 임계 영역 (Critical Section)

> 공유 자원에 접근하는 코드 영역

```python
# 임계 영역 문제 예시
balance = 1000

def withdraw(amount):
    global balance
    # 임계 영역 시작
    temp = balance
    temp = temp - amount
    balance = temp
    # 임계 영역 종료

# 두 스레드가 동시에 실행되면?
# Thread 1: withdraw(100)
# Thread 2: withdraw(200)
# 예상: balance = 700
# 실제: balance = 800 또는 900 (경쟁 상태 발생)
```

#### 임계 영역 해결 조건

1. **상호 배제** : 한 번에 하나의 프로세스만 임계 영역 진입
2. **진행** : 임계 영역에 아무도 없다면 진입 허용
3. **한정 대기** : 무한정 대기하지 않음

### 동기화 기법

#### 뮤텍스 (Mutex)

> 상호 배제를 위한 가장 기본적인 동기화 기법

```python
import threading

class BankAccount:
    def __init__(self, initial_balance):
        self.balance = initial_balance
        self.lock = threading.Lock()  # 뮤텍스
    
    def withdraw(self, amount):
        with self.lock:  # 임계 영역 보호
            if self.balance >= amount:
                temp = self.balance
                # 다른 작업 시뮬레이션
                import time
                time.sleep(0.1)
                self.balance = temp - amount
                return True
            return False
    
    def deposit(self, amount):
        with self.lock:
            temp = self.balance
            time.sleep(0.1)
            self.balance = temp + amount

# 사용 예시
account = BankAccount(1000)

def make_withdrawal():
    result = account.withdraw(100)
    print(f"출금 결과: {result}, 잔액: {account.balance}")

# 여러 스레드에서 동시 출금
threads = []
for i in range(5):
    thread = threading.Thread(target=make_withdrawal)
    threads.append(thread)
    thread.start()

for thread in threads:
    thread.join()
```

#### 세마포어 (Semaphore)

> 정해진 개수만큼의 자원에 대한 접근 제어

```python
import threading
import time

class DatabasePool:
    def __init__(self, max_connections):
        self.semaphore = threading.Semaphore(max_connections)
        self.connections = []
    
    def get_connection(self):
        self.semaphore.acquire()  # 자원 획득
        print(f"연결 획득: {threading.current_thread().name}")
        return "DB Connection"
    
    def release_connection(self, connection):
        print(f"연결 해제: {threading.current_thread().name}")
        self.semaphore.release()  # 자원 해제

def worker(db_pool, worker_id):
    conn = db_pool.get_connection()
    # 데이터베이스 작업 시뮬레이션
    time.sleep(2)
    db_pool.release_connection(conn)

# 최대 3개 연결만 허용하는 DB 풀
db_pool = DatabasePool(3)

# 10개의 워커가 동시에 DB 접근 시도
threads = []
for i in range(10):
    thread = threading.Thread(target=worker, args=(db_pool, i), name=f"Worker-{i}")
    threads.append(thread)
    thread.start()

for thread in threads:
    thread.join()
```

#### 조건 변수 (Condition Variable)

> 특정 조건이 만족될 때까지 대기

```python
import threading
import queue

class ProducerConsumer:
    def __init__(self, max_size):
        self.buffer = queue.Queue(maxsize=max_size)
        self.condition = threading.Condition()
        self.running = True
    
    def producer(self, producer_id):
        item_count = 0
        while self.running:
            with self.condition:
                # 버퍼가 가득 찰 때까지 기다림
                while self.buffer.full() and self.running:
                    print(f"Producer {producer_id}: 버퍼 가득참, 대기 중...")
                    self.condition.wait()
                
                if not self.running:
                    break
                
                item = f"item_{producer_id}_{item_count}"
                self.buffer.put(item)
                item_count += 1
                print(f"Producer {producer_id}: {item} 생산")
                
                # 소비자에게 알림
                self.condition.notify_all()
            
            time.sleep(0.5)
    
    def consumer(self, consumer_id):
        while self.running:
            with self.condition:
                # 버퍼가 비어있을 때까지 기다림
                while self.buffer.empty() and self.running:
                    print(f"Consumer {consumer_id}: 버퍼 비어있음, 대기 중...")
                    self.condition.wait()
                
                if not self.running:
                    break
                
                item = self.buffer.get()
                print(f"Consumer {consumer_id}: {item} 소비")
                
                # 생산자에게 알림
                self.condition.notify_all()
            
            time.sleep(1)

# 사용 예시
pc = ProducerConsumer(5)

# 생산자 2개, 소비자 3개 생성
threads = []
for i in range(2):
    thread = threading.Thread(target=pc.producer, args=(i,))
    threads.append(thread)
    thread.start()

for i in range(3):
    thread = threading.Thread(target=pc.consumer, args=(i,))
    threads.append(thread)
    thread.start()

# 10초 후 종료
time.sleep(10)
pc.running = False

for thread in threads:
    thread.join()
```

#### 모니터 (Monitor)

> 클래스나 모듈 수준에서 동기화 제공

```python
class Monitor:
    def __init__(self):
        self._lock = threading.RLock()  # 재진입 가능한 락
        self._conditions = {}
    
    def __enter__(self):
        self._lock.acquire()
        return self
    
    def __exit__(self, exc_type, exc_val, exc_tb):
        self._lock.release()
    
    def wait(self, condition_name):
        if condition_name not in self._conditions:
            self._conditions[condition_name] = threading.Condition(self._lock)
        self._conditions[condition_name].wait()
    
    def signal(self, condition_name):
        if condition_name in self._conditions:
            self._conditions[condition_name].notify()
    
    def signal_all(self, condition_name):
        if condition_name in self._conditions:
            self._conditions[condition_name].notify_all()

class BoundedBuffer:
    def __init__(self, size):
        self.buffer = []
        self.size = size
        self.monitor = Monitor()
    
    def put(self, item):
        with self.monitor:
            while len(self.buffer) >= self.size:
                self.monitor.wait('not_full')
            
            self.buffer.append(item)
            self.monitor.signal('not_empty')
    
    def get(self):
        with self.monitor:
            while len(self.buffer) == 0:
                self.monitor.wait('not_empty')
            
            item = self.buffer.pop(0)
            self.monitor.signal('not_full')
            return item
```

### 교착상태 (Deadlock)

> 두 개 이상의 프로세스가 서로의 자원을 기다리며 무한히 대기하는 상태

#### 교착상태 발생 조건

1. **상호 배제** : 자원을 한 번에 하나의 프로세스만 사용
2. **점유 대기** : 자원을 보유한 채 다른 자원 대기
3. **비선점** : 자원을 강제로 빼앗을 수 없음
4. **순환 대기** : 프로세스들이 순환적으로 자원 대기

#### 교착상태 예시

```python
import threading
import time

# 교착상태 발생 예시
lock1 = threading.Lock()
lock2 = threading.Lock()

def worker1():
    print("Worker1: lock1 획득 시도")
    lock1.acquire()
    print("Worker1: lock1 획득 완료")
    
    time.sleep(1)  # 다른 워커가 lock2를 획득할 시간을 줌
    
    print("Worker1: lock2 획득 시도")
    lock2.acquire()  # 여기서 무한 대기
    print("Worker1: lock2 획득 완료")
    
    lock2.release()
    lock1.release()

def worker2():
    print("Worker2: lock2 획득 시도")
    lock2.acquire()
    print("Worker2: lock2 획득 완료")
    
    time.sleep(1)
    
    print("Worker2: lock1 획득 시도")
    lock1.acquire()  # 여기서 무한 대기
    print("Worker2: lock1 획득 완료")
    
    lock1.release()
    lock2.release()

# 교착상태 발생!
# t1 = threading.Thread(target=worker1)
# t2 = threading.Thread(target=worker2)
# t1.start()
# t2.start()
```

### 교착상태 해결 방법

#### 1. 예방 (Prevention)

```python
# 자원 순서 정하기로 순환 대기 방지
def safe_worker1():
    # 항상 lock1 → lock2 순서로 획득
    with lock1:
        print("Worker1: lock1 획득")
        time.sleep(1)
        with lock2:
            print("Worker1: lock2 획득")
            # 작업 수행

def safe_worker2():
    # 동일한 순서로 자원 획득
    with lock1:
        print("Worker2: lock1 획득")
        time.sleep(1)
        with lock2:
            print("Worker2: lock2 획득")
            # 작업 수행
```

#### 2. 회피 (Avoidance) - 은행원 알고리즘

```python
class BankersAlgorithm:
    def __init__(self, processes, resources, allocation, max_need):
        self.processes = processes
        self.resources = resources
        self.allocation = allocation
        self.max_need = max_need
        self.available = [resources[i] - sum(allocation[j][i] 
                         for j in range(processes)) 
                         for i in range(len(resources))]
    
    def is_safe_state(self):
        work = self.available[:]
        finish = [False] * self.processes
        safe_sequence = []
        
        for _ in range(self.processes):
            found = False
            for i in range(self.processes):
                if not finish[i]:
                    need = [self.max_need[i][j] - self.allocation[i][j] 
                           for j in range(len(self.resources))]
                    
                    # 자원이 충분한지 확인
                    if all(need[j] <= work[j] for j in range(len(work))):
                        # 프로세스 완료 시뮬레이션
                        for j in range(len(work)):
                            work[j] += self.allocation[i][j]
                        finish[i] = True
                        safe_sequence.append(i)
                        found = True
                        break
            
            if not found:
                return False, []
        
        return True, safe_sequence
    
    def request_resource(self, process_id, request):
        # 요청이 가능한지 확인
        need = [self.max_need[process_id][j] - self.allocation[process_id][j] 
               for j in range(len(self.resources))]
        
        if not all(request[j] <= need[j] for j in range(len(request))):
            return False, "요청이 최대 필요량 초과"
        
        if not all(request[j] <= self.available[j] for j in range(len(request))):
            return False, "사용 가능한 자원 부족"
        
        # 가상으로 할당해보기
        for j in range(len(request)):
            self.available[j] -= request[j]
            self.allocation[process_id][j] += request[j]
        
        # 안전 상태인지 확인
        is_safe, sequence = self.is_safe_state()
        
        if is_safe:
            return True, f"안전 시퀀스: {sequence}"
        else:
            # 롤백
            for j in range(len(request)):
                self.available[j] += request[j]
                self.allocation[process_id][j] -= request[j]
            return False, "불안전 상태로 인한 거부"

# 사용 예시
banker = BankersAlgorithm(
    processes=3,
    resources=[10, 5, 7],
    allocation=[[0, 1, 0], [2, 0, 0], [3, 0, 2]],
    max_need=[[7, 5, 3], [3, 2, 2], [9, 0, 2]]
)

success, message = banker.request_resource(1, [1, 0, 2])
print(f"자원 요청 결과: {message}")
```

#### 3. 탐지 및 복구 (Detection & Recovery)

```python
class DeadlockDetector:
    def __init__(self):
        self.wait_for_graph = {}  # 대기 그래프
    
    def add_wait_relation(self, waiting_process, holding_process):
        if waiting_process not in self.wait_for_graph:
            self.wait_for_graph[waiting_process] = []
        self.wait_for_graph[waiting_process].append(holding_process)
    
    def has_cycle(self):
        visited = set()
        rec_stack = set()
        
        def dfs(node):
            if node in rec_stack:
                return True  # 순환 발견
            
            if node in visited:
                return False
            
            visited.add(node)
            rec_stack.add(node)
            
            if node in self.wait_for_graph:
                for neighbor in self.wait_for_graph[node]:
                    if dfs(neighbor):
                        return True
            
            rec_stack.remove(node)
            return False
        
        for process in self.wait_for_graph:
            if process not in visited:
                if dfs(process):
                    return True
        
        return False
    
    def find_victim_process(self):
        """교착상태 해결을 위한 희생자 프로세스 선택"""
        if not self.has_cycle():
            return None
        
        # 간단한 휴리스틱: 가장 적은 자원을 사용하는 프로세스
        min_cost_process = min(self.wait_for_graph.keys())
        return min_cost_process

# 사용 예시
detector = DeadlockDetector()
detector.add_wait_relation("P1", "P2")
detector.add_wait_relation("P2", "P3")
detector.add_wait_relation("P3", "P1")  # 순환 형성

if detector.has_cycle():
    victim = detector.find_victim_process()
    print(f"교착상태 탐지! 희생자 프로세스: {victim}")
```

### 실제 프로그래밍에서의 동기화

#### Python asyncio

```python
import asyncio

class AsyncBankAccount:
    def __init__(self, initial_balance):
        self.balance = initial_balance
        self.lock = asyncio.Lock()
    
    async def withdraw(self, amount):
        async with self.lock:
            if self.balance >= amount:
                # 비동기 I/O 시뮬레이션
                await asyncio.sleep(0.1)
                self.balance -= amount
                return True
            return False
    
    async def deposit(self, amount):
        async with self.lock:
            await asyncio.sleep(0.1)
            self.balance += amount

async def main():
    account = AsyncBankAccount(1000)
    
    # 동시 작업 실행
    tasks = []
    for i in range(5):
        task = asyncio.create_task(account.withdraw(100))
        tasks.append(task)
    
    results = await asyncio.gather(*tasks)
    print(f"최종 잔액: {account.balance}")

# asyncio.run(main())
```

#### JavaScript Promise와 동기화

```javascript
class AsyncMutex {
    constructor() {
        this.locked = false;
        this.queue = [];
    }
    
    async acquire() {
        return new Promise((resolve) => {
            if (!this.locked) {
                this.locked = true;
                resolve();
            } else {
                this.queue.push(resolve);
            }
        });
    }
    
    release() {
        if (this.queue.length > 0) {
            const resolve = this.queue.shift();
            resolve();
        } else {
            this.locked = false;
        }
    }
}

class BankAccount {
    constructor(initialBalance) {
        this.balance = initialBalance;
        this.mutex = new AsyncMutex();
    }
    
    async withdraw(amount) {
        await this.mutex.acquire();
        try {
            if (this.balance >= amount) {
                // 비동기 작업 시뮬레이션
                await new Promise(resolve => setTimeout(resolve, 100));
                this.balance -= amount;
                return true;
            }
            return false;
        } finally {
            this.mutex.release();
        }
    }
}

// 사용 예시
const account = new BankAccount(1000);

Promise.all([
    account.withdraw(100),
    account.withdraw(200),
    account.withdraw(150)
]).then(results => {
    console.log('출금 결과:', results);
    console.log('최종 잔액:', account.balance);
});
```

### 성능 고려사항

#### 락 경합 최소화

```python
# 나쁜 예: 락 범위가 너무 큼
class BadExample:
    def __init__(self):
        self.data = {}
        self.lock = threading.Lock()
    
    def process_data(self, key, value):
        with self.lock:
            # 긴 계산 작업
            result = expensive_computation(value)
            # 파일 I/O
            save_to_file(result)
            # 데이터 저장
            self.data[key] = result

# 좋은 예: 락 범위 최소화
class GoodExample:
    def __init__(self):
        self.data = {}
        self.lock = threading.Lock()
    
    def process_data(self, key, value):
        # 락 밖에서 계산과 I/O 수행
        result = expensive_computation(value)
        save_to_file(result)
        
        # 필요한 부분만 락으로 보호
        with self.lock:
            self.data[key] = result
```

#### 락프리 프로그래밍

```python
import queue
import threading

class LockFreeCounter:
    def __init__(self):
        self.value = 0
    
    def increment(self):
        # 원자적 연산 사용
        while True:
            current = self.value
            new_value = current + 1
            # Compare-And-Swap 시뮬레이션
            if self.compare_and_swap(current, new_value):
                break
    
    def compare_and_swap(self, expected, new_value):
        # 실제로는 하드웨어 지원 필요
        if self.value == expected:
            self.value = new_value
            return True
        return False

# 실제 Python에서는 queue.Queue 사용 권장
lock_free_queue = queue.Queue()  # 내부적으로 락프리 구현
```