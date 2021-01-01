# fast_bitrix24
API wrapper для Питона для быстрого получения данных от Битрикс24 через REST API.

![Статистика тестов](https://github.com/leshchenko1979/fast_bitrix24/workflows/Tests%20on%20push%20%28Ubuntu%2FPython%203.9%29/badge.svg)
[![Статистика загрузок](https://img.shields.io/pypi/dm/fast-bitrix24.svg)](https://pypistats.org/packages/fast-bitrix24)
[![Sourcery](https://img.shields.io/badge/Sourcery-enabled-brightgreen)](https://sourcery.ai)


## Основная функциональность

### Высокая скорость обмена данными

![Тест скорости](speed_test.gif)

- На больших списках скорость обмена данными с сервером достигает тысяч элементов в секунду.
- Использование асинхронных запросов через `asyncio` и `aiohttp` позволяет экономить время, делая несколько запросов к серверу параллельно.
- Автоматическая упаковка запросов в батчи сокращает количество требуемых запросов к серверу и ускоряет обмен данными.

### Избежание отказов сервера
- Скорость отправки запросов к серверу Битрикс контролируется для избежания ошибки HTTP `503 Service unavailable`.
- Если сервер для сложных запросов начинает возвращать `500 Internal Server Error`, можно в одну строку понизить скорость запроосов.

### Удобство кода
- Высокоуровневые списочные методы для сокращения количества необходимого кода. Большинство операций занимают только одну строку кода. Обработка параллельных запросов, упаковка запросов в батчи и многое другое убрано "под капот".
- Позволяет задавать параметры запроса именно в таком виде, как они приведены в [документации к Bitrix24 REST API](https://dev.1c-bitrix.ru/rest_help/index.php). Параметры проверяются на корректность для облегчения отладки.
- Выполнение запросов автоматически сопровождается прогресс-баром из пакета `tqdm`, иллюстрирующим не только количество обработанных элементов, но и прошедшее и оставшееся время выполнения запроса.

### Синхронный и асинхронный клиенты
- Наличие асинхронного клиента позволяет использовать библиотеку для написания веб-приложений (например, телеграм-ботов).

## Начало
Установите модуль через `pip`:
```shell
pip install fast_bitrix24
```

Далее в python:

```python
from fast_bitrix24 import *

# замените на ваш вебхук для доступа к Bitrix24
webhook = "https://your_domain.bitrix24.ru/rest/1/your_code/"
b = Bitrix(webhook)
```

Методы полученного объекта `b` в дальнейшем используются для взаимодействия с сервером Битрикс24.

## Использование

### `get_all()`

Чтобы получить полностью список сущностей, используйте метод `get_all()`:

```python
# список пользователей
users = b.get_all('user.get')
```

Метод `get_all()` возвращает список, где каждый элемент списка является словарем, описывающим одну сущность из запрошенного списка.

Вы также можете использовать параметр `params`, чтобы кастомизировать запрос:

```python
# список сделок в работе, включая пользовательские поля
deals = b.get_all('crm.deal.list', params={
    'select': ['*', 'UF_*'],
    'filter': {'CLOSED': 'N'}
})
```

### `get_by_ID()`
Если у вас есть список ID сущностей, то вы можете получить их свойства при помощи метода `get_by_ID()`
и использовании методов вида `*.get`:

```python
'''
получим список всех контактов, привязанных к сделкам, в виде
[
    (ID сделки1, [контакт1, контакт2, ...]),
    (ID сделки2, [контакт1, контакт2, ...]),
    ...
]
'''

contacts = b.get_by_ID('crm.deal.contact.items.get',
    [d['ID'] for d in deals])
```
Метод `get_by_ID()` возвращает список кортежей вида `(ID, result)`, где `result` - ответ сервера относительно этого `ID`.

### `call()`
Чтобы создавать, изменять или удалять список сущностей, используйте метод `call()`:

```python
# вставим в начало названия всех сделок их ID
tasks = [
    {
        'ID': d['ID'],
        'fields': {
            'TITLE': f'{d["ID"]} - {d["TITLE"]}'
        }
    }
    for d in deals
]

b.call('crm.deal.update', tasks)
```
Метод `call()` возвращает список ответов сервера по каждому элементу переданного списка.

### `call_batch()`
Если вы хотите вызвать пакетный метод, используйте `call_batch()`:

```python
results = b.call_batch ({
    'halt': 0,
    'cmd': {
        'deals': 'crm.deal.list', # берем список сделок
        # и берем список дел по первой из них
        'activities': 'crm.activity.list?filter[ENTITY_TYPE]=3&filter[ENTITY_ID]=$result[deals][0][ID]'
    }
})

```
### Асинхронные вызовы
Если требуется использование бибилиотеки в асинхронном коде, то вместо клиента `Bitrix()` создавайте клиент класса `BitrixAsync()`:
```python
from fast_bitrix24 import BitrixAsync
b = BitrixAsync(webhook)
```
Все методы у него - синхронные аналоги методов из `Bitrix()`, описанных выше:
```python
leads = await b.get_all('crm.lead.list')
```
## Как это работает
1. Перед обращением к серверу во всех методах класса `Bitrix` происходит проверка корректности самых популярных параметров, передаваемых к серверу, и поднимаются исключения `TypeError` и `ValueError` при наличии ошибок.
2. Cоздаются запросы на получение всех элементов из запрошенного списка.
3. Созданные запросы упаковываются в батчи по 50 запросов в каждом.
4. Полученные батчи параллельно отправляются на сервер с соблюдением установленных скоростных ограничений (см. ниже "Как Битрикс24 ограничивает скорость заросов").
5. Ответы (содержимое поля `result`) собираются в единый плоский список и возвращаются пользователю.
    - Поднимаются исключения класса `aiohttp.ClientError`, если сервер Битрикс вернул HTTP-ошибку, и `RuntimeError`, если код ответа был `200`, но ошибка сдержалась в теле ответа сервера.
    - Происходит сортировка ответов (кроме метода `get_all()`) - порядок элементов в списке результатов совпадает с порядком соответствующих запросов в списке запросов.

В случае с методом `get_all()` пункт 2 выше выглядит немного сложнее:
  - `get_all()` делает первый запрос к серверу Битрикс24 с указанным методом и параметрами.
  - Сервер возвращает первую страницу (50 элементов) и параметр `total` - общее количество элементов, найденных по запросу.
  - Исходя из полученного общего количества элементов, создаются запросы на каждую из страниц (всего `total // 50 - 1` запросов), необходимых для получения всех запрошенных элементов.

В связи с тем, что выполнение `get_all()` по длинным спискам может занимать долгое время, в течение которого пользователи могут добавлять новые элементы в список, может возникнуть ситуация, когда общее полученное количество элементов может не соответствовать изначальному значению `total`. В таких случаях будет выдано стандартное питоновское предупреждение (`warning`).

### Как Битрикс24 ограничивает скорость запросов
1. Существует пул из 50 запросов, которые можно направить без ожидания.
2. Пул пополняется со скоростью 2 запроса в секунду.
3. При исчерпании пула и несоблюдении режима ожидания сервер выдаёт ответ `403 Too Many Requests`.

## Подробный справочник по классу `Bitrix`
Объект класса `Bitrix` создаётся, чтобы через него выполнять все запросы к серверу Битрикс24.

Внутри объекта ведётся учёт скорости отправки запросов к серверу, поэтому важно, чтобы все запросы приложения в отношении одного аккаунта с одного IP-адреса отправлялись из одного экземпляра `Bitrix`.

### Метод ` __init__(self, webhook: str, verbose: bool = True):`
Создаёт экземпляр объекта `Bitrix`.

#### Параметры
* `webhook: str` - URL вебхука, полученного от сервера Битрикс.

* `verbose: bool = True` - показывать прогрессбар при выполнении запроса.

### Метод `get_all(self, method: str, params: dict = None) -> list | dict`
Получить полный список сущностей по запросу `method`.

`get_all()` самостоятельно обрабатывает постраничные ответы сервера, чтобы вернуть полный список (подробнее см. "Как это работает" выше).

#### Параметры
* `method: str` - метод REST API для запроса к серверу.

* `params: dict` - параметры для передачи методу. Используется именно тот формат, который указан в документации к REST API Битрикс24. `get_all()` не поддерживает параметры `start`, `limit` и `order`.

Возвращает полный список сущностей, имеющихся на сервере, согласно заданным методу и параметрам.

### Метод `get_by_ID(self, method: str, ID_list: Sequence, ID_field_name: str = 'ID', params: dict = None) -> dict`
Получить список сущностей по запросу `method` и списку ID.

Используется для случаев, когда нужны не все сущности, имеющиеся в базе, а конкретный список поименованных ID, либо в REST API отсутствует способ  получения сущностей одним вызовом.

Например, чтобы получить все контакты, привязанные к сделкам в работе, нужно выполнить следующий код:

```python
deals = b.get_all('crm.deal.list', params={
    'filter': {'CLOSED': 'N'}
})

contacts = b.get_by_ID('crm.deal.contact.item.get',
    [d['ID'] for d in deals])
```

#### Параметры

* `method: str` - метод REST API для запроса к серверу.

* `ID_list: Sequence` - список ID, в отношении которых будут выполняться запросы.

* `ID_field_name: str` - название поля, в которое будут подставляться значения из списка `ID_list`. По умолчанию `'ID'`.

* `params: dict` - параметры для передачи методу.
    Используется именно тот формат, который указан в
    документации к REST API Битрикс24. Если среди параметров,
    указанных в `params`, указан параметр `ID`, то
    поднимается исключение `ValueError`.

Возвращает словарь вида:

```python
{
    ID_1: результат_выполнения_запроса_по_ID_1,
    ID_2: результат_выполнения_запроса_по_ID_2,
    ...
}
```

Ключом каждого элемента возвращаемого словаря будет ID из списка `ID_list`. Значением будет результат выполнения запроса относительно этого ID. Это может быть, например, список связанных сущностей или пустой список, если не найдено ни одной привязанной сущности.

`get_by_ID()` гарантированно возвращает словарь такой же длины, как и поданный на вход `ID_list`.

### Метод `call(self, method: str, items: dict | Iterable[dict]) -> dict | list[dict]`

Вызвать метод REST API. Самый универсальный метод,
применяемый, когда `get_all` и `get_by_ID` не подходят.

#### Параметры
* `method: str` - метод REST API

* `items: dict | Iterable[dict]` - параметры вызываемого метода. Может быть списком, и тогда метод будет вызван для каждого элемента списка, а может быть одним словарем параметров для единичного вызова.

`call()` вызывает `method`, последовательно подставляя в параметры запроса все элементы `items`, и возвращает список ответов сервера для каждого из отправленных запросов. Либо, если `items` - не список, а словарь с параметрами, то происходит единичный вызов и возвращается его результат.


### Метод `call_batch(self, params: dict) -> dict`

Вызвать метод `batch` ([см. официальную документацию по методу `batch`](https://dev.1c-bitrix.ru/rest_help/general/batch.php)).

Поддерживается примение результатов выполнения одной команды в следующей при помощи ключевого слова `$result`:

```python
results = b.call_batch ({
    'halt': 0,
    'cmd': {
        'deals': 'crm.deal.list', # берем список сделок
        # и берем список дел по первой из них
        'activities': 'crm.activity.list?filter[ENTITY_TYPE]=3&filter[ENTITY_ID]=$result[deals][0][ID]'
    }
})
```

Возвращает словарь вида:
```python
{
    'имя_команды_1': результаты_выполнения_команды_1,
    'имя_команды_1': результаты_выполнения_команды_1,
    ...
}
```
### `slow(requests_per_second: float = 0.5)`
Снижает скорость запросов. По вызовам Bitrix, происходящим внутри вызова этого контекстного менеджера:
* скорость запросов гарантированно не превысит `requests_per_second`
* механика "пула запросов" отключается - считается, что размер пула равен 0, и запросы подаются равномерно

После выхода из контекстного менеджера механика пула восстанавливается, однако пул считается исчерпанным и начинает наполняться с обычной скоростью 2 запроса в секунду.

Иногда, когда серверу Битрикса посылается запрос, отбирающий много ресурсов сервера
(например, на создание 2500 лидов), то сервер не выдерживает даже стандартных
темпов подачи запросов, описанных в официальной документации, возвращая `500 Internal Server Error`
после нескольких первых запросов.

В такой ситуации помогает замедление запросов при
помощи контекстного менеджера `slow`:

```python
# временно снижаем скорость до 1.2 запроса в секунду
slower_speed = 1.2
with b.slow(slower_speed):
    b.call('crm.lead.add', [{} for x in range(2500)])

# а теперь несемся с прежней скоростью
leads = b.get_all('crm.lead.list')
...
```
#### Параметры
* `requests_per_second: float = 0.5` - требуемая замедленная скорость запросов. По умолчанию 0.5 запросов в секунду.

# Советы и подсказки

### А как мне сформировать запрос к Битриксу, чтобы  ...?

1. Поищите в [официальной документации по REST API](https://dev.1c-bitrix.ru/rest_help/).
1. Если на ваш вопрос там нет ответа - попробуйте задать его в [группе "Партнерский REST API" в Сообществе разработчиков Битрикс24](https://dev.bitrix24.ru/workgroups/group/34/).
1. Спросите в Телеграме в [группе разработчиков Битрикс24](https://t.me/bit24dev).
1. Спросите в Телеграме в [группе пользователей fast_bitrix24](https://t.me/fast_bitrix24).
1. Спросите на [русском StackOverflow](https://ru.stackoverflow.com/questions/tagged/битрикс24).

### Я хочу добавить несколько лидов списком, но получаю ошибку сервера.

Оберните вызов `call()` в `slow`, установив скорость запросов в 1 - 1,3 в секунду:

```python
with b.slow(1.2):
    results = b.call('crm.lead.add', tasks)
```

### Я хочу вызвать `call()` только один раз, а не по списку.
Передавайте параметры запроса методу `call()`, он может делать как запросы по списку, так и единичный запрос:

```python
method = 'crm.lead.add'
params = {'fields': {
    'TITLE': 'Чпок'
}}
b.call(method, params)
```
Результатом будет ответ сервера по этому одному элементу.

Однако, если такие вызовы делаются несколько раз, то более эффективно формировать из них список и вызывать `call()` единожды по всему списку.

### Как сортируются результаты при вызове `get_all()`?
Пока что никак.

Все обращения к серверу происходят асинхронно и список результатов отсортирован в том порядке, в котором сервер возвращал ответы. Если вам требуется сортировка, то вам нужно делать ее самостоятельно, например:

```python
deals = b.get_all('crm.deal.list')
deals.sort(key = lambda d: int(d['ID']))
```


## Как связаться с автором
- telegram: https://t.me/fast_bitrix24
- создать новый github issue: https://github.com/leshchenko1979/fast_bitrix24/issues/new
