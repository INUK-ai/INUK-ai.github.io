---
title: 운영체제 #3 - CPU 스케줄링
date: 2025-04-20 +09:00
categories: [AIBE, CS]
tags: [os, scheduling, cpu, 운영체제, 스케줄링]
---

### CPU 스케줄링이란?

> 여러 프로세스가 CPU를 효율적으로 사용할 수 있도록 순서를 정하는 것

#### 스케줄링이 필요한 이유

- **멀티프로그래밍** : 여러 프로세스가 동시에 실행
- **CPU 활용률 극대화** : 유휴 시간 최소화
- **응답 시간 개선** : 사용자 경험 향상

### 프로세스 상태

```
[새로운] → [준비] → [실행] → [종료]
           ↑  ↓      ↓
           [대기] ←──┘

- 새로운(New) : 프로세스가 생성된 상태
- 준비(Ready) : CPU를 할당받기를 기다리는 상태
- 실행(Running) : CPU를 할당받아 실행 중인 상태  
- 대기(Waiting) : I/O 등을 기다리는 상태
- 종료(Terminated) : 실행이 완료된 상태
```

### 스케줄링 알고리즘

#### FCFS (First Come First Served)

> 먼저 온 순서대로 처리

```python
class FCFS:
    def __init__(self):
        self.queue = []
    
    def add_process(self, process):
        self.queue.append(process)
    
    def schedule(self):
        if self.queue:
            return self.queue.pop(0)  # 먼저 온 프로세스 실행
        return None

# 예시
processes = [
    {'name': 'P1', 'burst_time': 10, 'arrival_time': 0},
    {'name': 'P2', 'burst_time': 5, 'arrival_time': 1},
    {'name': 'P3', 'burst_time': 8, 'arrival_time': 2}
]

# 실행 순서: P1 → P2 → P3
# 평균 대기 시간: (0 + 9 + 12) / 3 = 7
```

**장점** : 구현이 간단  
**단점** : 긴 프로세스가 먼저 오면 평균 대기 시간 증가 (Convoy Effect)

#### SJF (Shortest Job First)

> 실행 시간이 가장 짧은 프로세스를 먼저 처리

```python
class SJF:
    def __init__(self):
        self.queue = []
    
    def add_process(self, process):
        # 실행 시간 기준으로 정렬하여 삽입
        self.queue.append(process)
        self.queue.sort(key=lambda x: x['burst_time'])
    
    def schedule(self):
        if self.queue:
            return self.queue.pop(0)
        return None

# 위와 같은 프로세스로 예시
# 실행 순서: P2 → P3 → P1
# 평균 대기 시간: (0 + 4 + 11) / 3 = 5
```

**장점** : 평균 대기 시간이 최소  
**단점** : 긴 프로세스가 계속 밀림 (Starvation), 실행 시간 예측 어려움

#### Round Robin (RR)

> 각 프로세스에 동일한 시간 할당량(Time Quantum) 부여

```python
class RoundRobin:
    def __init__(self, time_quantum):
        self.time_quantum = time_quantum
        self.ready_queue = []
        self.current_time = 0
    
    def add_process(self, process):
        self.ready_queue.append(process)
    
    def schedule(self):
        if not self.ready_queue:
            return None
            
        process = self.ready_queue.pop(0)
        
        if process['remaining_time'] <= self.time_quantum:
            # 프로세스 완료
            self.current_time += process['remaining_time']
            return {'process': process, 'completed': True}
        else:
            # 시간 할당량만큼 실행 후 다시 큐에 삽입
            process['remaining_time'] -= self.time_quantum
            self.current_time += self.time_quantum
            self.ready_queue.append(process)
            return {'process': process, 'completed': False}

# Time Quantum = 3인 경우
# P1(10) → P2(5) → P3(8)
# 실행: P1(3) → P2(3) → P3(3) → P1(3) → P2(2) → P3(3) → P1(3) → P3(2) → P1(1)
```

**장점** : 공정함, 응답 시간 개선  
**단점** : 시간 할당량 설정에 따라 성능 좌우, 문맥 교환 오버헤드

#### Priority Scheduling

> 각 프로세스에 우선순위를 부여하여 처리

```python
class PriorityScheduling:
    def __init__(self):
        self.queue = []
    
    def add_process(self, process):
        self.queue.append(process)
        # 우선순위 기준으로 정렬 (숫자가 작을수록 높은 우선순위)
        self.queue.sort(key=lambda x: x['priority'])
    
    def schedule(self):
        if self.queue:
            return self.queue.pop(0)
        return None

# Aging 기법으로 Starvation 방지
def aging(processes, aging_rate=1):
    for process in processes:
        if process['waiting_time'] > 10:  # 10초 이상 대기 시
            process['priority'] -= aging_rate  # 우선순위 상승
```

**비선점형** : 실행 중인 프로세스를 중단하지 않음  
**선점형** : 더 높은 우선순위 프로세스가 오면 현재 프로세스 중단

#### MLFQ (Multi-Level Feedback Queue)

> 여러 개의 큐를 사용하여 동적으로 우선순위 조정

```python
class MLFQ:
    def __init__(self):
        self.queues = [
            {'level': 0, 'quantum': 2, 'processes': []},   # 최고 우선순위
            {'level': 1, 'quantum': 4, 'processes': []},   # 중간 우선순위  
            {'level': 2, 'quantum': 8, 'processes': []}    # 최저 우선순위
        ]
    
    def add_process(self, process):
        # 새로운 프로세스는 최고 우선순위 큐에 삽입
        process['queue_level'] = 0
        self.queues[0]['processes'].append(process)
    
    def schedule(self):
        # 상위 큐부터 확인
        for queue in self.queues:
            if queue['processes']:
                process = queue['processes'].pop(0)
                return self.execute_process(process, queue)
        return None
    
    def execute_process(self, process, queue):
        if process['remaining_time'] <= queue['quantum']:
            # 프로세스 완료
            return {'process': process, 'completed': True}
        else:
            # 시간 할당량 소진 시 하위 큐로 이동
            process['remaining_time'] -= queue['quantum']
            if process['queue_level'] < 2:
                process['queue_level'] += 1
                self.queues[process['queue_level']]['processes'].append(process)
            else:
                self.queues[2]['processes'].append(process)
            return {'process': process, 'completed': False}
```

### 스케줄링 성능 평가

#### 주요 지표

```python
def calculate_metrics(processes):
    """스케줄링 성능 지표 계산"""
    total_turnaround = 0
    total_waiting = 0
    total_response = 0
    
    for process in processes:
        # 반환 시간 = 완료 시간 - 도착 시간
        turnaround_time = process['completion_time'] - process['arrival_time']
        
        # 대기 시간 = 반환 시간 - 실행 시간
        waiting_time = turnaround_time - process['burst_time']
        
        # 응답 시간 = 첫 실행 시간 - 도착 시간
        response_time = process['first_run_time'] - process['arrival_time']
        
        total_turnaround += turnaround_time
        total_waiting += waiting_time
        total_response += response_time
    
    n = len(processes)
    return {
        'avg_turnaround': total_turnaround / n,
        'avg_waiting': total_waiting / n,
        'avg_response': total_response / n,
        'throughput': n / max(p['completion_time'] for p in processes)
    }
```

### 실제 운영체제에서의 스케줄링

#### Linux CFS (Completely Fair Scheduler)

```bash
# 프로세스 우선순위 확인
ps -eo pid,ni,pri,comm

# nice 값으로 우선순위 조정 (-20 ~ 19, 낮을수록 높은 우선순위)
nice -n 10 ./my_program

# 실행 중인 프로세스의 우선순위 변경
renice 5 -p 1234
```

#### Node.js Event Loop

```javascript
// 이벤트 루프 스케줄링 예시
console.log('1');

setTimeout(() => console.log('2'), 0);      // Timer Queue

setImmediate(() => console.log('3'));       // Check Queue

process.nextTick(() => console.log('4'));   // Next Tick Queue

Promise.resolve().then(() => console.log('5')); // Microtask Queue

console.log('6');

// 출력 순서: 1 → 6 → 4 → 5 → 2 → 3
```

#### JavaScript 웹 브라우저

```javascript
// 태스크 우선순위 관리
function scheduleTask(task, priority) {
    if (priority === 'high') {
        // 마이크로태스크 큐 (높은 우선순위)
        Promise.resolve().then(task);
    } else if (priority === 'normal') {
        // 매크로태스크 큐 (일반 우선순위)
        setTimeout(task, 0);
    } else {
        // 유휴 시간에 실행
        if (window.requestIdleCallback) {
            requestIdleCallback(task);
        } else {
            setTimeout(task, 0);
        }
    }
}

// 사용 예시
scheduleTask(() => updateUI(), 'high');
scheduleTask(() => saveData(), 'normal'); 
scheduleTask(() => cleanup(), 'low');
```

### 실시간 스케줄링

#### Rate Monotonic (RM)

```python
class RateMonotonic:
    def __init__(self):
        self.tasks = []
    
    def add_task(self, name, execution_time, period):
        task = {
            'name': name,
            'execution_time': execution_time,
            'period': period,
            'priority': 1 / period,  # 주기가 짧을수록 높은 우선순위
            'next_deadline': period
        }
        self.tasks.append(task)
        # 우선순위 기준으로 정렬
        self.tasks.sort(key=lambda x: x['priority'], reverse=True)
    
    def is_schedulable(self):
        """스케줄링 가능성 검사"""
        n = len(self.tasks)
        utilization = sum(task['execution_time'] / task['period'] 
                         for task in self.tasks)
        
        # Liu & Layland 조건
        bound = n * (2 ** (1/n) - 1)
        return utilization <= bound

# 예시
rm = RateMonotonic()
rm.add_task('T1', 3, 20)  # 실행시간 3, 주기 20
rm.add_task('T2', 2, 5)   # 실행시간 2, 주기 5
rm.add_task('T3', 2, 10)  # 실행시간 2, 주기 10

print(f"스케줄링 가능: {rm.is_schedulable()}")
```

#### EDF (Earliest Deadline First)

```python
class EDF:
    def __init__(self):
        self.tasks = []
        self.current_time = 0
    
    def add_task(self, name, execution_time, deadline, arrival_time):
        task = {
            'name': name,
            'execution_time': execution_time,
            'deadline': arrival_time + deadline,
            'arrival_time': arrival_time,
            'remaining_time': execution_time
        }
        self.tasks.append(task)
    
    def schedule(self):
        # 현재 시간에 도착한 태스크들 중 데드라인이 가장 빠른 것 선택
        available_tasks = [task for task in self.tasks 
                          if task['arrival_time'] <= self.current_time 
                          and task['remaining_time'] > 0]
        
        if not available_tasks:
            return None
        
        # 데드라인 기준으로 정렬
        available_tasks.sort(key=lambda x: x['deadline'])
        return available_tasks[0]
```

### 스케줄링 최적화

#### CPU 바운드 vs I/O 바운드

```python
def optimize_scheduling(process_type):
    if process_type == 'cpu_bound':
        # CPU 집약적 작업: 긴 시간 할당량
        return {
            'algorithm': 'SJF',
            'time_quantum': 100,
            'priority_boost': False
        }
    elif process_type == 'io_bound':
        # I/O 집약적 작업: 짧은 시간 할당량, 높은 우선순위
        return {
            'algorithm': 'Round Robin',
            'time_quantum': 10,
            'priority_boost': True
        }
    else:  # mixed
        return {
            'algorithm': 'MLFQ',
            'adaptive': True
        }
```

#### 멀티코어 스케줄링

```python
class MultiCoreScheduler:
    def __init__(self, num_cores):
        self.num_cores = num_cores
        self.core_queues = [[] for _ in range(num_cores)]
        self.global_queue = []
    
    def load_balance(self):
        """로드 밸런싱"""
        avg_load = len(self.global_queue) / self.num_cores
        
        for i, queue in enumerate(self.core_queues):
            if len(queue) < avg_load and self.global_queue:
                # 작업을 해당 코어로 이동
                task = self.global_queue.pop(0)
                queue.append(task)
    
    def work_stealing(self, core_id):
        """워크 스틸링 - 유휴 코어가 다른 코어의 작업 가져오기"""
        if self.core_queues[core_id]:
            return None
        
        # 가장 많은 작업을 가진 코어에서 작업 가져오기
        max_queue_idx = max(range(self.num_cores), 
                           key=lambda i: len(self.core_queues[i]))
        
        if self.core_queues[max_queue_idx]:
            return self.core_queues[max_queue_idx].pop()
        return None
```