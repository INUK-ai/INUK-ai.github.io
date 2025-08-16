---
title: 운영체제 #2 - 메모리 관리
date: 2025-04-19 +09:00
categories: [AIBE, CS]
tags: [os, memory, 운영체제, 메모리관리]
---

### 메모리 관리란?

> 운영체제가 메모리를 효율적으로 할당하고 회수하는 과정

#### 메모리 관리가 필요한 이유

- **제한된 메모리** : 물리적 메모리는 한정적
- **다중 프로세스** : 여러 프로세스가 동시에 실행
- **효율성** : 메모리 공간을 최적화해야 함

### 메모리 할당 방식

#### 연속 메모리 할당

**고정 분할 (Fixed Partition)**
- 메모리를 고정된 크기로 미리 분할
- 간단하지만 내부 단편화 발생

**가변 분할 (Variable Partition)**
- 프로세스 크기에 맞게 메모리 분할
- 외부 단편화 발생 가능

```
메모리 상태 예시:
[프로세스A][빈공간][프로세스B][빈공간][프로세스C]
         ↑          ↑
    내부 단편화   외부 단편화
```

#### 불연속 메모리 할당

**페이징 (Paging)**
- 물리 메모리를 고정 크기 블록(페이지)으로 분할
- 논리 주소를 물리 주소로 변환

```
논리 주소: [페이지 번호][페이지 오프셋]
        ↓ 페이지 테이블 참조
물리 주소: [프레임 번호][페이지 오프셋]
```

**세그멘테이션 (Segmentation)**
- 논리적 단위(세그먼트)로 메모리 분할
- 코드, 데이터, 스택 등을 별도 세그먼트로 관리

### 가상 메모리

> 물리 메모리보다 큰 프로그램을 실행할 수 있게 하는 기술

#### 주요 개념

**페이지 폴트 (Page Fault)**
- 요청한 페이지가 물리 메모리에 없을 때 발생
- 디스크에서 해당 페이지를 메모리로 로드

```python
# 페이지 폴트 처리 과정
def page_fault_handler(page_number):
    if page_number not in physical_memory:
        # 1. 디스크에서 페이지 로드
        page_data = load_from_disk(page_number)
        
        # 2. 물리 메모리에 공간이 없으면 교체
        if physical_memory.is_full():
            victim_page = select_victim_page()  # LRU, FIFO 등
            swap_out(victim_page)
        
        # 3. 새 페이지를 메모리에 로드
        physical_memory.load(page_data)
        
        # 4. 페이지 테이블 업데이트
        update_page_table(page_number)
```

### 페이지 교체 알고리즘

#### FIFO (First In First Out)

```python
class FIFO:
    def __init__(self, capacity):
        self.capacity = capacity
        self.memory = []
        self.queue = []
    
    def access_page(self, page):
        if page not in self.memory:
            if len(self.memory) >= self.capacity:
                # 가장 먼저 들어온 페이지 제거
                oldest_page = self.queue.pop(0)
                self.memory.remove(oldest_page)
            
            self.memory.append(page)
            self.queue.append(page)
            return "Page Fault"
        return "Hit"
```

#### LRU (Least Recently Used)

```python
class LRU:
    def __init__(self, capacity):
        self.capacity = capacity
        self.memory = {}
        self.access_order = []
    
    def access_page(self, page):
        if page in self.memory:
            # 최근 사용으로 갱신
            self.access_order.remove(page)
            self.access_order.append(page)
            return "Hit"
        
        if len(self.memory) >= self.capacity:
            # 가장 오래 사용되지 않은 페이지 제거
            lru_page = self.access_order.pop(0)
            del self.memory[lru_page]
        
        self.memory[page] = True
        self.access_order.append(page)
        return "Page Fault"
```

#### LFU (Least Frequently Used)

```python
class LFU:
    def __init__(self, capacity):
        self.capacity = capacity
        self.memory = {}
        self.frequency = {}
    
    def access_page(self, page):
        if page in self.memory:
            self.frequency[page] += 1
            return "Hit"
        
        if len(self.memory) >= self.capacity:
            # 가장 적게 사용된 페이지 찾기
            min_freq = min(self.frequency.values())
            lfu_page = next(p for p in self.frequency 
                          if self.frequency[p] == min_freq)
            
            del self.memory[lfu_page]
            del self.frequency[lfu_page]
        
        self.memory[page] = True
        self.frequency[page] = 1
        return "Page Fault"
```

### 메모리 단편화

#### 내부 단편화 (Internal Fragmentation)

```
할당된 메모리 블록:
[프로세스 데이터][사용되지 않는 공간]
                 ↑ 내부 단편화
```

- 할당된 메모리 블록 내부에서 사용되지 않는 공간
- 고정 크기 할당에서 주로 발생

#### 외부 단편화 (External Fragmentation)

```
전체 메모리:
[프로세스A][빈공간][프로세스B][빈공간][프로세스C]
          ↑ 2KB     ↑ 3KB
          
새로운 4KB 프로세스 → 할당 불가 (총 5KB 여유 공간이 있지만)
```

- 할당되지 않은 메모리가 작은 조각들로 나뉘어져 있는 상태
- 가변 크기 할당에서 발생

#### 압축 (Compaction)

```python
def memory_compaction(memory_blocks):
    """메모리 압축을 통한 외부 단편화 해결"""
    allocated = []
    free_space = 0
    
    for block in memory_blocks:
        if block.is_allocated:
            allocated.append(block)
        else:
            free_space += block.size
    
    # 할당된 블록들을 앞으로 이동
    compacted_memory = allocated + [FreeBlock(free_space)]
    return compacted_memory
```

### 실제 운영체제에서의 메모리 관리

#### Linux 메모리 관리

```bash
# 메모리 사용량 확인
free -h

# 프로세스별 메모리 사용량
ps aux --sort=-%mem | head

# 가상 메모리 정보 확인
cat /proc/meminfo
```

#### JavaScript V8 엔진

```javascript
// 힙 메모리 사용량 확인
const memUsage = process.memoryUsage();
console.log('Heap Used:', memUsage.heapUsed);
console.log('Heap Total:', memUsage.heapTotal);

// 가비지 컬렉션 강제 실행
if (global.gc) {
    global.gc();
    console.log('GC executed');
}
```

### 메모리 누수 방지

#### JavaScript 예시

```javascript
// 문제가 있는 코드
let users = [];
function addUser(userData) {
    users.push(userData); // 계속 누적됨
}

// 개선된 코드
let users = new Map();
function addUser(userId, userData) {
    users.set(userId, userData);
    
    // 일정 크기 초과 시 오래된 데이터 제거
    if (users.size > MAX_USERS) {
        const oldestKey = users.keys().next().value;
        users.delete(oldestKey);
    }
}
```

#### Python 예시

```python
# 문제가 있는 코드
cache = {}
def get_data(key):
    if key not in cache:
        cache[key] = expensive_operation(key)  # 무한정 증가
    return cache[key]

# 개선된 코드 - LRU Cache 사용
from functools import lru_cache

@lru_cache(maxsize=128)
def get_data(key):
    return expensive_operation(key)
```

### 성능 최적화 팁

**지역성 (Locality) 활용**
```python
# 나쁜 예 - 공간 지역성 위반
for j in range(cols):
    for i in range(rows):
        matrix[i][j] = process_data(i, j)

# 좋은 예 - 공간 지역성 활용
for i in range(rows):
    for j in range(cols):
        matrix[i][j] = process_data(i, j)
```

**메모리 풀 사용**
```python
class MemoryPool:
    def __init__(self, block_size, pool_size):
        self.block_size = block_size
        self.free_blocks = [bytearray(block_size) 
                           for _ in range(pool_size)]
        self.used_blocks = []
    
    def allocate(self):
        if self.free_blocks:
            block = self.free_blocks.pop()
            self.used_blocks.append(block)
            return block
        return None
    
    def deallocate(self, block):
        if block in self.used_blocks:
            self.used_blocks.remove(block)
            self.free_blocks.append(block)
```