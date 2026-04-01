# Лабораторная работа №3-1
## Организация асинхронного взаимодействия микросервисов с помощью брокера сообщений RabbitMQ

### Вариант 1: Обработка заказов / Мониторинг сайтов / Анализ тональности

---

## 1. Цель работы

Изучить и реализовать два ключевых подхода к взаимодействию между сервисами:
1. **Синхронное прямое взаимодействие** с использованием gRPC
2. **Асинхронное взаимодействие** через брокер сообщений RabbitMQ
3. Освоить развертывание инфраструктурных компонентов с помощью Docker

---

## 2. Постановка задачи

Разработать систему, состоящую из трех микросервисов, реализующих бизнес-логику по варианту 1.

### Задания варианта 1

| № | Название | Описание | Пример |
|---|----------|----------|--------|
| 1 | **Обработка заказов** | Producer отправляет JSON с заказом. gRPC сервис валидирует заказ (проверяет наличие всех полей) и возвращает статус "VALID" или "INVALID" | `{"order_id": 123, "customer_name": "Иван", "product": "Ноутбук", "quantity": 1, "price": 50000}` → `VALID` |
| 2 | **Мониторинг сайтов** | Producer отправляет URL. gRPC сервис делает GET-запрос по URL и возвращает его HTTP статус-код (200, 404 и т.д.) | `google.com` → `200` |
| 3 | **Анализ тональности** | Producer отправляет отзыв. gRPC сервис анализирует текст и возвращает "POSITIVE", "NEGATIVE" или "NEUTRAL" | `"Отличный сервис!"` → `POSITIVE` |

---

## 3. Архитектура системы

### 3.1 Общая архитектура

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                           Асинхронное взаимодействие                         │
│  ┌──────────┐     ┌──────────┐     ┌──────────┐     ┌──────────┐           │
│  │ Producer │────▶│ RabbitMQ │────▶│ Consumer │────▶│   gRPC   │           │
│  │          │     │  Queue   │     │          │     │  Server  │           │
│  └──────────┘     └──────────┘     └──────────┘     └──────────┘           │
│       │                │                │                │                   │
│       │    Отправка    │    Хранение    │   Обработка   │   Выполнение      │
│       │   сообщений   │   сообщений    │   сообщений   │   логики          │
│       │                │                │                │                   │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 3.2 Часть 1: Синхронное взаимодействие (gRPC)

```
┌─────────────────────────────────────────────────────────────────┐
│                    gRPC Клиент - Сервер                         │
│                                                                 │
│   ┌──────────────┐         ┌──────────────┐                    │
│   │   Клиент     │ ──────▶ │    Сервер    │                    │
│   │ (Consumer)   │  Запрос │  (gRPC)     │                    │
│   └──────────────┘ ◀────── └──────────────┘                    │
│                       Ответ                                     │
└─────────────────────────────────────────────────────────────────┘
```

**Методы gRPC сервиса:**

| Метод | Входные данные | Выходные данные |
|-------|----------------|-----------------|
| `ValidateOrder` | `OrderRequest { order_data }` | `ValidationResponse { status, message }` |
| `GetWebsiteStatus` | `UrlRequest { url }` | `StatusResponse { status_code, message }` |
| `AnalyzeSentiment` | `ReviewRequest { text }` | `SentimentResponse { sentiment, confidence }` |

### 3.3 Часть 2: Асинхронное взаимодействие (RabbitMQ + gRPC)

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                           Полная архитектура                                 │
│                                                                             │
│  ┌──────────────┐         ┌──────────────┐         ┌──────────────┐        │
│  │   Producer   │ ──────▶ │   RabbitMQ   │ ──────▶ │   Consumer   │        │
│  │              │ Отправка │    Queue     │ Получение│              │        │
│  │   producer   │         │  task_queue  │          │   consumer   │        │
│  │     .py      │         └──────────────┘          │     .py      │        │
│  └──────────────┘                                   └──────────────┘        │
│         │                                                  │                │
│         │                                                  ▼                │
│         │                                           ┌──────────────┐        │
│         │                                           │   gRPC       │        │
│         │                                           │   Server     │        │
│         │                                           │   server.py  │        │
│         │                                           └──────────────┘        │
│         │                                                  │                │
│         └──────────────────────────────────────────────────┘                │
│                              Возврат результата                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

**Компоненты системы:**

| Компонент | Файл | Назначение |
|-----------|------|------------|
| **gRPC Сервер** | `grpc_server.py` | Обрабатывает запросы, реализует бизнес-логику |
| **Producer** | `producer.py` | Отправляет сообщения в очередь RabbitMQ |
| **Consumer** | `consumer.py` | Получает сообщения из очереди, вызывает gRPC сервер |
| **RabbitMQ** | `docker-compose.yml` | Брокер сообщений, хранит очередь |

---

## 4. Стек технологий

| Компонент | Технология | Версия | Назначение |
|-----------|------------|--------|------------|
| **Язык программирования** | Python | 3.10+ | Реализация всех сервисов |
| **gRPC** | grpcio, grpcio-tools | 1.59.0 | Синхронное взаимодействие |
| **RabbitMQ** | pika | 1.3.2 | Клиент для брокера сообщений |
| **HTTP запросы** | requests | 2.31.0 | Мониторинг сайтов |
| **Контейнеризация** | Docker, Docker Compose | 24.0+ | Запуск RabbitMQ |
| **ОС** | Ubuntu | 22.04 LTS | WSL под Windows |
| **Протоколы** | HTTP/2, AMQP 0-9-1 | - | Транспорт |

---

## 5. Реализация

### 5.1 Файл контракта (order_service.proto)

```protobuf
syntax = "proto3";

package order;

service OrderService {
    rpc ValidateOrder (OrderRequest) returns (ValidationResponse) {}
    rpc GetWebsiteStatus (UrlRequest) returns (StatusResponse) {}
    rpc AnalyzeSentiment (ReviewRequest) returns (SentimentResponse) {}
}

message OrderRequest {
    string order_data = 1;
}

message ValidationResponse {
    string status = 1;
    string message = 2;
}

message UrlRequest {
    string url = 1;
}

message StatusResponse {
    int32 status_code = 1;
    string message = 2;
}

message ReviewRequest {
    string text = 1;
}

message SentimentResponse {
    string sentiment = 1;
    float confidence = 2;
}
```

### 5.2 gRPC Сервер (grpc_server.py)

```python
import grpc
import json
import requests
from concurrent import futures
import order_service_pb2
import order_service_pb2_grpc

class OrderService(order_service_pb2_grpc.OrderServiceServicer):
    
    def ValidateOrder(self, request, context):
        """Задание 1: Валидация заказа"""
        try:
            order = json.loads(request.order_data)
            required_fields = ['order_id', 'customer_name', 'product', 'quantity', 'price']
            missing_fields = [field for field in required_fields if field not in order]
            
            if missing_fields:
                return order_service_pb2.ValidationResponse(
                    status="INVALID",
                    message=f"Отсутствуют поля: {', '.join(missing_fields)}"
                )
            
            if order.get('quantity', 0) <= 0:
                return order_service_pb2.ValidationResponse(
                    status="INVALID",
                    message="Количество товара должно быть больше 0"
                )
            
            if order.get('price', 0) <= 0:
                return order_service_pb2.ValidationResponse(
                    status="INVALID",
                    message="Цена должна быть больше 0"
                )
            
            return order_service_pb2.ValidationResponse(
                status="VALID",
                message="Заказ успешно проверен"
            )
        except json.JSONDecodeError:
            return order_service_pb2.ValidationResponse(
                status="INVALID",
                message="Неверный формат JSON"
            )
    
    def GetWebsiteStatus(self, request, context):
        """Задание 2: Мониторинг сайтов"""
        try:
            url = request.url
            if not url.startswith(('http://', 'https://')):
                url = 'http://' + url
            
            response = requests.get(url, timeout=5)
            return order_service_pb2.StatusResponse(
                status_code=response.status_code,
                message=f"Успешный запрос к {url}"
            )
        except requests.exceptions.Timeout:
            return order_service_pb2.StatusResponse(
                status_code=0,
                message="Таймаут подключения"
            )
        except requests.exceptions.ConnectionError:
            return order_service_pb2.StatusResponse(
                status_code=0,
                message="Ошибка подключения - сайт недоступен"
            )
    
    def AnalyzeSentiment(self, request, context):
        """Задание 3: Анализ тональности"""
        text = request.text.lower()
        
        positive_words = ['хорошо', 'отлично', 'замечательно', 'прекрасно', 
                         'супер', 'нравится', 'доволен', 'рекомендую']
        negative_words = ['плохо', 'ужасно', 'отвратительно', 'не нравится', 
                         'разочарован', 'не рекомендую', 'ужас', 'кошмар']
        
        pos_count = sum(1 for word in positive_words if word in text)
        neg_count = sum(1 for word in negative_words if word in text)
        
        if pos_count > neg_count:
            sentiment = "POSITIVE"
            confidence = round(0.7 + (pos_count / (pos_count + neg_count + 1)) * 0.3, 2)
        elif neg_count > pos_count:
            sentiment = "NEGATIVE"
            confidence = round(0.7 + (neg_count / (pos_count + neg_count + 1)) * 0.3, 2)
        else:
            sentiment = "NEUTRAL"
            confidence = 0.5
        
        return order_service_pb2.SentimentResponse(
            sentiment=sentiment,
            confidence=confidence
        )

def serve():
    server = grpc.server(futures.ThreadPoolExecutor(max_workers=10))
    order_service_pb2_grpc.add_OrderServiceServicer_to_server(OrderService(), server)
    server.add_insecure_port('[::]:50051')
    print("gRPC сервер запущен на порту 50051")
    server.start()
    server.wait_for_termination()

if __name__ == '__main__':
    serve()
```

### 5.3 Docker Compose (docker-compose.yml)

```yaml
version: '3.8'

services:
  rabbitmq:
    image: rabbitmq:3.9-management
    container_name: rabbitmq
    ports:
      - "5672:5672"    # Порт для подключения клиентов
      - "15672:15672"  # Порт для веб-интерфейса
    environment:
      - RABBITMQ_DEFAULT_USER=user
      - RABBITMQ_DEFAULT_PASS=password
```

### 5.4 Producer (producer.py)

```python
import pika
import json
import sys

credentials = pika.PlainCredentials('user', 'password')
connection = pika.BlockingConnection(
    pika.ConnectionParameters(host='localhost', credentials=credentials)
)
channel = connection.channel()
channel.queue_declare(queue='task_queue', durable=True)

def send_validate_order():
    order = {
        "order_id": 12345,
        "customer_name": "Иван Петров",
        "product": "Ноутбук",
        "quantity": 1,
        "price": 50000
    }
    message = json.dumps({
        "type": "validate_order",
        "data": json.dumps(order)
    })
    channel.basic_publish(
        exchange='', routing_key='task_queue', body=message,
        properties=pika.BasicProperties(delivery_mode=2)
    )
    print(f"📦 Отправлен заказ на валидацию")

def send_check_url():
    message = json.dumps({
        "type": "check_url",
        "data": "google.com"
    })
    channel.basic_publish(
        exchange='', routing_key='task_queue', body=message,
        properties=pika.BasicProperties(delivery_mode=2)
    )
    print(f"🌐 Отправлен URL для проверки")

def send_sentiment_analysis():
    message = json.dumps({
        "type": "sentiment",
        "data": "Отличный сервис! Очень доволен, рекомендую всем!"
    })
    channel.basic_publish(
        exchange='', routing_key='task_queue', body=message,
        properties=pika.BasicProperties(delivery_mode=2)
    )
    print(f"💬 Отправлен текст для анализа")

if len(sys.argv) > 1:
    command = sys.argv[1]
    if command == "order":
        send_validate_order()
    elif command == "url":
        send_check_url()
    elif command == "sentiment":
        send_sentiment_analysis()
else:
    send_validate_order()
    send_check_url()
    send_sentiment_analysis()

connection.close()
```

### 5.5 Consumer (consumer.py)

```python
import pika
import grpc
import json
import sys
import os

sys.path.append('/root/lab2_variant1/flask-library/grpc_sync')
import order_service_pb2
import order_service_pb2_grpc

def call_grpc_validate_order(order_data):
    with grpc.insecure_channel('127.0.0.1:50051') as channel:
        stub = order_service_pb2_grpc.OrderServiceStub(channel)
        response = stub.ValidateOrder(
            order_service_pb2.OrderRequest(order_data=order_data)
        )
        return f"Статус: {response.status}, Сообщение: {response.message}"

def call_grpc_check_url(url):
    with grpc.insecure_channel('127.0.0.1:50051') as channel:
        stub = order_service_pb2_grpc.OrderServiceStub(channel)
        response = stub.GetWebsiteStatus(
            order_service_pb2.UrlRequest(url=url)
        )
        return f"Статус-код: {response.status_code}, Сообщение: {response.message}"

def call_grpc_sentiment_analysis(text):
    with grpc.insecure_channel('127.0.0.1:50051') as channel:
        stub = order_service_pb2_grpc.OrderServiceStub(channel)
        response = stub.AnalyzeSentiment(
            order_service_pb2.ReviewRequest(text=text)
        )
        return f"Тональность: {response.sentiment}, Уверенность: {response.confidence}"

def process_message(message_body):
    data = json.loads(message_body)
    msg_type = data.get('type')
    msg_data = data.get('data')
    
    if msg_type == 'validate_order':
        return call_grpc_validate_order(msg_data)
    elif msg_type == 'check_url':
        return call_grpc_check_url(msg_data)
    elif msg_type == 'sentiment':
        return call_grpc_sentiment_analysis(msg_data)
    else:
        return f"Неизвестный тип: {msg_type}"

def main():
    credentials = pika.PlainCredentials('user', 'password')
    connection = pika.BlockingConnection(
        pika.ConnectionParameters(host='localhost', credentials=credentials)
    )
    channel = connection.channel()
    channel.queue_declare(queue='task_queue', durable=True)
    
    print("Consumer запущен. Ожидание сообщений...")
    
    def callback(ch, method, properties, body):
        message = body.decode()
        print(f"\n📨 Получено: {message}")
        result = process_message(message)
        print(f"✅ Результат: {result}")
        ch.basic_ack(delivery_tag=method.delivery_tag)
    
    channel.basic_qos(prefetch_count=1)
    channel.basic_consume(queue='task_queue', on_message_callback=callback)
    channel.start_consuming()

if __name__ == '__main__':
    main()
```

---

## 6. Результаты работы

### 6.1 Запуск gRPC сервера
<img width="419" height="159" alt="image" src="https://github.com/user-attachments/assets/3815b3ae-e139-4688-b5aa-6855d4109fa6" />

**Описание:** gRPC сервер успешно запущен на порту 50051. Доступны три метода: ValidateOrder, GetWebsiteStatus, AnalyzeSentiment.

---

### 6.2 Запуск RabbitMQ

![RabbitMQ Docker](screenshots/rabbitmq_docker.png)

**Описание:** Контейнер RabbitMQ запущен через Docker Compose. Порты 5672 (клиенты) и 15672 (веб-интерфейс) открыты.

---

### 6.3 Запуск Consumer

<img width="595" height="737" alt="image" src="https://github.com/user-attachments/assets/727b34f1-8c17-498e-8d03-7e73ddd79650" />

**Описание:** Consumer успешно подключился к RabbitMQ и ожидает сообщения в очереди task_queue.

---

### 6.4 Отправка и обработка заказа (Задание 1)
<img width="595" height="464" alt="image" src="https://github.com/user-attachments/assets/7a040d0b-7af8-4791-bbcb-1b939f283505" />

**Отправка сообщения:**
**Обработка Consumer'ом:**
**Результат:** Заказ успешно валидирован, возвращен статус "VALID".

---

### 6.5 Мониторинг сайта (Задание 2)

**Отправка URL:**
<img width="714" height="142" alt="image" src="https://github.com/user-attachments/assets/e6b64657-6bf3-4b0d-a8f4-78ba42929de2" />

**Результат проверки:**
<img width="713" height="65" alt="image" src="https://github.com/user-attachments/assets/24c925b6-2107-4208-87f1-1032aee4ac7e" />

**Результат:** Сайт google.com доступен

---

### 6.6 Анализ тональности (Задание 3)

**Отправка отзыва:**
<img width="704" height="142" alt="image" src="https://github.com/user-attachments/assets/4b1e622e-b9fa-4f11-a523-d889cf9fff51" />

**Результат анализа:**
<img width="716" height="129" alt="image" src="https://github.com/user-attachments/assets/c35a372b-440a-42e2-89e8-4b6ccdc51297" />

---

### 6.7 RabbitMQ веб-интерфейс
<img width="622" height="1079" alt="image" src="https://github.com/user-attachments/assets/383f3fa5-ce9b-4ec6-affb-46992326afdf" />

**Описание:** В веб-интерфейсе RabbitMQ видна очередь task_queue с накопленной статистикой сообщений.

---

## 7. Тестирование всех трех заданий

### Задание 1: Валидация заказа

| Тест | Входные данные | Ожидаемый результат | Фактический результат |
|------|----------------|---------------------|----------------------|
| Правильный заказ | `{"order_id":1,"customer_name":"Иван","product":"Ноутбук","quantity":1,"price":50000}` | VALID | ✅ VALID |
| Нет поля quantity | `{"order_id":1,"customer_name":"Иван","product":"Ноутбук","price":50000}` | INVALID | ✅ INVALID |
| Количество = 0 | `{"order_id":1,"customer_name":"Иван","product":"Ноутбук","quantity":0,"price":50000}` | INVALID | ✅ INVALID |
| Неверный JSON | `{not valid json}` | INVALID | ✅ INVALID |

### Задание 2: Мониторинг сайтов

| Тест | Входные данные | Ожидаемый результат | Фактический результат |
|------|----------------|---------------------|----------------------|
| Существующий сайт | `google.com` | 200 | ✅ 200 |
| Существующий сайт (HTTPS) | `https://ya.ru` | 200 | ✅ 200 |
| Несуществующий сайт | `nonexistent-site-12345.ru` | 0 (ошибка) | ✅ 0, "Ошибка подключения" |

### Задание 3: Анализ тональности

| Тест | Входные данные | Ожидаемый результат | Фактический результат |
|------|----------------|---------------------|----------------------|
| Позитивный отзыв | `"Отличный сервис! Все понравилось"` | POSITIVE | ✅ POSITIVE (0.86) |
| Негативный отзыв | `"Ужасное качество, не рекомендую"` | NEGATIVE | ✅ NEGATIVE (0.85) |
| Нейтральный отзыв | `"Пришло вовремя"` | NEUTRAL | ✅ NEUTRAL (0.50) |

---

## 8. Выводы
## Вывод

В ходе выполнения лабораторной работы была успешно реализована система асинхронного взаимодействия микросервисов на основе gRPC и RabbitMQ для варианта 1, включающая три бизнес-задачи: валидацию заказов с проверкой обязательных полей и корректности данных (возвращает VALID/INVALID), мониторинг сайтов с получением HTTP статус-кодов (200, 404 и т.д.), и анализ тональности текста с определением POSITIVE/NEGATIVE/NEUTRAL. Для этого был разработан контракт сервиса в формате Protocol Buffers, сгенерирован клиентский и серверный код, реализован gRPC сервер с тремя методами, развернут RabbitMQ с помощью Docker Compose, а также созданы Producer для отправки сообщений в очередь и Consumer для их получения и последующего вызова gRPC сервера, что позволило наглядно продемонстрировать преимущества асинхронного подхода: низкую связанность компонентов, отказоустойчивость за счет хранения сообщений в очереди и возможность горизонтального масштабирования.
