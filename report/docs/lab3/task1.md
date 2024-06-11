# Упаковка FastAPI приложения, базы данных и парсера данных в Docker

### Задачи

1. Релизовать возможность вызова парсера по http.
2. Создать Dockerfile для упаковки FastAPI приложения и приложения с паресером. В Dockerfile указать базовый образ, установить необходимые зависимости, скопировать исходные файлы в контейнер и определить команду для запуска приложения.
3. Написать docker-compose.yml для управления оркестром сервисов, включающих FastAPI приложение, базу данных и парсер данных. 

### Создание нового приложения

В рамках лабораторной работы было создано приложение web_parser, которое бы позволяло парсить информацию о книгах с Google Boooks. Реализация была взята из Лабораторной работы №2.

``` py title="main.py"
app = FastAPI()


@app.on_event("startup")
def on_startup():
    init_db()


@app.get("/")
def main():
    return "Main page"


@app.post("/parse_books")
def parse(session: Session = Depends(get_session)):
    urls = fetch_random_book_ids(20)
    threads = []
    for url in urls:
        thread = threading.Thread(target=parse_and_save, args=(url,))
        threads.append(thread)
        thread.start()

    for thread in threads:
        thread.join()
```

### Создание Dockerfile

Для упаковки созданного приложения, а также основного сервиса, было написано два `Dockerfile`, а также создан файл `requirements.txt`. Содержимое в них идентично, за исключением путей.

``` py title="Dockerfile"
# 
FROM python:3.10.5

# 
WORKDIR /web_parser

# 
COPY ./requirements.txt /site/requirements.txt

# 
RUN pip install --no-cache-dir --upgrade -r /site/requirements.txt

# 
COPY . .

```

### Создание docker-compose

В конце, для управления созданием контейнерами, был написан `docker-compose.yml`. В этом файле содежится три сервиса: исходное приложение, приложение-парсер и база данных.

``` py title="docker-compose.yml"
version: '3.8'

services:
  db:
    image: postgres:16
    container_name: db
    environment:
      - POSTGRES_USER=postgres
      - POSTGRES_PASSWORD=M1m1m1m1
      - POSTGRES_DB=library
      - POSTGRES_HOST_AUTH_METHOD=trust
      - PGDATA=/var/lib/postgresql/data/pgdata
    ports:
      - "5434:5432"

  web:
    build:
      context: .
      dockerfile: /app/Dockerfile
    restart: always
    depends_on:
      - db
      - web_parser
    environment:
      DB_ADMIN: postgresql://postgres:M1m1m1m1@db/library
    command: sh -c "sleep 10 && uvicorn app.main:app --host 0.0.0.0 --port 8000"
    ports:
      - "8000:8000"

  web_parser:
    build:
      context: .
      dockerfile: /web_parser/Dockerfile
    restart: always
    depends_on:
      - db
    environment:
      DB_ADMIN: postgresql://postgres:M1m1m1m1@db/library
    command: sh -c "sleep 10 && uvicorn web_parser.main:app --host 0.0.0.0 --port 8002"
    ports:
      - "8002:8002"

```
