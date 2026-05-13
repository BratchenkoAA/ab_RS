

# Лабораторная работа №4. Реализация механизмов безопасности в распределенной системе

## Цель работы

Разработать распределенную систему, обеспечивающую защищенную передачу данных с использованием взаимной аутентификации (mTLS) и симметричного шифрования, а также продемонстрировать механизмы отказоустойчивости (failover) через автоматическое переключение между узлами.

---

## Задачи

| № | Задача | Описание |
|:-:|--------|----------|
| 1 | **Настройка PKI** | Сгенерировать сертификаты X.509 для центра сертификации (CA), серверов и клиентов. |
| 2 | **Обеспечение безопасности** | Реализовать HTTPS-соединения с взаимной аутентификацией (mTLS) и шифрование полезной нагрузки алгоритмом Fernet. |
| 3 | **Разработка компонентов** | Реализовать серверы обработки данных, клиентское приложение и координатор (балансировщик нагрузки). |
| 4 | **Реализация отказоустойчивости** | Настроить логику автоматического перенаправления запросов на резервный сервер при недоступности основного. |
| 5 | **Выполнение индивидуального задания** | Расширить функциональность системы согласно варианту. |

---

## Индивидуальное задание (Вариант 1)

> **Метод получения данных (get_data)**  
> Реализовать метод `get_data()` на сервере и обновить клиент для его использования. Сервер должен возвращать реальные данные в зависимости от запроса клиента (info, status, version и т.д.).

---

## Схема потоков данных

<img width="3866" height="993" alt="deepseek_mermaid_20260513_c5ac86" src="https://github.com/user-attachments/assets/bea98889-de32-4bb4-8199-6edd8827c756" />

---

## 1. Генерация сертификатов (PKI)

### Скрипт `generate_certificates.sh`

```bash
#!/bin/bash
echo "=== Генерация PKI для mTLS ==="

# 1. Генерация ключа и сертификата CA
openssl genrsa -out ca_key.pem 2048
openssl req -new -x509 -days 365 -key ca_key.pem -out ca_cert.pem -subj "/CN=CA"

# 2. Генерация серверного ключа и сертификата
openssl genrsa -out server_key.pem 2048
openssl req -new -key server_key.pem -out server_csr.pem -subj "/CN=localhost"
openssl x509 -req -days 365 -in server_csr.pem -CA ca_cert.pem -CAkey ca_key.pem -CAcreateserial -out server_cert.pem

# 3. Генерация клиентского ключа и сертификата
openssl genrsa -out client_key.pem 2048
openssl req -new -key client_key.pem -out client_csr.pem -subj "/CN=client"
openssl x509 -req -days 365 -in client_csr.pem -CA ca_cert.pem -CAkey ca_key.pem -out client_cert.pem

# 4. Удаление временных файлов
rm -f server_csr.pem client_csr.pem ca.srl

echo "=== Готово ==="
```

### Запуск скрипта

```bash
chmod +x generate_certificates.sh
./generate_certificates.sh
```

### Созданные файлы сертификации

<img width="859" height="158" alt="image" src="https://github.com/user-attachments/assets/6942f420-a20f-4b8f-910f-0954764a4724" />

| Файл | Назначение |
|------|------------|
| `ca_cert.pem` | Публичный сертификат центра сертификации |
| `ca_key.pem` | Закрытый ключ CA (секретно) |
| `server_cert.pem` | Сертификат сервера |
| `server_key.pem` | Закрытый ключ сервера |
| `client_cert.pem` | Сертификат клиента |
| `client_key.pem` | Закрытый ключ клиента |

---

## 2. Генерация ключа шифрования Fernet

### Скрипт `generate_key.py`

```python
from cryptography.fernet import Fernet

key = Fernet.generate_key()
with open('encryption_key.txt', 'wb') as f:
    f.write(key)
print("Key generated and saved to encryption_key.txt")
```

### Запуск скрипта

```bash
python3 generate_key.py
```

---

## 3. Установка необходимых библиотек Python

```bash
pip install flask requests cryptography
```

| Библиотека | Назначение |
|------------|-------------|
| **Flask** | Микрофреймворк для создания веб-приложений. Используется для реализации координатора и серверов. |
| **requests** | HTTP-библиотека для отправки запросов. Используется клиентом и координатором. |
| **cryptography** | Набор криптографических средств. Используется для Fernet (симметричное шифрование). |

---

## 4. Запуск системы (4 терминала)

### Терминал 1: Запуск основного сервера (порт 5001)

<img width="268" height="305" alt="image" src="https://github.com/user-attachments/assets/0197f73f-c96e-4365-98bf-69a551d0ef71" />

```bash
python3 server.py 5001
```

### Терминал 2: Запуск резервного сервера (порт 5002)

<img width="414" height="221" alt="image" src="https://github.com/user-attachments/assets/2ca68394-8bc0-455e-bae3-ba8f75e36a8d" />

```bash
python3 server.py 5002
```

### Терминал 3: Запуск координатора (порт 8000)

<img width="420" height="132" alt="image" src="https://github.com/user-attachments/assets/0b4bff8b-ffb2-4e14-b867-0bb7977c9013" />

```bash
python3 coordinator.py
```

---

## 5. Индивидуальное задание (Вариант 1): Реализация метода get_data()

На сервере была создана база данных (словарь) с учебными данными:

```python
FAKE_DB = {
    "info": "Distributed system with mTLS and Fernet",
    "status": "active",
    "version": "1.0"
}
```

Метод `get_data()` обрабатывает входящий запрос:

```python
if decrypted.startswith("get:"):
    item = decrypted[4:]
    result = FAKE_DB.get(item, f"Not found: {item}")
    return {'result': result}
```

Клиент отправляет зашифрованные команды вида `get:info`, `get:status`, `get:version`.

<img width="378" height="87" alt="image" src="https://github.com/user-attachments/assets/701edb09-2097-49c4-9734-c04e6a1470b9" />

*Запрос в терминале клиента*

<img width="356" height="57" alt="image" src="https://github.com/user-attachments/assets/70f74d6b-9ced-4ebc-91de-ab81d7a73025" />

*Ответ сервера (Терминал 1)*

---

## 6. Запуск клиента и вывод результатов

```bash
python3 client.py
```

### Результат выполнения

| Запрос | Ответ |
|--------|-------|
| `get:info` | `{'result': 'Distributed system with mTLS and Fernet'}` |
| `get:status` | `{'result': 'active'}` |
| `get:version` | `{'result': '1.0'}` |
| `get:unknown` | `{'result': 'Not found: unknown'}` |

---

## 7. Демонстрация отказоустойчивости (Failover)

### Штатный режим работы

<img width="429" height="429" alt="image" src="https://github.com/user-attachments/assets/921cec19-53af-4458-9b23-c597b58be0b4" />

*Лог сервера при успешном запросе: `[SERVER] Received: get:info`*

### Имитация отказа основного сервера

**1. Остановка сервера на порту 5001 (Ctrl+C в терминале 1)**

<img width="434" height="82" alt="image" src="https://github.com/user-attachments/assets/8bbf49f8-a48a-488e-b90e-15125fa59814" />

**2. Повторный запуск клиента**

<img width="385" height="85" alt="image" src="https://github.com/user-attachments/assets/7870540e-19fa-4be4-9f81-32d5ae1a960b" />

**3. Лог координатора при отказе**

<img width="442" height="265" alt="image" src="https://github.com/user-attachments/assets/aa66e1fe-5973-430b-810e-39c2a1660da1" />

Клиент успешно получает ответ от резервного сервера 5002.

---

## Итоги выполнения

| Проверка | Результат |
|----------|-----------|
| Запуск сервера 5001 | ✅ Успешно |
| Запуск сервера 5002 | ✅ Успешно |
| Запуск координатора (8000) | ✅ Успешно |
| Отправка запроса `get:info` | ✅ Статус 200, данные получены |
| Отправка запроса `get:status` | ✅ Статус 200, данные получены |
| Отправка запроса `get:version` | ✅ Статус 200, данные получены |
| Отправка запроса `get:unknown` | ✅ Обработка ошибки |
| Отказоустойчивость (отказ 5001) | ✅ Автоматическое переключение на 5002 |

---

## Выводы

В ходе выполнения лабораторной работы была разработана распределенная система, обеспечивающая защищенную передачу данных с использованием взаимной аутентификации (mTLS) и симметричного шифрования Fernet, а также отказоустойчивость через автоматическое переключение (failover) между основным (порт 5001) и резервным (порт 5002) серверами при недоступности основного узла. В рамках индивидуального задания (Вариант 1) реализован метод `get_data()`, который позволяет клиенту отправлять зашифрованные запросы вида `get:info`, `get:status`, `get:version` и получать от сервера соответствующие реальные данные, а также корректно обрабатывать запросы несуществующих ключей. Дополнительно была произведена автоматизация генерации сертификатов с помощью bash-скрипта, что упрощает развёртывание системы на любом окружении.

---

## Структура проекта

```
lab4_variant1/
├── server.py                 # Сервер обработки данных
├── client.py                 # Клиентское приложение
├── coordinator.py            # Балансировщик нагрузки
├── generate_certificates.sh  # Скрипт генерации сертификатов
├── generate_key.py           # Скрипт генерации ключа Fernet
├── requirements.txt          # Зависимости Python
├── README.md                 # Отчёт
│
├── ca_cert.pem               # Сертификат CA
├── ca_key.pem                # Ключ CA
├── server_cert.pem           # Сертификат сервера
├── server_key.pem            # Ключ сервера
├── client_cert.pem           # Сертификат клиента
├── client_key.pem            # Ключ клиента
└── encryption_key.txt        # Ключ Fernet
```

---

## Использованные технологии

| Технология | Применение |
|------------|------------|
| **mTLS** | Взаимная аутентификация клиента и сервера |
| **Fernet** | Симметричное шифрование (AES-128-CBC + HMAC-SHA256) |
| **X.509** | Инфраструктура открытых ключей (PKI) |
| **Failover** | Автоматическое переключение при отказе узла |
| **Flask** | Веб-фреймворк для серверов и координатора |
| **OpenSSL** | Генерация сертификатов |
```
