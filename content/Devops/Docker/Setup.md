# 🐳 Інсталяція Docker на Ubuntu — покроково

---

## 🔧 1. Підготовка системи

```bash
sudo apt-get update
sudo apt-get install ca-certificates curl gnupg
```

- `ca-certificates`: для HTTPS-з'єднань
- `curl`: інструмент для завантаження файлів
- `gnupg`: для перевірки підписів (GPG)

---

## 🔐 2. Додавання GPG-ключа Docker

```bash
sudo install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | \
  sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
sudo chmod a+r /etc/apt/keyrings/docker.gpg
```

- Створення директорії для ключів
- Завантаження і конвертація GPG-ключа Docker
- Налаштування прав доступу

---

## 📦 3. Додавання репозиторію Docker

```bash
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu \
  $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
```

- Додає офіційне джерело пакетів Docker до APT
- Вказується архітектура і кодова назва Ubuntu

---

## 🔄 4. Оновлення списку пакетів

```bash
sudo apt-get update
```

---

## 📥 5. Встановлення Docker Engine

```bash
sudo apt-get install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```

- `docker-ce`: Docker Engine
- `docker-ce-cli`: клієнт для Docker
- `containerd.io`: runtime
- `docker-buildx-plugin`: плагін для `docker buildx`
- `docker-compose-plugin`: плагін для `docker compose`

---

## 🚦 6. Перевірка сервісу Docker

```bash
systemctl status docker
```

- Перевірка, чи активний Docker (`active (running)`)

---

## 🧪 7. Тестування Docker

```bash
sudo docker run hello-world
```

- Завантажує тестовий образ
- Виводить повідомлення, якщо Docker працює правильно

---

✅ **Готово! Docker успішно встановлений.**
