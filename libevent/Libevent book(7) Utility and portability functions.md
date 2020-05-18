# Helper functions and types for Libevent

[TOC]

` <event2/util.h>` 头文件定义了许多函数，可以有助于使用Libevent实现可移植应用。Libevent在内部使用这些类型和函数。

## Basic types

### evutil_socket_t

```c
#ifdef WIN32
#define evutil_socket_t intptr_t
#else
#define evutil_socket_t int
#endif
```

### Standard integer types

| Type        | Width | Signed | Maximum       | Minimum      |
| ----------- | ----- | ------ | ------------- | ------------ |
| ev_uint64_t | 64    | No     | EV_UINT64_MAX | 0            |
| ev_int64_t  | 64    | Yes    | EV_INT64_MAX  | EV_INT64_MIN |
| ev_uint32_t | 32    | No     | EV_UINT32_MAX | 0            |
| ev_int32_t  | 32    | Yes    | EV_INT32_MAX  | EV_INT32_MIN |
| ev_uint16_t | 16    | No     | EV_UINT16_MAX | 0            |
| ev_int16_t  | 16    | Yes    | EV_INT16_MAX  | EV_INT16_MIN |
| ev_uint8_t  | 8     | No     | EV_UINT8_MAX  | 0            |
| ev_int8_t   | 8     | Yes    | EV_INT8_MAX   | EV_INT8_MIN  |

说起来，当前C/C++标准已经定义了相应的各种长度的整型类型，上面表格定义的类型也就不需要了，直接使用标准定义的即可。

### Miscellaneous compatibility types

- ev_ssize_t在有ssize_t定义的平台上就定义为ssize_t，否则就定义为一个该平台上合理的类型。最大值为EV_SSIZE_MAX，最小值为EV_SSIZE_MIN。对应的如果平台没有SIZE_MAX的定义，则size_t的最大值为EV_SIZE_MAX。
- ev_off_t用来表示一个文件或者一块内存的偏移量。在有合理的off_t定义的平台上就定义为off_t，在Windows上则是ev_int64_t。
- ev_socklen_t在有socklen_t定义的平台上就定义为socklen_t，否则就定义为一个合适的类型。
- ev_intptr_t是一个有符号的整型，长度要足以保存一个指针的全部长度。
- ev_uintptr_t是一个无符号的整型，同样长度要足以保存一个指针的全部长度。

## Timer portability functions

不是所有的平台都提供了timeval的操作函数，所以Libevent提供了自己的。

### evutil_timeradd

```c
#define evutil_timeradd(tvp, uvp, vvp) /* ... */
```

对前两个参数相加，并将结果放到第三个参数中。

###  evutil_timersub

```c
#define evutil_timersub(tvp, uvp, vvp) /* ... */
```

第一个参数减去第二个参数，并将结果保存到第三个参数中。

### evutil_timerclear

```c
#define evutil_timerclear(tvp) /* ... */
```

清除一个timeval，将其置为0值。

### evutil_timerisset

```c
define evutil_timerisset(tvp) /* ... */
```

检查一个timeval是否设置，如果设置了返回真，也就是非0值，如果没有设置，则返回假，也就是0。

### evutil_timercmp

```c
#define evutil_timercmp(tvp, uvp, cmp)
```

比较两个timeval，如果由所提供的关系运算符cmp所代表的关系为真，则返回真。比方说`evutil_timercmp(t1, t2, <=) `表示`"是否t1 <= t2?"`。Libevent支持所有的C关系运算符，也就是<, >, ==, !=, <=, 和>=。

### evutil_gettimeofday

```c
int evutil_gettimeofday(struct timeval *tv, struct timezone *tz);
```

把tv设置为当前时间，tz未使用。注意这是一个距epoch的秒数。

### 例子

```c
struct timeval tv1, tv2, tv3;

/* Set tv1 = 5.5 seconds */
tv1.tv_sec = 5; tv1.tv_usec = 500*1000;

/* Set tv2 = now */
evutil_gettimeofday(&tv2, NULL);

/* Set tv3 = 5.5 seconds in the future 
* 这里的描述可能会有误导性，
* 一定要清楚，此时tv2是个距epoch的秒数，而不是0
*/
evutil_timeradd(&tv1, &tv2, &tv3);

/* all 3 should print true */
if (evutil_timercmp(&tv1, &tv1, ==))  /* == "If tv1 == tv1" */
   puts("5.5 sec == 5.5 sec");
if (evutil_timercmp(&tv3, &tv2, >=))  /* == "If tv3 >= tv2" */
   puts("The future is after the present.");
if (evutil_timercmp(&tv1, &tv2, <))   /* == "If tv1 < tv2" */
   puts("It is no longer the past.");
```

## Socket API compatibility

### evutil_closesocket

```c
int evutil_closesocket(evutil_socket_t s);

#define EVUTIL_CLOSESOCKET(s) evutil_closesocket(s)
```

在UNIX，这是`close()`的别名，在Windows则对应`closesocket()`。

### error相关宏

```c
#define EVUTIL_SOCKET_ERROR()
#define EVUTIL_SET_SOCKET_ERROR(errcode)
#define evutil_socket_geterror(sock)
#define evutil_socket_error_to_string(errcode)
```

- EVUTIL_SOCKET_ERROR() 返回线程中最后一个socket操作的全局错误码。
- evutil_socket_geterror() 返回一个指定socket的错误码。在类UNIX系统中这两个宏返回的都是errno。
- EVUTIL_SET_SOCKET_ERROR() 设置当前的socket错误码。类似于在Unix中设置errno。
- evutil_socket_error_to_string() 返回一个代表指定socket错误的字符串错误信息。类似于Unix中的strerror。

提供这些函数或者宏的原因是Windows对socket函数的错误不使用errno，而是使用WSAGetLastError()。

注意Windows的socket错误与errno表示的标准C错误是不同的。

### evutil_make_socket_nonblocking

```c
int evutil_make_socket_nonblocking(evutil_socket_t sock);
```

这个函数把一个socket(或者来自于`socket()`或者来自于`accept()`)设置为非堵塞的。在Unix设置O_NONBLOCK，而在Windows设置FIONBIO。可见即便是设置一个socket的非堵塞IO也是不可移植到Windows的。

### evutil_make_listen_socket_reuseable

```c
int evutil_make_listen_socket_reuseable(evutil_socket_t sock);
```

在Unix中设置SO_REUSEADDR，在Windows则什么也不做，在Windows中SO_REUSEADDR的意义与Unix不同。

### evutil_make_socket_closeonexec

```c
int evutil_make_socket_closeonexec(evutil_socket_t sock);
```

告诉操作系统在调用exec()时关闭socket，注意是exec，启动一个新进程并执行程序或者命令，而不是exit。在Unix中设置FD_CLOEXEC，在Windows什么也不做。

### evutil_socketpair

```c
int evutil_socketpair(int family, int type, int protocol,
        evutil_socket_t sv[2]);
```

这个函数在Unix中与`socketpair() `是一样的：把两个socket连接起来，通常的应用场景是接下来在父子进程中各自使用一个socket来进行通信。创建成功返回0，失败返回-1.

在Windows，只支持family AF_INET，type SOCK_STREAM，以及protocol 0。要注意在某些Windows主机，由于防火墙设置可能无法使用127.0.0.1进行自我通信，通过这两个socket通信会失败。

## Portable string manipulation functions

### evutil_strtoll

```c
ev_int64_t evutil_strtoll(const char *s, char **endptr, int base);
```

与strtol一样，但是只处理64位整型，在某些平台，只支持base10。

### evutil_*nprintf

```c
int evutil_snprintf(char *buf, size_t buflen, const char *format, ...);
int evutil_vsnprintf(char *buf, size_t buflen, const char *format, va_list ap);
```

这两个函数分别与标准对应的函数相同。与c99标准一样，返回实际写到buf的字符的长度，不考虑结尾的NULL，注意在Windows，如果实际字符长度与buf长度不匹配，会返回一个负值。

## Locale-independent string manipulation functions

有时候，当实现基于ASCII的协议时，可能只打算处理遵从ASCII字符集类型的字符串，而不考虑当前的地区编码。

### evutil_ascii_str[n]casecmp

```c
int evutil_ascii_strcasecmp(const char *str1, const char *str2);
int evutil_ascii_strncasecmp(const char *str1, const char *str2, size_t n);
```

这两个函数与`strcasecmp()/strncasecmp()`的区别只在于总是使用ASCII字符集比较，而不考虑当前地区编码。

## IPv6 helper and portability functions

### evutil_inet_ntop/evutil_inet_pton

```c
const char *evutil_inet_ntop(int af, const void *src, char *dst, size_t len);
int evutil_inet_pton(int af, const char *src, void *dst);
```

这两个函数与标准的` inet_ntop()`和`inet_pton()`是一致的，都可以用来处理IPv4和IPv6。因此：

调用`evutil_inet_ntop()`时

- 对于IPv4地址，af设置为AF_INET，src指向一个in_addr结构，dst指向一个长度为len的字符缓冲区。
- 对于IPv6地址，af设置为AF_INET6，src指向一个in6_addr结构。

调用`evutil_inet_pton()`时，af分别设置为AF_INET或者AF_INET6，src是要解析的地址字符串，dst则适当的指向一个in_addr或者in_addr6结构。

`evutil_inet_ntop()`失败返回NULL，成功则返回指向dst的指针。

`evutil_inet_pton()`成功返回0，失败返回-1。

### evutil_parse_sockaddr_port

```c
int evutil_parse_sockaddr_port(const char *str, struct sockaddr *out,
    int *outlen);
```

这个函数解析str表示的地址，然后将结果写入out，outlen作为传入参数时是out的大小，作为传出参数时是作为结果的out的大小。

成功返回0，失败返回-1。

支持以下地址格式：

- `[ipv6]:port`，比如`"[ffff::]:80"`

- ipv6 ，比如 `"ffff::"`

- `[ipv6]` ，比如` "[ffff::]"`

- ipv4:port 比如`"1.2.3.4:80"`

- ipv4比如`"1.2.3.4"`

如果没有给定port，则sockaddr中的port就设置为0。

### evutil_sockaddr_cmp

```c
int evutil_sockaddr_cmp(const struct sockaddr *sa1,
    const struct sockaddr *sa2, int include_port);
```

如果sa1地址在sa2前面，则返回负数，如果相等则返回0，如果sa2在前面则返回正数。只对AF_INET 和 AF_INET6地址有效，其它地址返回未定义。比较是全序的，但是具体顺序可能依Libevent版本而变。

如果include_port参数为假，也就是为0，则比较不包括端口。

## Structure macro portability functions

### evutil_offsetof

```c
#define evutil_offsetof(type, field) /* ... */
```

与标准的偏移宏一样，这个宏得到从type的起始位到域field的字节数，也就是filed在type中的偏移量。

## Secure random number generator

### evutil_secure_rng_get_bytes

```c
void evutil_secure_rng_get_bytes(void *buf, size_t n);
```

这个函数用n个字节的随机数据填充一个n字节的缓冲区。

如果平台提供`arc4random()`函数，则Libevent使用这函数。否则，Libevent使用自己的`arc4random()`实现，随机数种子来自于操作系统的熵池(CryptGenRandom on Windows, /dev/urandom everywhere else)。

### evutil_secure_rng_init

```c
int evutil_secure_rng_init(void);
```

通常并不需要手动初始化安全随机数生成器，但是如果想要确保其确实初始化了，可以调用`evutil_secure_rng_init()`，如果没有设定随机数种子，则使用RNG作为随机数种子。成功返回0，如果返回-1，表明Libevent不能在操作系统中找到一个合适的熵源，也就不能安全的使用RNG，除非自己进行初始化。

如果程序需要改变权限，比方说使用chroot()，那么就要在此之前调用`evutil_secure_rng_init()`。

### evutil_secure_rng_add_bytes
```c
void evutil_secure_rng_add_bytes(const char *dat, size_t datlen);
```
这个函数可以增加熵池生成的数据的长度，通常并不需要这么做。