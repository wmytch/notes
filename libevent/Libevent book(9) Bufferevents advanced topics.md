# Bufferevents: advanced topics

[TOC]

## Paired bufferevents

有时候可能需要一个网络程序自己与自己通信。比方说，有一个程序可能需要向一个用户隧道连接写入数据，这个隧道连接是建立在某种协议基础上的，而程序可能在某些时候想通过这个协议来建立与自身的隧道连接。这可以与程序自己的监听端口建立连接来实现，当然，通过网络栈与自己通信是要浪费资源的。

于是，可以创建一对成对的bufferevent来实现相互之间的通信，而不需要真正使用平台的socket。

### bufferevent_pair_new

```c
int bufferevent_pair_new(struct event_base *base, int options,
    struct bufferevent *pair[2]);
```

调用bufferevent_pair_new会把`pair[0]`和`pair[1]`设置为一对相互连接的bufferevent。除了不支持BEV_OPT_CLOSE_ON_FREE，因为其没有意义，以及BEV_OPT_DEFER_CALLBACKS，因为其总是支持的，之外通常的可选项都是支持的。

为什么成对bufferevent要延迟运行回调？有个很常见的场景，对成对bufferevent中的一个进行操作，从而调用其回调改变了该bufferevent，接下来又会调用另外一个bufferevent的回调，然后不断的反复。如果回调不延时，这个调用链可能会很频繁，以致栈溢出，吞噬其它连接，然后使得所有回调重入。

成对bufferevent支持刷新：把bufferevent_flush的mode参数设置为BEV_NORMAL或者BEV_FLUSH可以忽略水线设置，强制把相关数据从一个bufferevent向另一个bufferevent传送。把mode设置为BEV_FINISHED则为对端bufferevent额外的生成一个EOF事件。

释放一个成员，不会自动的释放另一个，也不会产生EOF事件。仅仅只是使另一个成员变成未链接的。当一个bufferevent处于未链接时，将不再能够成功的读写或者产生事件。

### bufferevent_pair_get_partner

```c
struct bufferevent *bufferevent_pair_get_partner(struct bufferevent *bev)
```

这个函数返回另一端的bufferevent，如果存在的话，否则返回NULL。

## Filtering bufferevents

有时候可能需要通过另一个bufferevent对象来传输数据，比方说要添加一个压缩层，或者把协议封装到另一个协议中。

### bufferevent_filter_new

```c
enum bufferevent_filter_result {
        BEV_OK = 0,
        BEV_NEED_MORE = 1,
        BEV_ERROR = 2
};
typedef enum bufferevent_filter_result (*bufferevent_filter_cb)(
    struct evbuffer *source, struct evbuffer *destination, ev_ssize_t dst_limit,
    enum bufferevent_flush_mode mode, void *ctx);


struct bufferevent *bufferevent_filter_new(struct bufferevent *underlying,
        bufferevent_filter_cb input_filter,
        bufferevent_filter_cb output_filter,
        int options,
        void (*free_context)(void *),
        void *ctx);
```

创建一个新的过滤bufferevent，将一个已经存在的”underlying"bufferevent封装起来。所有底层bufferevent收到的数据都由input_filter进行处理，然后再送到过滤bufferevent。过滤bufferevent所有发送的数据也都先由output_filter处理，然后再送到底层的bufferevent。

给一个底层bufferevent添加一个过滤器，会替换底层bufferevent的回调。不过仍然可以给底层bufferevent的evbuffer添加回调，但是如果希望过滤有效，就不能给底层bufferevent本身再设置回调。

options支持所有的常规可选项。如果设置了BEV_OPT_CLOSE_ON_FREE，则释放过滤bufferevent也会释放底层bufferevent。ctx参数是传给过滤bufferevent的任意指针，如果提供了free_context函数，在过滤bufferevent关闭前用来对ctx进行处理。

只要底层输入缓冲有数据到达，就会调用输入过滤函数。只要过滤输出缓冲有数据，就会调用输出过滤函数。过滤函数接收两个evbuffer，从*source* 中读取数据，向*destination*写入数据。*dst_limit*参数描述了向*destination*添加的数据的上界，以字节为单位，过滤函数可以忽略这个参数，但是这样会违反高水位线或者传输率限制。dst_limit为-1表示没有限制。mode参数告诉过滤器以什么方式写入。BEV_NORMAL表示只要能够方便的转换就尽可能的写入，BEV_FLUSH表示尽可能的写入，BEV_FINISHED表示在流的最后做一些额外的清理工作。最后，ctx参数是bufferevent_filter_new提供的一个void指针。

只要有数据成功的写入目标缓冲，过滤函数返回BEV_OK，如果只有在从输入得到更多的数据，或者使用了其它的刷新模式，才能继续向输出缓冲写入时，就会返回BEV_NEED_MORE，如果过滤器发生不可恢复的错误，则返回BEV_ERROR。

创建过滤器就启用了底层bufferevent的可读和可写。不需要在自己管理读写，过滤器在不需要读的时候会自动挂起从底层bufferevent的读取。对2.0.8-rc及之后的版本，对底层bufferevent的读写设定可以独立于过滤器，但是如果真的这样做的话，可能会使得过滤器不能成功在需要时得到数据。

输入过滤和输出过滤函数不需要全部指定，被忽略的过滤函数会被直接传输而不转换的函数替代。

## Limiting maximum single read/write size

缺省情况下，事件循环的每次调用中，bufferevent都不会读或者写最大可能数量的数据，那样会造成不公平以及资源饥渴。但是另一方面，缺省情况并不适用于所有情形。

### bufferevent_set_max_single_read
### bufferevent_set_max_single_write
### bufferevent_get_max_single_read
### bufferevent_get_max_single_write
```c
int bufferevent_set_max_single_read(struct bufferevent *bev, size_t size);
int bufferevent_set_max_single_write(struct bufferevent *bev, size_t size);
ev_ssize_t bufferevent_get_max_single_read(struct bufferevent *bev);
ev_ssize_t bufferevent_get_max_single_write(struct bufferevent *bev);
```

两个set函数分别替换当前的读写最大限制，如果size为0或者大于EV_SSIZE_MAX，则替换最大值为缺省值。成功返回0，失败返回-1。

两个get函数分别返回当前每次循环读写的最大限制。

## Bufferevents and Rate-limiting

一些程序需要限制单一个bufferevent或者一组bufferevent的带宽。

### The rate-limiting model

Libevent的速率限制通过一个令牌桶来决定一次读或者写多少字节的数据。每一个速率限制的对象，在任意给定时间，都有一个读桶和一个写桶，其大小决定了这个对象能够马上读和写的字节数。每个桶有一个重填率，一个最大爆发大小，和一个计时单位或者说"tick”。每当过去一个计时单位，桶就依重填率装填数据，如果桶的数据超过了其爆发大小，超出的字节都会丢失。

因此，重填率决定了一个对象发送或者接收数据的最大平均速率，爆发大小决定了一次爆发所发送或者接收的最大数量，计时单位决定了流量的平滑性。

### Setting a rate limit on a bufferevent

#### ev_token_bucket_cfg

```c
#define EV_RATE_LIMIT_MAX EV_SSIZE_MAX
struct ev_token_bucket_cfg;
```
*ev_token_bucket_cfg*结构代表了一对令牌桶的设置值。

#### ev_token_bucket_cfg_new

```c
struct ev_token_bucket_cfg *ev_token_bucket_cfg_new(
        size_t read_rate, size_t read_burst,
        size_t write_rate, size_t write_burst,
        const struct timeval *tick_len);
```
ev_token_bucket_cfg_new用来创建一个*ev_token_bucket_cfg*并提供最大平均读速率，最大爆发读速率，最大写速率，最大爆发写速率，以及一个tick的长度。如果tick_len为NULL，则tick的长度为缺省的一秒。如果出错，则函数返回NULL。

注意*read_rate*和*write_rate*参数会以字节每tick为单位进行调整，也就是说，如果tick是一秒的长度，而read_rate是300，则最大平均读速率为3000字节每秒。这一段说明在ev_token_bucket_cfg_new的代码里没有找到对应的内容，所以应该是在别的地方做的调整。所以，先忽略吧。

不支持超过EV_RATE_LIMIT_MAX的速率和爆发值。

#### ev_token_bucket_cfg_free

```c
void ev_token_bucket_cfg_free(struct ev_token_bucket_cfg *cfg);
```
ev_token_bucket_cfg_free用来释放一个ev_token_bucket_cfg。注意如果一个ev_token_bucket_cfg有bufferevent仍然在使用，则释放是不安全的。

#### bufferevent_set_rate_limit

```c
int bufferevent_set_rate_limit(struct bufferevent *bev,
    struct ev_token_bucket_cfg *cfg);
```
bufferevent_set_rate_limit用来限制bufferevent的传输率，可以把同一个ev_token_bucket_cfg应用到任意的bufferevent上。要移除一个bufferevent的速率限制，调用bufferevent_set_rate_limit时把cfg参数设为NULL即可。

### Setting a rate limit on a group of bufferevents

一个bufferevent同一时间只能属于不超过一个的速率限制组。可以同时给bufferevent设置一个单独的读写速率限制和一个组速率限制，如果同时设置了两者，则比较低的那个设置起效。

#### bufferevent_rate_limit_group_new
```c
struct bufferevent_rate_limit_group;

struct bufferevent_rate_limit_group *bufferevent_rate_limit_group_new(
        struct event_base *base,
        const struct ev_token_bucket_cfg *cfg);
```
用来构造一个速率限制组。成功返回0，失败返回-1。

#### bufferevent_rate_limit_group_set_cfg

```c
int bufferevent_rate_limit_group_set_cfg(
        struct bufferevent_rate_limit_group *group,
        const struct ev_token_bucket_cfg *cfg);
```
用来改变一个现有的速率限制组的限制设置。成功返回0，失败返回-1.

#### bufferevent_rate_limit_group_free

```c
void bufferevent_rate_limit_group_free(struct bufferevent_rate_limit_group *);
```
移除限制组的所有成员，并释放这个组的结构。

#### bufferevent_add_to_rate_limit_group

```c
int bufferevent_add_to_rate_limit_group(struct bufferevent *bev,
    struct bufferevent_rate_limit_group *g);
```
用来向组中添加bufferevent。需要说明的是，这里除了操作组的结构外，还要设置bufferevent结构中的相关域。成功返回0，失败返回-1。

#### bufferevent_remove_from_rate_limit_group

```c
int bufferevent_remove_from_rate_limit_group(struct bufferevent *bev);
```

用来将一个bufferevent从速率限制组中移除，注意这里没有带组参数，因为属于哪个组是在bufferevent的结构里设置了的。成功返回0，失败返回-1。

### Inspecting current rate-limit values

#### bufferevent_get_read_limit
#### bufferevent_get_write_limit
#### bufferevent_rate_limit_group_get_read_limit
#### bufferevent_rate_limit_group_get_write_limit

```c
ev_ssize_t bufferevent_get_read_limit(struct bufferevent *bev);
ev_ssize_t bufferevent_get_write_limit(struct bufferevent *bev);
ev_ssize_t bufferevent_rate_limit_group_get_read_limit(
        struct bufferevent_rate_limit_group *);
ev_ssize_t bufferevent_rate_limit_group_get_write_limit(
        struct bufferevent_rate_limit_group *);
```

函数分别返回各自名字所示的令牌桶的当前大小。要注意的是如果在某种操作时，一个bufferevent实际的令牌桶被强制的超出其所分配的大小，返回值可能为负值，比方说刷新一个bufferevent可能会造成这种情况。

#### bufferevent_get_max_to_read
#### bufferevent_get_max_to_write
```c
ev_ssize_t bufferevent_get_max_to_read(struct bufferevent *bev);
ev_ssize_t bufferevent_get_max_to_write(struct bufferevent *bev);
```

返回当前bufferevent可以读或写的字节数，综合考虑了bufferevent的任意速率限制，组速率限制(如果有的话)，以及Libevent限定的最大的一次读写的数据量。

#### bufferevent_rate_limit_group_get_totals
#### bufferevent_rate_limit_group_reset_totals
```c
void bufferevent_rate_limit_group_get_totals(
    struct bufferevent_rate_limit_group *grp,
    ev_uint64_t *total_read_out, ev_uint64_t *total_written_out);
void bufferevent_rate_limit_group_reset_totals(
    struct bufferevent_rate_limit_group *grp);
```

每一个bufferevent_rate_limit_group都会追踪其成员发送的数据的总量。调用bufferevent_rate_limit_group_get_totals可以通过total_read_out和total_written_out分别获取一个组中所有成员读和写的数据的总量。

当组创建时，起始值设为0，调用bufferevent_rate_limit_group_reset_totals可以把起始值重设为0。

### Manually adjusting rate limits

对于一些需求比较复杂的程序，可以手工调整令牌桶的当前值。

#### bufferevent_decrement_read_limit
#### bufferevent_decrement_write_limit
#### bufferevent_rate_limit_group_decrement_read
#### bufferevent_rate_limit_group_decrement_write
```c
int bufferevent_decrement_read_limit(struct bufferevent *bev, ev_ssize_t decr);
int bufferevent_decrement_write_limit(struct bufferevent *bev, ev_ssize_t decr);
int bufferevent_rate_limit_group_decrement_read(
        struct bufferevent_rate_limit_group *grp, ev_ssize_t decr);
int bufferevent_rate_limit_group_decrement_write(
        struct bufferevent_rate_limit_group *grp, ev_ssize_t decr);
```

函数功能如名字所示，注意decr是有符号的，如果想增加桶容量，就传入一个负值。

### Setting the smallest share possible in a rate-limited group

通常，并不想把一个速率限制组的限制字节数平均的分配给组中的成员。比如，如果一个组有10000个活动的bufferevent，每一个tick可传输10000个字节，如果让每一个bufferevent每tick只传输1个字节那必然是低效的。

为此，每一个速率限制组都有一个"minimum share”。在上面的情况下，不是每个bufferevent每tick只能写1个字节，而是10,000/SHARE个bufferevent可以每tick写SHARE字节的数据，其余的则被禁止传输。哪些bufferevent可写是每tick随机选择的。

缺省的最小共享值的选择是出于合适的性能考虑，当前是64。也可以自己调整。

#### bufferevent_rate_limit_group_set_min_share

```c
int bufferevent_rate_limit_group_set_min_share(
        struct bufferevent_rate_limit_group *group, size_t min_share);
```

把min_share设置为0则完全禁用最小共享。Libevent的速率限制从开始就是支持最小共享的。

### Limitations of the rate-limiting implementation

对于Libevent 2.0，速率限制的实现是有一些限制的：

- 不是每个bufferevent类型都能很好的支持速率限制，有的干脆就不支持。
- bufferevent速率限制组不能嵌套，一个bufferevent同一时间只能属于一个速率限制组。
- 速率限制组的实现只统计作为TCP报文数据传输的字节数，并不包括TCP的报文头。
- 读限制的实现依赖于TCP的栈noticing，应用只能以一定的速率消费数据，当缓冲满的时候把数据发送到TCP连接的另一端。
- 一些bufferevent的实现(特别是Windows IOCP实现)可能会overcommit，大体上就是实际可分配内存不足。
- 这里的原文比较拗口，就说个大概意思，bufferevent的速率限制是通过tick来计算的，计算的时候如果tick不是个整数，比方说是N.1，那么就会按照N+1来计算。
- tick的值不能小于一毫秒，任何比一毫秒还小的分量就忽略了。所以如果tick设置了微秒字段，这个微秒会除以1000再计入。

## Bufferevents and SSL

### Setting up and using an OpenSSL-based bufferevent

#### bufferevent_openssl_filter_new
#### bufferevent_openssl_socket_new

```c
enum bufferevent_ssl_state {
        BUFFEREVENT_SSL_OPEN = 0,
        BUFFEREVENT_SSL_CONNECTING = 1,
        BUFFEREVENT_SSL_ACCEPTING = 2
};

struct bufferevent *
bufferevent_openssl_filter_new(struct event_base *base,
    struct bufferevent *underlying,
    SSL *ssl,
    enum bufferevent_ssl_state state,
    int options);

struct bufferevent *
bufferevent_openssl_socket_new(struct event_base *base,
    evutil_socket_t fd,
    SSL *ssl,
    enum bufferevent_ssl_state state,
    int options);
```

可以创建两种SSL bufferevent：

- bufferevent_openssl_filter_new创建基于过滤器的bufferevent，通过一个底层的bufferevent进行通信。
- bufferevent_openssl_socket_new创建基于socket的bufferevent，OpenSSL可以直接通过网络通信。

每一种情况，都需要提供一个SSL对象和一个表示SSL对象状态的描述。

如果SSL当前是作为一个客户端正在进行协商，那么状态就是BUFFEREVENT_SSL_CONNECTING ；

如果SSL当前是作为服务端正在进行协商，那么状态就是BUFFEREVENT_SSL_ACCEPTING ；

如果SSL握手结束，则状态是BUFFEREVENT_SSL_OPEN。

options可以是任意的常规的可选项。BEV_OPT_CLOSE_ON_FREE在openssl bufferevent关闭时，除了会关闭底层的socket或者bufferevent外，还会关闭SSL对象。

握手一完成，新的bufferevent事件回调就会被调用，这时候的event标记是BEV_EVENT_CONNECTED。

如果创建一个基于socket的bufferevent时，SSL对象已经有了一个socket，就不需要在提供socket，fd设为-1就可以了，也可以在之后使用bufferevent_setfd来设置fd。

要注意的是当一个SSL bufferevent设置了BEV_OPT_CLOSE_ON_FREE时，对SSL连接的关闭操作并不是完全的。这会造成两个问题：首先，这个连接在另一端会被视为“中断”而不是完全关闭，另一方无法知道连接是被关闭的，还是被攻击者或者第三方中断的。其次，OpenSSL将把这次会话视为”坏”的，然后将其从会话缓存中移除。这会显著的降低SSL应用的性能。

目前唯一的避免方式是手工进行延期SSL关闭。但这会违背TLS RFC，RFC要求一旦连接关闭，会话必须仍然保持在缓存中。以下是这种方式的例子。

```c
SSL *ctx = bufferevent_openssl_get_ssl(bev);

/*
 * SSL_RECEIVED_SHUTDOWN告诉SSL_shutdown就好像我们已经从对端收到了一个close notify，
 * SSL_shutdown接着会在响应中发送final close notify，
 * 对端将会收到这个close notify并发送自己的close notify。
 * 这个时候我们已经关闭了socket，而对端真正的close notify将永远不会被接收到。
 * 于是，两端都认为自己已经完成了一次完整的关闭，并且保持对话有效。
 * 当socket不处于写就绪时，这种策略就会失败，因为这会导致不完整的关闭以及在另一端丢失会话。
 * 要注意这里是指SSL连接的关闭，不是TCP连接的关闭，
 * 关闭报文仍然是通过TCP发出去的，所以如果socket不能写，也就意味着关闭报文没有发出去这端就关闭了
 */

SSL_set_shutdown(ctx, SSL_RECEIVED_SHUTDOWN);
SSL_shutdown(ctx);
bufferevent_free(bev);
```
#### bufferevent_openssl_get_ssl
```c
SSL *bufferevent_openssl_get_ssl(struct bufferevent *bev);
```

函数返回一个OpenSSL bufferevent使用的SSL对象，如果bev不是一个基于OpenSSL的bufferevent则返回NULL。

#### bufferevent_get_openssl_error

```c
unsigned long bufferevent_get_openssl_error(struct bufferevent *bev);
```

对于一个指定的bufferevent，函数返回其操作的第一个挂起的OpenSSL错误。如果没有挂起的错误，则返回0。返回的错误的格式与openssl库中ERR_get_error() 返回的一样。

#### bufferevent_ssl_renegotiate

```c
int bufferevent_ssl_renegotiate(struct bufferevent *bev);
```

调用这个函数告知SSL进行协商，以及bufferevent要调用适当的回调函数。这是个高级话题，除非你知道你在做什么，否则就避免使用它，尤其是不少SSL版本都有许多与协商有关的安全问题。

#### bufferevent_openssl_get_allow_dirty_shutdown
#### bufferevent_openssl_set_allow_dirty_shutdown

```c
int bufferevent_openssl_get_allow_dirty_shutdown(struct bufferevent *bev);
void bufferevent_openssl_set_allow_dirty_shutdown(struct bufferevent *bev,
    int allow_dirty_shutdown);
```

所有良好的SSL协议(也就是SSlv3和所有的TLS版本)都支持一个授权的关闭操作，使得参与方能够区分是一个内部的关闭，还是在底层缓冲上由于偶然或者故意而引发的中断。缺省情况下，在关闭一个连接时，正常关闭之外的任意事情都被视为错误，这里指在。如果allow_dirty_shutdown 设置为1，则连接关闭被视为BEV_EVENT_EOF。

#### 例子: A simple SSL-based echo server

```c
/* Simple echo server using OpenSSL bufferevents */
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <sys/socket.h>
#include <netinet/in.h>
#include <arpa/inet.h>

#include <openssl/ssl.h>
#include <openssl/err.h>
#include <openssl/rand.h>

#include <event.h>
#include <event2/listener.h>
#include <event2/bufferevent_ssl.h>

static void
ssl_readcb(struct bufferevent * bev, void * arg)
{
    struct evbuffer *in = bufferevent_get_input(bev);

    printf("Received %zu bytes\n", evbuffer_get_length(in));
    printf("----- data ----\n");
    printf("%.*s\n", (int)evbuffer_get_length(in), evbuffer_pullup(in, -1));

    bufferevent_write_buffer(bev, in);
}

static void
ssl_acceptcb(struct evconnlistener *serv, int sock, struct sockaddr *sa,
             int sa_len, void *arg)
{
    struct event_base *evbase;
    struct bufferevent *bev;
    SSL_CTX *server_ctx;
    SSL *client_ctx;

    server_ctx = (SSL_CTX *)arg;
    client_ctx = SSL_new(server_ctx);
    evbase = evconnlistener_get_base(serv);

    bev = bufferevent_openssl_socket_new(evbase, sock, client_ctx,
                                         BUFFEREVENT_SSL_ACCEPTING,
                                         BEV_OPT_CLOSE_ON_FREE);

    bufferevent_enable(bev, EV_READ);
    bufferevent_setcb(bev, ssl_readcb, NULL, NULL, NULL);
}

static SSL_CTX *
evssl_init(void)
{
    SSL_CTX  *server_ctx;

    /* Initialize the OpenSSL library */
    SSL_load_error_strings();
    SSL_library_init();
    /* We MUST have entropy, or else there's no point to crypto. */
    if (!RAND_poll())
        return NULL;

    server_ctx = SSL_CTX_new(SSLv23_server_method());

    if (! SSL_CTX_use_certificate_chain_file(server_ctx, "cert") ||
        ! SSL_CTX_use_PrivateKey_file(server_ctx, "pkey", SSL_FILETYPE_PEM)) {
        puts("Couldn't read 'pkey' or 'cert' file.  To generate a key\n"
           "and self-signed certificate, run:\n"
           "  openssl genrsa -out pkey 2048\n"
           "  openssl req -new -key pkey -out cert.req\n"
           "  openssl x509 -req -days 365 -in cert.req -signkey pkey -out cert");
        return NULL;
    }
    SSL_CTX_set_options(server_ctx, SSL_OP_NO_SSLv2);

    return server_ctx;
}

int
main(int argc, char **argv)
{
    SSL_CTX *ctx;
    struct evconnlistener *listener;
    struct event_base *evbase;
    struct sockaddr_in sin;

    memset(&sin, 0, sizeof(sin));
    sin.sin_family = AF_INET;
    sin.sin_port = htons(9999);
    sin.sin_addr.s_addr = htonl(0x7f000001); /* 127.0.0.1 */

    ctx = evssl_init();
    if (ctx == NULL)
        return 1;
    evbase = event_base_new();
    listener = evconnlistener_new_bind(
                         evbase, ssl_acceptcb, (void *)ctx,
                         LEV_OPT_CLOSE_ON_FREE | LEV_OPT_REUSEABLE, 1024,
                         (struct sockaddr *)&sin, sizeof(sin));

    event_base_loop(evbase, 0);

    evconnlistener_free(listener);
    SSL_CTX_free(ctx);

    return 0;
}
```

我们来看看这个程序。

首先，一些变量声明，包括一个`SSL_CTX*`变量ctx，在初始化ctx的函数中调用了对openssl库的初始化，然后是对ctx的创建以及设置，这里包括了一段使用openssl自行生成秘钥文件的说明，可以单独列出来以便查阅

##### openssl 生成秘钥文件

~~~bash
openssl genrsa -out pkey 2048
openssl req -new -key pkey -out cert.req
openssl x509 -req -days 365 -in cert.req -signkey pkey -out cert
~~~

ctx创建并初始化之后，就是创建一个event_base，然后创建并绑定一个listener，evconnlistener_new_bind函数会在后面说明，这里要知道ctx是传递给accept事件回调函数的参数，用来在回调中创建一个SSL对象。

然后是ssl_acceptcb函数，在这个函数里，创建了一个SSL对象client_ctx，然后在创建一个ssl bufferevent对象后，设置了这个bufferevent的读事件，其回调函数与普通的读回调就没有区别了。

### Some notes on threading and OpenSSL

构建支持线程机制的Libevent并不涉及到OpenSSL的锁，OpenSSL使用了大量的全局变量，需要自己处理锁来保证使用OpenSSL是线程安全的。下面是一个自己实现加锁的范例。

#### 例子: A very simple example of how to enable thread safe OpenSSL

```c
/*
 * Please refer to OpenSSL documentation to verify you are doing this correctly,
 * Libevent does not guarantee this code is the complete picture, but to be used
 * only as an example.
 */
#include <stdio.h>
#include <stdlib.h>
#include <string.h>

#include <pthread.h>
#include <openssl/ssl.h>
#include <openssl/crypto.h>

pthread_mutex_t * ssl_locks;
int ssl_num_locks;

/* Implements a thread-ID function as requied by openssl */
static unsigned long
get_thread_id_cb(void)
{
    return (unsigned long)pthread_self();
}

static void
thread_lock_cb(int mode, int which, const char * f, int l)
{
    if (which < ssl_num_locks) {
        if (mode & CRYPTO_LOCK) {
            pthread_mutex_lock(&(ssl_locks[which]));
        } else {
            pthread_mutex_unlock(&(ssl_locks[which]));
        }
    }
}

int
init_ssl_locking(void)
{
    int i;

    ssl_num_locks = CRYPTO_num_locks();
    ssl_locks = malloc(ssl_num_locks * sizeof(pthread_mutex_t));
    if (ssl_locks == NULL)
        return -1;

    for (i = 0; i < ssl_num_locks; i++) {
        pthread_mutex_init(&(ssl_locks[i]), NULL);
    }

    CRYPTO_set_id_callback(get_thread_id_cb);
    CRYPTO_set_locking_callback(thread_lock_cb);

    return 0;
}
```

