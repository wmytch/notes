# Connection listeners: accepting TCP connections

[TOC]

## Creating or freeing an evconnlistener

### evconnlistener_new
```c
struct evconnlistener *evconnlistener_new(struct event_base *base,
    evconnlistener_cb cb, void *ptr, unsigned flags, int backlog,
    evutil_socket_t fd);
```
### evconnlistener_new_bind
```c
struct evconnlistener *evconnlistener_new_bind(struct event_base *base,
    evconnlistener_cb cb, void *ptr, unsigned flags, int backlog,
    const struct sockaddr *sa, int socklen);
```
两个`evconnlistener_new*()`函数都是分配并且返回一个新的连接监听对象。当有新的连接到来时，就调用提供的回调函数。

两个函数里的base参数都是监听器用来监听连接的event_base，或者说，监听器绑定的event_base。cb是新连接到达时调用的回调函数，ptr指针将传递给回调函数。flags参数控制监听器的行为，详细解释在下面。backlog参数控制任一时刻允许挂起等待的连接的最大数量，这些挂起的连接指的是网络栈中处于尚未接受状态而等待的连接，与标准`listen()`函数的同名参数意义是一致的。如果backlog是负数，则Libevent会尝试找一个合适的值，如果是0，则Libevent假定之前已经调用过`listen()`，并且在那里已经提供了这个值。通常来说，这个值是多少关系不大。

两个函数的不同之处在于如何设置监听的socket。evconnlistener_new函数假定已经将一个socket与监听端口绑定，fd参数就代表这个socket。evconnlistener_new_bind函数则是根据传入的sa和socklen创建一个socket，并将其与监听端口绑定。需要特别注意的是evconnlistener_new_bind会将创建的socket置为非堵塞的，对于evconnlistener_new则需要保证socket已经设置为非堵塞的，否则会有未定义的错误。

### evconnlistener_free

```c
void evconnlistener_free(struct evconnlistener *lev);
```

释放一个监听器。

### Recognized flags

- LEV_OPT_LEAVE_SOCKETS_BLOCKING

    缺省情况下，对于接受到的socket，会缺省设置为非堵塞的，如果设置了这个标记就会取消这个缺省行为，也就是保留连入的socket为堵塞的。

    另外evconnlistener_new会检查这个标记，如果没有设置，就会把传入的用于绑定的socket设置为非堵塞的。

- LEV_OPT_CLOSE_ON_FREE

    当释放监听器时会关闭底层的socket。

- LEV_OPT_CLOSE_ON_EXEC

    这个标记的意思是接下来如果调用 execv() 或者posix_spawn()系列函数，就自动关闭这个socket。

- LEV_OPT_REUSEABLE

    当关闭一个socket时，其它socket可以马上使用同样的一个端口，而不需要再等待一段时间。

- LEV_OPT_THREADSAFE

    为监听器分配锁。

- LEV_OPT_DISABLED

    初始化监听器之后并不马上启用，而是之后需要时调用evconnlistener_enable来启用。

- LEV_OPT_DEFERRED_ACCEPT

    告诉内核如果可能的话，只有当接收到socket上的数据时，才将socket视为已接受的，并且是可读的。如果所用的协议是只有在收到客户端发送数据时才开始工作，那就不要使用这个标记，这个标记可能会使得内核永远不会告知这个连接的存在。不是所有平台都支持这个选项，在不支持的平台设置这个选项没有效果。

### The connection listener callback

```c
typedef void (*evconnlistener_cb)(struct evconnlistener *listener,
    evutil_socket_t sock, struct sockaddr *addr, int len, void *ptr);
```

当一个新的连接到达就会调用所提供的回调函数。*listener*参数是接受连接的监听器。sock参数是新连入的socket本身，即accept返回的socket，addr和len参数分别是连入的地址结构及其长度。ptr参数则是传入`evconnlistener_new*()`的用户定义的指针。

## Enabling and disabling an evconnlistener

### evconnlistener_disable
### evconnlistener_enable
```c
int evconnlistener_disable(struct evconnlistener *lev);
int evconnlistener_enable(struct evconnlistener *lev);
```
临时禁用和重新启用监听。

## Adjusting an evconnlistener’s callback

### evconnlistener_set_cb

```c
void evconnlistener_set_cb(struct evconnlistener *lev,
    evconnlistener_cb cb, void *arg);
```
调整一个evconnlistener的回调函数及其参数
## Inspecting an evconnlistener
### evconnlistener_get_fd
### evconnlistener_get_base
```c
evutil_socket_t evconnlistener_get_fd(struct evconnlistener *lev);
struct event_base *evconnlistener_get_base(struct evconnlistener *lev);
```
分别返回一个监听器关联的socket和event_base。
## Detecting errors

可以设置在accept出错时的回调函数，如果出错时可能会导致进程锁死那么这个设置就很重要，不然就会不知道出错以及哪里出错了。
### evconnlistener_set_error_cb
```c
typedef void (*evconnlistener_errorcb)(struct evconnlistener *lis, void *ptr);
void evconnlistener_set_error_cb(struct evconnlistener *lev,
    evconnlistener_errorcb errorcb);
```
只要evconnlistener出错就会调用回调函数，回调的ptr参数是传入`evconnlistener_new*()`的用户定义的指针。

## 例子: an echo server.

```c
#include <event2/listener.h>
#include <event2/bufferevent.h>
#include <event2/buffer.h>

#include <arpa/inet.h>

#include <string.h>
#include <stdlib.h>
#include <stdio.h>
#include <errno.h>

static void
echo_read_cb(struct bufferevent *bev, void *ctx)
{
        /* This callback is invoked when there is data to read on bev. */
        struct evbuffer *input = bufferevent_get_input(bev);
        struct evbuffer *output = bufferevent_get_output(bev);

        /* Copy all the data from the input buffer to the output buffer.*/ 
        //bufferevent_write_buffer(bev,bufferevent_get_input(bev));也是一样的
        evbuffer_add_buffer(output, input);
}

static void
echo_event_cb(struct bufferevent *bev, short events, void *ctx)
{
        if (events & BEV_EVENT_ERROR)
                perror("Error from bufferevent");
        if (events & (BEV_EVENT_EOF | BEV_EVENT_ERROR)) {
                bufferevent_free(bev);
        }
}

static void
accept_conn_cb(struct evconnlistener *listener,
    evutil_socket_t fd, struct sockaddr *address, int socklen,
    void *ctx)
{
        /* We got a new connection! Set up a bufferevent for it. */
        struct event_base *base = evconnlistener_get_base(listener);
        struct bufferevent *bev = bufferevent_socket_new(
                base, fd, BEV_OPT_CLOSE_ON_FREE);

        bufferevent_setcb(bev, echo_read_cb, NULL, echo_event_cb, NULL);

        bufferevent_enable(bev, EV_READ|EV_WRITE);
}

static void
accept_error_cb(struct evconnlistener *listener, void *ctx)
{
        struct event_base *base = evconnlistener_get_base(listener);
        int err = EVUTIL_SOCKET_ERROR();
        fprintf(stderr, "Got an error %d (%s) on the listener. "
                "Shutting down.\n", err, evutil_socket_error_to_string(err));

        event_base_loopexit(base, NULL);
}

int
main(int argc, char **argv)
{
        struct event_base *base;
        struct evconnlistener *listener;
        struct sockaddr_in sin;

        int port = 9876;

        if (argc > 1) {
                port = atoi(argv[1]);
        }
        if (port<=0 || port>65535) {
                puts("Invalid port");
                return 1;
        }

        base = event_base_new();
        if (!base) {
                puts("Couldn't open event base");
                return 1;
        }

        /* Clear the sockaddr before using it, in case there are extra
         * platform-specific fields that can mess us up. */
        memset(&sin, 0, sizeof(sin));
        /* This is an INET address */
        sin.sin_family = AF_INET;
        /* Listen on 0.0.0.0 */
        sin.sin_addr.s_addr = htonl(0);
        /* Listen on the given port. */
        sin.sin_port = htons(port);

        listener = evconnlistener_new_bind(base, accept_conn_cb, NULL,
            LEV_OPT_CLOSE_ON_FREE|LEV_OPT_REUSEABLE, -1,
            (struct sockaddr*)&sin, sizeof(sin));
        if (!listener) {
                perror("Couldn't create listener");
                return 1;
        }
        evconnlistener_set_error_cb(listener, accept_error_cb);

        event_base_dispatch(base);
        return 0;
}
```