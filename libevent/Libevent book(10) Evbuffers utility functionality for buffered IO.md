# Evbuffers: utility functionality for buffered IO

[TOC]

Libevent 的evbuffer实现了一个字节队列，并对其在尾端添加数据和在前端移除数据的操作做了优化。Evbuffer专注于缓冲网络IO的缓冲部分的通用性，并不提供对IO的调度和IO的触发的操作，这些是bufferevent的工作。

## Creating or freeing an evbuffer

### evbuffer_new
### evbuffer_free
```c
struct evbuffer *evbuffer_new(void);
void evbuffer_free(struct evbuffer *buf);
```

evbuffer_new分配并返回一个新的空evbuffer，evbuffer_free删除一个evbuffer及其全部内容。

## Evbuffers and Thread-safety

### evbuffer_enable_locking

### evbuffer_lock

### evbuffer_unlock

```c
int evbuffer_enable_locking(struct evbuffer *buf, void *lock);
void evbuffer_lock(struct evbuffer *buf);
void evbuffer_unlock(struct evbuffer *buf);
```

缺省情况下，同时从多个线程访问一个evbuffer是不安全的。如果确实需要这么做，可以对evbuffer调用evbuffer_enable_locking。如果lock参数为NULL，Libevent会通过之前设置的evthread_set_lock_creation_callback函数来分配一个新的锁。否则，就是用所提供的锁。

evbuffer_lock和evbuffer_unlock分别获取和释放锁。如果evbuffer没有启用锁，则这些函数不做任何事情。

注意对于单独的操作并不需要调用锁函数，如果evbuffer的锁启用，那么单独的操作就已经是原子的了。只有在多个操作一起并且不希望其它线程插入的时候才加锁。

## Inspecting an evbuffer

### evbuffer_get_length

```c
size_t evbuffer_get_length(const struct evbuffer *buf);
```

返回一个evbuffer中存储的字节数。

### evbuffer_get_contiguous_space

```c
size_t evbuffer_get_contiguous_space(const struct evbuffer *buf);
```

返回一个evbuffer头部连续存储的字节数。evbuffer中的数据可能会存储在多个内存块当中，这个函数返回的是第一个块中当前存储的字节数。

## Adding data to an evbuffer: basics

### evbuffer_add

```c
int evbuffer_add(struct evbuffer *buf, const void *data, size_t datlen);
```

在buf的尾部添加长度为datlen的数据，这些数据由data指向。成功返回0，失败返回-1。

### evbuffer_add_printf

### evbuffer_add_vprintf 

```c
int evbuffer_add_printf(struct evbuffer *buf, const char *fmt, ...)
int evbuffer_add_vprintf(struct evbuffer *buf, const char *fmt, va_list ap);
```

这两个函数向buf的尾部添加格式化的数据，其参数分别与C库函数printf和vprintf类似处理。函数都返回添加的字节数。

### evbuffer_expand

```c
int evbuffer_expand(struct evbuffer *buf, size_t datlen);
```

改变buf中最后一个内存块的大小，或者添加一个新的内存块，使得buf有足够的空间来存放datlen这么多的数据，而不需要再进一步分配内存。

### 例子

```c
/* Here are two ways to add "Hello world 2.0.1" to a buffer. */
/* Directly: */
evbuffer_add(buf, "Hello world 2.0.1", 17);

/* Via printf: */
evbuffer_add_printf(buf, "Hello %s %d.%d.%d", "world", 2, 0, 1);
```

## Moving data from one evbuffer to another

出于效率，Libevent优化了在evbuffer之间移动数据的操作。

### evbuffer_add_buffer

### evbuffer_remove_buffer

```c
int evbuffer_add_buffer(struct evbuffer *dst, struct evbuffer *src);
int evbuffer_remove_buffer(struct evbuffer *src, struct evbuffer *dst,
    size_t datlen);
```

evbuffer_add_buffer把src的所有数据移动到dst的尾端。成功返回0，失败返回-1。

evbuffer_remove_buffer从src向dst尾端移动datlen那么多字节的数据，如果可移动数据的长度不足datlen，就全部移过去，但是任何时候都不会超过datlen个字节。成功则返回移动的数据长度。

## Adding data to thInterface

### evbuffer_prepend

### evbuffer_prepend_buffer 

```c
int evbuffer_prepend(struct evbuffer *buf, const void *data, size_t size);
int evbuffer_prepend_buffer(struct evbuffer *dst, struct evbuffer* src);
```

这些函数把数据移动到dst的头部，此外分别与evbuffer_add和evbuffer_add_buffer类似。

## Rearranging the internal layout of an evbuffer

有时候可能会需要检查evbuffer的头部的N个字节的数据是不是存放在一个连续的数组中，为此，首先要确保缓冲头部确实是连续的。

### evbuffer_pullup

```c
unsigned char *evbuffer_pullup(struct evbuffer *buf, ev_ssize_t size);
```

evbuffer_pullup对buf的最初size个字节进行线性化，需要时通过复制或者移动来确保这些数据是连续的，并且占据同一个内存块。如果size是负数，函数线性化整个缓冲，如果size大于缓冲中已有的字节数，函数返回NULL，否则，返回一个指向buf中第一个字节的指针。

调用这个函数在size很大时会变得很慢，因为其有潜在的可能去复制整个缓冲的内容。

### 例子

```c
#include <event2/buffer.h>
#include <event2/util.h>

#include <string.h>
/* 这个例子实际上提供了一种获取报文头信息的方式*/
int parse_socks4(struct evbuffer *buf, ev_uint16_t *port, ev_uint32_t *addr)
{
    /* Let's parse the start of a SOCKS4 request!  The format is easy:
     * 1 byte of version, 1 byte of command, 2 bytes destport, 4 bytes of
     * destip. */
    unsigned char *mem;

    mem = evbuffer_pullup(buf, 8);

    if (mem == NULL) {
        /* Not enough data in the buffer */
        return 0;
    } else if (mem[0] != 4 || mem[1] != 1) {
        /* Unrecognized protocol or command */
        return -1;
    } else {
        memcpy(port, mem+2, 2);
        memcpy(addr, mem+4, 4);
        *port = ntohs(*port);
        *addr = ntohl(*addr);
        /* Actually remove the data from the buffer now that we know we
           like it. */
        evbuffer_drain(buf, 8);
        return 1;
    }
}
```

### 注意

调用evbuffer_pullup时，如果size取evbuffer_get_contiguous_space返回的值将不会导致任何数据被复制或者移动。

## Removing data from an evbuffer

### evbuffer_drain

### evbuffer_remove

```c
int evbuffer_drain(struct evbuffer *buf, size_t len);
int evbuffer_remove(struct evbuffer *buf, void *data, size_t datlen);
```

evbuffer_remove从buf头部复制然后移除最前面的datlen那么多字节的数据，然后把这些数据放到data所指的内存中。如果可用数据少于*datlen*，则复制所有的数据，失败返回-1，否则返回所复制的字节数。

evbuffer_drain与evbuffer_remove不同的是并不复制数据，而只是移除缓冲头部指定长度的数据。成功返回0，失败返回-1。

## Copying data out from an evbuffer

有时候可能会需要从缓冲的头部复制但又不需要移除其中的数据。比如，可能只是想看看是否一个某类型的结构完整的到达了，这时候并不打算移除(比如evbuffer_remove)，也不打算重新布局缓冲(比如evbuffer_pullup）

### evbuffer_copyout

### evbuffer_copyout_from

```c
ev_ssize_t evbuffer_copyout(struct evbuffer *buf, void *data, size_t datlen);
ev_ssize_t evbuffer_copyout_from(struct evbuffer *buf,
     const struct evbuffer_ptr *pos,
     void *data_out, size_t datlen);
```

evbuffer_copyout与evbuffer_remove不同只是在于前者不从缓冲中移除任何数据。也就是说，只是把buf头部最前面的datlen长度的字节复制到data中去。如果buf中可用数据少于datlen字节，则复制全部数据。失败返回-1，否则返回复制的字节数。

evbuffer_copyout_from从buf中*pos*所指的位置复制数据，这点是与evbuffer_copyout的主要区别。关于evbuffer_ptr结构参见下文"Searching within an evbuffer”。

如果从缓冲复制数据太慢，可以使用evbuffer_peek。

### 例子

```c
#include <event2/buffer.h>
#include <event2/util.h>
#include <stdlib.h>
#include <stdlib.h>

int get_record(struct evbuffer *buf, size_t *size_out, char **record_out)
{
    /* Let's assume that we're speaking some protocol where records
       contain a 4-byte size field in network order, followed by that
       number of bytes.  We will return 1 and set the 'out' fields if we
       have a whole record, return 0 if the record isn't here yet, and
       -1 on error.  */
    size_t buffer_len = evbuffer_get_length(buf);
    ev_uint32_t record_len;
    char *record;

    if (buffer_len < 4)
       return 0; /* The size field hasn't arrived. */

   /* We use evbuffer_copyout here so that the size field will stay on
       the buffer for now. */
    evbuffer_copyout(buf, &record_len, 4);
    /* Convert len_buf into host order. */
    record_len = ntohl(record_len);
    if (buffer_len < record_len + 4)
        return 0; /* The record hasn't arrived */

    /* Okay, _now_ we can remove the record. */
    record = malloc(record_len);
    if (record == NULL)
        return -1;

    evbuffer_drain(buf, 4);
    evbuffer_remove(buf, record, record_len);

    *record_out = record;
    *size_out = record_len;
    return 1;
}
```

## Line-oriented input

### evbuffer_readln

```c
enum evbuffer_eol_style {
        EVBUFFER_EOL_ANY,
        EVBUFFER_EOL_CRLF,
        EVBUFFER_EOL_CRLF_STRICT,
        EVBUFFER_EOL_LF,
        EVBUFFER_EOL_NUL
};
char *evbuffer_readln(struct evbuffer *buffer, size_t *n_read_out,
    enum evbuffer_eol_style eol_style);
```

许多Internet协议都是基于行的格式。evbuffer_readln从evbuffer的头部获取一行的数据，为其分配一个NULL结尾的字符串，然后返回这个字符串。如果*n_read_out*不为NULL，则设置其为返回字符串的长度。如果未能读取一整行，函数返回NULL。行结束符不复制到返回的字符串中，因为返回的字符串已经带了一个了。

evbuffer_readln可以识别4种行结束格式：

- EVBUFFER_EOL_LF

    行以一个单独的换行符结束，也就是`\n`，ASCII值为0x0A。

- EVBUFFER_EOL_CRLF_STRICT

    行以一个回车加一个换行结束，也就是`\r\n`，ASCII值为0x0D 0x0A。

- EVBUFFER_EOL_CRLF

    行结束以一个可选的回车加一个换行符结束，也就是`\r\n`或者`\n`。这个格式在解析基于文本的Internet协议时很有用，因为尽管标准通常要求是`\r\n`，但一些客户端可能只用`\n`。

- EVBUFFER_EOL_ANY

    行结束可能是任意数量的回车和任意数量的换行。这个格式不是很有用，主要只是出于后向兼容性的考虑。

- EVBUFFER_EOL_NUL

    行结束于一个单独字节的0，也就是ASCII NUL。

注意如果用event_set_mem_functions覆盖了缺省的malloc，evbuffer_readln返回的字符串会由所指定的替换分配函数分配。

### 例子

```c
char *request_line;
size_t len;

request_line = evbuffer_readln(buf, &len, EVBUFFER_EOL_CRLF);
if (!request_line) {
    /* The first line has not arrived yet. */
} else {
    if (!strncmp(request_line, "HTTP/1.0 ", 9)) {
        /* HTTP 1.0 detected ... */
    }
    free(request_line);
}
```

## Searching within an evbuffer

evbuffer_ptr结构指向evbuffer中的一个位置，可以从那里开始迭代evbuffer中的数据。

### struct evbuffer_ptr

```c
struct evbuffer_ptr {
        ev_ssize_t pos;
        struct {
                /* internal fields */
        } _internal;
};
```

pos域是唯一的公开域，其它部分不应该被用户代码使用。pos表示距evbuffer起始的一个偏移量。

###  evbuffer_search
```c
struct evbuffer_ptr evbuffer_search(struct evbuffer *buffer,
    const char *what, size_t len, const struct evbuffer_ptr *start);
```
函数扫描缓冲，寻找长度为*len*的字符串*what，*如果存在，则返回包含这个字符串位置的evbuffer_ptr ，否则返回-1。

如果提供了start参数，则从start开始查找，否则，从头开始查找，注意，指的是buffer的开头，不是what字符串的开头，原文这里用string是有些误导的。

###  evbuffer_search_range
```c
struct evbuffer_ptr evbuffer_search_range(struct evbuffer *buffer,
    const char *what, size_t len, const struct evbuffer_ptr *start,
    const struct evbuffer_ptr *end);
```
实际上evbuffer_search是直接调用了evbuffer_search_range，只是把end置为NULL。所以两个函数的区别只是在于evbuffer_search_range可以指定一个结束位置，而evbuffer_search要查找到buffer最后。

### evbuffer_search_eol
```c
struct evbuffer_ptr evbuffer_search_eol(struct evbuffer *buffer,
    struct evbuffer_ptr *start, size_t *eol_len_out,
    enum evbuffer_eol_style eol_style);
```
evbuffer_search_eol检测行结束，但不会复制行，只是返回行结束符的起始位置。如果eol_len_out不为NULL，则设置为EOL字符串的长度，指的是组成行结束符的字符串的长度，不是整个字符串的长度。

### evbuffer_ptr_set

```c
enum evbuffer_ptr_how {
        EVBUFFER_PTR_SET,
        EVBUFFER_PTR_ADD
};
int evbuffer_ptr_set(struct evbuffer *buffer, struct evbuffer_ptr *pos,
    size_t position, enum evbuffer_ptr_how how);
```

evbuffer_ptr_set 调整buffer中pos的位置。如果how是EVBUFFER_PTR_SET，则指针在buffer中移动到一个绝对的位置，如果how是EVBUFFER_PTR_ADD，则指针向前移动。成功返回0，失败返回-1。

如下面例子所示，这个函数也可以用来把一个buffer和一个pos关联起来，或者说给一个pos赋个初值。

### 例子

```c
#include <event2/buffer.h>
#include <string.h>

/* Count the total occurrences of 'str' in 'buf'. */
int count_instances(struct evbuffer *buf, const char *str)
{
    size_t len = strlen(str);
    int total = 0;
    struct evbuffer_ptr p;

    if (!len)
        /* Don't try to count the occurrences of a 0-length string. */
        return -1;

    evbuffer_ptr_set(buf, &p, 0, EVBUFFER_PTR_SET);

    while (1) {
         p = evbuffer_search(buf, str, len, &p);
         if (p.pos < 0)
             break;
         total++;
         evbuffer_ptr_set(buf, &p, 1, EVBUFFER_PTR_ADD);
    }

    return total;
}
```

### 警告

任意改变evbuffer的内容或者布局的函数调用都会使得evbuffer_ptr 失效，从而对其的使用将变得不安全。

## Inspecting data without copying it

有时候，可能需要从evbuffer中读数据而不复制出来(比如evbuffer_copyout会复制数据)，或者重新安排evbuffer的内部存储布局(比如evbuffer_pullup)。有时候也可能想看看evbuffer中间部分的数据。这时候就可以使用：

### evbuffer_peek

```c
struct evbuffer_iovec {
        void *iov_base;
        size_t iov_len;
};

int evbuffer_peek(struct evbuffer *buffer, ev_ssize_t len,
    struct evbuffer_ptr *start_at,
    struct evbuffer_iovec *vec_out, int n_vec);
```

调用evbuffer_peek的时候，要提供一个长度为*n_vec*的evbuffer_iovec结构数组*vec_out*。结构中的*iov_base*域指向evbuffer内部的存储块，iov_len则是对应内存块的长度。

如果len小于0，evbuffer_peek会尽可能地填写所提供的所有evbuffer_iovec结构，如果evbuffer内部内存块的数量大于n_vec就填n_vec个，否则有多少就填多少，返回的是实际上填写的evbuffer_iovec的数量。

如果len大于0，则所填写的evbuffer_iovec结构的个数取决于这些evbuffer_iovec对应的内存块的数据总量，实际上获取到的全部内存块数据的总量可能会大于len。如果函数能够获取所要求的所有数据，那么就会返回实际上使用的evbuffer_iovec结构的数量。如果函数不能够获取所要求的的数据量，则返回所需要提供的evbuffer_iovec结构的数量，也就是说，如果len长度那么多的数据不足以通过n_vec个evbuffer_iovec结构来获取，那么就返回为了满足这个len长度要求而提供的evbuffer_iovec数量，换句话说就是，这个时候在evbuffer内部有这么多个内存块，它们的数据总量满足len，但是n_vec小于这个数字。

如果start_at为空，则evbuffer_peek从缓冲的起始开始，否则，就从start_at所指的位置开始，不过这时候所返回的第一个evbuffer_iovec的iov_base指向的不一定是对应内存块的起始位置，可能会有一个偏移，iov_len也不是完整的内存块的长度，而是从iov_base到末尾的长度。注意一下，原文这里是ptr而不是start_at，从上下文看，应该是start_at。不过这时候

### 例子

```c
{
    /* Let's look at the first two chunks of buf, and write them to stderr. */
    int n, i;
    struct evbuffer_iovec v[2];
    n = evbuffer_peek(buf, -1, NULL, v, 2);
    for (i=0; i<n; ++i) { /* There might be less than two chunks available. */
        fwrite(v[i].iov_base, 1, v[i].iov_len, stderr);
    }
}

{
    /* Let's send the first 4906 bytes to stdout via write. */
    int n, i, r;
    struct evbuffer_iovec *v;
    size_t written = 0;

    /* determine how many chunks we need. */
    n = evbuffer_peek(buf, 4096, NULL, NULL, 0);
    /* Allocate space for the chunks.  This would be a good time to use
       alloca() if you have it. */
    v = malloc(sizeof(struct evbuffer_iovec)*n);
    /* Actually fill up v. */
    n = evbuffer_peek(buf, 4096, NULL, v, n);
    for (i=0; i<n; ++i) {
        size_t len = v[i].iov_len;
        if (written + len > 4096)
            len = 4096 - written;
        r = write(1 /* stdout */, v[i].iov_base, len);
        if (r<=0)
            break;
        /* We keep track of the bytes written separately; if we don't,
           we may write more than 4096 bytes if the last chunk puts
           us over the limit. */
        written += len;
    }
    free(v);
}

{
    /* Let's get the first 16K of data after the first occurrence of the
       string "start\n", and pass it to a consume() function. */
    struct evbuffer_ptr ptr;
    struct evbuffer_iovec v[1];
    const char s[] = "start\n";
    int n_written;

    ptr = evbuffer_search(buf, s, strlen(s), NULL);
    if (ptr.pos == -1)
        return; /* no start string found. */

    /* Advance the pointer past the start string. */
    if (evbuffer_ptr_set(buf, &ptr, strlen(s), EVBUFFER_PTR_ADD) < 0)
        return; /* off the end of the string. */

    while (n_written < 16*1024) {
        /* Peek at a single chunk. */
        if (evbuffer_peek(buf, -1, &ptr, v, 1) < 1)
            break;
        /* Pass the data to some user-defined consume function */
        consume(v[0].iov_base, v[0].iov_len);
        n_written += v[0].iov_len;

        /* Advance the pointer so we see the next chunk next time. */
        if (evbuffer_ptr_set(buf, &ptr, v[0].iov_len, EVBUFFER_PTR_ADD)<0)
            break;
    }
}
```

### 注意

- 改变evbuffer_iovec所指向的数据可能会导致未定义行为。
- 如果有别的函数改变了evbuffer，那么之前evbuffer_peek得到的结果可能就会变得无效。
- 如果evbuffer 要在多线程下使用，确保在调用evbuffer_peek前将其用evbuffer_lock锁住，然后在使用完evbuffer_peek返回的结果之后解锁。话说虽然evbuffer_peek内部也会锁定一次evbuffer，但是显然两次锁定的粒度是不同的。

## Adding data to an evbuffer directly

有时候可能会需要把数据直接插入到evbuffer中，而不是把数据先写入一个字符数组再用evbuffer_add把数组复制过去。evbuffer_reserve_space和evbuffer_commit_space借助evbuffer_peek可以通过evbuffer_iovec实现这个目的。

### evbuffer_reserve_space

### evbuffer_commit_space

```c
int evbuffer_reserve_space(struct evbuffer *buf, ev_ssize_t size,
    struct evbuffer_iovec *vec, int n_vecs);
int evbuffer_commit_space(struct evbuffer *buf,
    struct evbuffer_iovec *vec, int n_vecs);
```

evbuffer_reserve_space给出evbuffer内部的一片空间的指针，需要的话会增加内存块的数量适应size的要求。这些指针以及各自长度会存放在evbuffer_iovec结构数组vec中，n_vecs则是这个数组的大小。

n_vecs必须至少为1，如果n_vecs真的为1，则Libevent会确保提供在一个单独的内存块内的满足要求的连续空间，但这可能会造成缓冲的重新布局或者内存浪费。为了更好的性能，至少提供两个evbuffer_iovec。函数会返回为了满足空间要求所需要的evbuffer_iovec的数量。

向这些内存块写入的数据只有在evbuffer_commit_space调用之后才真正加入到evbuffer。如果提交的数据少于之前请求的空间，可以减少evbuffer_iovec中iov_len的值，当然所提交的evbuffer_iovec的数量也可以少于之前给出的。evbuffer_commit_space函数成功返回0，失败返回-1。

### 注意

- 对evbuffer重新布局或者添加数据都会使evbuffer_reserve_space得到的指针失效。
- 原文说不论n_vecs是多少，当时版本只使用两个evbuffer_iovec，不过目前版本的源码并没有看到这个限制。
- 不论调用evbuffer_reserve_space多少次都是安全的。
- 如果要在多线程中使用evbuffer，那么在调用evbuffer_reserve_space前加锁，在evbuffer_commit_space之后解锁。

### 例子

```c
/* Suppose we want to fill a buffer with 2048 bytes of output from a
   generate_data() function, without copying. */
struct evbuffer_iovec v[2];
int n, i;
size_t n_to_add = 2048;

/* Reserve 2048 bytes.*/
n = evbuffer_reserve_space(buf, n_to_add, v, 2);
if (n<=0)
   return; /* Unable to reserve the space for some reason. */
/*
* 注意一下，这里的循环没有修改n_to_add，
* 因为要写多少数据已经在返回的evbuffer_iovec的iov_len域设置好了
* 至于调整len，只是为了in case，或者只是举个例子
*/
for (i=0; i<n && n_to_add > 0; ++i) {
   size_t len = v[i].iov_len;
   if (len > n_to_add) /* Don't write more than n_to_add bytes. */
      len = n_to_add;
   if (generate_data(v[i].iov_base, len) < 0) {
      /* If there was a problem during data generation, we can just stop
         here; no data will be committed to the buffer. */
      return;
   }
   /* Set iov_len to the number of bytes we actually wrote, so we
      don't commit too much. */
   v[i].iov_len = len;
}

/* We commit the space here.  Note that we give it 'i' (the number of
   vectors we actually used) rather than 'n' (the number of vectors we
   had available. */
if (evbuffer_commit_space(buf, v, i) < 0)
   return; /* Error committing */
```

### 不好的例子

```c
/* Here are some mistakes you can make with evbuffer_reserve().
   DO NOT IMITATE THIS CODE. */
struct evbuffer_iovec v[2];

{
  /* Do not use the pointers from evbuffer_reserve_space() after
     calling any functions that modify the buffer. */
  evbuffer_reserve_space(buf, 1024, v, 2);
  evbuffer_add(buf, "X", 1);
  /* WRONG: This next line won't work if evbuffer_add needed to rearrange
     the buffer's contents.  It might even crash your program. Instead,
     you add the data before calling evbuffer_reserve_space. */
  memset(v[0].iov_base, 'Y', v[0].iov_len-1);
  evbuffer_commit_space(buf, v, 1);
}

{
  /* Do not modify the iov_base pointers. */
  const char *data = "Here is some data";
  evbuffer_reserve_space(buf, strlen(data), v, 1);
  /* WRONG: The next line will not do what you want.  Instead, you
     should _copy_ the contents of data into v[0].iov_base. */
  v[0].iov_base = (char*) data;
  v[0].iov_len = strlen(data);
  /* In this case, evbuffer_commit_space might give an error if you're
     lucky */
  evbuffer_commit_space(buf, v, 1);
}
```

## Network IO with evbuffers

### evbuffer_read

```c
int evbuffer_read(struct evbuffer *buffer, evutil_socket_t fd, int howmuch);
```

函数从socket fd中读取howmuch字节的数据到buffer的尾端。返回成功读取的数据量，EOF返回0，错误返回-1。注意错误可能表示一个非堵塞的操作不成功，这时候需要检查错误码是不是EAGAIN，或者在Windows下的WSAEWOULDBLOCK 。如果howmuch为负数，evbuffer_read会猜测一个需要读取的数字，当然不是瞎猜，而是调用一个函数看看fd有多少就绪的数据。

### evbuffer_write_atmost

```c
int evbuffer_write_atmost(struct evbuffer *buffer, evutil_socket_t fd,
        ev_ssize_t howmuch);
```

将buffer头部最多howmuch字节的数据写入socket fd。返回写成果的数据量，失败返回-1。与evbuffer_read一样，需要检查错误是不是由于非堵塞引起的。如果howmuch为负值，则尝试把buffer的全部数据写入fd。

### evbuffer_write

```c
int evbuffer_write(struct evbuffer *buffer, evutil_socket_t fd);
```

与调用evbuffer_write_atmost时howmuch为负值是一样的。

在Unix，这些函数支持所有可以读写的文件描述符，在Windows，只支持socket。

注意当使用bufferevent时不需要直接调用这些函数，bufferevent本身就会调用这些函数。

## Evbuffers and callbacks

为了知道数据什么时候添加到evbuffer或者从evbuffer中移除，Libevent为evbuffer提供了一套通用的回调机制

### evbuffer_cb_func

```c
struct evbuffer_cb_info {
        size_t orig_size;
        size_t n_added;
        size_t n_deleted;
};

typedef void (*evbuffer_cb_func)(struct evbuffer *buffer,
    const struct evbuffer_cb_info *info, void *arg);
```

evbuffer回调在数据添加到evbuffer或者从evbuffer中移除时调用。evbuffer_cb_info 结构的orit_size域记录buffer原始的字节数，n_added记录向buffer添加的字节数，n_deleted 记录移除的字节数。

### evbuffer_add_cb

```c
struct evbuffer_cb_entry;
struct evbuffer_cb_entry *evbuffer_add_cb(struct evbuffer *buffer,
    evbuffer_cb_func cb, void *cbarg);
```

函数用来给evbuffer添加一个回调，并返回一个指向这个回调函数实例的指针以备后用。cb和cbarg的用途跟以前的类似就不说明了。

可以为单独一个evbuffer添加多个回调，添加新的回调不会移除旧的回调。

### 例子

```c
#include <event2/buffer.h>
#include <stdio.h>
#include <stdlib.h>

/* Here's a callback that remembers how many bytes we have drained in
   total from the buffer, and prints a dot every time we hit a
   megabyte. */
struct total_processed {
    size_t n;
};
void count_megabytes_cb(struct evbuffer *buffer,
    const struct evbuffer_cb_info *info, void *arg)
{
    struct total_processed *tp = arg;
    size_t old_n = tp->n;
    int megabytes, i;
    tp->n += info->n_deleted;
    megabytes = ((tp->n) >> 20) - (old_n >> 20);
    for (i=0; i<megabytes; ++i)
        putc('.', stdout);
}

void operation_with_counted_bytes(void)
{
    struct total_processed *tp = malloc(sizeof(*tp));
    struct evbuffer *buf = evbuffer_new();
    tp->n = 0;
    evbuffer_add_cb(buf, count_megabytes_cb, tp);

    /* Use the evbuffer for a while.  When we're done: */
    evbuffer_free(buf);
    free(tp);
}
```

注意释放一个非空的evbuffer时不会视为从中排空数据，因此释放时不会调用回调，也不会释放传入回调的用户提供的数据指针，比方说上面的tp。

### evbuffer_remove_cb_entry

```c
int evbuffer_remove_cb_entry(struct evbuffer *buffer,
    struct evbuffer_cb_entry *ent);
```
通过添加回调时返回的回调项指针来移除一个回调。成功返回0，失败返回-1。

### evbuffer_remove_cb

```c
int evbuffer_remove_cb(struct evbuffer *buffer, evbuffer_cb_func cb,
    void *cbarg);
```
通过一个回调函数及传入的参数来移除一个回调。成功返回0，失败返回-1。

### evbuffer_cb_set_flags

### evbuffer_cb_clear_flags
```c
#define EVBUFFER_CB_ENABLED 1
int evbuffer_cb_set_flags(struct evbuffer *buffer,
                          struct evbuffer_cb_entry *cb,
                          ev_uint32_t flags);
int evbuffer_cb_clear_flags(struct evbuffer *buffer,
                          struct evbuffer_cb_entry *cb,
                          ev_uint32_t flags);
```

设置或者清除一个回调的标记。当前只支持一个用户可见的标记EVBUFFER_CB_ENABLED。这个标记是缺省设置的，如果清除，则表示该回调不被调用，或者时候暂停使用直到下次设置。

### evbuffer_defer_callbacks

```c
int evbuffer_defer_callbacks(struct evbuffer *buffer, struct event_base *base);
```

也可以延迟调用evbuffer的回调，也就是说可以和bufferevent的回调一样，不在事件触发时马上调用，而由event base的事件循环来调用。理由也是一样的，避免多个evbuffer相互之间多次交换数据造成的栈压平。

当evbuffer的回调延迟调用时，最终会累记多次操作的结果。

与bufferevent一样，evbuffer内部也是引用计数的，所以即使一个延迟回调还没有执行释放一个evbuffer也是安全的。

## Avoiding data copies with evbuffer-based IO

为了尽可能的减少数据复制，Libevent提供了一些机制。

### evbuffer_add_reference

```c
typedef void (*evbuffer_ref_cleanup_cb)(const void *data,
    size_t datalen, void *extra);

int evbuffer_add_reference(struct evbuffer *outbuf,
    const void *data, size_t datlen,
    evbuffer_ref_cleanup_cb cleanupfn, void *extra);
```

函数把一块数据的引用添加到evbuffer的尾端，没有复制，evbuffer只是存储了一个指针data，指向长度为datlen的一片数据区。因而，这个指针在被evbuffer使用期间必须保持有效。当evbuffer不需要数据时，会调用所提供的cleanupfn函数来清理数据，其参数为data指针，datlen值及extra指针。成功返回0，失败返回-1。

### 例子

```c
#include <event2/buffer.h>
#include <stdlib.h>
#include <string.h>

/* In this example, we have a bunch of evbuffers that we want to use to
   spool a one-megabyte resource out to the network.  We do this
   without keeping any more copies of the resource in memory than
   necessary. */

#define HUGE_RESOURCE_SIZE (1024*1024)
struct huge_resource {
    /* We keep a count of the references that exist to this structure,
       so that we know when we can free it. */
    int reference_count;
    char data[HUGE_RESOURCE_SIZE];
};

struct huge_resource *new_resource(void) {
    struct huge_resource *hr = malloc(sizeof(struct huge_resource));
    hr->reference_count = 1;
    /* Here we should fill hr->data with something.  In real life,
       we'd probably load something or do a complex calculation.
       Here, we'll just fill it with EEs. */
    memset(hr->data, 0xEE, sizeof(hr->data));
    return hr;
}

void free_resource(struct huge_resource *hr) {
    --hr->reference_count;
    if (hr->reference_count == 0)
        free(hr);
}

static void cleanup(const void *data, size_t len, void *arg) {
    free_resource(arg);
}

/* This is the function that actually adds the resource to the
   buffer. */
void spool_resource_to_evbuffer(struct evbuffer *buf,
    struct huge_resource *hr)
{
    ++hr->reference_count;
    evbuffer_add_reference(buf, hr->data, HUGE_RESOURCE_SIZE,
        cleanup, hr);
}
```

## Adding a file to an evbuffer

一些操作系统可以把文件直接写入到网络上，而不需要从用户空间复制数据。

### evbuffer_add_file

```c
int evbuffer_add_file(struct evbuffer *output, int fd, ev_off_t offset,
    size_t length);
```

函数假定fd是一个已经打开的文件的描述符，不能是socket，fd要可读。从文件的offset开始，读取length长度的内容，添加到output的尾端。成功返回0，失败返回-1。

### 警告

如果操作系统支持`splice()`或者`sendfile()`，Libevent就会在调用evbuffer_write时直接从fd向网络发送数据，而不需要把数据复制到用户RAM中。如果不存在这样的函数，但是有`mmap()`，Libevent就会mmap文件，然后操作系统可能就会认为不需要把数据复制到用户空间。除此之外，Libevent就只能把数据从磁盘复制到RAM。

当数据被从evbuffer中刷新后或者evbuffer被释放后，就会关闭文件。如果这不是所需要的，或者需要更精细的控制，就往下看。

## Fine-grained control with file segments

evbuffer_add_file在对同一个文件多次操作时效率不高，因为其需要拿走文件的所有权。当然，这里的意思是独占文件或者说需要打开文件的全部内容，文件的属性显然不能随意变更的。

### evbuffer_file_segment_new

```c
struct evbuffer_file_segment;
```
```c
struct evbuffer_file_segment *evbuffer_file_segment_new(
        int fd, ev_off_t offset, ev_off_t length, unsigned flags);
```
函数创建并返回一个新的evbuffer_file_segment 对象，该对象代表一个底层文件的片段，fd是这个底层文件的文件描述符，这个片段从文件的offset开始，长度为length字节。如果出错，函数返回NULL。

文件片段是由 sendfile, splice, mmap, CreateFileMapping, 或者malloc-and-read实现的，具体哪一个要具体而定，选择的依据是从最轻量的支持机制到最重量级的支持机制。比方说，如果操作系统支持sendfile和mmap，那么文件片段就会只通过sendfile来实现，直到你打算检查文件的内容，这时候就会由mmap来实现。

可以通过以下标记来精细控制文件片段的行为：

- EVBUF_FS_CLOSE_ON_FREE

    调用evbuffer_file_segment_free释放文件片段时关闭底层文件。

- EVBUF_FS_DISABLE_MMAP

    文件片段将不会使用内存映射类型的后端来实现，如CreateFileMapping和mmap，即使它们是合适的。

- EVBUF_FS_DISABLE_SENDFILE

    文件片段将不会使用sendfile类型的后端来实现，比附sendfile、splice，即使它们是合适的。

- EVBUF_FS_DISABLE_LOCKING

    将不会为文件片段分配锁，也就意味着在多线程下使用文件片段是不安全的。


### evbuffer_file_segment_free
```c
void evbuffer_file_segment_free(struct evbuffer_file_segment *seg);
```
当不再需要使用一个文件片段时，可以调用evbuffer_file_segment_free来释放。不过只有到了没有别的evbuffer引用该文件片段时，才会真正释放空间。

### evbuffer_add_file_segment

```c
int evbuffer_add_file_segment(struct evbuffer *buf,
    struct evbuffer_file_segment *seg, ev_off_t offset, ev_off_t length);
```

使用evbuffer_add_file_segment来添加一个evbuffer_file_segment，可以是全部内容也可以是部分内容，offset指的是文件片段的偏移，而不是文件本身的偏移。

### evbuffer_file_segment_add_cleanup_cb

```c
typedef void (*evbuffer_file_segment_cleanup_cb)(
    struct evbuffer_file_segment const *seg, int flags, void *arg);

void evbuffer_file_segment_add_cleanup_cb(struct evbuffer_file_segment *seg,
        evbuffer_file_segment_cleanup_cb cb, void *arg);
```

evbuffer_file_segment_add_cleanup_cb用来设置一个回调函数，以供当对文件片段的最后一个引用被释放，随即要释放文件片段时调用。这个回调函数**不能**试图再激活该文件片段，不能将其添加到任意的缓冲上，等等。

## Adding an evbuffer to another by reference

可以把一个evbuffer的引用添加到另一个evbuffer上，而不是将一个evbuffer的内容添加到另一个evbuffer，并将原evbuffer的内容抹掉，这时候在另一个evbuffer上使用这个引用，就好像把原evbuffer的数据全部复制过来一样。

### evbuffer_add_buffer_reference

```c
int evbuffer_add_buffer_reference(struct evbuffer *outbuf,
    struct evbuffer *inbuf);
```

函数把outbuf的引用添加到inbuf上，不会进行任何不必要的复制。成功返回0，失败返回-1。

注意之后改变inbuf的内容的不会反射回outbuf，函数只是添加了对evbuffer的内容的引用，而不是evbuffer本身。

也要注意不能嵌套引用缓冲，一个缓存已经作为evbuffer_add_buffer_reference的outbuf后，就不能再成为其他的inbuf。

实际上，看源码似乎文档这里把方向搞反了。在源码里边是把inbuf的引用添加到outbuf的附加链上，并且被设置为不可更改。所以，实际上这个函数并没有人真正使用过？比较前面的evbuffer_add_buffer和evbuffer_remove_buffer也可以发现，源和目标参数在evbuffer各个函数里出现的顺序或者说位置是蛮混乱的。

## Making an evbuffer add- or remove-only

### evbuffer_freeze

### evbuffer_unfreeze 

```c
int evbuffer_freeze(struct evbuffer *buf, int at_front);
int evbuffer_unfreeze(struct evbuffer *buf, int at_front);
```

用来临时禁用对evbuffer头部或者尾部的变更。bufferevent内部使用来避免对输出缓冲的头部或者输入缓冲的尾部偶然的变更。

 