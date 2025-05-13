_Which command is used to clear the DNS cache in Windows?_  
**Відповідь:** a. ipconfig /flushdns  
**Пояснення:** Ця команда очищає DNS-кеш системи, що корисно при зміні DNS-записів або вирішенні проблем із доступом до сайтів через старі дані.

_Which tool is commonly used to test network connectivity by sending ICMP echo requests?_  
**Відповідь:** d. Ping  
**Пояснення:** Команда `ping` надсилає ICMP echo-запити до цільового хоста й очікує на відповідь, що дозволяє перевірити, чи доступний вузол у мережі та яка затримка з'єднання. Це один із найпростіших і найпоширеніших інструментів для первинної діагностики мережі.

_What does the "ARP" stand for in networking?_  
**Відповідь:** d. Address Resolution Protocol  
**Пояснення:** **ARP (Address Resolution Protocol)** — це мережевий протокол, який використовується для визначення MAC-адреси пристрою за його IP-адресою в локальній мережі. Він дозволяє комп’ютерам у мережі зв’язуватись через фізичні адреси, необхідні для передачі даних по Ethernet.

**Питання 14:**  
_Which tool is used to display the route that packets take to reach a destination IP address?_  
**Відповідь:**  
✅ **a. traceroute**  
✅ **e. mtr**

**Пояснення:**

- **`traceroute`** (в Linux/macOS) або **`tracert`** (у Windows) показує шлях, яким проходять пакети до IP-адреси, включаючи всі проміжні маршрутизатори.
    
- **`mtr`** — більш сучасний інструмент, який поєднує функціональність `ping` і `traceroute`, дає динамічний огляд шляху з додатковими метриками втрат і затримок.
**Питання 16:**  
_Which command can be used to test DNS resolution by querying a specific DNS server?_  
**Відповідь:** **c. `nslookup`**

**Пояснення:**  
Команда **`nslookup`** дозволяє виконувати DNS-запити до вказаного DNS-сервера. Це зручно для перевірки, чи правильно працює сервер і як він розв’язує доменні імена в IP-адреси.

📌 **Приклад:**

bash

CopyEdit

`nslookup google.com 8.8.8.8`

Цей запит виконує розв'язання імені `google.com` через DNS-сервер Google (8.8.8.8).