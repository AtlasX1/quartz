# Основи мереж: Теоретичні основи та концепції виробництва

## Зміст
1. Мережні моделі та архітектура по шарах
2. IP адресація та основи маршрутизації
3. DNS: архітектура системи назв доменів
4. Протоколи транспортного шару (TCP та UDP)
5. Протоколи шару програм (HTTP/HTTPS)
6. Основи мережної безпеки
7. Доставка вмісту та балансування навантаження
8. Виробнича мережа та міркування продуктивності

---

## 1. Мережні моделі та архітектура по шарах

### 1.1 Модель OSI

**Модель Open Systems Interconnection (OSI)** - концептуальна структура, що визначає мережне спілкування в 7 шарах:

$$\text{Application} \rightarrow \text{Presentation} \rightarrow \text{Session} \rightarrow \text{Transport} \rightarrow \text{Network} \rightarrow \text{Data Link} \rightarrow \text{Physical}$$

Кожен шар надає сервіси шарові вище та використовує сервіси від шару нижче.

**Шар 1: Фізичний шар**
- Передача сирих потоків біт через фізичне середовище
- Характеристики: Рівні напруги, типи кабелів, частоти
- Обладнання: Hub'и, repeater'и, кабелі
- Приклад: Мідний дріт несе 0V (binary 0) та 5V (binary 1)

**Шар 2: Шар каналу даних**
- Надійне спілкування через локальний сегмент мережі
- Кадри з MAC адресами (Media Access Control)
- Обладнання: Комутатори, мости
- Протоколи: Ethernet, PPP, HDLC
- Функція: Синхронізація кадрів, виявлення помилок

**Шар 3: Мережний шар**
- Маршрутизація через мережі (IP)
- Логічна адресація (IP адреси)
- Обладнання: Маршрутизатори
- Протоколи: IP, ICMP, IGP, BGP
- Функція: Найкраще-вирішування перенесення пакетів

**Шар 4: Транспортний шар**
- End-to-end надійність спілкування
- Port-based сервіси
- Протоколи: TCP (надійний), UDP (ненадійний)
- Функція: Контроль потоку, виправлення помилок, мультиплексування

**Шар 5: Сеансовий шар**
- Контроль діалогу та синхронізація
- Встановлення сеансу, підтримання, завершення
- Протоколи: NetBIOS, RPC
- Функція: Управління розмовою half-duplex/full-duplex

**Шар 6: Шар представлення**
- Трансляція формату даних та шифрування
- Кодування символів (ASCII, Unicode)
- Стиснення та шифрування
- Функція: Трансляція даних між програмою та форматами мережі

**Шар 7: Шар програм**
- Сервіси користувача та програми
- Протоколи: HTTP, SMTP, FTP, DNS, SSH, Telnet
- Функція: Надає сервіси мережі безпосередньо до програм користувача

**Інкапсуляція даних (PDU на кожному шарі):**
```
Шар 7: Data
       ↓ (add application header)
Шар 6: APDU (Application Protocol Data Unit)
       ↓ (add presentation header)
Шар 5: PPDU
       ↓ (add session header)
Шар 4: SPDU → Segment (TCP) або Datagram (UDP)
       ↓ (add transport header)
Шар 3: TPDU → Packet (з IP header)
       ↓ (add network header)
Шар 2: NPDU → Frame (з Ethernet header)
       ↓ (add data link header)
Шар 1: Фізичні біти передаються через дріт
```

### 1.2 Модель TCP/IP (DoD модель)

Практичніша 4-шарова модель, що відображає фактичну архітектуру інтернету:

**Шар 1: Link шар**
- Комбінує фізичний та Data Link OSI
- Адресація апаратури (MAC)
- Фізична передача

**Шар 2: Internet шар**
- IP маршрутизація та логічна адресація
- Еквівалент до OSI мережного шару
- Протоколи: IP, ICMP, IGMP

**Шар 3: Транспортний шар**
- TCP (орієнтований на з'єднання, надійний)
- UDP (без з'єднання, ненадійний)
- Еквівалент до OSI транспортного шару

**Шар 4: Application шар**
- Комбінує OSI Session, Presentation, Application
- HTTP, SMTP, DNS, SSH, FTP, Telnet
- Сервіси користувача

**Порівняння відображення:**
```
OSI модель              TCP/IP модель
─────────────────────────────────────────
7. Application        │
6. Presentation       ├→ 4. Application
5. Session            │
4. Transport        → 3. Transport
3. Network          → 2. Internet
2. Data Link        │
1. Physical         ├→ 1. Link
```

### 1.3 Структура пакета та потік

**Інкапсуляція пакета через шари:**
```
Дані програм: "GET / HTTP/1.1\r\n..."
        ↓ (додати TCP header: sport=random, dport=80, seq, ack, flags)
TCP Segment: [TCP Header | HTTP Request]
        ↓ (додати IP header: src=192.168.1.10, dst=93.184.216.34)
IP Packet: [IP Header | TCP Header | HTTP Request]
        ↓ (додати Ethernet header: src_MAC=aa:bb:cc:dd:ee:ff, dst_MAC=gateway)
Ethernet Frame: [Ethernet | IP | TCP | Data] [CRC]
        ↓ (перетворити на біти та передати)
Фізичні біти: 101010110010101010...
```

**Шлях через мережу:**
```
Source Host:
  Application → Шар 4 (TCP додає port 80)
             → Шар 3 (IP додає адресу призначення)
             → Шар 2 (Ethernet додає MAC gateway)
             → Шар 1 (передати на дріт)

Router 1:
  Шар 1 → 2 (отримати Ethernet frame)
       → 2 (перевірити MAC назначения - це мій? так)
       → 3 (розглянути IP header, перевірити таблицю маршрутизації)
       → 3 (інкапсулювати в новий Ethernet frame до Router 2)
       → 1 (передати на вихідний інтерфейс)

Destination Host:
  Шар 1 → 2 (отримати frame, видалити Ethernet header)
       → 3 (отримати IP packet, перевірити IP призначення - це мій? так)
       → 4 (отримати TCP segment, перевірити port 80)
       → 7 (доставити HTTP request до web server)
```

---

## 2. IP адресація та основи маршрутизації

### 2.1 IPv4 адресація

**IPv4 адреса** - це 32-бітний ідентифікатор, представлений у **точкованій десятковій нотації**:

$$\text{Адреса} = \text{Біти мережі} + \text{Біти хосту}$$

**Приклад: 192.168.1.100**
```
Binary: 11000000.10101000.00000001.01100100
         ↑      ↑         ↑         ↑
         192    168       1         100
```

**Classful адресація (Legacy):**
```
Клас A: 0xxxxxxx.xxxxxxxx.xxxxxxxx.xxxxxxxx (1-126.x.x.x)
        1 byte мережі, 3 bytes хосту
        Мережі: 126, Хости per мережа: 16,777,214

Клас B: 10xxxxxx.xxxxxxxx.xxxxxxxx.xxxxxxxx (128-191.x.x.x)
        2 bytes мережі, 2 bytes хосту
        Мережі: 16,384, Хости per мережа: 65,534

Клас C: 110xxxxx.xxxxxxxx.xxxxxxxx.xxxxxxxx (192-223.x.x.x)
        3 bytes мережі, 1 byte хосту
        Мережі: 2,097,152, Хости per мережа: 254

Клас D: 1110xxxx (224-239.x.x.x) - Multicast
Клас E: 1111xxxx (240-255.x.x.x) - Reserved
```

**CIDR нотація (Classless Inter-Domain Routing):**
```
192.168.1.0/24
         └─ /24 означає перші 24 біти - мережа
            залишаючи 32-24=8 біти для хосту

Адреса мережі: 192.168.1.0 (біти хосту = 0)
Broadcast: 192.168.1.255 (біти хосту = всі 1s)
Використовувані хости: 192.168.1.1 до 192.168.1.254
Всього адрес: 2^8 = 256, використовуємо = 254
```

**Маска підмережі:**
```
CIDR /24 = Маска підмережі 255.255.255.0
           11111111.11111111.11111111.00000000

Щоб знайти адресу мережі:
192.168.1.100 AND 255.255.255.0 = 192.168.1.0

Щоб знайти біти хосту:
192.168.1.100 AND 0.0.0.255 = 0.0.0.100
```

**Спеціальні адреси:**
```
127.0.0.1 - Loopback (localhost)
0.0.0.0 - Ця мережа
255.255.255.255 - Broadcast
169.254.x.x - Link-local (APIPA)
224.0.0.0/4 - Multicast
```

### 2.2 Основи маршрутизації

**Процес маршрутизації:**
```
Пакет наступає на маршрутизатор:
  1. Вилучити IP призначення
  2. Консультуватись з таблицею маршрутизації для відповідного маршруту
  3. Знайти найдовший префіксний матч (найбільш специфічний маршрут)
  4. Отримати вихідний інтерфейс та next-hop gateway
  5. Інкапсулювати в Ethernet frame з next-hop MAC
  6. Перенести до next-hop
```

**Приклад таблиці маршрутизації:**
```
Destination       Netmask           Gateway         Interface
─────────────────────────────────────────────────────────────
192.168.1.0      255.255.255.0     192.168.1.1     eth0
192.168.2.0      255.255.255.0     192.168.1.254   eth0
0.0.0.0          0.0.0.0           192.168.1.1     eth0 (default)

Пакет до 192.168.2.50:
  Перевірити 192.168.2.50 & 255.255.255.0 = 192.168.2.0 ✓
  Gateway: 192.168.1.254
  Відправити до 192.168.1.254
```

**Алгоритм найдовшого префіксного матчу:**
```
Призначення: 10.1.2.3

Route 1: 10.0.0.0/8 (відповідає першим 8 бітам)
Route 2: 10.1.0.0/16 (відповідає першим 16 бітам) ← ВИБРАТИ ЦЕ
Route 3: 10.1.2.0/24 (відповідає першим 24 бітам) ← АБО ЦЕ (найбільш специфічна)

Найбільш специфічна виграє (найдовший префікс = /24)
```

**Distance Vector vs. Link State маршрутизація:**

**Distance Vector (RIP):**
- Кожен маршрутизатор знає: Distance (hop count) до кожного призначення
- Маршрутизатори обмінюються таблицями маршрутизації з сусідами
- "Маршрут через сусіда X, щоб досягти призначення D"
- Проблема: Повільна конвергенція, count-to-infinity

**Link State (OSPF):**
- Кожен маршрутизатор знає: Повна топологія мережі
- Маршрутизатори наводнюють інформацію про зв'язок усім маршрутизаторам
- Кожен маршрутизатор обчислює найкоротший шлях (алгоритм Dijkstra)
- Перевага: Швидка конвергенція, точна маршрутизація

---

## 3. DNS: архітектура системи назв доменів

### 3.1 DNS ієрархія та розв'язання

**DNS ієрархія:**
```
Root Name Servers (13 по всьому світу)
  ↓
Top-Level Domain (TLD) Servers (.com, .org, .edu)
  ↓
Authoritative Name Servers (власність власника домену)
```

**DNS процес розв'язання (Рекурсивна):**
```
Клієнт: Розв'язати example.com?
  ↓ (query)
Recursive Resolver (ISP DNS): Не в кеші, запитати root
  ↓ (query)
Root Server: Спробуйте TLD server для .com
  ↓ (referral до TLD)
TLD Server: Спробуйте authoritative для example.com
  ↓ (referral до authoritative)
Authoritative Server: example.com = 93.184.216.34
  ↓ (answer)
Recursive Resolver: Кеш результату, повернути до клієнта
  ↓ (answer)
Клієнт: example.com → 93.184.216.34
```

**Типи запитів:**
```
A Record: IPv4 адреса (example.com → 93.184.216.34)
AAAA Record: IPv6 адреса (example.com → 2606:2800:220:1...)
MX Record: Mail exchange (example.com → mail.example.com priority 10)
CNAME Record: Canonical name (alias → example.com)
NS Record: Name server (example.com authority → ns1.example.com)
TXT Record: Text data (використовується для SPF, DKIM, DMARC)
SOA Record: Start of Authority (zone метадані)
```

**DNS кешування:**
```
TTL (Time-To-Live): Як довго можна кешувати запис

Високий TTL (86400 seconds = 1 день):
  ✓ Менше запитів до DNS servers
  ✗ Повільніше поширювати зміни (1 день затримка)

Низький TTL (300 seconds = 5 хвилин):
  ✓ Швидке поширення змін
  ✗ Більше DNS запитів (вище навантаження)

Типово: 3600 seconds (1 година)
```

**Приклади DNS запиту:**
```
$ dig example.com
; <<>> DiG 9.10.6 <<>> example.com
; (1 server знайдений)
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR
;; flags: qr rd ra; QUERY: 1, ANSWER: 1, AUTHORITY: 3, ADDITIONAL: 0

;; QUESTION SECTION:
;example.com.                   IN      A

;; ANSWER SECTION:
example.com.            3600    IN      A       93.184.216.34

;; Query time: 53 msec
;; SERVER: 8.8.8.8#53(8.8.8.8)
```

---

## 4. Протоколи транспортного шару (TCP та UDP)

### 4.1 UDP (User Datagram Protocol)

**Характеристики:**
- Без з'єднання (немає handshake)
- Ненадійний (пакети можуть бути втрачені)
- Низька затримка (без контролю потоку або передачі)
- Мінімальний over (8-byte header)
- Без стану (без відстеження з'єднання)

**UDP Header:**
```
Source Port (16 bits)
Destination Port (16 bits)
Length (16 bits) = header + data
Checksum (16 bits) = опціональне виявлення помилок
Data (variable)
```

**Випадки використання:**
- DNS запити (потрібна швидка відповідь, низький over)
- Потокова передача відео (періодична втрата прийнятна)
- Online ігри (швидкі оновлення > точність)
- VoIP (real-time, втрата прийнятна)
- DHCP (простий пошук)

**UDP спілкування:**
```
Клієнт (port 50000):           Server (port 53)
  send("8.8.8.8") ─────────→
                   ←───────── recv("93.184.216.34")
  send("google.com") ────────→
                   ←───────── recv("142.251.32.142")
  (без з'єднання, може послати багато запитів без налаштування)
```

### 4.2 TCP (Transmission Control Protocol)

**Характеристики:**
- Орієнтований на з'єднання (three-way handshake)
- Надійний (гарантирована доставка, по порядку)
- Контроль потоку (запобігти переповнення приймача)
- Контроль перевантаження (поважати ємність мережі)
- Over: Вищий через послідовність/підтвердження

**Three-Way Handshake:**
```
Клієнт                                    Server
  │                                         │
  │─────── SYN (seq=1000) ──────────────→  │
  │                                    ACK отримано, послати SYN-ACK
  │  ←─── SYN-ACK (seq=5000, ack=1001) ─── │
  │  ACK отримано, з'єднання встановлено, послати ACK
  │─────── ACK (seq=1001, ack=5001) ──────→ │
  │                                    З'єднання встановлено
  │ ═══════════════════════════════════════ │
  │      З'єднано, можна обмінюватись дані  │
  │ ═══════════════════════════════════════ │
```

**TCP Header:**
```
Source Port (16 bits)
Destination Port (16 bits)
Sequence Number (32 bits) - Ідентифікує порядок даних
Acknowledgment Number (32 bits) - Підтверджує отримані дані
Flags: SYN, ACK, FIN, RST, PSH, URG
Window Size (16 bits) - Контроль потоку
Checksum (16 bits)
Urgent Pointer (16 bits)
Options (variable)
```

**Sequence Number та Acknowledgment:**
```
Клієнт надсилає 100 bytes з seq=1000:
  Bytes 1000-1099 передано
  
Server отримує та надсилає:
  ACK з ack=1100 (наступний очікуваний sequence)
  
Клієнт надсилає наступні 100 bytes з seq=1100:
  Bytes 1100-1199 передано
  
Якщо server виявляє відсутність byte 1050-1099:
  Все ще надсилає ack=1050 (останній отримано по порядку)
  
Клієнт передає bytes 1050-1099 знову
```

**Закінчення з'єднання (Four-Way Handshake):**
```
Клієнт                                    Server
  │                                         │
  │─────── FIN (seq=5000) ────────────────→ │
  │                                    Послати ACK
  │ ←─── ACK (ack=5001) ─────────────────── │
  │                              Закрити з'єднання, послати FIN
  │ ←─── FIN (seq=7000) ─────────────────── │
  │  Послати ACK
  │─────── ACK (ack=7001) ────────────────→ │
  │                                    З'єднання закрито
```

**TCP vs. UDP порівняння:**
```
                TCP                UDP
────────────────────────────────────────────
З'єднання      Oriented           Connectionless
Надійність    Guaranteed          Best effort
Упорядкування In-order            Без гарантії
Speed         Slower (over)       Faster
Over          High (20+ bytes)    Low (8 bytes)
Контроль потоку Yes                 No
Перевантаження Yes                 No
Використання   HTTP, SMTP, SSH     DNS, ігри, потокова
```

---

## 5. Протоколи шару програм (HTTP/HTTPS)

### 5.1 HTTP/1.1 основи

**Request-Response модель:**
```
Клієнт надсилає HTTP Request:
  Method (GET, POST, PUT, DELETE, HEAD, OPTIONS)
  URI (/index.html)
  HTTP Version (1.1)
  Headers (Host, User-Agent, Accept, etc.)
  Body (опціонально, для POST/PUT)

Server надсилає HTTP Response:
  Status Code (200, 404, 500, etc.)
  Reason Phrase (OK, Not Found, Internal Server Error)
  Headers (Content-Type, Content-Length, Server, etc.)
  Body (HTML, JSON, binary data, etc.)
```

**Приклад HTTP Request:**
```
GET /index.html HTTP/1.1
Host: example.com
User-Agent: Mozilla/5.0
Accept: text/html,application/xhtml+xml
Accept-Language: en-US,en;q=0.9
Accept-Encoding: gzip, deflate
Connection: keep-alive
```

**Приклад HTTP Response:**
```
HTTP/1.1 200 OK
Content-Type: text/html; charset=utf-8
Content-Length: 1234
Server: nginx/1.18.0
Cache-Control: public, max-age=3600
ETag: "abc123def456"
Set-Cookie: sessionid=abc123; Path=/; HttpOnly

<!DOCTYPE html>
<html>
  <head><title>Example Domain</title></head>
  <body>...</body>
</html>
```

**Codes стану:**
```
1xx: Informational (100 Continue, 101 Switching Protocols)
2xx: Success (200 OK, 201 Created, 204 No Content)
3xx: Redirection (301 Moved Permanently, 302 Found, 304 Not Modified)
4xx: Client Error (400 Bad Request, 401 Unauthorized, 403 Forbidden, 404 Not Found)
5xx: Server Error (500 Internal Server Error, 502 Bad Gateway, 503 Service Unavailable)
```

**HTTP методи:**
```
GET: Отримати ресурс (safe, idempotent)
POST: Створити ресурс (unsafe, not idempotent)
PUT: Оновити ресурс (idempotent)
DELETE: Видалити ресурс (idempotent)
PATCH: Partial оновлення (not idempotent)
HEAD: Як GET але без body (safe, idempotent)
OPTIONS: Описати можливості спілкування (safe, idempotent)
```

**HTTP/1.1 Keep-Alive:**
```
Без Keep-Alive:
  Request 1 → Response 1 → Закрити з'єднання
  Request 2 → Відкрити з'єднання → Response 2 → Закрити з'єднання
  Request 3 → Відкрити з'єднання → Response 3 → Закрити з'єднання
  (3 з'єднання, 3 handshakes = over)

З Keep-Alive (default у HTTP/1.1):
  Request 1 → Response 1 → Тримати з'єднання відкритим
  Request 2 → Response 2 → Тримати з'єднання відкритим
  Request 3 → Response 3 → Тримати з'єднання відкритим
  (1 з'єднання, 1 handshake = ефективне)
  
  Header: Connection: keep-alive
  Timeout: З'єднання закривається після простою часу
```

### 5.2 HTTPS (HTTP Secure)

**HTTPS = HTTP + TLS (Transport Layer Security)**

**TLS Handshake:**
```
Клієнт                                          Server
  │                                               │
  │─ ClientHello (TLS version, ciphers) ────────→│
  │                                        ServerHello + Certificate
  │←─────────────────────────────────────────────│
  │                                    ServerKeyExchange + ServerHelloDone
  │
  │─ ClientKeyExchange ──────────────────────────→│
  │   (ephemeral key + MAC verification)    │
  │─ ChangeCipherSpec ─────────────────────────→ │
  │ Finished                              ServerKeyExchange + Finished
  │←─────────────────────────────────────────────│
  │                                    ChangeCipherSpec
  │ ═════════════════════════════════════════════
  │    Зашифроване з'єднання встановлено
```

**Валідація цепі сертифіката:**
```
Server надсилає цеп сертифіката:
  1. Leaf Certificate (example.com)
     └─ Підписано Intermediate CA
        └─ Intermediate Certificate
           └─ Підписано Root CA
              └─ Root Certificate (довіряється ОС)

Клієнт валідує:
  1. Перевірити leaf certificate для example.com
  2. Верифікувати підпис (Intermediate підписав своїм private key)
  3. Верифікувати Intermediate підпис
  4. Верифікувати Root знаходиться в trust store
  ✓ Все валідно → Довіряти з'єднанню
  ✗ Будь-який невалідний → Відхилити з'єднання
```

**Cipher Suites:**
```
TLS_ECDHE_RSA_WITH_AES_256_GCM_SHA384
  │        │    │       │         │
  │        │    │       │         └─ Hash алгоритм (SHA384)
  │        │    │       └─────────── Шифрування (AES-256-GCM)
  │        │    └─────────────────── Key exchange (RSA)
  │        └─────────────────────── Key agreement (ECDHE - Perfect Forward Secrecy)
  └──────────────────────────────── Protocol (TLS)

Сучасна рекомендація:
  - Використовуйте ECDHE (Perfect Forward Secrecy)
  - Використовуйте AEAD (Authenticated Encryption: GCM, ChaCha20-Poly1305)
  - Вимкніть: RC4, DES, 3DES, MD5
```

**Perfect Forward Secrecy (PFS):**
```
З статичним RSA:
  Server long-term key скомпрометовано → Всі минулі sessions розшифровуються

З ECDHE (Ephemeral):
  Для кожного з'єднання:
    1. Server генерує ephemeral key pair
    2. Клієнт та server узгоджують ephemeral shared secret
    3. З'єднання зашифровано ephemeral secret
    4. Ephemeral key вилучається
    
  Якщо server long-term key скомпрометовано → Минулі sessions все ще безпечні
  (ephemeral keys були знищені)
```

---

## 6. Основи мережної безпеки

### 6.1 Поширені вектори атак

**Man-in-the-Middle (MITM):**
```
Атакуючий на тій же мережі:
  
Нормальний потік:
  Клієнт ──→ Server

MITM:
  Клієнт ──→ Атакуючий ──→ Server
       ▲                    │
       └────────────────────┘
  
Атакуючий бачить весь трафік, може модифікувати пакети.
Запобігання: HTTPS (TLS шифрування + аутентифікація)
```

**Packet Sniffing:**
```
Атакуючий з інструментом захоплення пакета (tcpdump, Wireshark):
  
HTTP трафік (незашифрований):
  GET /login?user=admin&pass=secret123

HTTPS трафік (зашифрований):
  Зашифрований blob (виглядає як випадкові дані)
  TLS забезпечує конфіденційність
```

**DNS Spoofing:**
```
Нормально:
  Клієнт: Що таке example.com?
  Server: 93.184.216.34

Spoof:
  Атакуючий перехоплює/відповідає першим:
  Клієнт: Що таке example.com?
  Атакуючий (видаючись за DNS): 192.168.1.100 (IP атакуючого!)
  Клієнт відвідує сайт атакуючого думаючи, це example.com
  
Запобігання: DNSSEC (криптографічно підпишіть DNS відповіді)
```

### 6.2 TLS та безпека сертифіката

**Pinning сертифіката:**
```
Під час розробки:
  Розробник програми фіксує (зберігає) public key сертифіката server

При підключенні:
  1. Нормальний TLS handshake
  2. Витягнути public key сертифіката server
  3. Порівняти з зафіксованим key
  4. Продовжувати тільки якщо keys збігаються
  
Запобігає: Скомпрометовані intermediate CAs видають підроблені сертифікати
Trade-off: Складно ротувати сертифікати (потрібне оновлення програми)
```

**Хешування та підписи:**
```
Server має:
  - Private key (секретна)
  - Public key (розповсюджується в сертифікаті)

Server підписує сертифікат:
  hash = SHA256(certificate_data)
  signature = RSA_sign(hash, private_key)
  
Клієнт верифікує:
  hash = SHA256(certificate_data)
  recovered_hash = RSA_verify(signature, public_key)
  якщо hash == recovered_hash:
    ✓ Сертифікат автентичний (підписаний власником private key)
```

---

## 7. Доставка вмісту та балансування навантаження

### 7.1 Content Delivery Networks (CDNs)

**Проблема:** Користувач в Сіднеї отримує вміст з сервера в Нью-Йорку
- Затримка: ~200ms (швидкість світла через fiber)
- Перевантаження: Закордонні канали дорогі та перевантажені
- User experience: Повільне завантаження сторінок

**CDN рішення:**
```
Origin Server (Нью-Йорк):
  └─ CDN Cache Layer (глобально розповсюджена)
     ├─ Edge Server (Сідней)
     ├─ Edge Server (Лондон)
     ├─ Edge Server (Токіо)
     └─ Edge Server (Сан-Паулу)

Користувач в Сіднеї:
  1. Request йде до найближчого edge server (Сідней)
  2. Якщо cache hit: Відповідь негайно (~5ms)
  3. Якщо cache miss: Отримати від origin, кешувати локально

Результат: 200ms → 5ms зменшення затримки
```

**Архітектура CDN:**
```
User request для example.com:
  1. DNS query для example.com
     ↓
  2. CNAME alias: example.com → cdn.example.com.cdnprovider.com
     ↓
  3. CDN DNS повертає IP найближчого edge server
     ↓
  4. User підключається до edge server
     ↓
  5. Edge server має вміст кешований (або отримує від origin)
     ↓
  6. User отримує вміст з швидкого, близького server
```

**Cache Headers для CDN:**
```
Cache-Control: public, max-age=3600
  - public: Будь-який кеш може зберігати
  - max-age=3600: Валідна 3600 секунд (1 година)
  
Cache-Control: private, max-age=0
  - private: Тільки browser кеш (не CDN)
  - max-age=0: Не кешувати (завжди отримувати свіжу)
  
ETag: "abc123def456"
  - Entity tag для валідації кеша
  - If-None-Match: "abc123def456"
    Server відповідає 304 Not Modified якщо без змін
```

### 7.2 Балансування навантаження

**Проблема:** Один server не може обробити мільйони requests

**Load Balancer:**
```
                    ┌─ Server 1 (port 8001)
Client 1 ─→         │
Client 2 ─→ Load ───┼─ Server 2 (port 8002)
Client 3 ─→ Balancer│
Client 4 ─→         │
            (public  └─ Server 3 (port 8003)
             IP)
             
Load balancer розподіляє requests через servers
```

**Алгоритми балансування:**

**Round Robin:**
```
Client 1 → Server 1
Client 2 → Server 2
Client 3 → Server 3
Client 4 → Server 1 (цикл повторюється)
```

**Найменші з'єднання:**
```
Server 1: 10 активних з'єднань
Server 2: 5 активних з'єднань ← Маршрут до цього
Server 3: 8 активних з'єднань
```

**Зважений Round Robin:**
```
Server 1: High-power машина, weight=3
Server 2: Low-power машина, weight=1

Розподіл: Server1:Server2:Server3 = 3:1
```

**IP Hash:**
```
hash(client_IP) % num_servers = server_index

Той же клієнт завжди маршрутизується до того же server (sticky)
Корисно для session state зберігається на servers
```

**Sticky Sessions:**
```
Клієнт A: Маршрут до Server 1 (перший request)
Клієнт A: Маршрут до Server 1 (наступні requests)
Клієнт B: Маршрут до Server 2 (перший request)
Клієнт B: Маршрут до Server 2 (наступні requests)

Впровадження:
  - Cookie з ID server
  - Load balancer перевіряє cookie, маршрутизує відповідно
  - Якщо server 1 down, повинен re-route та втратити session
  
Кращий підхід: Зберегти session у спільному кеші (Redis, Memcached)
```

**Перевірки здоров'я:**
```
Load balancer періодично перевіряє кожен server:
  
GET /health HTTP/1.1
Host: server1.internal

Response:
  200 OK {"status": "healthy"}
  
Якщо server не відповідає або повертає помилку:
  Позначити як unhealthy
  Припинити маршрутизацію requests до нього
  
Один раз здорів знову:
  Відновити маршрутизацію
```

---

## 8. Виробнича мережа та міркування продуктивності

### 8.1 Затримка та пропускна спроможність

**Затримка vs. Пропускна спроможність:**

$$\text{Затримка} = \text{Час для пакета досягти призначення}$$
$$\text{Пропускна спроможність} = \text{Максимальна швидкість передачі даних (bits/second)}$$

**Аналогія:**
- Затримка = Час подорожі від міста A до міста B
- Пропускна спроможність = Ширина автомагістралі (скільки машин одночасно)

**Типові затримки:**
```
L1 Cache hit: 1-2 ns
L2 Cache hit: 3-4 ns
L3 Cache hit: 10-20 ns
RAM доступ: 60-100 ns
Disk seek: 1-10 ms
Network round-trip (локальна): 1-10 ms
Network round-trip (intercontinental): 100-200 ms

Емпіричне правило:
  Кожна 10x повільніша = 10x більше затримка
```

**Композиція затримки для HTTP Request:**
$$\text{Total Latency} = \text{DNS lookup} + \text{TCP handshake} + \text{TLS handshake} + \text{HTTP request/response}$$

```
Приклад:
  DNS lookup: 50 ms (resolver query)
  TCP handshake: 30 ms (SYN, SYN-ACK, ACK)
  TLS handshake: 60 ms (ClientHello, ServerHello, etc.)
  HTTP request/response: 50 ms (network round-trip + processing)
  ──────────────────
  Всього: ~190 ms
```

**Розрахунок пропускної спроможності:**
```
100 Mbps internet з'єднання:
  100 bits за секунду = 12.5 MB за секунду
  
Завантажити 1 GB файл:
  1 GB / 12.5 MB/s = 80 секунд

Примітка: Real-world повільніше через:
  - Protocol over (IP, TCP headers)
  - Network перевантаження
  - Hardware обмеження
  - Реалістичні швидкості: 70-80% від теоретичної
```

### 8.2 Методики оптимізації мережі

**Оптимізація TCP:**

**Window Scaling:**
```
Default TCP window: 65,535 bytes (64 KB)
На high-latency посиланнях: Малий window = марна ємність

Приклад: 200 ms затримка, 100 Mbps:
  Bytes у польоті = 100 Mbps × 0.2 s = 2.5 MB
  Але window тільки 64 KB → Серйозно недовикористана
  
Рішення: Window scaling option
  Window size: 1 MB (дозволяє кращу утилізацію)
```

**Контроль перевантаження (CUBIC):**
```
Коли виявлена втрата пакета:
  1. Зменшити швидкість надсилання (congestion avoidance)
  2. Поступово збільшувати швидкість надсилання (recovery)
  3. Повторити
  
Сучасна: CUBIC алгоритм (замінює старіший Reno)
  - Швидкіша recovery
  - Краща справедливість
  - Обробляє high-BDP мережі
```

**Оптимізація DNS:**

**DNS Prefetching:**
```
HTML hint:
  <link rel="dns-prefetch" href="https://cdn.example.com">
  
Browser починає DNS lookup рано, економить 50-100 ms на першому request
```

**HTTP/2 та HTTP/3:**

**HTTP/2 Multiplexing:**
```
HTTP/1.1 (з keep-alive):
  Request 1 ──→ Response 1 ──→ Request 2 ──→ Response 2
  (послідовно)

HTTP/2 Multiplexing:
  Request 1 ──→ Response 1
  Request 2 ──→ Response 2
  (concurrent через одне з'єднання)
  
Вигода: Уникнути head-of-line blocking
Вартість: Одне з'єднання = одне congestion window
Trade-off: Краще для high-latency посилань
```

**HTTP/3 (QUIC):**
```
HTTP/2 поверх TCP:
  TCP надійність забезпечує упорядкування
  Якщо пакет втрачено → все з'єднання затримується
  
HTTP/3 поверх QUIC:
  QUIC мультиплексує streams
  Якщо stream 1 пакет втрачено → тільки stream 1 впливається
  Інші streams продовжуються
  
Вигода: Краще для lossy мереж (4G/5G мобільні)
Вартість: Складніше впровадження
```

### 8.3 Моніторинг продуктивності

**Ключові метрики:**
```
Затримка (p50, p95, p99):
  p50: 50th percentile (медіана)
  p99: 99th percentile (tail затримка)
  
  p50 = 10ms (типовий користувач)
  p99 = 500ms (незадоволені користувачі)
  
Пропускна спроможність:
  Requests за секунду (RPS)
  Kilobytes за секунду (KB/s)
  
Error Rate:
  Failed requests / Total requests
  Прийнятне: < 0.1%
  
Bandwidth Utilization:
  Used / Available
  Тримати нижче 70% (простір для spikes)
```

**Інструменти моніторингу:**
```
Аналіз пакетів: tcpdump, Wireshark
Мережні шляхи: traceroute, mtr
DNS розв'язання: nslookup, dig
Підключення: ping, nc (netcat)
Пропускна спроможність: iperf, speedtest
```

---

## Ключові висновки

1. **Модель OSI**: 7 шарів (Physical → Application) забезпечують концептуальну структуру. TCP/IP - практична 4-шарова модель.

2. **IP адресація**: IPv4 32-бітні адреси, CIDR нотація для subnetting. Маршрутизація через найдовший префіксний матч через таблиці маршрутизації.

3. **DNS**: Ієрархічна система (Root → TLD → Authoritative) розв'язує домени на IPs. Кешування критичне для продуктивності.

4. **Транспортний шар**:
   - **UDP**: Ненадійний, швидкий, без з'єднання (DNS, ігри)
   - **TCP**: Надійний, повільніший, орієнтований на з'єднання (HTTP, email)

5. **HTTP/1.1**: Request-response модель, keep-alive для ефективності, status codes для результатів.

6. **HTTPS**: HTTP + TLS шифрування. Handshake встановлює шифрування, сертифікати забезпечують аутентифікацію.

7. **Безпека**:
   - MITM запобігнута HTTPS
   - DNS spoofing запобігнута DNSSEC
   - Certificate pinning для додаткової безпеки

8. **CDNs**: Кеш вмісту глобально біля користувачів, драматично зменшити затримку (200ms → 5ms).

9. **Балансування навантаження**: Розподіліти requests через servers (round-robin, найменші з'єднання, IP hash, sticky sessions).

10. **Продуктивність**: 
    - Композиція затримки: DNS + TCP + TLS + HTTP
    - Пропускна спроможність vs. затримка (обидва важливі)
    - TCP window scaling, контроль перевантаження
    - HTTP/2 multiplexing, HTTP/3 QUIC

11. **Моніторинг**: Стежити p50/p99 затримка, пропускна спроможність, error rates, bandwidth utilization.

12. **Сучасні протоколи**: HTTP/2 для web, HTTP/3 для мобільні, TLS 1.3 для безпеку.
