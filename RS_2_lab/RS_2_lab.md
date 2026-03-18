## 📝 **ОТЧЕТ ПО ЛАБОРАТОРНОЙ РАБОТЕ №2**
### Вариант 1: HTTP-анализ github.com + API библиотеки книг + Nginx proxy



# 1. Цель работы
Изучение протокола HTTP, методов отправки запросов через curl, настройка веб-сервера Nginx в качестве обратного прокси, разработка REST API на Flask для управления библиотекой книг.



# 2. Теоретические основы
- **HTTP** - протокол прикладного уровня для передачи данных
- **REST API** - архитектурный стиль для создания веб-сервисов
- **Nginx** - веб-сервер и обратный прокси
- **Reverse Proxy** - сервер, который принимает запросы и перенаправляет их другим серверам



# 3. Часть 1. HTTP-анализ github.com

## 3.1 Анализ заголовков ответа

```bash
curl -I https://github.com
```

**Результат:**
```
HTTP/2 200 
server: GitHub.com
content-type: text/html; charset=utf-8
cache-control: no-cache
strict-transport-security: max-age=31536000; includeSubdomains; preload
```

## 3.2 Анализ редиректа с HTTP на HTTPS

```bash
curl -L -v http://github.com
```

**Результат:**
```
* Connected to github.com (140.82.121.3) port 80
> GET / HTTP/1.1
> Host: github.com
>
< HTTP/1.1 301 Moved Permanently
< Location: https://github.com/
```

**Анализ:** Сервер github.com перенаправляет все HTTP-запросы на HTTPS версию (код 301 Moved Permanently).

---

# 4. Часть 2. Разработка REST API "Библиотека книг"

## 4.1 Создание виртуального окружения и установка Flask

```bash
python3 -m venv venv
source venv/bin/activate
pip install flask
```

**Результат установки:**
```
Successfully installed flask-3.1.3
```

## 4.2 Код REST API (app.py)

```python
from flask import Flask, jsonify, request
from datetime import datetime

app = Flask(__name__)

# База данных в памяти
books = [
    {"id": 1, "title": "Война и мир", "author": "Лев Толстой"},
    {"id": 2, "title": "Преступление и наказание", "author": "Федор Достоевский"}
]
next_id = 3

@app.route('/api/books', methods=['GET'])
def get_books():
    """Получить список всех книг"""
    return jsonify(books)

@app.route('/api/books/<int:book_id>', methods=['GET'])
def get_book(book_id):
    """Получить книгу по ID"""
    book = next((b for b in books if b["id"] == book_id), None)
    if book:
        return jsonify(book)
    return jsonify({"error": "Книга не найдена"}), 404

@app.route('/api/books', methods=['POST'])
def create_book():
    """Создать новую книгу"""
    global next_id
    
    data = request.get_json()
    if not data or 'title' not in data or 'author' not in data:
        return jsonify({"error": "Поля title и author обязательны"}), 400
    
    new_book = {
        "id": next_id,
        "title": data["title"],
        "author": data["author"]
    }
    
    books.append(new_book)
    next_id += 1
    return jsonify(new_book), 201

@app.route('/api/books/<int:book_id>', methods=['DELETE'])
def delete_book(book_id):
    """Удалить книгу"""
    global books
    book = next((b for b in books if b["id"] == book_id), None)
    if book:
        books = [b for b in books if b["id"] != book_id]
        return jsonify({"message": "Книга удалена", "book": book})
    return jsonify({"error": "Книга не найдена"}), 404

if __name__ == "__main__":
    app.run(host="127.0.0.1", port=5000)
```

## 4.3 Запуск Flask сервера

**Команда:**
```bash
python3 app.py
```

**Вывод:**
```
 * Serving Flask app 'app'
 * Debug mode: off
 * Running on http://127.0.0.1:5000
```

## 4.4 Тестирование API напрямую (порт 5000)

**GET запрос:**
```bash
curl http://127.0.0.1:5000/api/books
```

**Ответ:**
```json
[{"author":"Лев Толстой","id":1,"title":"Война и мир"},{"author":"Федор Достоевский","id":2,"title":"Преступление и наказание"}]
```

**POST запрос:**
```bash
curl -X POST -H "Content-Type: application/json" \
  -d '{"title": "1984", "author": "Джордж Оруэлл"}' \
  http://127.0.0.1:5000/api/books
```

**Ответ:**
```json
{"author":"Джордж Оруэлл","id":3,"title":"1984"}
```

---

# 5. Часть 3. Настройка Nginx как обратного прокси

## 5.1 Установка Nginx

```bash
sudo apt update
sudo apt install nginx -y
sudo systemctl start nginx
sudo systemctl enable nginx
```

## 5.2 Настройка конфигурации Nginx

**Открытие конфигурации:**
```bash
sudo nano /etc/nginx/sites-available/default
```

**Итоговая конфигурация:**
```nginx
server {
    listen 80;
    server_name localhost;

    location /api/ {
        proxy_pass http://127.0.0.1:5000;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    }

    location / {
        root /var/www/html;
        index index.html;
    }
}
```
<img width="1259" height="384" alt="image" src="https://github.com/user-attachments/assets/08c185b7-2317-4008-9417-d97d9faebad0" />
Результат выолнения 

## 5.3 Проверка конфигурации и перезапуск

```bash
sudo nginx -t
```

**Результат:**
```
nginx: configuration file /etc/nginx/nginx.conf syntax is ok
nginx: configuration file /etc/nginx/nginx.conf test is successful
```

**Перезапуск:**
```bash
sudo systemctl restart nginx
```

---

# 6. Тестирование API через Nginx (порт 80)

## 6.1 GET запрос всех книг

```bash
curl -i http://localhost/api/books
```

**Ответ:**
```
HTTP/1.1 200 OK
Server: nginx/1.24.0 (Ubuntu)
Content-Type: application/json

[{"author":"Лев Толстой","id":1,"title":"Война и мир"},{"author":"Федор Достоевский","id":2,"title":"Преступление и наказание"},{"author":"Джордж Оруэлл","id":3,"title":"1984"}]
```

## 6.2 POST запрос (создание новой книги)

```bash
curl -X POST -H "Content-Type: application/json" \
  -d '{"title": "Мастер и Маргарита", "author": "Михаил Булгаков"}' \
  http://localhost/api/books
```

**Ответ:**
```json
{"author":"Михаил Булгаков","id":4,"title":"Мастер и Маргарита"}
```

## 6.3 GET запрос для проверки добавления

```bash
curl http://localhost/api/books | python3 -m json.tool
```

**Ответ (отформатированный):**
```json
[
    {
        "author": "Лев Толстой",
        "id": 1,
        "title": "Война и мир"
    },
    {
        "author": "Федор Достоевский",
        "id": 2,
        "title": "Преступление и наказание"
    }
]
```
<img width="548" height="354" alt="image" src="https://github.com/user-attachments/assets/55f3923e-8cab-4054-b68e-e14bf69b056a" />
Результат выолнения 

## 6.4 GET запрос книги по ID

```bash
curl http://localhost/api/books/1
```

**Ответ:**
```json
{"author":"Лев Толстой","id":1,"title":"Война и мир"}
```

## 6.5 DELETE запрос

```bash
curl -X DELETE http://localhost/api/books/1
```

**Ответ:**
```json
{"book":{"author":"Лев Толстой","id":1,"title":"Война и мир"},"message":"Книга удалена"}
```

---

# 7. Сравнение прямого доступа и через Nginx

| Параметр | Прямой доступ (порт 5000) | Через Nginx (порт 80) |
|----------|---------------------------|----------------------|
| URL | http://127.0.0.1:5000/api/books | http://localhost/api/books |
| Server | Werkzeug (Flask) | nginx/1.24.0 |
| Доступность | Только локально | Через стандартный порт HTTP |

---

# 8. Выводы
В ходе выполнения лабораторной работы были получены следующие навыки: **анализ HTTP-запросов и ответов** с помощью утилиты curl, **разработка REST API** на Flask для управления библиотекой книг, **настройка веб-сервера Nginx** в качестве обратного прокси, **конфигурирование proxy_pass** для перенаправления запросов с Nginx на Flask, а также **тестирование API** через различные эндпоинты (GET, POST, DELETE). В результате была создана полноценная клиент-серверная система, где Nginx выступает в роли **входного шлюза** (порт 80), а Flask-приложение реализует **бизнес-логику** (порт 5000). Таким образом, цель лабораторной работы достигнута.
