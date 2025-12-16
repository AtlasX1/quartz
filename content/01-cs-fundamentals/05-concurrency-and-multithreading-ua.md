# Конкурентність та Багатопотоковість: Теоретичні Основи та Практичні Паттерни

## Зміст
1. Основи конкурентності та моделі
2. Архітектура процесів та потоків
3. Примітиви синхронізації та механізми
4. Lock-Free структури даних та алгоритми
5. Event-Driven архітектура та асинк моделі
6. JavaScript/Node.js модель конкурентності
7. Поширені паттерни конкурентності та anti-patterns
8. Виробничі системи та міркування продуктивності

---

## 1. Основи конкурентності та моделі

### 1.1 Конкурентність vs. Паралелізм

Ці терміни часто плутаються, але представляють окремі концепції:

**Конкурентність**: Кілька завдань роблять прогрес на одному процесорі через чередування виконання.

**Паралелізм**: Кілька завдань виконуються одночасно на кількох процесорах.

**Візуальне розрізнення:**
```
Одиничне ядро (тільки конкурентність):
Task A: [===]   [===]   [===]
Task B:     [===]   [===]   [===]
        Час →

Багато-ядерна (справжній паралелізм):
Core 1: Task A [===========]
Core 2: Task B [===========]
        Час →
```

**Математична формулювання:**
$$\text{Пропускна спроможність} = \frac{\text{Завершені завдання}}{\text{Проходить часу}}$$

- Конкурентність підвищує пропускну спроможність на одному процесорі (перемикання задач)
- Паралелізм підвищує пропускну спроможність на багатьох процесорах (одночасне виконання)
- Справжній паралелізм вимагає як конкурентності, так і кількох процесорів

### 1.2 Моделі конкурентності

Різні моделі розглядають конкурентне виконання фундаментально по-різному:

#### 1.2.1 Модель спільної пам'яті

Кілька потоків ділять спільний простір пам'яті, спілкуючись через спільні змінні.

```
┌─────────────────────────────────┐
│      Спільна пам'ять            │
│  ┌──────────────────────────┐   │
│  │ Глобальні змінні, heap   │   │
│  └──────────────────────────┘   │
└─────────────────────────────────┘
    ▲         ▲         ▲
    │         │         │
┌───────┐ ┌───────┐ ┌───────┐
│ Thd 1 │ │ Thd 2 │ │ Thd 3 │
└───────┘ └───────┘ └───────┘
```

**Характеристики:**
- Неявне спілкування через спільний стан
- Низька затримка спілкування (та ж пам'ять)
- Вимагає синхронізації, щоб запобігти race conditions
- Складно міркувати про (складність чередування)

**Мови**: Java, C++, Python (з релаксацією GIL), C#

#### 1.2.2 Модель передачі повідомлень

Потоки/процеси спілкуються виключно через повідомлення, без спільної пам'яті.

```
┌──────────┐           Message           ┌──────────┐
│ Thread A │──────────────────────────────→│ Thread B │
└──────────┘                              └──────────┘
     ▲                                           │
     │──────────────────────────────────────────┘
```

**Характеристики:**
- Явне спілкування через повідомлення
- Вища затримка (серіалізація/транспорт повідомлень)
- Без race conditions (ізольована пам'ять)
- Легше міркувати про (явне спілкування)

**Мови**: Erlang, Go (channels), Rust (message passing), Actor frameworks

#### 1.2.3 Гібридна модель

Комбінує спільну пам'ять для локальних даних з передачею повідомлень для міжакторного спілкування.

**Приклад - Actor Model:**
```
┌─────────────────────────┐
│ Actor A                 │
│ ┌────────────────────┐  │
│ │ Ізольований стан   │  │ Message
│ └────────────────────┘  │─────────────→ [Message Queue]
└─────────────────────────┘
                          ▼
                    ┌─────────────────────────┐
                    │ Actor B                 │
                    │ ┌────────────────────┐  │
                    │ │ Ізольований стан   │  │
                    │ └────────────────────┘  │
                    └─────────────────────────┘
```

**Вигода:**
- Ізоляція стану (запобігає race conditions)
- Явне спілкування (легше міркувати)
- Природне розповсюдження (може бігти на різних машинах)

### 1.3 Модель пам'яті

Модель пам'яті визначає **які значення операція читання може повернути** коли кілька потоків звертаються до пам'яті одночасно.

**Без моделі пам'яті:**
- Компілятор міг би переупорядкувати читання/запису для оптимізації
- CPU міг би виконати операції поза порядком
- Невідповідність кешу непередбачувана
- Поведінка невизначена (кошмар для відлагодження)

**Java Memory Model (JMM) - Ключові гарантії:**

**Видимість**: Якщо Thread A пише до volatile змінної $x$, то Thread B читання $x$ бачить значення, яке Thread A написав (або пізніше):
$$\text{Write}(x, v_1) \xrightarrow{\text{happens-before}} \text{Read}(x) \Rightarrow \text{returns } v_1 \text{ or later}$$

**Впорядкування**: Дії синхронізації встановлюють впорядкування:
```
Thread A:                          Thread B:
lock(mutex)                        
write(x, 5)                        
unlock(mutex) ─┐                   ┌─ lock(mutex)
               └─happens-before─→ read(x) // Returns 5
```

**Атомарність**: Певні операції невідділювані:
```
// Неатомарна - може чередуватися:
int tmp = x;  // Load
x = tmp + 1;  // Increment
x;            // Store

// Атомарна операція (рівень апаратури):
atomic_increment(&x);  // Невідділювана
```

---

## 2. Архітектура процесів та потоків

### 2.1 Модель процесу

**Процес** - це ізольований контекст виконання з:
- Приватний віртуальний простір адреси (ізоляція пам'яті)
- Ресурси на рівні процесу (дескриптори файлів, сигнали)
- Захищені від інших процесів (ОС дотримується ізоляції)

**Над процесу:**
$$\text{Cost Context Switch} = \text{CPU time} + \text{TLB flushes} + \text{Cache invalidation}$$

Виміряно: 1-100 мікросекунди залежно від ОС та архітектури.

**Переваги:**
- Повна ізоляція (крах в одному не впливає на інших)
- Захищена пам'ять (один не може пошкодити іншого)
- Легше міркувати (незалежний стан)

**Недоліки:**
- Важке використання ресурсів (кожен процес ~10-100MB мінімум)
- Дорого перемикання контексту
- Спілкування вимагає IPC (Inter-Process Communication)

### 2.2 Модель потоку

**Поток** - це легкий контекст виконання, що ділить:
- Той же віртуальний простір адреси (спільна пам'ять)
- Те ж ресурси процесу
- Окремі стек та регістри per потік

**Над потоку:**
$$\text{Thread Creation} \approx 1-10\text{ms} \text{ (OS threads)}$$
$$\text{Context Switch Cost} \approx 1-10\text{ microseconds} \text{ (same address space)}$$

**Спільний стан потоку:**
```
┌─────────────────────────────────┐
│ Process (Спільна пам'ять)       │
│ ┌──────────────────────────┐    │
│ │ Heap                     │    │
│ │ Глобальні змінні         │    │
│ │ Code (text segment)      │    │
│ └──────────────────────────┘    │
└─────────────────────────────────┘
    ▲                       ▲
    │                       │
┌─────────────────┐  ┌─────────────────┐
│ Thread 1 Stack  │  │ Thread 2 Stack  │
└─────────────────┘  └─────────────────┘
```

**Переваги:**
- Легко (дешево створити)
- Низький over перемикання контексту
- Швидке спілкування через спільну пам'ять
- Менше використання ресурсів per потік

**Недоліки:**
- Race conditions (спільний стан без синхронізації)
- Складність відлагодження (чередування)
- Крах в одному впливає на весь процес
- Stack overflow в одному потоці може пошкодити інші

### 2.3 Green потоки та планування User-Space

Деякі runtime впроваджують **green потоки** - потоки, засновані в user-space замість kernel:

**Kernel потоки**: ОС scheduler управляє плануванням потоку
- Preemptive (ОС може переривати в будь-який час)
- Справжній паралелізм на multicore
- Context switch викликає ОС
- Вартість: Високий over контексту перемикання

**Green потоки**: Runtime scheduler управляє плануванням потоку
- Можуть бути preemptive або cooperative
- Легко (мільйони можливі)
- Context switch - це виклик функції
- Вартість: Низько, але без справжнього паралелізму (один kernel потік)

**Приклад - Go Goroutines:**
```
Go Runtime:
  ┌────────────────────────────────┐
  │ Scheduler (M goroutines)       │
  │ ┌──────────────────────────┐   │
  │ │ Goroutine pool management│   │
  │ │ Work stealing scheduler  │   │
  │ └──────────────────────────┘   │
  └────────────────────────────────┘
         ▼ (maps to)
    Kernel потоки (N потоків)
    Виконання на (P процесорів)
    
Де M >> N > P (мільйони goroutines на кількох потоків)
```

---

## 3. Примітиви синхронізації та механізми

### 3.1 Mutex (Взаємне виключення блокування)

Найбільш фундаментальний примітив синхронізації. Забезпечує **тільки один потік тримає блокування одночасно**:

```
Thread A:                      Thread B:
lock(mutex)                    lock(mutex) // BLOCKED
critical_section()             // Чекання...
unlock(mutex) ─┐               ┌→ PROCEEDS
               └─ Available ──┘
```

**Spin Lock впровадження (спрощене):**
```c
void spin_lock(atomic_int *lock) {
  while (atomic_exchange(lock, LOCKED) == LOCKED) {
    // Spin (busy wait) до придбання блокування
  }
}

void spin_unlock(atomic_int *lock) {
  atomic_store(lock, UNLOCKED);
}
```

**Аналіз вартості:**
$$\text{Lock Acquisition} = \begin{cases}
O(1) & \text{if uncontended} \\
O(n) & \text{if n threads waiting (spin cost)}
\end{cases}$$

**Ризик deadlock:**
```
Thread A:                      Thread B:
lock(M1)                       lock(M2)
lock(M2) // BLOCKED           lock(M1) // BLOCKED
// Чекання на M2             // Чекання на M1
// DEADLOCK: Ні прогресу
```

**Стратегії запобігання:**
1. **Lock упорядкування**: Завжди придбати блокування в тому ж порядку
2. **Lock timeout**: Виявляти deadlock через timeout
3. **Try-lock pattern**: Non-blocking придбання блокування

### 3.2 Семафор

Примітив синхронізації з **лічильником**, дозволяючий кілька потоків (counting семафор) або один потік (binary семафор):

```
Семафор(count = 2):

Thread A: wait() → count=1 (дозволено)
Thread B: wait() → count=0 (дозволено)
Thread C: wait() // BLOCKED, count=0
Thread A: signal() → count=1, Thread C дозволено

Концептуально:
wait():
  while (count == 0) { sleep() }
  count--

signal():
  count++
  wake(waiting_thread)
```

**Випадок використання - Resource Pool:**
```
Семафор dbConnections(maxConnections = 5)

Worker A: wait() // Придбати з'єднання
  // Використання з'єднання
Worker A: signal() // Вилучити з'єднання
```

**Bounded Buffer Problem:**
```
Buffer size = N
itemCount = Семафор(0)  // Доступні товари
spaceCount = Семафор(N) // Доступні простори
mutex = Mutex()

Producer:
  acquire(spaceCount)
  lock(mutex)
    buffer[writePtr++] = item
  unlock(mutex)
  release(itemCount)

Consumer:
  acquire(itemCount)
  lock(mutex)
    item = buffer[readPtr++]
  unlock(mutex)
  release(spaceCount)
```

### 3.3 Умовні змінні

Дозволяє потокам **чекати на умову** утримуючи блокування, потім відновити коли сповіщено:

```
Thread A:                          Thread B:
lock(mutex)                        lock(mutex)
while (!condition) {               condition = true
  wait(condVar, mutex)  // BLOCK   notify_all(condVar)
  // Автоматично перепридбати блокування
}
// condition є істинна тепер
unlock(mutex)

wait() механізм:
1. Вилучити блокування
2. Sleep (atomically)
3. Пробуджені by signal/notify
4. Re-acquire блокування
5. Перевірити умову знову (spurious wakeup)
```

**Producer-Consumer Pattern:**
```
Queue queue
Mutex m
CondVar full, empty

Producer:
  lock(m)
  while (queue.size == MAX) {
    wait(full, m)  // Чекати якщо queue повна
  }
  queue.push(item)
  notify_one(empty)  // Пробудити consumer
  unlock(m)

Consumer:
  lock(m)
  while (queue.empty()) {
    wait(empty, m)  // Чекати якщо queue пуста
  }
  item = queue.pop()
  notify_one(full)  // Пробудити producer
  unlock(m)
```

### 3.4 Бар'єри

Точка синхронізації де **потоки чекають доки всі дійдуть бар'єру**:

```
Thread A: [===code===] barrier() ──┐
Thread B: [===code===] barrier() ──┼─→ [Всі потоки відновлюються разом]
Thread C: [===code===] barrier() ──┘
```

**Паттерн впровадження:**
```
Бар'єр(num_threads = N):
  count = 0
  condition = CondVar()
  
arrive():
  lock(mutex)
  count++
  if (count == N) {
    notify_all(condition)  // Пробудити всі
  } else {
    wait(condition, mutex)  // Sleep до прибуття всіх
  }
  unlock(mutex)
```

**Випадок використання - Синхронізація фази:**
```
Фаза 1: Всі робітники обчислюють локальні дані
barrier() // Всі повинні завершити Phase 1 перед Phase 2
Фаза 2: Всі робітники агрегують дані
barrier() // Всі повинні завершити Phase 2 перед Phase 3
Фаза 3: Результати фіналізовані
```

---

## 4. Lock-Free структури даних та алгоритми

### 4.1 Атомарні операції та Compare-And-Swap (CAS)

**Атомарні операції** виконуються невідділювано без переривання:

```
Звичайна операція (неатомарна):
Load x в регістр       // Step 1
Increment регістр      // Step 2
Store регістр в x      // Step 3
// Можуть чередуватися між кроків

Atomic increment:
atomic_increment(&x)   // Невідділювана - немає чередування
```

**Compare-And-Swap (CAS)** - Фундаментальний lock-free примітив:

$$\text{CAS}(address, expected, new) = \begin{cases}
\text{Success} & \text{if } [address] == expected \text{ (swap to } new) \\
\text{Failure} & \text{if } [address] \neq expected \text{ (no change)}
\end{cases}$$

**Впровадження апаратури** (x86):
```
lock cmpxchg mem, reg  // Atomic test-and-swap з memory lock
```

**Lock-Free Increment Використанням CAS:**
```
void atomic_increment(atomic_int *x) {
  while (true) {
    int old = *x;
    int new = old + 1;
    if (CAS(x, old, new)) {  // Succeeds якщо значення без змін
      break;
    }
    // Retry якщо значення змінилось між read та CAS
  }
}
```

### 4.2 Lock-Free Stack

Використанням CAS для впровадження thread-safe stack без блокувань:

```
struct Node {
  int value;
  Node* next;
};

struct LockFreeStack {
  atomic<Node*> top;
  
  void push(int value) {
    Node* newNode = new Node{value, nullptr};
    
    while (true) {
      Node* oldTop = top.load();
      newNode->next = oldTop;
      
      if (top.compare_exchange_strong(oldTop, newNode)) {
        break;  // Success
      }
      // Retry якщо інший потік модифікував top
    }
  }
  
  bool pop(int& value) {
    while (true) {
      Node* oldTop = top.load();
      
      if (oldTop == nullptr) {
        return false;  // Stack пустий
      }
      
      Node* newTop = oldTop->next;
      
      if (top.compare_exchange_strong(oldTop, newTop)) {
        value = oldTop->value;
        delete oldTop;
        return true;
      }
      // Retry якщо інший потік модифікував top
    }
  }
};
```

**Характеристики продуктивності:**
$$\text{Contention} = \frac{\text{CAS failures}}{\text{Total attempts}}$$

- **Uncontended**: CAS вдається в першої спроби (O(1) з дуже низьким over)
- **Contended**: Багато CAS невдач (retry loops, евентуальний успіх, але busy-wait)
- **Highly contended**: Lock-free гірше за блокування (spinning, cache thrashing)

### 4.3 ABA Problem

**Тонка помилка** в lock-free алгоритмах:

```
Початковий стан: [A] → [B] → [C]

Thread 1:
  ptr = top;  // [A]
  // Preempted...

Thread 2:
  pop();      // Remove [A]
  // Somehow [A] реапарує на top
  [A] → [D] → [C]

Thread 1:
  compare_exchange(A, B)  // Виглядає валідно! Але B тепер orphaned
  // [B] → [C] втрачено (memory leak або corruption)
```

**Рішення - Versioned Pointers:**

```
struct VersionedPtr {
  Node* ptr;
  uint64_t version;  // Incremented на кожну модифікацію
};

// CAS тепер порівнює як pointer, так і version
compare_exchange_strong(
  {oldPtr, version1},
  {newPtr, version1+1}
)
// ABA неможливо - version доказує справжню зміну
```

---

## 5. Event-Driven архітектура та асинк моделі

### 5.1 Event Loop модель

Обробляє **багато одночасних операцій з одним потоком** через event multiplexing:

```
┌──────────────────────────┐
│ Event Loop               │
│  while (!done) {         │
│    events = poll()       │
│    for event in events   │
│      handle(event)       │
│  }                       │
└──────────────────────────┘
    ▲                  │
    │                  ▼
[Pending Ops]   [Ready Events]
    ▲                  │
    │                  ▼
    └────────────────────┘
```

**Підтримка операційної системи - Select/Poll/Epoll:**

```
fd_set readfds, writefds;
FD_SET(socket1, &readfds);
FD_SET(socket2, &readfds);

// Block доки socket1 АБО socket2 має дані
int ready = select(maxfd+1, &readfds, &writefds, NULL, timeout);

// Kernel стежить стан дескриптора файлу, повертає ready fds
for (int i = 0; i < maxfd; i++) {
  if (FD_ISSET(i, &readfds)) {
    // Дескриптор файлу i має дані готові
    handle_read(i);
  }
}
```

**Масштабованість:**
$$\text{Max Concurrent Connections} = \begin{cases}
O(n) \text{ з select (scans всі fds)} \\
O(1) \text{ з epoll (повертає ready fds тільки)}
\end{cases}$$

**Блокування vs. Non-Blocking операцій:**
```
Блокування I/O:
Operation стартує → BLOCKS → Повертає результат

Non-Blocking I/O:
Operation стартує → Повертає негайно
  (з EAGAIN якщо не готовий)

Event-Driven:
Operation стартує (повертає негайно)
Коли готовий, event стреляє → callback викликається
```

### 5.2 Callback та Promise-Based Асинк

**Модель Callback** (continuation-passing стиль):

```javascript
fetchData(url, function(err, data) {
  if (err) handleError(err);
  else {
    processData(data, function(err, result) {
      if (err) handleError(err);
      else {
        saveResult(result, function(err) {
          if (err) handleError(err);
          else console.log("Done");
        });
      }
    });
  }
});
// Callback Hell / Pyramid of Doom
```

**Модель Promise** (composable асинк):

```javascript
fetchData(url)
  .then(data => processData(data))
  .then(result => saveResult(result))
  .then(() => console.log("Done"))
  .catch(err => handleError(err));
```

**Promise State Machine:**
```
┌──────────────┐
│  Pending     │
└──────┬───────┘
       │
       ├─→ Fulfilled (value доступна)
       │      └─→ .then() handlers виконуються
       │
       └─→ Rejected (помилка сталась)
              └─→ .catch() handlers виконуються
              
Один раз Fulfilled або Rejected → Immutable (немає змін стану)
```

**Promise впровадження (спрощене):**
```javascript
class MyPromise {
  constructor(executor) {
    this.state = 'PENDING';
    this.value = undefined;
    this.handlers = [];
    
    const resolve = (value) => {
      if (this.state === 'PENDING') {
        this.state = 'FULFILLED';
        this.value = value;
        this.handlers.forEach(h => h.onResolve(value));
      }
    };
    
    const reject = (error) => {
      if (this.state === 'PENDING') {
        this.state = 'REJECTED';
        this.value = error;
        this.handlers.forEach(h => h.onReject(error));
      }
    };
    
    executor(resolve, reject);
  }
  
  then(onResolve, onReject) {
    return new MyPromise((resolve, reject) => {
      const handler = {
        onResolve: (value) => {
          try {
            resolve(onResolve(value));
          } catch (e) {
            reject(e);
          }
        },
        onReject: (error) => {
          try {
            resolve(onReject(error));
          } catch (e) {
            reject(e);
          }
        }
      };
      
      if (this.state === 'PENDING') {
        this.handlers.push(handler);
      } else if (this.state === 'FULFILLED') {
        handler.onResolve(this.value);
      } else {
        handler.onReject(this.value);
      }
    });
  }
}
```

### 5.3 Async/Await синтаксис

**Синтаксичний цукор** над promises для послідовного-виглядаючого коду:

```javascript
// Версія Promise
function fetchAndProcess() {
  return fetchData(url)
    .then(data => processData(data))
    .then(result => saveResult(result))
    .then(() => "Done");
}

// Версія Async/Await (та ж логіка, послідовний вигляд)
async function fetchAndProcess() {
  try {
    const data = await fetchData(url);
    const result = await processData(data);
    await saveResult(result);
    return "Done";
  } catch (err) {
    handleError(err);
  }
}

// Виклик
fetchAndProcess().then(console.log).catch(console.error);
```

**Компіляція в Promise chains:**
```javascript
// Async/await
async function foo() {
  const a = await op1();
  const b = await op2(a);
  return op3(b);
}

// Компілюється до приблизно:
function foo() {
  return op1()
    .then(a => op2(a))
    .then(b => op3(b));
}
```

---

## 6. JavaScript/Node.js модель конкурентності

### 6.1 Single-Threaded Event Loop

JavaScript виконується в **single-threaded event loop** з асинк I/O:

```
┌─────────────────────────────────────────┐
│ Call Stack                              │
│ ┌─────────────────────────────────────┐ │
│ │ function виконання                  │ │
│ └─────────────────────────────────────┘ │
└─────────────────────────────────────────┘

┌─────────────────────────────────────────┐
│ Task Queue (Macrotasks)                 │
│ ┌─────────────┐ ┌─────────────┐         │
│ │ setTimeout  │ │ I/O готовий  │ ...     │
│ └─────────────┘ └─────────────┘         │
└─────────────────────────────────────────┘

┌─────────────────────────────────────────┐
│ Microtask Queue                         │
│ ┌─────────────┐ ┌─────────────┐         │
│ │ Promise     │ │ queueMicro  │ ...     │
│ └─────────────┘ └─────────────┘         │
└─────────────────────────────────────────┘

┌─────────────────────────────────────────┐
│ Event Loop                              │
│ while (taskQueue.hasTasks()) {          │
│   if (callStack.empty()) {              │
│     runAllMicrotasks();                 │
│     runOneTask();                       │
│     render();                           │
│   }                                     │
│ }                                       │
└─────────────────────────────────────────┘
```

**Порядок виконання:**
$$\text{CallStack} \xrightarrow{\text{empty}} \text{Microtasks} \xrightarrow{\text{empty}} \text{OneTask} \xrightarrow{\text{render}} \text{Repeat}$$

**Приклад - Часова шкала виконання:**
```javascript
console.log('1');           // → Synchronous, логує негайно

setTimeout(() => {
  console.log('2');         // → Macrotask (Task queue)
}, 0);

Promise.resolve()
  .then(() => console.log('3'))  // → Microtask (Microtask queue)
  .then(() => console.log('4'));

console.log('5');           // → Synchronous, логує негайно

// Вихід: 1, 5, 3, 4, 2
// Обґрунтування:
// 1. Call stack: 1, 5
// 2. Microtask queue: 3, 4
// 3. Task queue: 2
```

### 6.2 GIL в Python

Python має **Global Interpreter Lock (GIL)** - mutex запобігаючий справжній паралелізм:

```
┌──────────────────────────────────┐
│ Python Interpreter               │
│ ┌──────────────────────────────┐ │
│ │ GIL (Global Lock)            │ │
│ │ Тільки один потік виконує    │ │
│ │ Python bytecode одночасно    │ │
│ └──────────────────────────────┘ │
│                                  │
│ ┌─────────────┐ ┌─────────────┐ │
│ │ Thread 1    │ │ Thread 2    │ │
│ │ Утримує GIL │ │ Чекає       │ │
│ │ Виконує     │ │ на GIL      │ │
│ └─────────────┘ └─────────────┘ │
└──────────────────────────────────┘
```

**Вплив на продуктивність:**
```python
# CPU-bound роботи: GIL запобігає паралелізму
def cpu_task(n):
    sum = 0
    for i in range(n):
        sum += i
    return sum

# Послідовна: ~1 секунда для 2x100M iterations
result = cpu_task(100000000) + cpu_task(100000000)

# Багатопотокова: ~1 секунда все-таки (GIL запобігає паралелізму)
t1 = Thread(target=cpu_task, args=(100000000,))
t2 = Thread(target=cpu_task, args=(100000000,))
t1.start(); t2.start()
t1.join(); t2.join()

# Multiprocess: ~0.5 секунд (справжній паралелізм, окремі interpreters)
p1 = Process(target=cpu_task, args=(100000000,))
p2 = Process(target=cpu_task, args=(100000000,))
p1.start(); p2.start()
p1.join(); p2.join()
```

**I/O операцій вилучають GIL:**
```python
# I/O-bound роботи: GIL вилучена під час I/O
def io_task():
    response = requests.get('http://example.com')  # Вилучає GIL
    return response.text

# Багатопотокова: Concurrent I/O (GIL вилучена)
# Обидва потоки можуть робити I/O одночасно
threads = [Thread(target=io_task) for _ in range(10)]
for t in threads: t.start()
for t in threads: t.join()
```

---

## 7. Поширені паттерни конкурентності та anti-patterns

### 7.1 Паттерни безпеки потоків

**Immutability:**
```
// THREAD-SAFE: Немає спільного мутабельного стану
const config = Object.freeze({
  maxConnections: 100,
  timeout: 5000
});

// Всі потоки читають config безпечно
thread1.use(config);
thread2.use(config);
```

**Synchronized Access:**
```
// THREAD-SAFE: Охороняється блокуванням
class ThreadSafeCounter {
  private int count = 0;
  private final Object lock = new Object();
  
  public void increment() {
    synchronized(lock) {
      count++;  // Тільки один потік одночасно
    }
  }
  
  public int get() {
    synchronized(lock) {
      return count;
    }
  }
}
```

**Copy-on-Write:**
```
// THREAD-SAFE: Читачі не блокуються, писачи створюють копію
class CopyOnWriteList<T> {
  private volatile T[] array;
  
  public T get(int index) {
    return array[index];  // Ніколи не блокує
  }
  
  public void add(T element) {
    synchronized(this) {
      T[] newArray = Arrays.copyOf(array, array.length + 1);
      newArray[array.length] = element;
      array = newArray;  // Volatile запис
    }
  }
}
```

### 7.2 Поширені anti-patterns

**Race Condition - Check-Then-Act:**
```
// UNSAFE: Race condition між check та act
if (list.isEmpty()) {  // Thread A перевіряє
  // Thread B додає element тут
  list.add(element);   // Thread A додає (немає видимої race)
}

// SAFE: Atomic check-and-act
synchronized(list) {
  if (list.isEmpty()) {
    list.add(element);  // Atomic
  }
}
```

**Double-Checked Locking Anti-pattern:**
```
// BROKEN: Все ще має race condition у деяких мовах
class Singleton {
  private static Singleton instance;
  
  public static Singleton getInstance() {
    if (instance == null) {  // Перша перевірка (no lock)
      synchronized(Singleton.class) {
        if (instance == null) {  // Друга перевірка (locked)
          instance = new Singleton();  // Все ще може мати races
        }
      }
    }
    return instance;
  }
}

// CORRECT: Eager ініціалізація або volatile
class Singleton {
  private static volatile Singleton instance;  // Volatile!
  
  public static Singleton getInstance() {
    if (instance == null) {
      synchronized(Singleton.class) {
        if (instance == null) {
          instance = new Singleton();  // Volatile запис забезпечує видимість
        }
      }
    }
    return instance;
  }
}

// SIMPLER: Static ініціалізація
class Singleton {
  public static final Singleton INSTANCE = new Singleton();
}
```

**Deadlock сценарії:**
```
// DEADLOCK: Циклічна залежність блокування
Thread A:
  lock(M1)
  lock(M2)  // Чекає на M2

Thread B:
  lock(M2)
  lock(M1)  // Чекає на M1 - DEADLOCK!

// SOLUTION: Встановити упорядкування блокування
Завжди блокування: M1 → M2 (ніколи M2 → M1)
```

---

## 8. Виробничі системи та міркування продуктивності

### 8.1 Закон Амдала та Speedup

**Закон Амдала** кількісно визначає speedup від паралелізму:

$$S = \frac{1}{(1-p) + \frac{p}{n}}$$

Де:
- $S$ = коефіцієнт speedup
- $p$ = частка коду паралелізовна
- $n$ = кількість процесорів

**Приклади:**
```
Паралелізовна = 50%, Процесори = 4:
S = 1 / (0.5 + 0.5/4) = 1 / 0.625 = 1.6x speedup
(Не 4x попри 4 процесори!)

Паралелізовна = 90%, Процесори = 4:
S = 1 / (0.1 + 0.9/4) = 1 / 0.325 = 3.08x speedup

Паралелізовна = 99%, Процесори = 4:
S = 1 / (0.01 + 0.99/4) = 1 / 0.2575 = 3.88x speedup

Паралелізовна = 99%, Процесори = ∞:
S = 1 / (0.01 + 0) = 100x speedup (теоретична межа)
```

**Імплікація**: Навіть малі послідовні частини суворо обмежують паралелізм.

### 8.2 Thread Pool архітектура

Замість створення потоків per завдання (дорого), переиспользуйте потоки:

```
┌─────────────────────────────────┐
│ Task Queue                      │
│ [Task1] [Task2] [Task3] ...     │
└─────────────────────────────────┘
        ▲
        │
┌─────────────────────────────────┐
│ Thread Pool (N robotic потоків) │
│ ┌───────┐ ┌───────┐             │
│ │ Thd 1 │ │ Thd 2 │ ...         │
│ │(busy) │ │(idle) │             │
│ └───────┘ └───────┘             │
└─────────────────────────────────┘
```

**Налаштування параметра:**
$$N_{\text{threads}} = \begin{cases}
\text{cpu cores} & \text{CPU-bound завдання} \\
\text{cpu cores} \times (1 + \text{wait ratio}) & \text{I/O-bound завдання}
\end{cases}$$

Для I/O з 90% часу чекання:
$$N = \text{cores} \times (1 + 0.9) = \text{cores} \times 1.9$$

### 8.3 Пастки продуктивності

**Context Switch Thrashing:**
- Забагато потоків → надмірні context switches
- Вартість: 1-100 мікросекунди per switch
- Лікування: Thread pool, обмеження concurrent потоків

**Cache Invalidation:**
- Contention блокування → потоки інвалідують cache один одного
- Вартість: 100-300 cycle cache miss (vs 1-3 cycles для cache hit)
- Лікування: Зменшити contention, lock-free алгоритми, окремі cache lines

**GIL Thrashing (Python):**
- Потоки вилучаються/придбаються GIL швидко
- Вартість: Over перевищує паралелізм вигода
- Лікування: Використання multiprocessing, async/await, Cython

---

## Ключові висновки

1. **Моделі конкурентності**: Спільна пам'ять (складна, але швидка), передача повідомлень (складна логіка трирування, уникнута), гібридна (actor модель).

2. **Примітиви синхронізації**:
   - **Mutex**: Взаємне виключення (найпростіший, найпоширеніший)
   - **Semaphore**: Лічильник-базований (resource pools)
   - **Condition Variable**: Чекати на умови
   - **Barrier**: Синхронізація фази

3. **Lock-Free програмування**: Використання CAS для синхронізації без блокувань. Uncontended швидко, але contended гірше за блокування. Вирішує GC pause та priority inversion проблеми.

4. **Event Loop**: Single-threaded модель (JS, Node.js) обробляє мільйони concurrent з'єднань через I/O multiplexing.

5. **JavaScript виконання**: Synchronous код → Microtasks (Promises) → Macrotasks (setTimeout).

6. **Python GIL**: Запобігає справжній CPU паралелізм з потоками. Використання multiprocessing для CPU-bound, async для I/O-bound.

7. **Закон Амдала**: Speedup обмежений послідовними частинами. 10% послідовна = max 10x speedup навіть на ∞ процесорів.

8. **Thread Pools**: Переиспользуйте потоки, уникайте creation over. Розмір базований на CPU cores (CPU-bound) або cores × (1 + wait ratio) (I/O-bound).

9. **Поширені пастки**: Race conditions (check-then-act), deadlocks (встановити блокування упорядкування), context switch thrashing (обмеження потоків), cache invalidation (зменшити contention).

10. **Тестування concurrent коду**: Надзвичайно складно через nondeterminism. Використовуйте інструменти (ThreadSanitizer, Helgrind), stress тестування, property-based тестування.

11. **Сучасні підходи**: Перевагу immutability, async/await над callbacks, functional програмування без спільного стану.

12. **Розподілена конкурентність**: Передача повідомлень + event-driven (microservices, Kafka, distributed actors) уникає багатьох lock-based проблем.
