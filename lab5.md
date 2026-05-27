
# Лабораторная работа №5
## Проектирование и реализация комплексной микросервисной системы для автоматизации бизнес-процесса

**Выполнила:** Братченко А.А.  
**Вариант:** №1  
**Тема:** Учет билетов

---

##  Цель работы

Научиться запускать многоконтейнерные приложения, организовывать взаимодействие между сервисами, использовать Docker Compose для оркестрации, изменять бизнес-логику и инфраструктуру проекта, работать с Redis как с внешним сервисом хранения данных.

---

##  Задание (вариант №1)

В соответствии с моим вариантом я выполнила три изменения:

1. **Бизнес-логика (app.py):** изменила заголовок на "Учет билетов" и цвет текста на синий (`color:blue`)
2. **Инфраструктура (docker-compose.yml):** сменила внешний порт с 8000 на **8081**
3. **Среда сборки (Dockerfile):** добавила метку `LABEL maintainer="Братченко А.А."`

---

##  Архитектура проекта

| Компонент | Технология | Порт | Роль |
|-----------|------------|------|------|
| Web | Flask + Python 3.9-alpine | 8081 | Обработка запросов, увеличение счётчика |
| Redis | redis:alpine | 6379 | Хранение значения счётчика |

**Схема взаимодействия:**  
Пользователь → браузер → порт 8081 → контейнер web (Flask) → обращение к Redis (порт 6379) → увеличение счётчика → ответ пользователю

---

##  Состав проекта

| Файл | Назначение |
|------|------------|
| `app.py` | Основное приложение на Flask |
| `Dockerfile` | Сборка образа (python:3.9-alpine) |
| `docker-compose.yml` | Оркестрация сервисов |
| `requirements.txt` | Зависимости Python |
| `README.md` | Документация |

---

## 🔧 Содержание файлов

### 1. `requirements.txt`

```
Flask==2.0.1
Werkzeug==2.3.7
redis==4.6.0
```

### 2. `app.py`

```python
import time
import redis
from flask import Flask

app = Flask(__name__)

cache = redis.Redis(host='redis', port=6379)

def get_hit_count():
    retries = 5
    while True:
        try:
            return cache.incr('hits')
        except redis.exceptions.ConnectionError as exc:
            if retries == 0:
                raise exc
            retries -= 1
            time.sleep(0.5)

@app.route('/')
def hello():
    count = get_hit_count()
    return '''
    <h1 style="color:blue">Учет билетов</h1>
    <p>Продано билетов: <strong>{}</strong></p>
    '''.format(count)

if __name__ == "__main__":
    app.run(host="0.0.0.0", debug=True)
```

### 3. `Dockerfile`

```dockerfile
FROM python:3.9-alpine

LABEL maintainer="Братченко А.А."

WORKDIR /code

COPY requirements.txt requirements.txt
RUN pip install -r requirements.txt

COPY . .

CMD ["python", "app.py"]
```

### 4. `docker-compose.yml`

```yaml
services:
  web:
    build: .
    ports:
      - "8081:5000"
    depends_on:
      - redis

  redis:
    image: "redis:alpine"
```

---

##  Запуск проекта

```bash
docker compose up -d --build
```

---

## 📸 Скриншоты

### Скриншот 1. Содержимое `requirements.txt`

<img width="545" height="92" alt="image" src="https://github.com/user-attachments/assets/a4c8d11d-2fbc-4445-b7d5-e0ecc73b2167" />


---

### Скриншот 2. Содержимое `app.py` (виден синий цвет и заголовок "Учет билетов")

<img width="473" height="171" alt="image" src="https://github.com/user-attachments/assets/25d8e580-c54e-4e2d-938d-1a5be3c586ee" />


---

### Скриншот 3. Содержимое `Dockerfile` (видна метка maintainer)

<img width="383" height="202" alt="image" src="https://github.com/user-attachments/assets/157086da-d91b-4e18-9c5e-61375ce9ba91" />


---

### Скриншот 4. Содержимое `docker-compose.yml` (виден порт 8081)

<img width="267" height="169" alt="image" src="https://github.com/user-attachments/assets/3ff08869-c4fb-48e1-8aa7-66723dde4330" />


---

### Скриншот 5. Команда `docker ps` (работающие контейнеры)

<img width="665" height="215" alt="image" src="https://github.com/user-attachments/assets/6aa3c969-d36f-4028-bf50-0eab6937962f" />

---

### Скриншот 6. Браузер с адресом `http://localhost:8081`

<img width="432" height="206" alt="image" src="https://github.com/user-attachments/assets/c679f5e5-9af5-4953-b59f-346687dd25e0" />


---

##  Вывод

В ходе выполнения лабораторной работы я успешно запустила многоконтейнерное приложение, состоящее из двух сервисов (Flask и Redis), организовала их сетевое взаимодействие, использовала Docker Compose для оркестрации, внесла необходимые изменения в бизнес-логику (синий заголовок "Учет билетов"), инфраструктуру (порт 8081) и среду сборки (метка maintainer), а также настроила Redis в качестве внешнего хранилища данных для счётчика.

