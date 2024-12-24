# Основания


## Четыре способа установки обратной связи

* **Анонимная функция**

```php
$server->on('Request', function ($req, $resp) use ($a, $b, $c) {
    echo "hello world";
});
```
!> Можно использовать `use` для передачи параметров анонимной функции

* **Статический метод класса**

```php
class A
{
    static function test($req, $resp)
    {
        echo "hello world";
    }
}
$server->on('Request', 'A::Test');
$server->on('Request', array('A', 'Test'));
```
!> Соответствующий статический метод должен быть `public`

* **Функция**

```php
function my_onRequest($req, $resp)
{
    echo "hello world";
}
$server->on('Request', 'my_onRequest');
```

* **Метод объекта**

```php
class A
{
    function test($req, $resp)
    {
        echo "hello world";
    }
}

$object = new A();
$server->on('Request', array($object, 'test'));
```

!> Соответствующий метод должен быть `public`


## Синхронный/асинхронный IO

В `Swoole4+` все бизнес-кода написан синхронно (в эпоху `Swoole1.x` была поддержка асинхронного кода, но теперь асинхронные клиенты удалены, и соответствующие потребности можно полностью реализовать с помощью CORO-клиентов), что не требует дополнительных усилий для понимания, соответствует человеческому способу мышления, но на уровне реализации синхронный код может различаться между `синхронным IO/асинхронным IO`.

Независимо от того, является ли это синхронным IO/асинхронным IO, `Swoole/Server` может поддерживать большое количество подключений к `TCP` клиентам (см. [модель SWOOLE_PROCESS](/learn?id=swoole_process)). Ваш сервис является блокирующим или неблокирующим, и вам не нужно Configuring некоторые параметры отдельно, это зависит от того, есть ли в вашем коде операции с синхронным IO.

**Что такое синхронный IO:**

Простой пример - когда выполняется `MySQL->query`, этот процесс ничего не делает, ждет, пока MySQL вернет результат, и после получения результата продолжает выполнять код ниже, поэтому способность к параллели синхронного IO-сервиса очень плохая.

**Какой код является синхронным IO:**

 * Когда не включено [одностороннее CORO-обогащение](/runtime), большинство операций, связанных с IO в вашем коде, являются синхронными IO. После CORO-обогащения они станут асинхронными IO, и процесс не будет глупо ждать там, см. [ CORO-распределение](/coroutine?id=协程调度).
 * Некоторые `IO` нельзя一键式 CORO-обогатить, нельзя превратить синхронный IO в асинхронный IO, например, `MongoDB` (верю, что `Swoole` решит эту проблему), нужно обращать внимание при написании кода.

!> [CORO](/coroutine) предназначен для повышения параллели, если мое приложение не имеет высокой параллели, или мне необходимо использовать некоторые операции, которые нельзя асинхронизировать (например, упомянутый выше MongoDB), то вы можете полностью отказаться от [одностороннего CORO-обогащения](/runtime), выключить [enable_coroutine](/server/setting?id=enable_coroutine), увеличить количество `Worker` процессов, это та же модель, что и `Fpm/Apache`. Стоит отметить, что поскольку `Swoole` является постоянным процессом, даже производительность синхронного IO значительно улучшится, и многие компании на практике так делают.


### Конвертация синхронного IO в асинхронный IO

[В предыдущем разделе](/learn?id=同步io异步io) было представлено, что такое синхронный/асинхронный IO. В `Swoole` в некоторых случаях синхронные операции `IO` могут быть преобразованы в асинхронные IO.
 
 - После включения [одностороннего CORO-обогащения](/runtime) операции с `MySQL`, `Redis`, `Curl` и т.д. станут асинхронными IO.
 - Используя [Event](/event) модуль для ручного управления событиями, добавляя fd в [EventLoop](/learn?id=什么是eventloop), они становятся асинхронными IO, пример:

```php
//С помощью inotify наблюдать за изменениями в файлах
$fd = inotify_init();
//Добавьте $fd в EventLoop Swoole
Swoole\Event::add($fd, function () use ($fd){
    $var = inotify_read($fd);// После изменения файла прочитать измененный файл.
    var_dump($var);
});
```

Если в вышеупомянутом коде не вызвать `Swoole\Event::add`, чтобы асинхронизировать IO, прямая `inotify_read()` блокирует рабочий процесс, и другие запросы не будут обработаны.

 - Использование метода [sendMessage()](/server/methods?id=sendMessage) `Swoole\Server` для межпроцессного общения, по умолчанию `sendMessage` является синхронным IO, но в некоторых случаях он может быть преобразован `Swoole` в асинхронный IO, например, с использованием [User进程](/server/methods?id=addprocess):

```php
$serv = new Swoole\Server("0.0.0.0", 9501, SWOOLE_BASE);
$serv->set(
    [
        'worker_num' => 1,
    ]
);

$serv->on('pipeMessage', function ($serv, $src_worker_id, $data) {
    echo "#{$serv->worker_id} message from #$src_worker_id: $data\n";
    sleep(10);// Не принимать данные, отправленные с помощью sendMessage, буфер быстро наполняется
});

$serv->on('receive', function (swoole_server $serv, $fd, $reactor_id, $data) {

});

//Ситуация 1: Синхронный IO (по умолчанию)
$userProcess = new Swoole\Process(function ($worker) use ($serv) {
    while (1) {
        var_dump($serv->sendMessage("большой字符串", 0));// По умолчанию, когда буфер наполняется, здесь будет блокировка
    }
}, false);

//Ситуация 2: Включение поддержки CORO для процесса UserProcess с помощью параметра enable_coroutine, чтобы предотвратить нарушение расписания EventLoop другими корутинами,
//Swoole превратит sendMessage в асинхронный IO
$enable_coroutine = true;
$userProcess = new Swoole\Process(function ($worker) use ($serv) {
    while (1) {
        var_dump($serv->sendMessage("большой字符串", 0));// После того как буфер наполняется, процесс не будет заблокирован, будет抛出 ошибка
    }
}, false, 1, $enable_coroutine);

//Ситуация 3: Если в процессе UserProcess установлена асинхронная обратная связь (например, установка таймера, Swoole\Event::add и т.д.),
//чтобы предотвратить нарушение расписания EventLoop другими обратными связями, Swoole превратит sendMessage в асинхронный IO
$userProcess = new Swoole\Process(function ($worker) use ($serv) {
    swoole_timer_tick(2000, function ($interval) use ($worker, $serv) {
        echo "таймер\n";
    });
    while (1) {
        var_dump($serv->sendMessage("большой字符串", 0));// После того как буфер наполняется, процесс не будет заблокирован, будет抛出 ошибка
    }
}, false);

$serv->addProcess($userProcess);

$serv->start();
```

 - Точно так же, [Task进程](/learn?id=taskworker进程) также использует `sendMessage()` для межпроцессного общения, отличие заключается в том, что поддержка CORO для Task进程 включается путем настройки Server's [task_enable_coroutine](/server/setting?id=task_enable_coroutine), и нет "ситуации 3", то есть Task进程 не будет异步изировать sendMessage из-за включения асинхронной обратной связи.


## Что такое EventLoop

Так называемый `EventLoop`, то есть цикл событий, можно просто понимать как epoll_wait, который добавляет всеHandles, которые должны произойти события (fd), в epoll_wait, эти события включают чтение, письмо, ошибки и т.д.

Соответствующий процесс блокируется на этой функции内核 epoll_wait, и когда происходит событие (или просрочение) функция epoll_wait заканчивает блокировку и возвращает результат, после чего можно вызвать соответствующую PHP функцию обратной связи, например, когда поступает данные от клиента, вызывает обратную связь onReceive.

Когда большое количество fd помещается в epoll_wait, и одновременно происходит множество событий, функция epoll_wait возвращается, чтобы по очереди вызвать соответствующие обратные связи, это называется одним циклом событий, то есть многопутевым вводом/выводом, затем снова блокируется, чтобы вызвать epoll_wait для следующего цикла событий.
## Проблемы границы пакетов TCP

В ситуации без параллельности [код в процессе быстрого запуска](/start/start_tcp_server) может нормально работать, но при высокой параллельности возникают проблемы с границами пакетов TCP. protokol TCP на основе своих внутренних механизмов решает проблемы последовательности и пересылки потерянных пакетов, которые присутствуют в протоколе UDP, но по сравнению с UDP он приводит к новым проблемам. Протокол TCP является потоковым, и пакеты не имеют границ, что заставляет приложения, использующие TCP для коммуникации, сталкиваться с этими трудностями, обычно известными как проблема "пасты" TCP.

Поскольку коммуникация по TCP является потоковой, при приеме одного большого пакета он может быть разобран на несколько пакетов для отправки. Множественные `Send` на уровне также могут быть объединены в один для отправки. Здесь необходимы два действия для решения:

* Разбор пакетов: `Сервер` получает несколько пакетов и должен разделить их.
* Сложение пакетов: `Сервер` получает данные, которые являются лишь частью пакета, и должен缓存 данные для их объединения в полный пакет.

Таким образом, при сетевой коммуникации по TCP необходимо установить коммуникационный протокол. Среди распространенных общих протоколов TCP для сетевой коммуникации есть `HTTP`, `HTTPS`, `FTP`, `SMTP`, `POP3`, `IMAP`, `SSH`, `Redis`, `Memcache`, `MySQL`.

Стоит отметить, что Swoole встроил анализ многих распространенных общих протоколов для решения проблем с границами пакетов TCP на серверах этих протоколов, и для этого достаточно просто настроить, см. [открытие HTTP-протокола](/server/setting?id=open_http_protocol)/[открытие HTTP/2-протокола](/http_server?id=open_http2_protocol)/[открытие WebSocket-протокола](/server/setting?id=open_websocket_protocol)/[открытие MQTT-протокола](/server/setting?id=open_mqtt_protocol).

Помимо общих протоколов, можно также определить собственный протокол. Swoole поддерживает два типа собственных сетевых коммуникационных протоколов.

* **Протокол с концом пакета EOF**

Принцип работы протокола EOF заключается в том, что на конце каждого пакета добавляется серия специальных символов, обозначающих конец пакета. Например, `Memcache`, `FTP`, `SMTP` используют `\r\n` в качестве конца пакета. При отправке данных достаточно добавить `\r\n` в конец пакета. При использовании протокола EOF необходимо убедиться, что внутри пакета не появляется EOF, иначе это приведет к ошибкам в разбиении пакетов.

В коде `Сервера` и `Клиента` необходимо установить два параметра для использования протокола EOF.

```php
$server->set(array(
    'open_eof_split' => true,
    'package_eof' => "\r\n",
));
$client->set(array(
    'open_eof_split' => true,
    'package_eof' => "\r\n",
));
```

Однако вышеуказанное конфигурационное значение EOF будет иметь плохую производительность, Swoole будет проверять каждый байт, чтобы увидеть, является ли данные `\r\n`, помимо вышеупомянутого способа, можно также настроить так.

```php
$server->set(array(
    'open_eof_check' => true,
    'package_eof' => "\r\n",
));
$client->set(array(
    'open_eof_check' => true,
    'package_eof' => "\r\n",
));
```
Эта комбинация настроек будет значительно лучше по производительности, поскольку она не требует обхода данных, но она может решить только проблему разделения пакетов, не решая проблему объединения пакетов. То есть возможно, что `onReceive` будет вызваться несколько раз с одними и теми же запросами от клиента, и вам придется самостоятельно разделять их, например, с помощью `explode("\r\n", $data)`. Основная цель этой комбинации настроек заключается в том, что если вы используете сервис, который отвечает на запросы (например, командный ввод в терминале), вам не нужно беспокоиться о разделении данных. Это потому, что после отправки одного запроса клиент должен дождаться ответа от сервера на текущий запрос, прежде чем отправить следующий запрос, и не будет отправляться одновременно два запроса.

* **Протокол с фиксированным заголовком и телом пакета**

Метод с фиксированным заголовком очень распространен в серверных программах. Отличительной особенностью этого протокола является то, что каждый пакет всегда состоит из двух частей: заголовка и тела. Заголовок указывает длину тела или всего пакета, которая обычно выражается с помощью двухбайтного или четырехбайтного целого числа. После получения заголовка сервер может точно контролировать, сколько данных ему нужно принять для получения полного пакета. Конфигурация Swoole хорошо поддерживает этот протокол и позволяет гибко устанавливать четыре параметра для всех случаев.

Сервер обрабатывает пакет в [onReceive](/server/events?id=onreceive) обратном вызове. Когда установлен протокол обработки, событие [onReceive](/server/events?id=onreceive) будет вызвано только при получении полного пакета. После установки протокола обработки на клиенте, вызов [$client->recv()](/client?id=recv) больше не требует передачи длины. Функция `recv` возвращается после получения полного пакета или при возникновении ошибки.

```php
$server->set(array(
    'open_length_check' => true,
    'package_max_length' => 81920,
    'package_length_type' => 'n', //see php pack()
    'package_length_offset' => 0,
    'package_body_offset' => 2,
));
```

!> Для конкретного значения каждого конфигурационного элемента смотрите раздел [Конфигурация](/server/setting?id=open_length_check) в главе "Сервер/Клиент".

## Что такое IPC

Существует множество способов коммуникации между двумя процессами на одном и том же хосте (сокращенно IPC), и в Swoole используются два таких способа: `Unix Socket` и `sysvmsg`. Далее они будут представлены отдельно:

- **Unix Socket**  

    Полное название UNIX Domain Socket, в просторечии `UDS`, использует API сокетов (socket, bind, listen, connect, read, write, close и т.д.), в отличие от TCP/IP, где не требуется указывать IP и port, а представляет собой имя файла (например, `/tmp/php-fcgi.sock` между FPM и Nginx). UDS - это коммуникация в полной памяти, реализованная ядром Linux, без каких-либо `IO` затрат. В тесте, где один процесс пишет, а другой читает, 100 тысяч коммуникаций занимает всего 1.02 секунды, и это очень мощно. По умолчанию в Swoole используется именно этот способ IPC.  
      
    * **`SOCK_STREAM` и `SOCK_DGRAM`**  

        - В Swoole для коммуникации с использованием UDS есть два типа: `SOCK_STREAM` и `SOCK_DGRAM`, которые можно简单地 сравнить с TCP и UDP. Когда используется тип `SOCK_STREAM`, также необходимо учитывать проблему границы пакетов TCP.   
        - Когда используется тип `SOCK_DGRAM`, не нужно беспокоиться о проблеме границы пакетов TCP, каждое отправленное с помощью `send()` данные имеет границы, и вы получите столько же данных, сколько было отправлено, без потерь данных или их беспорядка в процессе передачи, и порядок отправки и приема данных с помощью `send` и `recv` полностью согласован. После успешного возврата `send` можно с уверенностью получить данные с помощью `recv`. 

Когда объем данных для IPC невелик, очень подходит использование метода `SOCK_DGRAM`. **Поскольку каждый IP-пакет имеет максимальный размер в 64 кбит, при использовании `SOCK_DGRAM` для IPC объем отправляемой в один раз данных не должен превышать 64 кбит, и следует обратить внимание на то, что слишком медленный прием пакетов может привести к тому, что операционная система будет сбрасывать пакеты из-за переполнения буфера, поскольку UDP позволяет терять пакеты, можно увеличить буфер.**


- **sysvmsg**
     
    Это механизм "сообщественных буферов" Linux, который позволяет общаться через имя файла в качестве "ключа". Этот способ IPC довольно не гибкий и не широко используется в реальных проектах, поэтому мы не будем его подробно описывать.

    * **Этот способ IPC полезен только в двух сценариях:**

        - Чтобы предотвратить потерю данных, если весь сервис宕ет, сообщения в очереди останутся, и их можно продолжить обрабатывать, **но также есть проблема с грязными данными**.
        - Можно снаружи доставлять данные, например, `Worker` процессы в Swoole могут доставлять задания `Task` процессам через сообщественные буферы, а внешние процессы также могут доставлять задания в очередь для обработки `Task`, и даже можно вручную добавить сообщения в очередь из командной строки.
## Разница и связь между Master процессом, Reactor потока, Worker процессами, Task процессами, Manager процессами: id=diff-process


### Master процесс

* Master процесс - это много线程овый процесс, см. [Структура процессов/потоков](/server/init?id=структура процессов_потоков)


### Reactor потока

* Reactor потока создаются в Master процессе
* Они отвечают за поддержание TCP соединений с клиентами, обработку сетевых IO, обработку протоколов, прием и отправку данных
* Не выполняют никаких PHP кодов
* Buffеризуют, собирают и расщепляют данные, 收到 от TCP клиентов, в целые запросные пакеты


### Worker процессы

* Принимают запросные пакеты, отправленные Reactor线程ами, и выполняют PHP обратные вызовы для обработки данных
* 生ируют ответные данные и отправляют их в Reactor线程, который потом отправить их TCP клиентам
* Могут работать как в асинхронном неблокирующем, так и в синхронном блокирующем режиме
* Worker работают в multitude процессов


### TaskWorker процессы

* Принимают задания, отправленные Worker процессами с помощью методов Swoole\Server->[task](/server/methods?id=task)/[taskwait](/server/methods?id=taskwait)/[taskCo](/server/methods?id=taskCo)/[taskWaitMulti](/server/methods?id=taskWaitMulti)
* Обрабатывают задания и возвращают результаты обратно Worker процессам (с помощью [Swoole\Server->finish](/server/methods?id=finish))
* Работают исключительно в синхронном блокирующем режиме
* TaskWorker работают в multitude процессов, [подробный пример задания](/start/start_task)


### Manager процесс

* Ответственен за создание/от银色ка Worker/Task процессов

Их отношения можно сравнить так: Reactor - это nginx, Worker - это PHP-FPM. Reactor线程 асинхронно и параллельно обрабатывают сетевые запросы, затем пересылают их Worker процессам для обработки. Reactor и Worker общаются друг с другом через [unixSocket](/learn?id=что такое IPC).

В приложениях PHP-FPM часто асинхронно отправляют задания в очередь, например, в Redis, и запускают некоторые PHP процессы для асинхронной обработки этих заданий. Swoole предоставляет TaskWorker - это более полное решение, объединяющее отправку заданий, очередь, управление процессами PHP для обработки заданий в одном. С помощью APIs, предоставляемых на нижнем уровне, можно очень просто реализовать асинхронную обработку заданий. Кроме того, TaskWorker может вернуть результаты выполнения задания Worker после завершения работы.

Reactor, Worker и TaskWorker Swoole могут тесно сочетаться друг с другом, предоставляя более высокие возможности использования.

Более простая метафора: предположим, что Server - это фабрика, то Reactor - это продавец, принимающий заказы от клиентов. А Worker - это рабочий, который начинает работать и производить вещи, которые хочет клиент, когда продавец получает заказ. TaskWorker можно сравнить с административным персоналом, который может помогать Worker с мелочами, чтобы Worker мог сосредоточиться на работе.

Как показано на рисунке:

![process_demo](_images/server/process_demo.png)


## Введение трех рабочих режимов Server

В третьем аргументе конструктора Swoole\Server можно указать одно из трех постоянных значений - [SWOOLE_BASE](/learn?id=swoole_base), [SWOOLE_PROCESS](/learn?id=swoole_process) и [SWOOLE_THREAD](/learn?id=swoole_thread). Далее будет представлено различие между этими тремя моделями, а также их преимущества и недостатки.


### SWOOLE_PROCESS

В модели SWOOLE_PROCESS все TCP соединения с клиентами Server устанавливаются с [главным процессом](/learn?id=reactor线程), внутреннее реализация довольно сложное, использует много механизмов межпроцессного общения и управления процессами. Подходит для复杂的 бизнес-логики. Swoole предоставляет полный набор средств управления процессами, защиты памяти.
В случаях очень сложной бизнес-логики, также может стабильно работать в долгосрочной перспективе.

В [Reактор](/learn?id=reactor线程)线程х Swoole предоставлен функционал `Buffer`, который может справиться с большим количеством медленных подключений и злонамеренных клиентов, которые читают данные по одному символу.

#### Преимущества режима процессов:

* Соединения и отправка данных разделены, что не позволяет `Worker` процессам быть несбалансированными из-за различной объема данных в некоторых подключениях
* Когда `Worker` процесс терпит смертельную ошибку, соединения не断аются
* Можно реализовать одновременное обращение по одному соединению, поддерживая всего несколько `TCP` подключений, а запросы могут быть одновременно обработаны в различных `Worker` процессах

#### Недостатки режима процессов:

* Существует `2`开销 IPC, `master` процесс и `worker` процесс должны общаться с использованием [unixSocket](/learn?id=что такое IPC)
* `SWOOLE_PROCESS` не поддерживает PHP ZTS, в таких случаях можно использовать только `SWOOLE_BASE` или установить [single_thread](/server/setting?id=single_thread) в true


### SWOOLE_BASE

Этот режим SWOOLE_BASE является традиционным асинхронным неблокирующим `Server`. Он полностью идентичен таким программам, как `Nginx` и `Node.js`.

Параметр [worker_num](/server/setting?id=worker_num) все еще действителен для режима `BASE`, он запустит несколько `Worker` процессов.

Когда приходит запрос на TCP соединение, все `Worker` процессы соревнуются за это соединение, и в конечном итоге один из `Worker` процессов успешно устанавливает прямую TCP связь с клиентом, после этого все данные для этого соединения напрямую общаются с этим `Worker`, не проходя через `Reactor`线程 главного процесса для пересылки.

* В режиме `BASE` нет роли `Master` процесса, есть только роль [Manager](/learn?id=manager进程).
* Каждый `Worker` процесс одновременно выполняет функции `Reactor`线程 и `Worker` процесса в режиме [SWOOLE_PROCESS](/learn?id=swoole_process).
* В режиме `BASE` менеджер进程 необязателен, когда установлено `worker_num=1` и не используются характеристики [Task](/server/methods?id=task) и [MaxRequest](/server/settings?id=max_requests), на дне будет создан отдельный `Worker` процесс напрямую, без создания Manager процесса

#### Преимущества режима BASE:

* В режиме BASE нет开销 IPC, производительность лучше
* Кода в режиме BASE проще, меньше вероятность ошибок

#### Недостатки режима BASE:

* TCP соединения поддерживаются внутри `Worker` процесса, поэтому когда какой-то `Worker`进程 падает, все соединения в этом `Worker` будут закрыты
* Небольшое количество длинных TCP подключений не может использовать все `Worker` процессы
* TCP соединения привязаны к Worker, в приложениях с длинными подключениями некоторые соединения имеют большой объем данных, и нагрузка на `Worker` процессов с этими подключениями будет очень высокой. Но некоторые соединения имеют малый объем данных, поэтому нагрузка на `Worker` процессов будет очень низкой, и различные `Worker` процессы не могут достичь баланса.
* Если в обратном вызове есть блокирующие операции, это может привести к тому, что Server превратится в синхронный режим, когда легко возникает проблема переполнения очереди ожидания TCP [backlog](/server/setting?id=backlog).

#### Подходящие сценарии для режима BASE:

Если между клиентами нет необходимости в взаимодействии, можно использовать режим BASE. Например, `Memcache`, HTTP серверы и т. д.

#### Ограничения режима BASE:

В режиме BASE, кроме методов [send](/server/methods?id=send) и [close](/server/methods?id=close), другие методы [Server](/server/methods) не поддерживают исполнения между процессами.

!> В версии v4.5.x только метод `send` поддерживает 跨процессное исполнение в режиме BASE; в версии v4.6.x поддерживаются только методы `send` и `close`.
### SWOOLE_THREAD

SWOOLE_THREAD - это новый режим выполнения, представленный в Swoole 6.0 с использованием режима PHP ZTS, что позволяет нам теперь запускать сервис в многопроцессном режиме.

Параметр [worker_num](/server/setting?id=worker_num) все еще применим к режиму THREAD, но теперь вместо создания многопроцессов он будет создавать многоп线程ов, что приведет к запуску нескольких рабочих потоков.

Будет только один процесс, а дочерние процессы превратятся в дочерние потоки, отвечающие за прием запросов от клиентов.

#### Преимущества режима THREAD:
* Коммуникация между процессами стала проще, нет дополнительных затрат на IPC-сообщение.
* Отладка программы стала удобнее, поскольку есть только один процесс, использование `gdb -p` будет проще.
* Имеется преимущества от синхронного выполнения IO с использованием协程, а также возможность并行ной исполнения и общего памяти для стека из-за многоп线程ов.

#### Недостатки режима THREAD:
* В случае падения или вызова `Process::exit()` весь процесс будет завершен, что требует реализации логики восстановления ошибок, таких как повторные попытки и Reconnection после обрыва связи на стороне клиента, а также использования supervisor и docker/k8s для автоматического возобновления процесса после его завершения.
* Операция с ZTS и блокировками может потребовать дополнительных затрат, и производительность может быть на 10% ниже, чем в многопроцессной модели NTS. Если сервис безstata乌斯, все же рекомендуется использовать многопроцессный режим NTS.
* Не поддерживается передача объектов и ресурсов между потоками.

#### Применение режима THREAD:
Режим THREAD более эффективен для разработки игровых серверов и коммуникационных серверов.

## Различие между Process, Process\Pool и UserProcess :id=process-diff

### Process

[Process](/process/process) - это модуль управления процессами, предоставляемый Swoole, который заменяет PHP `pcntl`.

* Удобно реализовать межпроцессную коммуникацию;
* Поддерживает перенаправление стандартного ввода и вывода, в дочернем процессе `echo` не будет печатать на экран, а будет写入 трубу, чтение клавиатурного ввода можно перенаправить для чтения данных из трубы;
* Предоставляет [exec](/process/process?id=exec) интерфейс, созданные процессы могут выполнять другие программы и легко общаться с исходным PHP родителем;

!> В контексте协程 невозможно использовать модуль `Process`, его можно реализовать с помощью `runtime hook` + `proc_open`, см. [Управление процессами в协程](/coroutine/proc_open)

### Process\Pool

[Process\Pool](/process/process_pool) - это PHP класс, который упакует модуль управления процессами сервера в Swoole, позволяя использовать менеджера процессов Swoole в PHP коде.

В реальных проектах часто нужно писать долгосрочные скрипты, такие как потребители многопроцессных очередей на основе `Redis`, `Kafka`, `RabbitMQ`, многопроцессные веб-скаплеры и т.д., разработчики должны использовать расширительные библиотеки, связанные с `pcntl` и `posix`, для реализации многопроцессного программирования, но им также требуется глубокое знание системного программирования на Linux, иначе легко возникают проблемы. Использование менеджера процессов Swoole может значительно упростить работу с многопроцессными скриптами.

* Обеспечивает стабильность рабочих процессов;
* Поддерживает обработку сигналов;
* Поддерживает функции передачи сообщений в очереди и TCP-сокеты;

### UserProcess

`UserProcess` - это пользовательский рабочий процесс, добавленный с помощью [addProcess](/server/methods?id=addprocess), обычно используемый для создания специального рабочего процесса для мониторинга, отчетности или других специальных задач.

Хотя `UserProcess` будет hosted в [Manager процессе](/learn?id=manager进程), он является относительно независимым процессом по сравнению с [Worker процессами](/learn?id=worker进程) и используется для выполнения пользовательских функций.