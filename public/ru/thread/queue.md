# Безопасный конкурентный контейнер Queue

Создание конкурентного `Queue` объекта, который может быть передан в качестве аргумента для дочерних线程. Читая и пиша, он可见 для других线程.




## Особенности
- `Thread\Queue` представляет собой структуры данныхFIFO.


- `Map`, `ArrayList`, `Queue` автоматически распределяют память, не требуется фиксированное распределение как у `Table`.


- В основе лежит автоматическое блокирование, что делает его безопасным для线程


- 可传递的变量类型参见 [线程参数传递](thread/transfer.md)


- Не поддерживается итератор, в основе используется `C++ std::queue`, поддерживаются только операции FIFO


- `Map`, `ArrayList`, `Queue` объекты должны быть переданы в качестве线程参数 дочерним线程м до создания线程


- `Thread\Queue` поддерживает только вставку и удаление элементов, доступ к элементам нельзя осуществлять произвольно


- Встроен в `Thread\Queue` условный переменный线程, который позволяет разбудить или ждать других线程 во время операций `push/pop`


## Пример

```php
use Swoole\Thread;
use Swoole\Thread\Queue;

$args = Thread::getArguments();
$c = 4;
$n = 128;

if (empty($args)) {
    $threads = [];
    $queue = new Queue;
    for ($i = 0; $i < $c; $i++) {
        $threads[] = new Thread(__FILE__, $i, $queue);
    }
    while ($n--) {
        $queue->push(base64_encode(random_bytes(16)), Queue::NOTIFY_ONE);
        usleep(random_int(10000, 100000));
    }
    $n = 4;
    while ($n--) {
        $queue->push('', Queue::NOTIFY_ONE);
    }
    for ($i = 0; $i < $c; $i++) {
        $threads[$i]->join();
    }
    var_dump($queue->count());
} else {
    $queue = $args[1];
    while (1) {
        $job = $queue->pop(-1);
        if (!$job) {
            break;
        }
        var_dump($job);
    }
}
```


## Константы



Название | Функция
---|---
`Queue::NOTIFY_ONE` | Разбудить одну нить
`Queue::NOTIFY_ALL` | Разбудить все нити


## Список методов


### __construct()
Конструктор безопасного конкурентного контейнера `Queue`

```php
Swoole\Thread\Queue->__construct()
```


### push()
Записать данные в конец очереди

```php
Swoole\Thread\Queue()->push(mixed $value, int $notify_which = 0): void
```

  * **Параметры**
      * `mixed $value`
          * Функция: содержание данных для записи.
          * По умолчанию: нет.
          * Другие значения: нет.

      !> Чтобы избежать歧义和 ошибок, не следует записывать в канал `null` и `false`
  
      * `int $notify`
          * Функция: должно ли уведомляться нить, ожидающая чтения данных.
          * По умолчанию: `0`, не будит ни одной нити
          * Другие значения: `Swoole\Thread\Queue::NOTIFY_ONE` будит одну нить, `Swoole\Thread\Queue::NOTIFY_ALL` будит все нити.



### pop()
Искать данные из начала очереди

```php
Swoole\Thread\Queue()->pop(float $timeout = 0): mixed
```

* **Параметры**
    * `float $wait`
        * Функция: время ожидания.
        * По умолчанию: `0`, означает ожидание без времени.
        * Другие значения: если не `0`, означает ожидание производителя `push()` данных в течение `$timeout` секунд, если отрицательно, то ожидание бесконечно.

* **Возвращаемое значение**
    * Возвращает данные из начала очереди, если очередь пуста, то возвращает `NULL`.

> Когда используется `Queue::NOTIFY_ALL` для будки всех нитей, только одна нить получит данные, записанные в операции `push()`


### count()
Получить количество элементов в очереди

```php
Swoole\Thread\Queue()->count(): int
```

* **Возвращаемое значение**
    * Возвращает количество элементов в очереди.

### clean()
Очистить все элементы

```php
Swoole\Thread\Queue()->clean(): void
```