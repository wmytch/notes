# Setting up the Libevent library

[TOC]

libevent有一些全局设置，会影响整个库。对这些设置的任何改变必须在调用libevent库的其它部分前进行，不能在使用过程中进行更改，否则会引发不一致问题。明确的说，就是在这些设置必须使用libevent前进行设置。

## Log messages in Libevent

Libevent可以记录内部错误和警告，也可以记录debug信息，这些消息缺省的输出到stderr，也可以使用自定义的日志函数来记录。

### 接口

~~~c
#define EVENT_LOG_DEBUG 0
#define EVENT_LOG_MSG   1
#define EVENT_LOG_WARN  2
#define EVENT_LOG_ERR   3

/* Deprecated; see note at the end of this section */
#define _EVENT_LOG_DEBUG EVENT_LOG_DEBUG
#define _EVENT_LOG_MSG   EVENT_LOG_MSG
#define _EVENT_LOG_WARN  EVENT_LOG_WARN
#define _EVENT_LOG_ERR   EVENT_LOG_ERR

typedef void (*event_log_cb)(int severity, const char *msg);

void event_set_log_callback(event_log_cb cb);
~~~

这里定义了log的级别，以及一个日志记录函数的签名，给event_set_log_callback传递一个符合这个签名的函数的指针就可以覆盖libevent缺省的日志操作，如果给event_set_log_callback传递一个NULL，则可以恢复libevent的缺省日志操作，也就是前面说过的输出都stderr。

### 例子

```c
#include <event2/event.h>
#include <stdio.h>

static void discard_cb(int severity, const char *msg)
{
    /* This callback does nothing. */
}

static FILE *logfile = NULL;
static void write_to_file_cb(int severity, const char *msg)
{
    const char *s;
    if (!logfile)
        return;
    switch (severity) {
        case _EVENT_LOG_DEBUG: s = "debug"; break;
        case _EVENT_LOG_MSG:   s = "msg";   break;
        case _EVENT_LOG_WARN:  s = "warn";  break;
        case _EVENT_LOG_ERR:   s = "error"; break;
        default:               s = "?";     break; /* never reached */
    }
    fprintf(logfile, "[%s] %s\n", s, msg);
}

/* Turn off all logging from Libevent. */
void suppress_logging(void)
{
    event_set_log_callback(discard_cb);
}

/* Redirect all Libevent log messages to the C stdio file 'f'. */
void set_logfile(FILE *f)
{
    logfile = f;
    event_set_log_callback(write_to_file_cb);
}
```

程序很直白，就不多说了。只是要注意，在用户自定义的event_log_cb回调函数中调用libevent的函数是不安全的，因为这里记录的是libevent本身的日志信息，道理想想就能明白，也就不多说了

### 调试记录

通常调试记录是被禁用的，那么也可以手工启用，如果构建libevent时选择了支持这个操作。

~~~c
#define EVENT_DBG_NONE 0
#define EVENT_DBG_ALL 0xffffffffu

void event_enable_debug_logging(ev_uint32_t which);
~~~

调试记录是很详细或者说冗长的，并且大多数情况下是没有必要的。调用`event_enable_debug_logging() `时，以EVENT_DBG_NONE为参数调用可以得到缺省行为，以EVENT_DBG_ALL为参数则打开所有调试记录支持。目前就这两个参数，将来可能会有更为精细的开关。

这些函数在`<event2/event.h>`中声明。

## Handling fatal errors

当libevent检测到一个不可恢复的内部错误时，比如数据结构崩溃，缺省行为是调用`exit()`或者`abort()`离开当前进程。这些错误基本上总是意味着bug，不是在你自己的代码中，就是在libevent本身。

可以自定义一个函数来覆盖libevent的缺省行为，这样在libevent调用退出时可以处理一些事情。要注意的是，退出肯定是会退出的，只是在退出前可以做些事情。

### 接口

```c
typedef void (*event_fatal_cb)(int err);
void event_set_fatal_callback(event_fatal_cb cb);
```

首先，定义一个符合event_fatal_cb签名的函数，然后把这个函数作为参数调用event_set_fatal_callback，这样在发生致命错误时，libevent就会回调这个函数。

注意，这个函数不能把控制权返回libevent，也就是说，在这个函数里不能再调用任何libevent的函数。否则行为是未定义的，也就是可能没问题，如果有问题，程序就直接退出了。

这些函数在`<event2/event.h>`中声明

## Memory management

通常，libevent使用C的内存管理函数来管理内存，当然也可以自己提供，如果需要的话。

### 接口

```c
void event_set_mem_functions(void *(*malloc_fn)(size_t sz),
                             void *(*realloc_fn)(void *ptr, size_t sz),
                             void (*free_fn)(void *ptr));
```

### 例子

```c
#include <event2/event.h>
#include <sys/types.h>
#include <stdlib.h>

/* This union's purpose is to be as big as the largest of all the
 * types it contains. */
union alignment {
    size_t sz;
    void *ptr;
    double dbl;
};
/* We need to make sure that everything we return is on the right
   alignment to hold anything, including a double.
   实际上，这里自定义的分配函数会把所分配的空间的大小保存在分配空间的前部，
   所以，需要这个宏以及前面的union。
   这个操作只是为了示例说明自定义内存管理函数能做什么，而不是必须要这么做，
   毕竟，如果不做点什么其它的，自定义内存管理函数的意义在哪里呢？
   话又说回来，这里的例子我觉得使用类似c++的placement new的操作可能更好
   */
#define ALIGNMENT sizeof(union alignment)

/* We need to do this cast-to-char* trick on our pointers to adjust
   them; doing arithmetic on a void* is not standard. */
#define OUTPTR(ptr) (((char*)ptr)+ALIGNMENT)
#define INPTR(ptr) (((char*)ptr)-ALIGNMENT)
/*注意一下，这里的分配函数实际分配的空间比直接调用系统函数所分配的空间大ALIGNMENT，
并且会将所分配的大小的值放在这多出来的空间里，所返回的指针则是经过上面的宏调整后的位置。
实际上，这也是系统函数的操作，所以分配和释放函数是必须配套使用的。*/
static size_t total_allocated = 0;
static void *replacement_malloc(size_t sz)
{
    void *chunk = malloc(sz + ALIGNMENT);
    if (!chunk) return chunk;
    total_allocated += sz;
    *(size_t*)chunk = sz;
    return OUTPTR(chunk);
}
/*注意，对于sz=0的情况，
这里并没有如系统realloc那样完全将空间返回系统，而是保留了一块ALIGNMENT的空间，
只有在调用自定义的free函数之后才会完全释放，注意不能调用系统的free函数*/
static void *replacement_realloc(void *ptr, size_t sz)
{
    size_t old_size = 0;
    if (ptr) {
        ptr = INPTR(ptr);
        old_size = *(size_t*)ptr;
    }
    ptr = realloc(ptr, sz + ALIGNMENT);
    if (!ptr)
        return NULL;
    *(size_t*)ptr = sz;
    total_allocated = total_allocated - old_size + sz;
    return OUTPTR(ptr);
}
static void replacement_free(void *ptr)
{
    ptr = INPTR(ptr);
    total_allocated -= *(size_t*)ptr;
    free(ptr);
}
void start_counting_bytes(void)
{
    event_set_mem_functions(replacement_malloc,
                            replacement_realloc,
                            replacement_free);
}
```

### 注意

- 替换内存管理函数影响libevent中所有的内存分配、重设大小和释放操作。所以，必须确保在调用其它任意的libevent函数前安装这几个函数。
- 你的malloc和realloc函数返回的内存块的对齐必须与C函数返回的一致。
- 你的realloc函数必须要能正确地处理 realloc(NULL, sz)，也就是说，等同于malloc(sz)。
- 你的realloc函数必须要能正确地处理realloc(ptr, 0)，也就是说，等同于free(ptr)。
- 你的free函数不需要处理 free(NULL)。
- 你的malloc函数不需要处理malloc(0)。
- 自定义的内存管理函数必须是线程安全的。
- Libevent会使用这些替换函数来分配内存，因此，如果你替换了内存分配函数，那么你也必须使用自定义的free函数来释放内存。

Libevent在构建时可以禁用`event_set_mem_functions()`，这样使用event_set_mem_functions 的程序就不会被编译和链接。 在Libevent 2.0.2-alpha和之后, 可以通过检查 EVENT_SET_MEM_FUNCTIONS_IMPLEMENTED 是否有定义，来检测`event_set_mem_functions()`是否存在。

这些函数在`<event2/event.h>`中声明

## Locks and threading

对于多线程情况，Libevent的结构总的来说有三种工作方式。

- 一些结构是有生具来的单线程，在多线程下总是不安全的。
- 一些结构加锁是可选的，也就是说你可以自己决定是否需要及如何在多线程下使用这些对象。
- 一些结构总是加锁的，如果Libevent运行时支持锁定，那么这些结构在多线程时总是安全的。

要在Libevent中使用锁，必须告知Libevent使用哪个锁函数，如果需要在多线程下共享一些数据结构，这项工作必须在调用分配这些结构的任一Libevent函数前进行。

### Ptheads和Windows接口

Libevent预定义了一些函数来设定使用正确的pthreads或者Windows的线程库。

```c
#ifdef WIN32
int evthread_use_windows_threads(void);
#define EVTHREAD_USE_WINDOWS_THREADS_IMPLEMENTED
#endif
#ifdef _EVENT_HAVE_PTHREADS
int evthread_use_pthreads(void);
#define EVTHREAD_USE_PTHREADS_IMPLEMENTED
#endif
```

所有函数成功返回0，失败返回-1。

### 其它接口

如果需要使用其他的线程库，就需要自定义使用这些库的函数来实现下面这些机制：

- Locks
- locking
- unlocking
- lock allocation
- lock destruction
- Conditions
- condition variable creation
- condition variable destruction
- waiting on a condition variable
- signaling/broadcasting to a condition variable
- Threads
- thread ID detection

然后使用evthread_set_lock_callbacks和evthread_set_id_callback接口告知Libevent这些函数。

```c
#define EVTHREAD_WRITE  0x04
#define EVTHREAD_READ   0x08
#define EVTHREAD_TRY    0x10

#define EVTHREAD_LOCKTYPE_RECURSIVE 1
#define EVTHREAD_LOCKTYPE_READWRITE 2

#define EVTHREAD_LOCK_API_VERSION 1

struct evthread_lock_callbacks {
       int lock_api_version;
       unsigned supported_locktypes;
       void *(*alloc)(unsigned locktype);
       void (*free)(void *lock, unsigned locktype);
       int (*lock)(unsigned mode, void *lock);
       int (*unlock)(unsigned mode, void *lock);
};

int evthread_set_lock_callbacks(const struct evthread_lock_callbacks *);

void evthread_set_id_callback(unsigned long (*id_fn)(void));

struct evthread_condition_callbacks {
        int condition_api_version;
        void *(*alloc_condition)(unsigned condtype);
        void (*free_condition)(void *cond);
        int (*signal_condition)(void *cond, int broadcast);
        int (*wait_condition)(void *cond, void *lock,
            const struct timeval *timeout);
};

int evthread_set_condition_callbacks(
        const struct evthread_condition_callbacks *);
```

evthread_lock_callbacks结构描述了锁回调函数和它们的作用。对于上面描述的版本：

- lock_api_version域必须设定为 EVTHREAD_LOCK_API_VERSION
- supported_locktypes域必须设定为由`EVTHREAD_LOCKTYPE_*`常量组合成的一个bitmask，这些常量描述了可以支持的锁(对于 2.0.4-alpha, EVTHREAD_LOCK_RECURSIVE是强制的，而 EVTHREAD_LOCK_READWRITE是未使用的)
- alloc*函数必须返回一个指定类型的新的锁
-  *free* 函数必须释放一个指定的锁所拥有的全部资源
-  *lock* 函数必须尝试获取指定类型的锁，如果成功返回0，失败返回非0值
- *unlock* 函数必须尝试解锁，成功返回0，失败返回非0值。 

已知的所类型有：

- 0

    ​        一个常规的，非必须递归的锁。

- EVTHREAD_LOCKTYPE_RECURSIVE

    ​        一个线程已经拥有这个锁时，如果再次请求这个锁，该线程不会堵塞。拥有这个锁的线程释放这个锁后，其它线程的申请不受限制，就像一个新的锁一样。 

- EVTHREAD_LOCKTYPE_READWRITE

    ​       多个读线程可以共享这个锁，但是只能被一个写线程独占。写线程排除其它所有的读线程。 

已知的锁模式:

- EVTHREAD_READ

    ​        只用于READWRITE锁：在读的时候获取和释放。

- EVTHREAD_WRITE

    ​        只用于READWRITE锁：在写的时候获取和释放。

- EVTHREAD_TRY

    ​        只用于锁定：只有当锁可以马上获取的时候获取锁。

id_fn是一个函数指针，必须指向一个返回值为unsigned long的函数，这个返回值标识调用该函数的线程。对同一个线程，这个返回值必须一致，不同的线程，这个值必须不同，即使这些线程同时调用了同一个函数。

evthread_condition_callbacks结构描述了与条件变量相关的回调函数。 同样，对于上面描述的版本：

- lock_api_version域必须设定为EVTHREAD_CONDITION_API_VERSION.  
- alloc_condition函数必须返回一个指向新的条件变量的指针，接受0作为参数。
- free_condition函数必须释放一个条件变量拥有的存储空间和资源。
- wait_condition函数必须使用三个参数，一个由alloc_condition分配的条件变量，一个由evthread_lock_callbacks.alloc函数分配的锁，以及一个可选的timeout。函数一被调用就会获取这个锁，获取成功后就会马上释放这个锁然后等待，直到条件信号触发或者超时(可选)。这个函数在发生错误时返回-1，条件信号触发时返回0，超时时返回1。这个函数返回前必须确保再次获得锁，这也意味这这个函数的调用者需要手动释放这个锁。
- signal_condition函数应当可以唤醒等待这个条件的一个线程(如果其broadcast参数为false，在C语言情况下为0)，或者所有等待这个条件的线程(如果其broadcast参数为true，同样，在C语言情况下为非0)。当然，只有在获取与条件关联的锁后，才能够调用或者说触发这个函数。

### 例子

参见evthread_pthread.c和evthread_win32.c。

这些函数在`<event2/thread.h>`中声明。

## Debugging lock usage

To help debug lock usage, Libevent has an optional "lock debugging" feature that wraps its locking calls in order to catch typical lock errors, including为了便于调试锁的使用，Libevent有一个可选的"lock debugging”特性，封装了其与锁相关的调用，可以捕获典型锁相关错误：

- 解锁一个没有所有权的锁
- 对一个非递归锁再次加锁

如果发生了这样的错误，Libevent就会以一个assertion失败退出。

### 接口

```c
void evthread_enable_lock_debugging(void);
/*注意下面宏的拼写，此外没别的意义了*/
#define evthread_enable_lock_debuging() evthread_enable_lock_debugging() 
```

### 注意

这个函数必须在任意的锁被创建或者使用前调用。为了安全，在设置了线程函数之后马上调用这个函数。

## Debugging event usage

在使用事件时有一些常见错误是Libevent可以检测并且可以报告的。包括：

- 把一个未初始化的结构事件当成一个已经初始化的结构事件来处理，这里的结构事件大体上可以理解为bufferevent。
- 试图重新初始化一个挂起的结构事件。

要追踪哪一个事件是初始化了的需要使用额外的内存和CPU时间，因此应该只在确实需要调试程序的时候再启用这项调试模式。

### 接口

~~~c
void event_enable_debug_mode(void);
void event_debug_unassign(struct event *ev);
~~~

### 注意

event_enable_debug_mode必须在任意的event_base创建之前调用，注意是创建event_base之前，不是调用其它任意Libevent函数之前。

在调试模式启用时，如果程序使用`event_assign()`而不是`event_new()`创建了大量的事件，那么就有可能耗尽内存。这是因为Libevent没有办法告知一个由`event_assign()`创建的事件什么时候失效，而使用`event_free()`来释放一个`event_new()`事件时是可以知道这个事件是否有效的。所以如果要在调试时避免内存耗尽，可以显式的调用event_debug_unassign告诉Libevent这样的事件不再被视为是 assigned的。当然，在调试模式未启用的时候调用event_debug_unassign是无效的。

特别要说明的是，使用`event_assign()`会耗费大量内存只是在debug模式下，正常使用并不必然会如此。

### 例子

```c
#include <event2/event.h>
#include <event2/event_struct.h>

#include <stdlib.h>

void cb(evutil_socket_t fd, short what, void *ptr)
{
    /* We pass 'NULL' as the callback pointer for the heap allocated
     * event, and we pass the event itself as the callback pointer
     * for the stack-allocated event. 
     * 所谓heap allocated event，指的是用event_new()创建的event。
     * 而stack-allocated event，指的是一个局部变量
     */
    
    struct event *ev = ptr;

    if (ev)
        event_debug_unassign(ev);
}

/* Here's a simple mainloop that waits until fd1 and fd2 are both
 * ready to read. */
void mainloop(evutil_socket_t fd1, evutil_socket_t fd2, int debug_mode)
{
    struct event_base *base;
    struct event event_on_stack, *event_on_heap;

    if (debug_mode)
       event_enable_debug_mode();

    base = event_base_new();

    event_on_heap = event_new(base, fd1, EV_READ, cb, NULL);
    event_assign(&event_on_stack, base, fd2, EV_READ, cb, &event_on_stack);

    event_add(event_on_heap, NULL);
    event_add(&event_on_stack, NULL);

    event_base_dispatch(base);

    event_free(event_on_heap);
    event_base_free(base);
}
```

这个程序只是说明了event_debug_unassign的使用方式，此外没有别的特别意义。

详细的事件调试作为一个特性，只能在编译时使用参数"-DUSE_DEBUG”来启用，不能在调用API是启用或者禁用，这里指的的是编译程序时在Makefile中设置一个编译参数，具体怎么做，可以给CFLAGS变量赋个值，也可以直接加到编译命令中去。如果设置了这个标记，那么使用Libevent的程序都会在后端输出一个冗长并且接近底层的详细记录，输出到哪里需要结合上面关于日志那部分来看。这些信息包括但不限于：

- 添加事件
- 删除事件
- 平台特定的事件通知信息

## Detecting the version of Libevent

至于为什么要检测Libevent的版本，就不废话了。

### 接口

```c
#define LIBEVENT_VERSION_NUMBER 0x02000300
#define LIBEVENT_VERSION "2.0.3-alpha"
const char *event_get_version(void);
ev_uint32_t event_get_version_number(void);
```

Libevent的版本信息可以通过宏在编译器获取，也可以通过函数在运行时获取。需要注意的是如果使用动态链接库，这些版本信息可能会不一致。

这些宏和函数在` <event2/event.h>`中定义.  

Libevent版本信息有两种形式：字符串适用于显示给用户，4字节的整型数适用于数值比较。整型数格式使用高位字节表示主版本号，第二个字节表示小版本号，第三字节表示补丁版本，最低字节表示发布状态(0表示发布版本，非0表示某个发布版本之后的开发版本)。

比如，Libevent 2.0.1-alpha的版本号是[02 00 01 00]，或者0x20000100，一个介于2.0.1-alpha和2.0.2-alpha之间的开发版本的版本号可能会是[02 00 01 08]，或者0x20000108。

### 例子：编译时检查

```c
#include <event2/event.h>

#if !defined(LIBEVENT_VERSION_NUMBER) || LIBEVENT_VERSION_NUMBER < 0x02000100
#error "This version of Libevent is not supported; Get 2.0.1-alpha or later."
#endif

int
make_sandwich(void)
{
        /* Let's suppose that Libevent 6.0.5 introduces a make-me-a
           sandwich function. */
#if LIBEVENT_VERSION_NUMBER >= 0x06000500
        evutil_make_me_a_sandwich();
        return 0;
#else
        return -1;
#endif
}
```

### 例子：运行时检查

```c
#include <event2/event.h>
#include <string.h>

int
check_for_old_version(void)
{
    const char *v = event_get_version();
    /* This is a dumb way to do it, but it is the only thing that works
       before Libevent 2.0. */
    if (!strncmp(v, "0.", 2) ||
        !strncmp(v, "1.1", 3) ||
        !strncmp(v, "1.2", 3) ||
        !strncmp(v, "1.3", 3)) {

        printf("Your version of Libevent is very old.  If you run into bugs,"
               " consider upgrading.\n");
        return -1;
    } else {
        printf("Running with Libevent version %s\n", v);
        return 0;
    }
}

int
check_version_match(void)
{
    ev_uint32_t v_compile, v_run;
    v_compile = LIBEVENT_VERSION_NUMBER;
    v_run = event_get_version_number();
    if ((v_compile & 0xffff0000) != (v_run & 0xffff0000)) {
        printf("Running with a Libevent version (%s) very different from the "
               "one we were built with (%s).\n", event_get_version(),
               LIBEVENT_VERSION);
        return -1;
    }
    return 0;
}
```

## Freeing global Libevent structures

即使已经释放了所有通过Libevent分配的对象，也总还是会遗留一些全局对象，这通常不是一个问题，因为程序退出时就会清理掉了，但是这些对象可能会使得一些调试工具认为Libevent发生了资源泄露，如果需要确认Libevent释放了所有库内部使用的全局数据结构，可以调用

### 接口

```
void libevent_global_shutdown(void);
```

这个函数不会释放Libevent函数返回给用户的任何数据结构。如果需要在退出前释放所有的东西，那么就需要手动释放所有的event、event_base、bufferevent，等等。

调用` libevent_global_shutdown()`可能会使得其它Libevent函数的行为不可预测，因此把它作为程序中最后一个调用的Libevent函数。不过这个函数是幂等的，也就是说在调用过之后还可以继续调用，不会有任何影响。

这个函数在` <event2/event.h>`中声明。