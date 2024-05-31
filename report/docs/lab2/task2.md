# Задание и модели

### Текст задания

Тема: **Разработка веб-приложения для буккросинга**


**Задача 2. Параллельный парсинг веб-страниц с сохранением в базу данных** 

Задача: Напишите программу на Python для параллельного парсинга нескольких веб-страниц с сохранением данных в базу данных с использованием подходов threading, multiprocessing и async. Каждая программа должна парсить информацию с нескольких веб-сайтов, сохранять их в базу данных.

![Database ERD](pics/1.jpg "Database ERD")

Подробности задания:

1. Напишите три различных программы на Python, использующие каждый из подходов: threading, multiprocessing и async.
2. Каждая программа должна содержать функцию parse_and_save(url), которая будет загружать HTML-страницу по указанному URL, парсить ее, сохранять заголовок страницы в базу данных и выводить результат на экран.
3. Используйте базу данных из лабораторной работы номер 1 для заполенния ее данными. Если Вы не понимаете, какие таблицы и откуда Вы могли бы заполнить с помощью парсинга, напишите преподавателю в общем чате потока.
4. Для threading используйте модуль threading, для multiprocessing - модуль multiprocessing, а для async - ключевые слова async/await и модуль aiohttp для асинхронных запросов.
5. Создайте список нескольких URL-адресов веб-страниц для парсинга и разделите его на равные части для параллельного парсинга.
6. Запустите параллельный парсинг для каждой программы и сохраните данные в базу данных.
7. Замерьте время выполнения каждой программы и сравните результаты.

### Модель и источник

Во второй работе при работе с данными я воспользовался наработками первой работы. Мой вариант - система буккроссинга, поэтому я решил спарсить книги. В качестве источника я выбрал сервис Google Books, с которого через API я подключался и собирал данные о случайных книгах. Во всех реализациях я парсил и вносил в таблицу 20 книг.

**Функция для парсера**

``` py title="book_parser.py"
def get_book_details(book_id, api_key="AIzaSyAroYMo_JrN4UAdptepmm3DMp1ubMb0YEo"):
    url = f"https://www.googleapis.com/books/v1/volumes?q={book_id}&key={api_key}"
    response = requests.get(url)

    if response.status_code == 200:
        data = response.json()
        if "items" in data and len(data["items"]) > 0:
            random_book = data["items"][0]["volumeInfo"]
            return {
                "title": random_book.get("title", "Нет названия"),
                "author": ", ".join(random_book.get("authors", "Нет автора")),
                "publication_year": random_book.get("publishedDate", 2000)[:4],
                "description": random_book.get("description", "Нет описания"),
                "available": True,
            }
    return None
``` 

### Наивный подход

Подход собирает через цикл 20 книг. Получается довольно долго - в среднем 20 секунд.

``` py title="book_naive.py"
def parse_and_save(url):
    book_details = get_book_details(url)
    if book_details:
        save_to_db([book_details])
        print(f"Saved book: {book_details['title']}")


start_time = time.time()
urls = fetch_random_book_ids(20)

for url in urls:
    parse_and_save(url)
```

### Threading

Трединг неплохо подходит для I/O Bound задач, поэтому способ неплохо показывает себя при сборе информации о книгах. Каждая ссылка - отдельный тред.

``` py title="book_threading.py"
def parse_and_save(url):
    book_details = get_book_details(url)
    if book_details:
        save_to_db([book_details])
        print(f"Saved book: {book_details['title']}")


start_time = time.time()
threads = []
urls = fetch_random_book_ids(20)

for url in urls:
    thread = threading.Thread(target=parse_and_save, args=(url,))
    threads.append(thread)
    thread.start()

for thread in threads:
    thread.join()
```

### Multiprocessing

Многопроцессорность хоть и выигрывает наивный метод, но все еще медленнее асинхронного и разбиения на потоки.

``` py title="book_multiprocessing.py"
def parse_and_save_multiprocessing(book_id):
    book_data = get_book_details(book_id)
    if book_data:
        save_to_db([book_data])
        print(f"Saved: {book_data['title']}")


def parse_and_save(book_ids):
    start_time = time.time()
    pool = multiprocessing.Pool(processes=multiprocessing.cpu_count())
    pool.map(parse_and_save_multiprocessing, book_ids)
    pool.close()
    pool.join()

    print(f"Multiprocessing version took {time.time() - start_time}")

```

### Async

Асинхронность показывает результаты на уровне с многопоточностью, а иногда и обгоняет его по времени.

``` py title="sum_async.py"
async def parse_and_save_async(book_id, http_session, db_session):
    book_data = await fetch_book_details(book_id, http_session)
    if book_data:
        await save_to_db([book_data], async_SessionFactory)
        print(f"Saved: {book_data['title']}")


async def async_version(book_ids):
    async with aiohttp.ClientSession() as http_session:
        async with async_SessionFactory() as db_session:
            tasks = [
                parse_and_save_async(book_id, http_session, db_session)
                for book_id in book_ids
            ]
            await asyncio.gather(*tasks)


if __name__ == "__main__":
    urls = fetch_random_book_ids(20)
    start_time = time.time()
    asyncio.run(async_version(urls))
    print(f"Multiprocessing version took {time.time() - start_time}")

```

### Результаты

| Испытание      | Наивный   | Multiprocessing | Threading | Async   |
|---------------|---------|-----------------|-----------|---------|
| 1             | 22.026       | 4.579     | 1.220   | 1.715 |
| 2             | 19.286       | 4.619         | 1.669  | 1.427 |
| 3             | -       | 4.452     | 1.220 | 1.989 |

Видно, что наивный метод сильно проигрывает перед любым из параллельных методов, поэтому при задаче на парсинг стоит использовать рассмотренные подходы, чтобы сэкономить время.