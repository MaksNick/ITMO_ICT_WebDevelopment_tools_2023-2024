# Задание и модели

### Текст задания

Тема: **Разработка веб-приложения для буккросинга**


**Задача 1. Различия между threading, multiprocessing и async в Python** 

Задача: Напишите три различных программы на Python, использующие каждый из подходов: threading, multiprocessing и async. Каждая программа должна решать считать сумму всех чисел от 1 до 1000000. Разделите вычисления на несколько параллельных задач для ускорения выполнения.

Подробности задания:

1. Напишите программу на Python для каждого подхода: threading, multiprocessing и async.
2. Каждая программа должна содержать функцию calculate_sum(), которая будет выполнять вычисления.
3. Для threading используйте модуль threading, для multiprocessing - модуль multiprocessing, а для async - ключевые слова async/await и модуль asyncio.
4. Каждая программа должна разбить задачу на несколько подзадач и выполнять их параллельно.
5. Замерьте время выполнения каждой программы и сравните результаты.

### Наивный подход

Подход считает через цикл сумму от 1 до 1 000 000. Задача нетяжелая, поэтому даже наивный подход справляется за сотые секунды.

``` py title="sum_naive.py"
def calculate_sum(from, to):
    return sum(range(from, to))
```

### Threading

Я создал 20 потоков, которые параллельно считали сумму. С другой стороны, от такого подхода питон не дает сильного преимущества, даже на больших числах.

``` py title="sum_threading.py"
def calculate_sum(start, end, result):
    total = sum(range(start, end))
    result.append(total)


def calculate():
    result = []
    threads = []

    for i in range(20):
        start = i * 50000 + 1
        end = (i + 1) * 50000 + 1
        thread = threading.Thread(target=calculate_sum, args=(start, end, result))
        threads.append(thread)
        thread.start()

    for thread in threads:
        thread.join()

    total_sum = sum(result)
    print("Total sum:", total_sum)

```

### Multiprocessing

Разбил задачу на 20 подзадач. Результаты значительно хуже наивного подхода, так как слишком много времени тратится на создание и управление процессами. Поэтому для таких легких задач многопроцессорность не подходит.

``` py title="sum_multiprocessing.py"
def calculate_sum(start, end, result):
    total = sum(range(start, end))
    result.put(total)


def main():
    result = multiprocessing.Queue()
    processes = []

    for i in range(20):
        start = i * 50000 + 1
        end = (i + 1) * 50000 + 1
        process = multiprocessing.Process(
            target=calculate_sum, args=(start, end, result)
        )
        processes.append(process)
        process.start()

    for process in processes:
        process.join()

    total_sum = sum([result.get() for _ in range(result.qsize())])
    print("Total sum:", total_sum)

```

### Async

Асинхронность справляется с задачей на уровне наивного метода, но не быстрее.

``` py title="sum_async.py"
async def calculate_sum(start, end):
    return sum(range(start, end))


async def calculate():
    tasks = []

    for i in range(20):
        start = i * 50000 + 1
        end = (i + 1) * 50000 + 1
        task = asyncio.create_task(calculate_sum(start, end))
        tasks.append(task)

    total_sum = sum(await asyncio.gather(*tasks))
    print("Total sum:", total_sum)

```

### Результаты

| Испытание      | Наивный   | Multiprocessing | Threading | Async   |
|---------------|---------|-----------------|-----------|---------|
| 1             | 0.0224       | 0.447     | 0.030   | 0.022 |
| 2             | -       | 0.417         | 0.028  | 0.021 |
| 3             | -       | 0.430     | 0.030 | 0.022 |