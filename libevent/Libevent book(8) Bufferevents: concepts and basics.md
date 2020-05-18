# Bufferevents: concepts and basics

[TOC]

大多数情况下，一个应用不仅仅是响应事件，还需要对缓冲数据做某些操作。比如，当我们需要写数据时，模式通常是这样的：

- 需要给一个连接写入数据，把这些数据放入缓冲区。
- 等待该连接变得可写。
- 写入尽可能多的数据。
- 如果还有数据要写，记录写了多少数据，等待连接下一次变得可写入。

Libevent为这种常见的缓冲IO模式提供了一套通用机制，也就是“Bufferevent”，包括了底层的传输(类似socket)，一个读缓冲，一个写缓冲。常规的event，是通过回调函数来处理底层传输的可读或者可写。而对于bufferevent，则是调用用户提供的回调函数来处理有足够的数据可读或者可写。这里主要的区别不在于对回调的形容，而是常规event处理的是传输层，比方说socket的可读可写事件，bufferevent处理的是数据本身，或者说缓冲区里数据的数量引发的事件。

下面是目前为止所有的bufferevent的类型，它们共享一个公共的接口：

- 基于socket的bufferevent

    通过底层的流socket来发送和接收数据，使用`event_*`接口作为后端

- 异步IObufferevent

    使用Windows IOCP接口来发送和接收数据，底层也还是流socket。(只对Windows，实验性质)

- 过滤bufferevent

    在把传入或者传出的数据传入底层的bufferevent对象前，预先进行一些处理的bufferevent，比方说压缩或者转换。

- 成对bufferevent

    两个相互传输数据的bufferevent。

**注意**

- 下面将要描述的接口并不一定都适用于所有的bufferevent类型。
- 目前bufferevent只支持面向流的协议，如TCP，将来也许会支持面向数据报的协议，如UDP。
- 本节所述的所有函数和类型都在event2/bufferevent.h里声明。
- 与evbuffer相关的一些函数在 event2/buffer.h中声明。

## Bufferevents and evbuffers

每一个bufferevent都有一个输入缓冲和一个输出缓冲，它们的类型是所谓的evbuffer结构。当把数据写入到一个bufferevent，就是把数据添加到输出缓冲中，当一个bufferevent有数据可读，则是从输入缓冲中获取，或者说排出。

evbuffer在后面章节讨论。

## Callbacks and watermarks

每个bufferevent都有两个数据相关的回调：读回调和写回调。缺省情况下，当从底层传输中读到任何数据是都会调用读回调，当输出缓冲中的数据足够排空到底层传输时就会调用写回调。这些行为可以通过设置bufferevent的读或者写的水线来改变。

这里需要明确一下，写回调针对的是输出缓冲，而不是底层的传输，底层传输的情况由bufferevent处理，或者说，底层传输可写的话，bufferevent会自动将输出缓冲中的数据加入到底层传输的缓冲中，当bufferevent的输出缓冲中有足够的空间可以写入时，就会调用写回调。

bufferevent有四个水线：

- Read low-water mark

    当一个读操作使得bufferevent的输入缓冲的数据达到或者高于这个线时，就会调用读回调。缺省为0，所以每一次读都会调用读回调。这里的读指的是从底层传输中获取数据，或者说一个read系统调用。

- Read high-water mark

    如果bufferevent的输入缓冲达到了这条线，bufferevent就会停止读取，直到从输入缓冲中排出足够的数据，或者说应用从bufferevent的输入缓冲中取出了足够的数据，使得其中的数据低于这条线为止。缺省是无限的，因此也就不会因为输入缓冲的大小而停止读取数据。同样，这里的读取指的是从底层传输读取。

- Write low-water mark

    当一个写操作使得输出缓冲的数据达到或者低于这条线时，就会调用写回调。缺省是0，因此，除非输出缓冲空了，否则是不会调用写回调的。同样，这里的写操作指的是向底层传输的写入。

- Write high-water mark

    Libevent不直接使用这个水线，当一个bufferevent用来作为另一个Libevent的底层传输时，这条水线有特殊的含义，参见下面过滤bufferevent的说明。

bufferevent还有”error”或者”event”回调，用来告知应用一些与数据无关的事件，比方说连接关闭或者错误发生。有如下的事件标志定义：

- BEV_EVENT_READING

    在对bufferevent进行读操作时发生的事件。需要查看别的事件标记来确定是什么事件。

- BEV_EVENT_WRITING

    在对bufferevent进行写操作时发生的事件。需要查看别的事件标记来确定是什么事件。

- BEV_EVENT_ERROR

     在操作bufferevent是发生的错误，调用`EVUTIL_SOCKET_ERROR()`来获知是什么错误。

- BEV_EVENT_TIMEOUT

    bufferevent超时

- BEV_EVENT_EOF

    在bufferevent得到一个文件结束标记。

- BEV_EVENT_CONNECTED

    完成一个bufferevent上的连接请求。

##  Deferred callbacks

缺省情况下，一个bufferevent回调当对应情况发生时就会马上执行，对应evbuffer回调也是如此。这种即时调用在依赖性比较复杂时可能会造成麻烦，比方说，假定一个回调在evbuffer A变空时向其写入数据，另一个回调在evbuffer A满的时候处理其中的数据，由于这些回调是在栈上发生的，当依赖性变得一团糟时可能就会有栈溢出的风险。

为了解决这个问题，可以告知bufferevent或者evbuffer，其回调应该延时调用。当一个延时回调的条件满足时，并不会马上调用，而是在`event_loop()`中排队，如常规事件一样等待被调用。

## Option flags for bufferevents

创建bufferevent时可以设置一些标记来改变其行为。可识别的标记有：

- BEV_OPT_CLOSE_ON_FREE

    当bufferevent被释放时，关闭底层的传输。包括关闭底层的socket，释放底层的bufferevent，等等。

- BEV_OPT_THREADSAFE

    自动分配为bufferevent锁，这样可以在多线程中安全的使用。

- BEV_OPT_DEFER_CALLBACKS

    延时调用所有的回调，参见上面的解释。

- BEV_OPT_UNLOCK_CALLBACKS

    缺省情况下，bufferevent是设置为线程安全的，当一个用户提供的回调调用时，bufferevent都会锁定。这个参数使得bufferevent在调用回调是释放其锁。

## Working with socket-based bufferevents

基于socket的bufferevent是最易于使用的，其使用Libevent的底层传输机制来检测底层网络socket可以读或写的时机，并且使用底层网络调用(如readv、writev、WSASend、WSARecv)来发送和接收数据。

### Creating a socket-based bufferevent

#### bufferevent_socket_new

```c
struct bufferevent *bufferevent_socket_new(
    struct event_base *base,
    evutil_socket_t fd,
    enum bufferevent_options options);
```

base是一个event_base，options是bufferevent可选项(BEV_OPT_CLOSE_ON_FREE等)的bitmask，fd是一个socket的文件描述符，如果打算之后再设置这个文件描述符，可以将其设为-1。

注意确保把提供给bufferevent_socket_new的socket设置为非堵塞的。Libevent提供的evutil_make_socket_nonblocking可以很方便的用来实现这点。

成功返回一个bufferevent，失败返回NULL。注意是指针。

### Launching connections on socket-based bufferevents

#### bufferevent_socket_connect

```c
int bufferevent_socket_connect(struct bufferevent *bev,
    struct sockaddr *address, int addrlen);
```

address和addrlen参数与标准的`connect()`一致。如果bufferevent还没有设置一个socket，这个函数会分配一个新的流socket，并将其置为非堵塞。这里只是说此函数会自动分配并设置非堵塞，并不表明所有使用bev的函数都会如此。

如果bufferevent**确实**已经有了一个socket，这个函数会告知Libevent该socket还没有连接，不应该在这个socket上读或写，直到连接操作成功。

在连接结束前向输出缓冲添加数据时可以的。

如果连接成功函数返回0，如果发生错误返回-1。

### 例子

```c
#include <event2/event.h>
#include <event2/bufferevent.h>
#include <sys/socket.h>
#include <string.h>

void eventcb(struct bufferevent *bev, short events, void *ptr)
{
    if (events & BEV_EVENT_CONNECTED) {
         /* We're connected to 127.0.0.1:8080.   Ordinarily we'd do
            something here, like start reading or writing. */
    } else if (events & BEV_EVENT_ERROR) {
         /* An error occured while connecting. */
    }
}

int main_loop(void)
{
    struct event_base *base;
    struct bufferevent *bev;
    struct sockaddr_in sin;

    base = event_base_new();

    memset(&sin, 0, sizeof(sin));
    sin.sin_family = AF_INET;
    sin.sin_addr.s_addr = htonl(0x7f000001); /*或者 inet_addr("127.0.0.1") */
    sin.sin_port = htons(8080); /* Port 8080 */

    bev = bufferevent_socket_new(base, -1, BEV_OPT_CLOSE_ON_FREE);

    bufferevent_setcb(bev, NULL, NULL, eventcb, NULL);

    if (bufferevent_socket_connect(bev,
        (struct sockaddr *)&sin, sizeof(sin)) < 0) {
        /* Error starting connection */
        bufferevent_free(bev);
        return -1;
    }

    event_base_dispatch(base);
    return 0;
}
```

注意只有在使用`bufferevent_socket_connect()`时才会得到BEV_EVENT_CONNECTED事件，如果直接调用`connect()`，则连接会被报告为写。

如果想自己调用`connect()`，但又想在连接成功时得到BEV_EVENT_CONNECTED事件，则可以在调用`connect()`以errno为EAGAIN或者EINPROGRESS返回-1后，调用`bufferevent_socket_connect(bev, NULL, 0)`。当然要清楚，这时候connect所使用的socket必须是已经设置为非堵塞的了。不大想象得出这种用法的使用场景，列在这里以便万一需要的时候做个参考。

### Launching connections by hostname

#### bufferevent_socket_connect_hostname

```c
int bufferevent_socket_connect_hostname(struct bufferevent *bev,
    struct evdns_base *dns_base, int family, const char *hostname,
    int port);
```
#### bufferevent_socket_get_dns_error
```c
int bufferevent_socket_get_dns_error(struct bufferevent *bev);
```

函数解析DNS名称hostname，查找类型为family的地址。允许的family类型是AF_INET, AF_INET6, AF_UNSPEC。如果名字解析失败，调用event回调，如果成功，则发起与bufferevent_connect一样的连接操作。

dns_base参数是可选的，如果为NULL，则Libevent在名字查找完成前堵塞，这通常并不是所想要的。如果提供了，则Libevent使用这个参数以异步方式查找主机名。具体可参见后面关于DNS的章节。

与`bufferevent_socket_connect()`一样，这个函数在解析完成以及连接成功之前，都会告知Libevent，bufferevent已有的socket还没有连接，不应该在其上进行读或写操作。

如果有错误发生，可能是个DNS主机名查找错误。通过调用`bufferevent_socket_get_dns_error()`可以找到最近发生的错误。如果返回的错误码为0，则表示没有检测到DNS错误。

#### 例子：Trivial HTTP v0 client.

```c
/* Don't actually copy this code: it is a poor way to implement an
   HTTP client.  Have a look at evhttp instead.
*/
#include <event2/dns.h>
#include <event2/bufferevent.h>
#include <event2/buffer.h>
#include <event2/util.h>
#include <event2/event.h>

#include <stdio.h>

void readcb(struct bufferevent *bev, void *ptr)
{
    char buf[1024];
    int n;
    struct evbuffer *input = bufferevent_get_input(bev);
    while ((n = evbuffer_remove(input, buf, sizeof(buf))) > 0) {
        fwrite(buf, 1, n, stdout);
    }
}

void eventcb(struct bufferevent *bev, short events, void *ptr)
{
    if (events & BEV_EVENT_CONNECTED) {
         printf("Connect okay.\n");
    } else if (events & (BEV_EVENT_ERROR|BEV_EVENT_EOF)) {
         struct event_base *base = ptr;
         if (events & BEV_EVENT_ERROR) {
                 int err = bufferevent_socket_get_dns_error(bev);
                 if (err)
                         printf("DNS error: %s\n", evutil_gai_strerror(err));
         }
         printf("Closing\n");
         bufferevent_free(bev);
         event_base_loopexit(base, NULL);
    }
}

int main(int argc, char **argv)
{
    struct event_base *base;
    struct evdns_base *dns_base;
    struct bufferevent *bev;

    if (argc != 3) {
        printf("Trivial HTTP 0.x client\n"
               "Syntax: %s [hostname] [resource]\n"
               "Example: %s www.google.com /\n",argv[0],argv[0]);
        return 1;
    }

    base = event_base_new();
    dns_base = evdns_base_new(base, 1);

    bev = bufferevent_socket_new(base, -1, BEV_OPT_CLOSE_ON_FREE);
    bufferevent_setcb(bev, readcb, NULL, eventcb, base);
    bufferevent_enable(bev, EV_READ|EV_WRITE);
    evbuffer_add_printf(bufferevent_get_output(bev), "GET %s\r\n", argv[2]);
    bufferevent_socket_connect_hostname(
        bev, dns_base, AF_UNSPEC, argv[1], 80);
    event_base_dispatch(base);
    return 0;
}
```

## Generic bufferevent operations

### Freeing a bufferevent

#### bufferevent_free

```c
void bufferevent_free(struct bufferevent *bev);
```

释放一个bufferevent。Bufferevent内部是引用计数的，所以在释放一个buffevent的时候，如果有延时回调正处于挂起等待状态，则会等待所有这些回调完成之后才会删除这个bufferevent。

不过，bufferevent_free是只要有可能就会尽快释放bufferevent。如果有挂起的数据等待写入bufferevent，那么在bufferevent被释放前这些数据可能不会被写入到bufferevent中，就是说释放bufferevent可能会丢失一部分数据。

如果设置了BEV_OPT_CLOSE_ON_FREE标记，并且bufferevent由socket或者底层bufferevent来传输数据，当释放bufferevent时，传输会关闭。

### Manipulating callbacks, watermarks, and enabled operations

#### 回调函数原型

```c
typedef void (*bufferevent_data_cb)(struct bufferevent *bev, void *ctx);
typedef void (*bufferevent_event_cb)(struct bufferevent *bev,
    short events, void *ctx);
```
#### bufferevent_setcb
```c
void bufferevent_setcb(struct bufferevent *bufev,
    bufferevent_data_cb readcb, bufferevent_data_cb writecb,
    bufferevent_event_cb eventcb, void *cbarg);
```
bufferevent_setcb用来设置或者改变指定bufferevent的回调函数。

- bufev是指定的bufferevent，其余几个参数所代表的的事件都发生在这个bufferevent上。
- readcb在有足够的数据可读时调用，writecb在有足够的数据可写时调用，eventcb在有事件发生时调用，这几个回调函数的相关问题可以参照前面的一些说明，这里说的只是大概情况，并不十分严格或者说确切。如果参数被置为NULL，则表示禁用相应的回调。
- cbarg_ptr是用来给回调函数传递参数的，或者说这个参数会变成如上面回调函数签名里的ctx，传入回调函数，通常会在回调函数里对ctx做一个类型转换，然后再使用，或者说，对一个回调函数而言，ctx提供了一个泛型机制。所有设置的回调函数共享这个参数，所以要注意其设置。
- bufferevent_event_cb的events参数是事件标记的一个bitmask，当然这是在回调发生时由Bufferevent自动传入的。

#### bufferevent_getcb

```c
void bufferevent_getcb(struct bufferevent *bufev,
    bufferevent_data_cb *readcb_ptr,
    bufferevent_data_cb *writecb_ptr,
    bufferevent_event_cb *eventcb_ptr,
    void **cbarg_ptr);
```

bufferevent_getcb用来查询bufferevent当前的回调设置。要注意的是其参数都是指针，这几个参数都是作为输入输出参数传入函数，被设为NULL的参数表示忽略相应的内容。所有的参数输入前只需要声明，但不需要分配空间，函数只是把bufev结构的域的值简单的赋给参数，返回后，因为函数指针和函数名是等价的，不会有什么使用上的区别或者说误解，就不多说了了，只是要稍微说下cbarg_ptr这个参数，作为输出参数后续使用时要先做转换，转换的对象是`*cbarg_ptr`，而不是cbarg_ptr，因为cbarg_ptr所指向的内存区存放是一个地址，我们要获取的是这个地址，而不是cbarg_ptr本身的地址，所以要对`*cbarg_ptr`进行操作，而不是对cbarg_ptr，至于为什么传入一个`void **`，而不是一个`void *`，这涉及到对c语言指针和函数参数传递机制的的理解，就不废话了。

#### bufferevent_enable

#### bufferevent_disable

#### bufferevent_get_enabled

```c
void bufferevent_enable(struct bufferevent *bufev, short events);
void bufferevent_disable(struct bufferevent *bufev, short events);
short bufferevent_get_enabled(struct bufferevent *bufev);
```

可以设置或者禁止bufferevent上的EV_READ、EV_WRITE、或者EV_READ|EV_WRITE事件。当没有设置读或写事件是，bufferevent将不会去读或者写数据。这里的读或者写指的是从底层传输读取或者向底层传输写入。

当输出缓冲空时并不需要禁止写，bufferevent会自动停止，当有数据时会自动启动。

类似的，当输入缓冲达到高水位线时也不需要禁止读，bufferevent也会自动停止读，然后当有空间的时候再自动启动。

缺省情况下，新创建的bufferevent会设置可写，但不设置可读。

`bufferevent_get_enabled()`用来查看bufferevent当前启用了哪些事件。 

#### bufferevent_setwatermark

```c
void bufferevent_setwatermark(struct bufferevent *bufev, short events,
    size_t lowmark, size_t highmark);
```

用来调整一个bufferevent的读写水线。如果events设为EV_READ，则调整读水线，如果设置为EV_WRITE，则调整读水位线。但是除非打算给读写两类水线设为一样的值，否则没有理由设置events为EV_READ|EV_WRITE，虽然是允许的。

高水位线设为0表示无限。

#### 例子：

```c
#include <event2/event.h>
#include <event2/bufferevent.h>
#include <event2/buffer.h>
#include <event2/util.h>

#include <stdlib.h>
#include <errno.h>
#include <string.h>

struct info {
    const char *name;
    size_t total_drained;
};

void read_callback(struct bufferevent *bev, void *ctx)
{
    struct info *inf = ctx;
    struct evbuffer *input = bufferevent_get_input(bev);
    size_t len = evbuffer_get_length(input);
    if (len) {
        inf->total_drained += len;
        evbuffer_drain(input, len);
        printf("Drained %lu bytes from %s\n",
             (unsigned long) len, inf->name);
    }
}

void event_callback(struct bufferevent *bev, short events, void *ctx)
{
    struct info *inf = ctx;
    struct evbuffer *input = bufferevent_get_input(bev);
    int finished = 0;

    if (events & BEV_EVENT_EOF) {
        size_t len = evbuffer_get_length(input);
        printf("Got a close from %s.  We drained %lu bytes from it, "
            "and have %lu left.\n", inf->name,
            (unsigned long)inf->total_drained, (unsigned long)len);
        finished = 1;
    }
    if (events & BEV_EVENT_ERROR) {
        printf("Got an error from %s: %s\n",
            inf->name, evutil_socket_error_to_string(EVUTIL_SOCKET_ERROR()));
        finished = 1;
    }
    if (finished) {
        free(ctx);
        bufferevent_free(bev);
    }
}

struct bufferevent *setup_bufferevent(void)
{
    struct bufferevent *b1 = NULL;
    struct info *info1;

    info1 = malloc(sizeof(struct info));
    info1->name = "buffer 1";
    info1->total_drained = 0;

    /* ... Here we should set up the bufferevent and make sure it gets
       connected... */

    /* Trigger the read callback only whenever there is at least 128 bytes
       of data in the buffer. */
    bufferevent_setwatermark(b1, EV_READ, 128, 0);

    bufferevent_setcb(b1, read_callback, NULL, event_callback, info1);

    bufferevent_enable(b1, EV_READ); /* Start reading. */
    return b1;
}
```

### Manipulating data in a bufferevent

#### bufferevent_get_input

####  bufferevent_get_output

```c
struct evbuffer *bufferevent_get_input(struct bufferevent *bufev);
struct evbuffer *bufferevent_get_output(struct bufferevent *bufev);
```

这两个函数分别返回输入和输出缓冲，详细信息可以查看下一章的evbuffer类型。

注意应用对输入缓冲只能移除数据而不能添加数据，对输出缓冲只能添加而不能移除数据。

如果bufferevent由于输出缓冲的数据过少而停止了向传输写入，那么就会自动启动向输出缓冲的数据添加。如果输入缓冲的数据过多而使得从传输读取数据停止了，bufferevent会等到从输入缓冲移除一定量的数据之后再自动启动读取。

#### bufferevent_write

#### bufferevent_write_buffer

```c
int bufferevent_write(struct bufferevent *bufev,
    const void *data, size_t size);
int bufferevent_write_buffer(struct bufferevent *bufev,
    struct evbuffer *buf);
```

这两个函数都是把数据添加到bufev的输出缓冲中。

bufferevent_write把内存中位于data的size字节的数据添加到bufev的输出缓冲的末端。

bufferevent_write_buffer移除buf的所有数据然后放到bufev的输出缓冲的末端。

两个函数都是成功返回0，失败返回-1。

#### bufferevent_read

#### bufferevent_read_buffer

```c
size_t bufferevent_read(struct bufferevent *bufev, void *data, size_t size);
int bufferevent_read_buffer(struct bufferevent *bufev,
    struct evbuffer *buf);
```

这两个函数都是从bufev的输入缓冲中移除数据。

bufferevent_read从输入缓冲移除最多size字节的数据，然后存储到data所指的内存中，返回实际移除的数据。

bufferevent_read_buffer把输入缓冲的所有内容排空，并把它们放到buf中，成功返回0，失败返回-1。

要注意对于bufferevent_read，data指向的内存块必须要有足够大的空间能够存放size字节的数据。

#### 例子

```c
#include <event2/bufferevent.h>
#include <event2/buffer.h>

#include <ctype.h>

void
read_callback_uppercase(struct bufferevent *bev, void *ctx)
{
        /* This callback removes the data from bev's input buffer 128
           bytes at a time, uppercases it, and starts sending it
           back.

           (Watch out!  In practice, you shouldn't use toupper to implement
           a network protocol, unless you know for a fact that the current
           locale is the one you want to be using.)
         */

        char tmp[128];
        size_t n;
        int i;
        while (1) {
                n = bufferevent_read(bev, tmp, sizeof(tmp));
                if (n <= 0)
                        break; /* No more data. */
                for (i=0; i<n; ++i)
                        tmp[i] = toupper(tmp[i]);
                bufferevent_write(bev, tmp, n);
        }
}

struct proxy_info {
        struct bufferevent *other_bev;
};
void
read_callback_proxy(struct bufferevent *bev, void *ctx)
{
        /* You might use a function like this if you're implementing
           a simple proxy: it will take data from one connection (on
           bev), and write it to another, copying as little as
           possible. */
        struct proxy_info *inf = ctx;

        bufferevent_read_buffer(bev,
            bufferevent_get_output(inf->other_bev));
}

struct count {
        unsigned long last_fib[2];
};

void
write_callback_fibonacci(struct bufferevent *bev, void *ctx)
{
        /* Here's a callback that adds some Fibonacci numbers to the
           output buffer of bev.  It stops once we have added 1k of
           data; once this data is drained, we'll add more. */
        struct count *c = ctx;

        struct evbuffer *tmp = evbuffer_new();
        while (evbuffer_get_length(tmp) < 1024) {
                 unsigned long next = c->last_fib[0] + c->last_fib[1];
                 c->last_fib[0] = c->last_fib[1];
                 c->last_fib[1] = next;

                 evbuffer_add_printf(tmp, "%lu", next);
        }

        /* Now we add the whole contents of tmp to bev. */
        bufferevent_write_buffer(bev, tmp);

        /* We don't need tmp any longer. */
        evbuffer_free(tmp);
}
```

### Read- and write timeouts

与其他事件一样，可以设置bufferevent读或写的超时时间，如果超时了，就可以调用超时事件回调。

#### bufferevent_set_timeouts

```c
void bufferevent_set_timeouts(struct bufferevent *bufev,
    const struct timeval *timeout_read, const struct timeval *timeout_write);
```

把超时设为NULL本来是要移除超时设置，但是在Libevent 2.1.2-alpha之前对所有事件这都不不起效。之后大概也行可以了，但是也不好说。

当bufferevent在要读的时候如果等待了至少*timeout_read* 秒，就会触发读超时。如果在要写的时候等待了至少*timeout_write* ，则会触发写超时。

注意只有当bufferevent打算开始读或者写的时候才开始计时。换句话说，如果bufferevent禁止读，或者当输入缓冲满了(达到高水线)，那么读超时就不会启用。类似的，如果写被禁止，或者没有数据可写，则写超时也不会启用。简单的说，就是bufferevent要开始读或写了，才会开始计时，如果根本就没有或者打算进行这个操作，那么就不会计时。

但读或者写超时发生时，在bufferevent上相应的读或写操作会被禁止，然后调用事件回调函数，其标记参数为BEV_EVENT_TIMEOUT|BEV_EVENT_READING或者BEV_EVENT_TIMEOUT|BEV_EVENT_WRITING。

### Initiating a flush on a bufferevent

#### bufferevent_flush

```c
int bufferevent_flush(struct bufferevent *bufev,
    short iotype, enum bufferevent_flush_mode state);
```

刷新一个bufferevent表示bufferevent要强制地使底层传输读入或者发送尽可能多的数据，并忽略其它任何的限制。注意指的是把数据接收到底层传输或者从底层传输发送出去，而不是从bufferevent的输入缓冲把数据读取到应用，或者把应用的数据写入到输出缓冲。

iotype参数应该是EV_READ、EV_WRITE或者EV_READ|EV_WRITE。state参数可以是BEV_NORMAL、BEV_FLUSH、BEV_FINISHED之一，BEV_FINISHED表示另一端应该告知没有更多数据发送了，BEV_NORMAL和BEV_FLUSH的区别取决于bufferevent的类型。

返回-1表示失败，0表示没有数据被刷新，1表示刷新了一些数据。

到Libevent 2.0.5-beta，bufferevent_flush只应用于某些bufferevent类型，特别的是，到目前版本为止，基于socket的bufferevent也还是不支持的。

## Type-specific bufferevent functions

下面的函数并不是所有的bufferevent类型都支持。

### bufferevent_priority_set
### bufferevent_get_priority

```c
int bufferevent_priority_set(struct bufferevent *bufev, int pri);
int bufferevent_get_priority(struct bufferevent *bufev);
```

bufferevent_priority_set调整的是bufev对应的events的优先权。成功返回0，失败返回-1。

bufferevent_get_priority获取的是bufev对应的events的优先权，不存在失败的情况，至少会返回一个event_base的缺省优先权。

仅适用于基于socket的bufferevent。

#### bufferevent_setfd

####bufferevent_getfd 

```c
int bufferevent_setfd(struct bufferevent *bufev, evutil_socket_t fd);
evutil_socket_t bufferevent_getfd(struct bufferevent *bufev);
```

设置或者返回基于文件描述符事件的文件描述符。只有基于socket的bufferevent支持setfd。失败都返回-1，setfd成功返回0。

#### bufferevent_get_base

```c
struct event_base *bufferevent_get_base(struct bufferevent *bev);
```

就不废话l了。

#### bufferevent_get_underlying

```c
struct bufferevent *bufferevent_get_underlying(struct bufferevent *bufev);
```

返回作为底层传输的另一个bufferevent，当然只适用于filtering bufferevents。

## Manually locking and unlocking a bufferevent

有时候需要确保对bufferevent包括evbuffer的操作的原子性。

#### bufferevent_lock
#### bufferevent_unlock

```c
void bufferevent_lock(struct bufferevent *bufev);
void bufferevent_unlock(struct bufferevent *bufev);
```

注意创建bufferevent时如果没有指定BEV_OPT_THREADSAFE标记，或者Libevent的线程支持没有激活，锁定一个bufferevent是没有效果的。

锁定一个bufferevent也会同时锁定其关联的evbuffer。这些函数是递归的，锁定一个已经锁定的bufferevent是安全的。当然，每次锁定都必须对应一次解锁。

