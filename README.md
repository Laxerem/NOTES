# Работа с сокетами C++
### Дисклеймер
>Я не являюсь специалистом в данной теме, я обычный новичок, поэтому всё что я расскажу ниже является моим субъективным опытом, в объяснении которого могут присутствовать ошибки. 
Иначе говоря что-то я буду объяснять вам так, как понял это я, но некоторую информацию я тупо вставил с интернета.  
~Так что не судите строго :( 

# Сокет

**Сокет** - это абстракция, конечная точка соединения между сервером и клиентом, сокет представляет собой *файловый дескриптор*, в котором содержатся все параметры для соединения, именно через сокет и передаются многие данные.

Существуют несколько протоколов сокетов, основные из них: 
**TCP** и **UDP**.
```
TCP - протокол который гарантирует цельную доставку данных получателю.

Сначала одна сторона отправляет запрос на соединение, 
вторая подтверждает его получение и готовность к общению, а затем первая 
подтверждает получение этого подтверждения.
Только после этого начинается обмен данными. 
Если не все данные были получены, они отправляются повторно.

Этот процесс гарантирует, что соединение установлено надежно, 
прежде чем начнется передача данных.
```

```
UDP - протокол быстрой передачи данных, который не гарантирует 
целостность доставки этих данных.

При протоколе UDP нет никакого запроса на соединение и 
подтверждение этого соединения, клиент отправляет данные вразнабой.

Этот протокол используют, когда скорость передачи важнее надёжности
соединения и целостность доставки данных.

Например: для потоковой передачи видео.
```
## Взаимодействие с сокетами

Существует два вида взаимодействия с сокетами: **Синхронное** и **Асинхронное**

**Синхронное** взаимодействие означает, что ваши операции не будут выполнятся до тех пор, пока вы не получите какие то данные от сокета. Т.е ваша **программа будет ждать сообщения** (listen) и пока она его не получит, не приступит к другим задачам.

При **Асинхронном** взаимодействии ваша программа не будет чего то ждать, она будет лишь проверять, не пришли ли к вам какие то данные, и если они не пришли, то продолжать выполнять какую то программу. Иначе говоря **выполнение вашей программы не будет зависеть от клиента.**


## Создание сокета

**Сокет** включает в себе три атрибута: *домен*, *тип*, *протокол*.

Для создания сокета используется функция socket, имеющая следующий прототип:

```C++
#include <sys/types.h>
#include <sys/socket.h>

int socket(int domain, int type, int protocol);
```

**Домен** определяет пространство адресов, в котором располагается *сокет*, и множество протоколов, которые используются для передачи данных.

Основные семейства адресов (*AF* - "address family") домена следующие:
* Константа **AF_INET** соответствует *Internet*-домену. Сокеты, размещённые в этом домене, могут использоваться для работы в любой IP-сети.
* При задании **AF_UNIX** для передачи данных используется файловая система ввода/вывода *UNIX*.

>В данной статье я буду использовать сокеты для передачи данных по *сети*, поэтому всё что я буду говорить ниже будет в контексте того, что вы выбрали атрибут **AF_INET**

**Тип** сокета определяет способ передачи данных по сети.

* **SOCK_STREAM** - Передача потока данных с предварительной установкой соединения. Для реализации этого параметра используется протокол **TCP**.
* **SOCK_DGRAM** - Передача данных в виде отдельных сообщений (датаграмм). Является быстрым но не надёжным, т.к для реализации этого параметра используется протокол **UDP**
* **SOCK_RAW** - Этот тип присваивается низкоуровневым (т. н. "сырым") сокетам. Их отличие от обычных сокетов состоит в том, что с их помощью программа может взять на себя формирование некоторых заголовков, добавляемых к сообщению.

**Протокол**. Так как мы по типу сокета уже можем понять протокол, можете просто перадать 0, или же следующие аргументы (основные):
* **IPPROTO_TCP** - сокет **TCP**
* **IPPROTO_UDP** - сокет **UDP**

# Установление соединения

> Примечание: На операционных системах **UNIX** сокеты могут использоваться для передачи данных между процессами на вашем устройстве, такие сокеты будут являться локальными и вместо указания IP адреса нужно будет указывать # ЧТО ТО

Для установления **локального сетевого** соединения между *клиентом* и *сервером* необходимо создать, и соединить между собой два *сокета*:

Сокет **клиента** (отправителя) и сокет **сервера** (получателя)

## Сокет сервера

**Основная задача сервера** - *слушать* и *принимать* запросы, которые в последствии можно будет обработать.

Итак, для того чтобы мы могли получать данные от клиента, нам необходимо создать серверный сокет, этот сокет также называют слушающим, потому что он будет постоянно ждать, до тех пор пока он не получит запрос.

```C++
int server = socket(AF_INET, SOCK_STREAM, 0);
```

Функция *socket()* запрашивает у ОС свободный номер индекса **файлового дескриптора**, после чего возвращает его.

После создания сокета, необходимо создать и описать уже имеющуюся в библиотеке структуру **sockaddr_in**, которая будет хранить параметры сервера.

Содержит структура следующие параметры:
```C++
struct sockaddr_in {
    short int          sin_family;  // Семейство адресов
    unsigned short int sin_port;    // Номер порта
    struct in_addr     sin_addr;    // IP-адрес клиента
    unsigned char      sin_zero[8]; // "Дополнение" до размера структуры sockaddr
};
```

**sin_family** - Семейство адресов, так как мы хотим передавать данные через интернет, в этот параметр нужно внести **AF_INET**

**sin_port** - Порт к которому привязан ваш сервер, клиент также должен быть привязан к порту сервера.

**sin_addr** - IP адрес, с которым будет общаться ваш сервер, вы можете прописать его под конкретный IP, а можете поставить *INADDR_ANY*, который означает, что сервер будет принимать данные от любого IP адреса в вашей сети.

>**Примечание:** Структура **in_addr**, в которой и находится IP адрес sin_addr, содержит лишь **один элемент**, потому что раньше *in_addr* **представляла собой объединение** (union), содержащее гораздо большее число полей. Сейчас, когда в ней осталось всего одно поле, она продолжает использоваться для **обратной совместимости**.

### Привязка сокета к параметрам

Хорошо, теперь у нас есть пустой сокет, давайте же привяжем его к параметрам структуры **sockaddr_in** для того чтобы сокет понимал с кем и где общаться.

Для того чтобы привязать сокет к его адресу и порту, используется функция **bind()**

```C++
#include <sys/types.h>
#include <sys/socket.h>

int bind(int socket, struct sockaddr *addr, int addrlen);
```
**socket** - сокет, который вы хотите привязать к адресу.

**addr** - указатель на структуру *sockaddr_in*, параметры которой будут переданы в базовую структуру sockaddr.

**addrlen** - размер структуры *sockaddr_in* которую вы передаёте для привязки к сокету. Чтобы получить этот размер в байтах, используйте функцию sizeof() в которую передайте ваш *sockaddr_in*.

>**Номер порта** сокета **не должен быть занят** другим устройством, поэтому если в процессе привязки произойдёт ошибка, попробуйте сменить номер порта.

### Прослушивание соединений с сервером

После того как мы соединили сокет с адресом, можно сказать что ваш сокет настроен и готов к использованию.
Давайте же начнём слушать соединения с нашим сервером.

Для того чтобы сервер слушал входящие соединения, необходимо использовать функцию **listen()**:

```C++
int listen(int socket_server, int backlog)
```
**socket_server** - сокет сервера, который будет слушать входящие соединения.

**backlog** - размер очереди запросов. Иначе говоря: количество запросов, которые будут находится в очереди, до тех пор пока сервер не обработает первый запрос. Если очередь равна единице, все последующие запросы будут проигнорированы и не выполнены, до тех пор пока сервер не обработает запрос.

В примере у нас будет только один запрос, без очереди, поэтому можете ставить 1.

>Примечание: функция **listen()** отправит вашу программу в ожидание (while), т.е пока сервер не получит сообщение от клиента, программа не начнёт выполнять код который следует после listen()

### Принятие запроса клиента

Наш **сокет сервера и клиента не должны общаться на прямую**, потому что задача серверного сокета - просто слушать входящие соединения. 

Если бы мы общались напрямую, то другие клиенты просто не смогли бы подключится к нашему серверу, потому что сокет сервера был бы занят обработкой другого соединения.

Поэтому после того как наш серверный (слушающий) сокет принял запрос от клиента, создаётся новый (клиентский) сокет, через который и будет происходить передача данных. После чего серверный сокет будет готов принимать следующие запросы.

Для того чтобы принять запрос и вернуть новый сокет используется функция **accept()**

```C++
#include <sys/socket.h>

int accept(int socket, void *addr, int *addrlen); 
//возвращает новый сокет клиента
```
**socket** - **серверный** (слушающий) **сокет**, на который поступил запрос.

**addr** - указатель на **структуру клиента**, которая будет содержать данные о его адресе. Если вам не нужны эти данные, можете ставить NULL или nullptr.

**addrlen** - функция записывает в переменную длину размера структуры addr, тоже можете ставить NULL, если вам это не нужно.

### Считывание данных

Теперь у нас есть отдельный сокет для общения с конкретным клиентом, и осталось лишь получить те данные, которые отправил нам клиент на наш новый сокет.

Для этого используется функция **recv()**

```C++
#include <sys/socket.h>

int recv(int new_sock, void *buf, int len, int flags);
```
**Первый аргумент** - новый сокет, который мы получили из функции accept().

**buf** - указатель на переменную массива *char*. 

**len** - размер массива buf

**flags** - **комбинация битовых флагов**, управляющих режимами чтения. Если аргумент flags **равен нулю**, то считанные данные **удаляются из сокета**. Если значение flags есть **MSG_PEEK**, то **данные не удаляются** и могут быть считаны последущим вызовом (или вызовами) *recv*.

>Функция возвращает число считанных байтов или -1 в случае ошибки. Следует отметить, что нулевое значение не является ошибкой. Оно сигнализирует об отсутствии записанных в сокет процессом-поставщиком данных.

### Закрытие соединения

Если вы **не закрыли** соединение, то это может привести к ошибке при попытке подключения следующих запросов. Поэтому после того как вы приняли данные от клиента, **обязательно закройте с ним соединение!!**

Для того чтобы отсоединится от сокета, необходимо воспользоваться функцией **close(socket)**

>WARNING! Если вы не отключились от сокета, и у вас возникают ошибки, попробуйте изменить порт сервера.

## Пример работы сервера

>Примечание: В моём коде я буду использовать встроенные библиотеки для *Linux*, если же у вас *Windows* вам нужно будет установить библиотеки по типу *winsock* и т.д. Эти библиотеки схожи и имеют одну и ту же реализацию.

```C++
#include <iostream>
#include <sys/types.h>
#include <sys/socket.h>
#include <netinet/in.h>
#include <arpa/inet.h>
#include <unistd.h>


using namespace std;

int main(int argc, char *argv[]) {

    sockaddr_in addr;
    addr.sin_family = AF_INET;
    addr.sin_port = htons(7000); // преобразование порта в сетевой порядок байт
    
    addr.sin_addr.s_addr = htons(INADDR_ANY); // Принимать любой запрос

    int server = socket(AF_INET, SOCK_STREAM, 0);

    if (bind(server, (struct sockaddr *) &addr, sizeof(addr)) < 0) {
        cout << "Не удалось связать сокет сервера с адресом." << endl;
        return 1;
    }
    else 
        cout << "Порт открыт" << endl;
    
    if (listen(server, 1) == 0) {
        cout << "Сокет стал в режим прослушки" << endl;
        
        int client; //Файловый дескриптор клиента

        if ((client = accept(server, nullptr, nullptr)) < 0) {
            cout << "Не получилось соединится с клиентом" << endl;
            close(server);
            return -1;
        }
        else
            cout << "Успешное соединение с клиентом" << endl;

        char buf[1024]; // Сообщение которое пришлёт клиент
        int bytes_read; // Размер сообщения в байтах

        if ((bytes_read = recv(client, buf, 1024, 0)) == 0) {
            cout << "Клиент ничего не отправил" << endl;
            return 1;
        }
        else {
            cout << "Получено: " << bytes_read << " байт" << endl;
            cout << buf << endl; //Вывод сообщения от клиента
        }

        close(client);
    }
    else {
        close(server);
        cout << "Не удалось прослушать сокет";
        return 1;
    }

    close(server);

    return 0;
}

```
В данном коде сервер принимает запрос от клиента, создаёт с ним отдельную связь (сокет), через который принимаются данные, которые в последствии выводятся нам в консоль. После чего соединение с клиентом закрывается, как и серверный сокет.

## Сокет клиента

Реализация клиента проще, потому что всё что нужно клиенту - **отправить серверу данные**. Для этого мы должны указать сокету клиента адрес сервера (sockaddr_in), чтобы он знал куда отправлять данные.

Я не буду повторять всё то что мы уже делали (создание сокета и т.д), я расскажу в чём отличия в их реализации.

После того как вы создали сокет клиента, в структуре **sockaddr_in** вы должны написать всё то же самое(порты, семейство), но только теперь в *s_addr* **вы должны указать IP вашего сервера**, и **НЕ ЗАБУДЬТЕ НАПИСАТЬ inet_addr()** для того чтобы перевести понятный нами IP адрес (из чисел и точек) в двоичный код в сетевом порядке расположения байтов. Если входящий адрес неверен, то возвращается *INADDR_NONE* (обычно -1)

Вот пример sockaddr_in:

```C++
#include <sys/socket.h>

sockaddr_in addr;
  addr.sin_family = AF_INET;
  addr.sin_port = htons("ПОРТ ВАШЕГО СЕРВЕРА");
  addr.sin_addr.s_addr = inet_addr("IP СЕРВЕРА");
```
Тут можно заметить, что мы используем разные функции, т.к **htons преобразует целые числа** в сетевой порядок байт, а **inet_addr() - строку.**

>Примечание: Вы можете реализовать сервер и клиент на одной машине, по этому клиенту просто укажите IP адрес вашего устройства.

### Подключение сокета к серверу

После того вы создали **структуру**, содержащую адрес сервера, ваш сокет необходимо **подключить** (не связать!) к серверу по этому адресу.

Для этого используется функция **connect()**

Обратите внимание, что при подключении ваш сервер уже должен работать и слушать входящие соединения listen().

```C++
#include <sys/types.h>
#include <sys/socket.h>

int connect(int sockfd, struct sockaddr *serv_addr, int addrlen);
```
**sockfd** - сокет клиента, которого будете подключать к серверу.

**serv_addr** - указатель на вашу структуру *sockaddr_in*

**addrlen** - размер (в байтах) вашей структуры. Для получения размера просто используйте функцию sizeof().

Если ошибка не возникает, функция возвращает **0**, если иначе тогда **меньше 0**.

### Отправка данных

После того как ваш сокет подключился к серверу, вы, наконец, можете передавать ему данные.

Для того чтобы отправить данные серверу используется функция **send()**

```C++
#include <sys/types.h>
#include <sys/socket.h>

int send(int sockfd, const void *msg, int len, int flags);
```
**sockfd** - сокет клиента.

**msg** - указатель на буфер с данными.

**len** - размер (в байтах) ваших данных.

**flags** - набор **битовых флагов**, управляющих работой функции (если флаги не используются, передайте функции 0).

После того как вы отправили данные серверу, не забудьте закрыть все соединения.

## Пример работы клиента

```C++
#include <iostream>
#include <sys/types.h>
#include <sys/socket.h>
#include <netinet/in.h>
#include <arpa/inet.h>

using namespace std;

int main(int argc, char *argv[]) {

  char *server_ip = argv[1];
  char *message = argv[2];

  int client;
  sockaddr_in addr;
  addr.sin_family = AF_INET;
  addr.sin_port = htons(7000); 
  addr.sin_addr.s_addr = inet_addr(server_ip);
  

  // файловый дескриптор (число)
  client = socket(AF_INET, SOCK_STREAM, 0);

  if (client >= 0) 
    cout << "Сокет успешно создался" << endl;
  else {
    cout << "Проблема создания сокета" << endl;
    return 1;
  }

  if (connect(client, (struct sockaddr *) &addr, sizeof(addr)) < 0) {
    cout << "Не удалось подключится" << endl;
    return 1;
  } 
  else {
    send(client, message, sizeof(message), 0);
    cout << "Отправил ваше сообщение: " << message << endl;
    cout << "Задействовано памяти: " << sizeof(message) << " байт" << endl;
  }

  return 0;
}
```

В функции main() содержаться аргументы, это некий стандарт при работе с сокетами, это можно использовать для того чтобы передавать параметры прямо в консоли

Например:

```bash
./app IP MESSAGE
```
Где **./app** - скомпилированный файл программы клиента

**IP** - IP вашего сервера

**MESSAGE** - ваше сообщение