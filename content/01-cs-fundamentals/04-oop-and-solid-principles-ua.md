# ООП та SOLID Принципи: Теоретичні Основи та Архітектурні Паттерни

## Зміст
1. Парадигма об'єктно-орієнтованого програмування
2. Основні концепції ООП та математичні основи
3. SOLID принципи: теорія та застосування
4. Design Patterns та їх класифікація
5. Архітектурні паттерни та корпоративний дизайн
6. Injection залежностей та Inversion of Control
7. GRASP принципи та розподіл відповідальності
8. Виробничі системи та компроміси

---

## 1. Парадигма об'єктно-орієнтованого програмування

### 1.1 Історичний контекст та зміна парадигми

Об'єктно-орієнтоване програмування виникло в 1960-х-1970-х роках як відповідь на обмеження процедурного програмування при масштабуванні. Mentre процедурне програмування організує код навколо **функцій** (поведінка), ООП організує код навколо **об'єктів** (дані + поведінка). Це фундаментальна зміна вирішує кілька ключових проблем:

**Порівняння процедурної та об'єктної декомпозиції:**
- Процедурна: Функції оперують спільним глобальним станом → сильна зв'язаність, складна підтримка
- ООП: Об'єкти інкапсулюють стан та поведінку → слабка зв'язаність, інкапсуляція, модульність

Теоретичне підґрунтя ООП спирається на три стовпи:
1. **Інкапсуляція**: Пакування даних з методами, контроль доступу
2. **Спадкування**: Встановлення ієрархії типів та повторне використання коду
3. **Поліморфізм**: Варіація поведінки під час виконання через перевизначення методів

### 1.2 Проблема абстракції

ООП вирішує **проблему абстракції** у розробці ПО: як представити складні системи, використовуючи простіші ментальні моделі. Це відображає концепції теорії категорій у математиці:

- Об'єкти - це екземпляри **категорій** з визначеними інтерфейсами
- Спадкування створює **ієрархічні категоріальні відносини**
- Диспетчеризація методів - це форма **природної трансформації** між типами об'єктів

Принцип абстракції стверджує: *"Вибрана абстракція визначає, які аспекти системи видимі розробнику, а які приховані."*

Наприклад, абстракція `PaymentProcessor` приховує:
- Деталі конвертації валют
- Специфіку платіжного шлюзу
- Логіку повторних спроб та обробки помилок
- Генерацію журналу аудиту

Водночас розкриває:
- `processPayment(amount: Money, card: CreditCard): Result<PaymentId>`
- `refund(paymentId: PaymentId): Result<RefundId>`

### 1.3 Системи типів та поліморфізм підтипу

ООП фундаментально про **поліморфізм підтипу**, формалізований принципом Ліскова:

*Якщо S є підтипом T, то об'єкти типу S можуть бути замінені об'єктами типу T без порушення бажаних властивостей програми.*

Це можна виразити математично. Для типів $S$ та $T$:
$$S <: T \iff \forall \text{ операції } f \text{ де } T \text{ валідна, } S \text{ дає еквівалентні результати}$$

**Приклад - Поведінковий підтип:**
```
Type Circle extends Shape
Type Square extends Shape

// Обидва повинні задовольняти контракт Shape
Shape.getArea(): Number
Shape.getPerimeter(): Number
```

Порушення LSP: `Rectangle extends Square` порушує LSP, оскільки:
- Square очікує: `width == height` інваріант
- Rectangle порушує цей інваріант
- LSP передбачає проблеми підтримки (які дійсно виникають на практиці)

---

## 2. Основні концепції ООП та математичні основи

### 2.1 Інкапсуляція: Приховування інформації

Інкапсуляція - це **пакування даних з методами** та приховування внутрішніх деталей. Це вирішує фундаментальну проблему управління зв'язаністю:

**Метрика зв'язаності:**
$$\text{Зв'язаність} = \frac{\text{Зовнішні залежності розкриті}}{(\text{Всього методів} + \text{Всього полів})}$$

Інкапсуляція зменшує це відношення через:
1. Приватні поля (не розкриті)
2. Розкриття поведінки обмеженими, навмисними інтерфейсами
3. Контроль як внутрішнього стану доступ та модифікація

**Збереження інваріантів:**
Інкапсуляція дозволяє підтримувати об'єктні інваріанти. Для `BankAccount`:
```
Інваріант: balance >= 0
Інваріант: transactions.length <= maxTransactionHistory
```

Приватні методи як `_validateBalance()` та `_rotateTransactions()` підтримують ці інваріанти без зовнішнього втручання.

**Рівні доступу та інформаційні межі:**
- `private`: Без зовнішнього доступу (тільки всередину класу)
- `protected`: Доступно підклассам (внутрішня ієрархія)
- `package/internal`: Доступно в межах модуля/пакета
- `public`: Повний зовнішній доступ

Кожен рівень представляє компроміс між гнучкістю та силою інкапсуляції.

### 2.2 Спадкування: Ієрархія типів та повторне використання коду

Спадкування встановлює **is-a відносини** між типами, дозволяючи:

1. **Повторне використання коду**: Спільна поведінка визначена один раз в батьківському класі
2. **Поліморфна поведінка**: Різні реалізації через перевизначення методів
3. **Ієрархія типів**: Встановлення замінних типів

**Компроміси спадкування:**

| Аспект | Вигода | Вартість |
|--------|--------|---------|
| Повторне використання коду | DRY принцип, зменшена дублікація | Сильна зв'язаність батько-дитини |
| Поліморфізм | Варіація поведінки під час виконання | Над поліморфної диспетчеризації |
| Чіткість ієрархії | Моделює природні IS-A відносини | Жорстка структура, крихка базова класу |
| Контракти інтерфейсу | Гарантує поведінку підкласу | LSP порушення якщо не обережні |

**Крихка проблема базової класу:**
Коли базовий клас змінюється, похідні класи можуть неочікувано зламатись:
```
Base клас: removeElement(index) → видаляє на індексі
Derived клас: List extends ArrayList
  Інваріант: підтримує відсортований порядок

// Якщо Base клас змінить removeElement() реалізацію,
// Інваріант відсортованого порядку Derived класу може бути порушений
```

**Одиночне vs. Множинне спадкування:**
Більшість сучасних мов ООП (Java, C#, Python 3 MRO) обмежують одиночне спадкування або використовують інтерфейси (множинне спадкування інтерфейсу, не реалізації):

- **Одиночне спадкування**: Простіше, уникає diamond problem, але менш виразне
- **Множинне спадкування**: Більш виразне, але створює неоднозначність (C++ diamond problem)
- **Множинне впровадження інтерфейсу**: Компромісне рішення (Java, C#, TypeScript)

### 2.3 Поліморфізм: Варіація поведінки під час виконання

Поліморфізм дозволяє **викликати той же назву методу на різних об'єктах** та отримувати різні поведінки. Це проявляється у трьох формах:

#### 2.3.1 Поліморфізм підтипу (Runtime диспетчеризація)

```
interface Shape {
  area(): number
  perimeter(): number
}

class Circle implements Shape {
  constructor(radius: number) { this.r = radius }
  area() { return Math.PI * this.r * this.r }
  perimeter() { return 2 * Math.PI * this.r }
}

class Rectangle implements Shape {
  constructor(w: number, h: number) { this.w = w; this.h = h }
  area() { return this.w * this.h }
  perimeter() { return 2 * (this.w + this.h) }
}

// Поліморфна поведінка
const shapes: Shape[] = [new Circle(5), new Rectangle(4, 6)]
shapes.forEach(s => console.log(s.area()))  // Різна реалізація per тип
```

**Таблиці віртуальних методів (VMT):**
Сучасні реалізації ООП використовують VMT для розв'язання викликів методів:
- Кожен об'єкт несе вказівник на VMT його класу
- VMT відображає назви методів на фактичні реалізації
- Пошук відбувається під час виконання (O(1) з хеш-таблицею або індексацією масиву)

#### 2.3.2 Ad-hoc поліморфізм (Перевантаження функцій)

Та сама назва методу, різні сигнатури. Розв'язується при **компіляції**:
```
class Calculator {
  add(a: int, b: int): int
  add(a: double, b: double): double
  add(a: string, b: string): string
}
```

#### 2.3.3 Параметричний поліморфізм (Generics)

Параметри типу дозволяють абстрактний код, що працює з кількома типами:
```
class Container<T> {
  private items: T[] = []
  
  add(item: T): void { this.items.push(item) }
  get(index: number): T { return this.items[index] }
}
```

Це фундаментально відрізняється від поліморфізму підтипу - той же шлях коду обробляє все типи, на відміну від різних реалізацій per тип.

### 2.4 Композиція vs. Спадкування

**Композиція (Has-A)**: Об'єкти містять інші об'єкти, делегуючи поведінку:

```
class Engine { start(): void }
class Car {
  private engine: Engine = new Engine()
  start(): void { this.engine.start() }
}
```

**Спадкування (Is-A)**: Об'єкти розширюють інші об'єкти, спадкуючи поведінку:

```
class Vehicle { start(): void }
class Car extends Vehicle { }
```

**Переваги композиції:**
1. Гнучка зміна поведінки під час виконання
2. Уникає крихкої проблеми базової класу
3. Множинні комбінації поведінки без множинного спадкування
4. Краща інкапсуляція (внутрішні об'єкти не розкриті)

**Переваги спадкування:**
1. Чистіші is-a відносини
2. Природний поліморфізм для ієрархії типів
3. Менш багатослівне для моделювання ієрархій

**Сучасна рекомендація**: *"Перевагу композиції над спадкуванням"* (Gang of Four)
- Використовуйте спадкування для **відносин типу тільки**
- Використовуйте композицію для **повторного використання та комбінації поведінки**

---

## 3. SOLID принципи: теорія та застосування

### 3.1 Single Responsibility Principle (SRP)

**Визначення**: Клас повинен мати одну, і тільки одну, причину для змін.

**Математична формулювання:**
$$R = \frac{\text{Кількість причин для змін}}{1} \leq 1$$

Клас має множинні відповідальності, якщо він має кілька причин для змін. Наприклад:

```
// ПОРУШУЄ SRP: Дві відповідальності
class User {
  saveToDatabase(): void { }  // Проблема persistence
  sendWelcomeEmail(): void { }  // Проблема комунікації
  validateEmail(): void { }  // Проблема валідації даних
}

// ВІДПОВІДАЄ SRP
class User {
  validateEmail(): void { }  // Тільки цілісність даних
}

class UserRepository {
  save(user: User): void { }  // Тільки persistence
}

class WelcomeEmailSender {
  send(user: User): void { }  // Тільки комунікація
}
```

**Метрика когезії:**
$$\text{Когезія} = \frac{\text{Методи що використовують спільні поля}}{\text{Всього методів}}$$

Вища когезія (методи що ділять стан) вказує на дотримання SRP.

**Вигода:**
- Зменшена зв'язаність: Зміни в persistence не впливають на email логіку
- Покращена тестованість: Кожна проблема тестується незалежно
- Підвищена повторюваність: Шар persistence можна використовувати в кількох контекстах
- Легша підтримка: Чіткі межі відповідальності

**Порушення на практиці:**
1. **God Classess**: Великі класи з багатьма відповідальностями
2. **Feature Envy**: Методи що звертаються до занадто багато зовнішнього стану
3. **Inappropriate Intimacy**: Класи що знають занадто багато один про одного

### 3.2 Open/Closed Principle (OCP)

**Визначення**: Сутності ПО повинні бути відкриті для розширення, але закриті для модифікації.

**Формальне вираження:**
Клас відповідає OCP якщо нова поведінка може бути додана без модифікації існуючого коду:
$$\text{OCP-відповідний} \iff \forall \text{ нові можливості } f, \text{ існуючий код без змін}$$

**Досягнення OCP через абстракцію:**

```
// ПОРУШУЄ OCP: Повинен модифікувати PaymentProcessor для кожного нового методу
class PaymentProcessor {
  process(method: string) {
    if (method === 'credit-card') { /* логіка кредитної карти */ }
    if (method === 'paypal') { /* логіка PayPal */ }
    if (method === 'crypto') { /* логіка крипто */ }  // НОВА: Повинна модифікувати!
  }
}

// ВІДПОВІДАЄ OCP: Нові методи без модифікації
interface PaymentMethod {
  process(amount: Money): Result<PaymentId>
}

class PaymentProcessor {
  private methods: Map<string, PaymentMethod> = new Map()
  
  register(name: string, method: PaymentMethod): void {
    this.methods.set(name, method)
  }
  
  process(methodName: string, amount: Money): Result<PaymentId> {
    return this.methods.get(methodName)?.process(amount) ?? fail()
  }
}

// НОВА: Додати без модифікації PaymentProcessor
class CryptoPayment implements PaymentMethod {
  process(amount: Money): Result<PaymentId> { /* логіка крипто */ }
}

processor.register('crypto', new CryptoPayment())
```

**Компроміс рівня абстракції:**
- **Занадто конкретна**: Сильно зв'язана, вимагає модифікації для кожної варіації
- **Занадто абстрактна**: Over-engineered, непотрібна непрямість
- **Оптимальна**: Абстрактна ровно на достатньому рівні для передбачуваних розширень

**Стратегії впровадження:**
1. **Template Method Pattern**: Абстрактна скелет алгоритму, підклащи заповнюють деталі
2. **Strategy Pattern**: Інкапсулювання варіацій алгоритму
3. **Decorator Pattern**: Динамічне додавання поведінки
4. **Dependency Injection**: Інжекціонування реалізацій замість хардкодування

### 3.3 Liskov Substitution Principle (LSP)

**Визначення**: Підтипи повинні бути замінні для своїх базових типів без порушення програми.

**Формальне твердження** (Barbara Liskov, 1987):
Якщо $S$ є підтипом $T$, то об'єкти типу $S$ можуть бути замінені об'єктами типу $T$ без порушення бажаних властивостей цієї програми.

**Збереження контракту:**
```
interface Shape {
  area(): number
  perimeter(): number
}

// ВІДПОВІДАЄ LSP
class Circle implements Shape {
  area() { return Math.PI * this.r * this.r }
  perimeter() { return 2 * Math.PI * this.r }
}

// ПОРУШУЄ LSP
class Square implements Shape {
  setWidth(w: number) { this.width = w; this.height = w }
  setHeight(h: number) { this.width = h; this.height = h }
  
  area() { return this.width * this.height }
  perimeter() { return 4 * this.width }
  
  // ПРОБЛЕМА: Викликаючий очікує незалежні width/height
  // Код що використовує Shape: shape.setWidth(5); shape.setHeight(3)
  // Очікування: width=5, height=3
  // Реальність (Square): width=3, height=3 (порушений інваріант!)
}
```

**Порушення контракту поведінки:**

| Тип порушення | Приклад | Вплив |
|---|---|---|
| Посилення передумови | Підклас вимагає більш обмежувальний вхід | Валідний вхід викликаючого відхилений |
| Послаблення постумови | Підклас дає слабші гарантії | Припущення викликаючого зламані |
| Порушення інваріанту | Підклас порушує класові інваріанти | Несподіване пошкодження стану |
| Додавання винятків | Підклас викидає несподівані винятки | Невідловлена виключення аварія |

**LSP в колекціях:**
```
// ПОРУШУЄ LSP
class CountdownList<T> extends ArrayList<T> {
  @Override add(element: T): boolean {
    // Додати на початок замість кінця
    this.items.unshift(element)
    return true
  }
}

// Викликаючий очікує поведінку ArrayList (FIFO)
let list: ArrayList<int> = new CountdownList()
list.add(1); list.add(2); list.add(3)
console.log(list[0])  // Очікував: 1, Отримав: 3 (LSP порушення!)
```

**Вигода:**
- Коректність програми гарантована через систему типів
- Безпека рефакторингу: Можна замінити реалізації без тестування всіх викликаючих
- Дозволяє поліморфні колекції без несподіванок під час виконання

### 3.4 Interface Segregation Principle (ISP)

**Визначення**: Клієнти не повинні залежати від інтерфейсів, які вони не використовують.

**Формальне вираження:**
$$\text{ISP-відповідний} \iff \forall \text{ клієнти } c, \text{ невикористані методи} = 0$$

**Anti-pattern надлишку залежностей:**

```
// ПОРУШУЄ ISP: Worker робить багато; Robots змушені залежати від всього
interface Worker {
  work(): void
  eat(): void
  sleep(): void
}

class Robot implements Worker {
  work(): void { /* робот працює */ }
  eat(): void { throw new Error('Роботи не їдять!') }
  sleep(): void { throw new Error('Роботи не спять!') }
}

// ВІДПОВІДАЄ ISP: Сегреговані інтерфейси
interface Workable {
  work(): void
}

interface Eatable {
  eat(): void
}

class Robot implements Workable {
  work(): void { /* робот працює */ }
}

class Human implements Workable, Eatable {
  work(): void { /* людина працює */ }
  eat(): void { /* людина їсть */ }
}
```

**Вигода ISP:**
1. **Зменшена зв'язаність**: Клієнти залежать тільки від потрібних методів
2. **Легше мокувати**: Подвійники тесту впроваджують тільки необхідну поведінку
3. **Краща документація**: Інтерфейс чітко показує очікуваний контракт
4. **Гнучкість**: Різні клієнти використовують різні підмножини інтерфейсу

**Розпізнавання anti-pattern:**
```
// Ознаки ISP порушення:
// 1. Великі інтерфейси (>6-8 методів)
// 2. NotImplementedException в реалізаціях
// 3. Методи що не відповідають основній відповідальності класу
```

### 3.5 Dependency Inversion Principle (DIP)

**Визначення**: 
1. High-level модулі не повинні залежати від low-level модулів. Обидва повинні залежати від абстракцій.
2. Абстракції не повинні залежати від деталей. Деталі повинні залежати від абстракцій.

**Візуальне представлення:**
```
// ПОРУШУЄ DIP: High-level залежить від low-level
BusinessLogic → DatabaseImpl → PhysicalDisk

// ВІДПОВІДАЄ DIP: Обидва залежать від абстракції
BusinessLogic → IRepository ← DatabaseImpl
                            ← FileSystemImpl
```

**Потік залежностей:**
```
// ANTI-PATTERN: Пряма зв'язаність
class OrderService {
  private mysql: MySQLDatabase = new MySQLDatabase()
  
  getOrder(id: string): Order {
    return this.mysql.query('SELECT * FROM orders WHERE id = ?', id)
  }
}

// PATTERN: Абстракція через DI
interface IRepository {
  getOrder(id: string): Promise<Order>
}

class OrderService {
  constructor(private repo: IRepository) { }
  
  async getOrder(id: string): Promise<Order> {
    return this.repo.getOrder(id)
  }
}
```

**Вигода:**
1. **Тестованість**: Інжекціонування mock repositories в тести
2. **Гнучкість**: Заміна реалізацій без змін business logic
3. **Підтримуваність**: Зміни у шарі persistence не каскадують
4. **Повторюваність**: Business logic незалежна від механізму сховища

**Практичне впровадження:**
```
// Виробництво
const repo = new MySQLRepository()
const service = new OrderService(repo)

// Тестування
const mockRepo = new MockRepository()
const service = new OrderService(mockRepo)

// Майбутнє: Зміна на document database
const repo = new MongoDBRepository()
const service = new OrderService(repo)  // Той же код!
```

---

## 4. Design Patterns та їх класифікація

### 4.1 Таксономія Gang of Four (GoF) Pattern

Design patterns - це **повторно використовувані рішення типових проблем** у об'єктно-орієнтованому дизайні. Каталогізовані Gang of Four (1994), вони поділяються на три категорії:

#### 4.1.1 Creational Patterns

Контролюють **механізми інстанціації об'єктів**:

**Singleton Pattern:**
```
class Configuration {
  private static instance: Configuration
  private constructor() { }
  
  static getInstance(): Configuration {
    if (!this.instance) {
      this.instance = new Configuration()
    }
    return this.instance
  }
}
```
- Використання: Координація одиночного екземпляру (логери, конфіг)
- Вартість: Глобальний стан, складність тестування, проблеми потокобезпеки

**Factory Pattern:**
```
interface AnimalFactory {
  create(): Animal
}

class DogFactory implements AnimalFactory {
  create(): Animal { return new Dog() }
}
```
- Використання: Розв'язати логіку створення від використання
- Вигода: Єдина точка для створення об'єкта

**Builder Pattern:**
```
class QueryBuilder {
  private query: Query = new Query()
  
  select(fields: string[]): QueryBuilder { this.query.fields = fields; return this }
  where(condition: string): QueryBuilder { this.query.condition = condition; return this }
  build(): Query { return this.query }
}

const q = new QueryBuilder()
  .select(['id', 'name'])
  .where('age > 18')
  .build()
```
- Використання: Складне конструювання об'єктів
- Вигода: Читабельний, текучий API

**Object Pool Pattern:**
```
class ConnectionPool {
  private available: Connection[] = []
  private inUse: Set<Connection> = new Set()
  
  acquire(): Connection {
    const conn = this.available.length > 0 ? this.available.pop() : new Connection()
    this.inUse.add(conn)
    return conn
  }
  
  release(conn: Connection): void {
    this.inUse.delete(conn)
    this.available.push(conn)
  }
}
```
- Використання: Повторне використання дорогих об'єктів (DB з'єднання, потоки)
- Вигода: Зменшене навантаження на розподіл

#### 4.1.2 Structural Patterns

Займаються **композицією об'єктів та відносинами**:

**Decorator Pattern:**
```
interface Component {
  operation(): string
}

class ConcreteComponent implements Component {
  operation(): string { return "Basic operation" }
}

abstract class Decorator implements Component {
  constructor(protected component: Component) { }
  operation(): string { return this.component.operation() }
}

class AuthDecorator extends Decorator {
  operation(): string {
    return `[Auth check] ${this.component.operation()}`
  }
}

const basic = new ConcreteComponent()
const decorated = new AuthDecorator(basic)
```
- Використання: Динамічно додати відповідальності
- Альтернатива до: Спадкування (більш гнучко)

**Adapter Pattern:**
```
interface TargetInterface {
  request(): string
}

class Adapter implements TargetInterface {
  constructor(private adaptee: IncompatibleInterface) { }
  
  request(): string {
    return this.adaptee.specificMethod()
  }
}
```
- Використання: Зробити несумісні інтерфейси сумісні
- Real-world: USB-C адаптер, конвертер напруги

**Facade Pattern:**
```
class LibraryFacade {
  private catalog: Catalog = new Catalog()
  private checkout: Checkout = new Checkout()
  private payment: Payment = new Payment()
  
  borrowBook(bookId: string, userId: string): Result {
    // Координувати складні взаємодії підсистеми
    const book = this.catalog.findBook(bookId)
    this.checkout.reserve(book, userId)
    const fee = this.payment.calculateFee(book, userId)
    return this.payment.charge(userId, fee)
  }
}
```
- Використання: Спростити складні підсистеми
- Вигода: Приховати внутрішню складність

**Proxy Pattern:**
```
interface Subject {
  request(): void
}

class RealSubject implements Subject {
  request(): void { console.log("Expensive operation") }
}

class Proxy implements Subject {
  private realSubject: RealSubject
  
  request(): void {
    if (this.isAccessAllowed()) {
      this.realSubject.request()
    }
  }
  
  private isAccessAllowed(): boolean { /* логіка аутентифікації */ }
}
```
- Використання: Контроль доступу, ледивого завантаження, кешування
- Приклад: Прокси DB з'єднання

#### 4.1.3 Behavioral Patterns

Контролюють **співпрацю об'єктів та відповідальність**:

**Strategy Pattern:**
```
interface SortingStrategy {
  sort(array: number[]): number[]
}

class QuickSort implements SortingStrategy {
  sort(array: number[]): number[] { /* быстрое сортировка */ }
}

class MergeSort implements SortingStrategy {
  sort(array: number[]): number[] { /* сортировка слиянием */ }
}

class Sorter {
  constructor(private strategy: SortingStrategy) { }
  
  sort(array: number[]): number[] {
    return this.strategy.sort(array)
  }
}
```
- Використання: Множинні реалізації алгоритму
- Вигода: Вибір алгоритму під час виконання

**Observer Pattern:**
```
interface Observer {
  update(data: any): void
}

class Subject {
  private observers: Observer[] = []
  
  attach(observer: Observer): void {
    this.observers.push(observer)
  }
  
  notify(data: any): void {
    this.observers.forEach(obs => obs.update(data))
  }
}
```
- Використання: Системи подій, реактивні оновлення
- Real-world: UI слухачі подій, pub/sub системи

**Command Pattern:**
```
interface Command {
  execute(): void
  undo(): void
}

class PrintCommand implements Command {
  constructor(private document: Document) { }
  
  execute(): void { this.document.print() }
  undo(): void { /* видалити з черги друку */ }
}

class Invoker {
  private history: Command[] = []
  
  execute(command: Command): void {
    command.execute()
    this.history.push(command)
  }
  
  undo(): void {
    this.history.pop()?.undo()
  }
}
```
- Використання: Undo/redo, черга транзакцій, макроси
- Вигода: Інкапсулювання запитів як об'єктів

**State Pattern:**
```
interface State {
  handle(context: Context): void
}

class OnState implements State {
  handle(context: Context): void {
    console.log("Already on")
    context.setState(new OnState())
  }
}

class Context {
  private state: State = new OffState()
  
  setState(state: State): void { this.state = state }
  
  request(): void { this.state.handle(this) }
}
```
- Використання: Об'єкти зі станозалежною поведінкою
- Приклад: Торговельний автомат, світлофор

---

## 5. Архітектурні паттерни та корпоративний дизайн

### 5.1 Layered Architecture

Найбільш поширений корпоративний паттерн. Організовує систему горизонтальними шарами:

```
┌─────────────────────────────────┐
│   Presentation Layer            │ (UI, API endpoints)
├─────────────────────────────────┤
│   Business Logic Layer          │ (Use cases, rules)
├─────────────────────────────────┤
│   Persistence Layer             │ (Database access)
├─────────────────────────────────┤
│   Infrastructure Layer          │ (Utilities, config)
└─────────────────────────────────┘
```

**Характеристики:**
- Кожен шар має специфічні відповідальності
- Шари спілкуються вертикально (presentation → business → persistence)
- Типово без пропуску шарів (presentation не повинна прямо звертатись до persistence)

**Переваги:**
- Простота розуміння та впровадження
- Чітке розділення проблем
- Легко тестувати кожен шар незалежно

**Недоліки:**
- Може призвести до "спагеті", якщо межі розмиваються
- Над послідовностю від перетину шарів
- Може заохочувати дизайн орієнтований на BD (anemic моделі)

### 5.2 Hexagonal Architecture (Ports and Adapters)

Альтернатива layered, наголошує ізоляцію домену:

```
        ┌────────────────────────────┐
        │   Domain Logic (Core)      │
        │   Business entities,       │
        │   use cases, rules         │
        └────────────────────────────┘
           ▲                    ▲
           │ Adapter            │ Adapter
           │                    │
    ┌──────┴──┐          ┌──────┴──┐
    │ Web API  │          │Database │
    │ Adapter  │          │Adapter  │
    └──────────┘          └──────────┘
```

**Ports**: Інтерфейси що визначають межі
**Adapters**: Реалізації специфічні для зовнішніх систем

**Переваги:**
- Логіка домену повністю незалежна від інфраструктури
- Дуже тестована (легко мокувати адаптери)
- Агностична щодо технології ядро
- Легко замінити реалізації

**Недоліки:**
- Складніше за layered
- Вимагає дисципльованої абстракції
- Може бути лишнім для простих систем

### 5.3 Event-Driven Architecture

Системи спілкуються через **события** замість прямих викликів:

```
┌──────────────┐         Event         ┌──────────────┐
│ Event Source │────────────────────────→ Event Handler│
└──────────────┘      (UserCreated)     └──────────────┘
                                        
                ┌─────────────────────────────────────┐
                │    Event Bus / Message Broker       │
                │  (RabbitMQ, Kafka, EventBridge)    │
                └─────────────────────────────────────┘
```

**Характеристики:**
- Компоненти спілкуються асинхронно через события
- Тимчасова розв'язаність: Продуценти не чекають на споживачів
- Event sourcing: Стан системи отримується з історії подій

**Переваги:**
- Масштабованість: Легко додати нових обробників
- Слаба зв'язаність: Компоненти не знають один про одного
- Журнал аудиту: События надають повну історію

**Недоліки:**
- Евентуальна консистентність: Важче гарантувати сильну консистентність
- Складність відлагодження: Потік важче відстежити
- Семантика at-least-once: Обробляти повторювані события

---

## 6. Injection залежностей та Inversion of Control

### 6.1 IoC контейнер

IoC контейнер управляє **життєвим циклом об'єктів та розв'язанням залежностей**:

```typescript
// Ручна DI
const db = new PostgresConnection()
const repo = new UserRepository(db)
const service = new UserService(repo)
const controller = new UserController(service)

// IoC контейнер (автоматична)
const container = new DIContainer()
container.register(UserService, { 
  dependencies: [UserRepository] 
})
container.register(UserRepository, { 
  dependencies: [Database] 
})

const service = container.resolve(UserService)
// Контейнер автоматично розв'язує залежності
```

**Вигода:**
1. **Зменшена boilerplate**: Контейнер обробляє проведення
2. **Централізована конфігурація**: Всі залежності в одному місці
3. **Управління життєвим циклом**: Singleton vs. transient vs. scoped екземплярів
4. **Легко тестувати**: Заміна реалізацій глобально

**Загальні IOC особливості:**
- **Singleton**: Один екземпляр за час життя контейнера
- **Transient**: Новий екземпляр кожного разу
- **Scoped**: Один екземпляр per запиту/scope
- **Factory функції**: Користувацька логіка інстанціації
- **Auto-wiring**: Автоматичне виявлення залежностей

### 6.2 Dependency Injection Patterns

#### 6.2.1 Constructor Injection (Рекомендована)

Залежності надаються через конструктор:
```typescript
class UserService {
  constructor(private repo: IUserRepository) { }
}
```

**Переваги:**
- Незмінні залежності
- Чіткі залежності (видимі в конструкторі)
- Неможливо використовувати неініціалізовану
- Чудово для тестування

**Недоліки:**
- Великі конструктори для багатьох залежностей (code smell)
- Виявлення циклічної залежності (час компіляції)

#### 6.2.2 Setter Injection

Залежності встановлені через властивості:
```typescript
class UserService {
  private repo: IUserRepository
  setRepository(repo: IUserRepository) {
    this.repo = repo
  }
}
```

**Переваги:**
- Опціональні залежності
- Без великих конструкторів
- Можна змінити залежності під час виконання

**Недоліки:**
- Може використовувати перед injection (помилки null reference)
- Залежності не відразу видимі
- Складніше тестувати

#### 6.2.3 Interface Injection

Спеціальний інтерфейс для реєстрації залежностей:
```typescript
interface InjectionTarget {
  inject(container: DIContainer): void
}

class UserService implements InjectionTarget {
  private repo: IUserRepository
  inject(container: DIContainer) {
    this.repo = container.resolve(IUserRepository)
  }
}
```

**Недоліки:**
- Багатослівно та інтрузивно
- Рідко використовується на практиці
- Менш типобезпечно від constructor injection

**Рекомендація**: Constructor injection для виробництва, property injection для опціональних cross-cutting стосунків.

---

## 7. GRASP принципи та розподіл відповідальності

GRASP (General Responsibility Assignment Software Patterns) надає евристики для призначення відповідальності класам:

### 7.1 Creator Pattern

**Хто повинен створити об'єкт X?**

**Евристика**: B повинен створити A якщо:
- B агрегує A
- B містить A
- B має дані необхідні для ініціалізації A
- B логує/стежить A

```typescript
// ДОБРЕ: OrderFactory знає як створити Order
class OrderFactory {
  createOrder(customerId: string, items: Item[]): Order {
    return new Order(customerId, items, DateTime.now())
  }
}

// ПОГАНО: Випадковий сервіс створює замовлення
class EmailService {
  sendOrderConfirmation(order: Order) {
    // Не повинен створювати замовлення тут
    const newOrder = new Order(...)
  }
}
```

### 7.2 Information Expert Pattern

**Хто повинен відповідати за дану задачу?**

**Евристика**: Призначити класу з більшою інформацією необхідною для виконання задачі.

```typescript
// ДОБРЕ: Order знає про товари та суми
class Order {
  getTotal(): Money {
    return this.items.reduce((sum, item) => sum + item.price, Money.zero())
  }
}

// ПОГАНО: Калькулятор не має контексту замовлення
class OrderCalculator {
  calculateTotal(items: Item[]): Money {
    // Слабкий дизайн - не повинен знати всі деталі товарів
    return items.reduce((sum, item) => sum + item.price, Money.zero())
  }
}
```

### 7.3 Low Coupling Pattern

**Як зменшити зв'язаність?**

**Евристика**: Мінімізувати залежності та абстрагувати.

```typescript
// ВИСОКА ЗБАЛАНІСТЬ: Прямі залежності на конкретних класах
class ReportGenerator {
  private mysql: MySQLDatabase = new MySQLDatabase()
  private fileWriter: LocalFileWriter = new LocalFileWriter()
  private emailer: GmailEmailer = new GmailEmailer()
}

// НИЗЬКА ЗBALАНІСТЬ: Залежності на абстракціях
class ReportGenerator {
  constructor(
    private db: IDatabase,
    private output: IReportOutput,
    private notifier: INotifier
  ) { }
}
```

### 7.4 High Cohesion Pattern

**Як зберегти класи сфокусованими?**

**Евристика**: Елементи повинні бути міцно пов'язані; відповідальності повинні бути пов'язані.

```typescript
// НИЗЬКА КОГЕЗІЯ: Змішані проблеми
class User {
  authenticate(): void { /* логіка аутентифікації */ }
  sendEmail(): void { /* логіка email */ }
  logActivity(): void { /* логіка логування */ }
  generateReport(): void { /* логіка звітності */ }
}

// ВИСОКА КОГЕЗІЯ: Сфокусована відповідальність
class User {
  authenticate(): void { /* логіка аутентифікації */ }
  getEmail(): string { }
  getId(): string { }
  // Все пов'язано з ідентичністю користувача та аутентифікацією
}
```

**Формула когезії:**
$$C = \frac{\sum \text{(методи що ділять поля)}}{\text{всіх пар методів}}$$

### 7.5 Polymorphism Pattern

**Як обробляти варіації на основі типу?**

**Евристика**: Використовувати поліморфізм замість switch виразів.

```typescript
// ANTI-PATTERN: Варіація на основі switch
class PaymentProcessor {
  process(type: string, amount: Money) {
    switch(type) {
      case 'credit': return this.creditCard(amount)
      case 'paypal': return this.paypal(amount)
      case 'crypto': return this.crypto(amount)  // Повинна модифікувати!
    }
  }
}

// PATTERN: Поліморфна варіація
interface PaymentMethod {
  process(amount: Money): Result
}

class PaymentProcessor {
  constructor(private method: PaymentMethod) { }
  
  process(amount: Money): Result {
    return this.method.process(amount)
  }
}
```

---

## 8. Виробничі системи та компроміси

### 8.1 Міркування про масштабованість

**Проблеми архітектури Vertical Layered:**
- Monolithic розгортання: Одна зміна вимагає повного розгортання
- Каскадні зміни: Модифікація шару впливає на всю систему
- Складність тестування: Потребує інтеграційного тестування для малих змін
- Зв'язаність з БД: Шар тісно зв'язаний з моделлю persistence

**Корпоративні рішення:**
- **Service-oriented architecture (SOA)**: Сервіси з чіткими межами
- **Microservices**: Fine-grained розкладання сервісів
- **Serverless**: Function-as-a-service, event-driven
- **CQRS**: Окремення модельей команд та запитів

### 8.2 Загальні виробничі anti-patterns

**Anemic Domain Model:**
```typescript
// ANTI-PATTERN: Об'єкти - просто контейнери даних
class Order {
  customerId: string
  items: OrderItem[]
  status: string
}

// PATTERN: Об'єкти інкапсулюють поведінку
class Order {
  private customerId: string
  private items: OrderItem[]
  private status: OrderStatus
  
  canCancel(): boolean { return this.status === OrderStatus.PENDING }
  cancel(): void { if (this.canCancel()) this.status = OrderStatus.CANCELLED }
}
```

**Leaky Abstractions:**
```typescript
// ANTI-PATTERN: БД витікає через абстракцію
interface Repository {
  find(id: string): Promise<Entity>  // Повертає концепцію БД
  executeSQL(sql: string): any       // Прямий SQL доступ
}

// PATTERN: Чиста абстракція
interface Repository {
  find(id: string): Promise<Entity>
  findByStatus(status: Status): Promise<Entity[]>
}
```

**Premature Abstraction:**
```typescript
// ANTI-PATTERN: Over-engineered для одного випадку використання
interface EmailProvider {
  send(email: Email): Promise<void>
}
class SMTPEmailProvider implements EmailProvider { }
class MockEmailProvider implements EmailProvider { }

// Коли тільки SMTP використовується. Краще:
class EmailService {
  send(email: Email): Promise<void> { /* SMTP */ }
}
```

### 8.3 Вплив на продуктивність

**Над ООП:**
| Операція | Вартість | Пом'якшення |
|-----------|---------|-----------|
| Virtual метод диспетчеризація | 1-5ns per виклик | JIT інлайнінг (сучасні VMs) |
| Розподіл об'єктів | 10-100ns | Object pooling |
| Шари абстракції | Latency per шар | Інлайн гарячі шляхи |
| Поліморфні колекції | Cache misses | Type спеціалізація |

**Виміряно на практиці:**
- Прямий метод виклик: ~1ns
- Virtual метод виклик: ~2-5ns
- Reflection-based доступ: ~100ns
- Interface диспетчеризація в поліморфних колекціях: ~5-20ns

Більша частина над незначна порівняно з I/O операціями (мережа: ~100,000ns, диск: ~1,000,000ns).

### 8.4 Стратегії тестування

**Unit тестування:**
```typescript
describe('UserService', () => {
  it('створює користувача з repository', () => {
    const mockRepo = new MockRepository()
    const service = new UserService(mockRepo)
    
    service.create({ name: 'John' })
    
    expect(mockRepo.created).toContain({ name: 'John' })
  })
})
```

**Integration тестування:**
```typescript
describe('UserService з реальною БД', () => {
  it('зберігає користувача в БД', async () => {
    const db = new TestDatabase()
    const repo = new PostgresRepository(db)
    const service = new UserService(repo)
    
    await service.create({ name: 'John' })
    
    const user = await db.query('SELECT * FROM users WHERE name = ?', 'John')
    expect(user).toBeDefined()
  })
})
```

---

## Ключові висновки

1. **Основи ООП**: Інкапсуляція, спадкування та поліморфізм формують основу для розширювальних, підтримуваних систем.

2. **SOLID принципи** надають дієві евристики:
   - SRP: Одна причина для змін per клас
   - OCP: Розширити без модифікації через абстракцію
   - LSP: Контракти поведінки важливі для типобезпеки
   - ISP: Клієнти не повинні залежати від невикористаних методів
   - DIP: Залежить від абстракцій, не конкретик

3. **Design Patterns** - це перевірені рішення типових проблем, класифіковані як:
   - **Creational**: Контроль інстанціації (Factory, Builder, Singleton)
   - **Structural**: Композиція об'єктів (Decorator, Adapter, Facade)
   - **Behavioral**: Співпраця об'єктів (Strategy, Observer, Command)

4. **Архітектурні паттерни** формують великі системи:
   - Layered: Простий, поширений, але схильний до размивання
   - Hexagonal: Domain-зосереджений, дуже тестований
   - Event-driven: Слабо зв'язаний, складне відлагодження

5. **Dependency Injection** через параметри конструктора - стандартний підхід для тестованості та гнучкості.

6. **GRASP принципи** керують розподілом відповідальності:
   - Creator, Information Expert, Low Coupling, High Cohesion, Polymorphism

7. **Виробничі компроміси**: Абстракція забезпечує гнучкість, але додає складність. Преждевременна абстракція гірше, ніж сильна зв'язаність спочатку; рефакторувати як вимоги стабілізуються.

8. **Тестування є першочерговим**: Сильна типізація та dependency injection дозволяють комплексне unit тестування, яке виявляє LSP порушення на ранніх етапах.

9. **Сучасна реальність**: Більшість систем об'єднують паттерни:
   - Layered архітектура + hexagonal ядро
   - Service-oriented + event-driven
   - Microservices + CQRS

10. **Мета-принцип**: *"Перевагу композиції над спадкуванням, залежить від абстракцій, та розподіліть відповідальність високо-когезивним, низько-зв'язаним класам."*
