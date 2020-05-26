# Working with events

[TOC]

Libevent的基本操作单位是事件，每一个事件代表一个条件或者说情况的集合，包括

- 一个已经可以用来读或写的文件描述符
- 一个即将变成可以读或写的文件描述符(只对边缘触发的IO）
- 超时
- 信号
- 用户触发的的事件

事件也有所谓的生命周期。调用Libevent函数设置一个事件并将其与一个event base关联起来，这个事件就是**初始状态**，然后可以向base添加事件，此时事件就处于**挂起**状态，**所谓挂起，就是等待事情发生的意思，并且是与某个base关联的**，当事件处于**挂起**状态时，如果出现了会触发事件的情况或者说条件，比方说文件描述符改变状态或者超时了，则事件变为**活动**状态，并且运行其(用户定义的)回调函数。如果事件被配置为**持久**的，此时会继续保持挂起，否则，当运行回调函数时该事件就会停止挂起，或者说，退出与base的关联。可以删除一个事件使其变成非挂起的，也可以添加一个非挂起的事件使其再次挂起。

## Constructing event objects

### `event_new()/event_free()`

```c
#define EV_TIMEOUT      0x01
#define EV_READ         0x02
#define EV_WRITE        0x04
#define EV_SIGNAL       0x08
#define EV_PERSIST      0x10
#define EV_ET           0x20

typedef void (*event_callback_fn)(evutil_socket_t, short, void *);

struct event *event_new(struct event_base *base, evutil_socket_t fd,
    short what, event_callback_fn cb,
    void *arg);

void event_free(struct event *event);
```

event_new函数分配并构造一个由base使用的新的event。what参数是上面列出的一系列标记。如果fd非负，就代表要观察读或写事件的文件。当事件变成活动时，Libevent就会调用所提供的cb函数，并将文件描述符fd、所有触发事件组成的位域，也就是what的一个子集、以及arg作为参数传入。

发生内部错误或者参数无效时，event_new返回NULL。

所有新的event都是初始化并且非挂起的。要使得一个event变成挂起，调动event_add。

要销毁一个event，调用event_free。一个事件处于挂起或者活动状态时调用event_free是安全的，在销毁之前会使其变成非挂起和非活动状态的。

以上函数在`<event2/event.h>`中定义。

### 例子

```c
#include <event2/event.h>

void cb_func(evutil_socket_t fd, short what, void *arg)
{
        const char *data = arg;
        printf("Got an event on socket %d:%s%s%s%s [%s]",
            (int) fd,
            (what&EV_TIMEOUT) ? " timeout" : "",
            (what&EV_READ)    ? " read" : "",
            (what&EV_WRITE)   ? " write" : "",
            (what&EV_SIGNAL)  ? " signal" : "",
            data);
}

void main_loop(evutil_socket_t fd1, evutil_socket_t fd2)
{
        struct event *ev1, *ev2;
        struct timeval five_seconds = {5,0};
        struct event_base *base = event_base_new();

        /* The caller has already set up fd1, fd2 somehow, and make them
           nonblocking. */

        ev1 = event_new(base, fd1, EV_TIMEOUT|EV_READ|EV_PERSIST, cb_func,
           (char*)"Reading event");
        ev2 = event_new(base, fd2, EV_WRITE|EV_PERSIST, cb_func,
           (char*)"Writing event");

        event_add(ev1, &five_seconds);
        event_add(ev2, NULL);
        event_base_dispatch(base);
}
```

### The event flags

- EV_TIMEOUT

    这个标记指明一个事件在超时后变成活动的。

    这个标记在构造一个event时是被忽略的：在添加事件时可以设置超时，也可以不设置。当超时发生时，会设置到回调函数的what参数中。

    这里的意思是，Libevent在构造event时，不会去查看这个标记，因此，在添加事件时，可以带超时参数，也可以不带，并不受这个参数影响。对于回调函数，如果是发生了超时事件，则会把这个标记作为what参数传入。

- EV_READ

    这个标记指明一个事件当fd可读时变成活动的。

- EV_WRITE

    这个标记指明一个事件当fd可写时变成活动的。

- EV_SIGNAL

    用来实现信号检测。参见下面的 "Constructing signal events”。

- EV_PERSIST

    指明一个事件是持久的。参见下面的"About Event Persistence”。

- EV_ET

    指明事件应该是边缘触发的，如果底层的event_base后端支持边缘触发事件。这个标记影响EV_READ和EV_WRITE的语义。

从Libevent 2.0.1-alpha开始，对同一个情形或者说条件，可能同时会有任意数量的事件在等待。比如，如果一个fd可读，可能有两个事件会变成活动的。其回调函数的调用顺序是未定义的。

这些标记在<event2/event.h>中定义。

### About Event Persistence

缺省情况下，当一个挂起事件变成活动的时候，在其回调函数执行前就会变成非挂起的，因此如果想要这个事件再次挂起，就需要在回调函数中对这个事件再次调用event_add。

如果设置了EV_PERSIST标记，则事件就是持久的。也就是说，即使其回调激活，事件仍然保持挂起。这时如果想在回调中使其变成非挂起的，则可以对事件调用`event_del()`。

对一个持久的事件，当事件的回调运行时，就会重置其超时。因此，如果一个事件带有EV_READ|EV_PERSIST标记，以及一个5秒超时，事件在下面情况下会变成活动的：

- socket变成可读。
- 距事件上一次活动状态过去了5秒。

## Creating an event as its own callback argument

### event_self_cbarg

```c
void *event_self_cbarg();
```

有时候可能需要创建一个可以把自身作为回调函数的参数传入的event，这里指的是把event的指针作为arg传入event_new，但是这并不能实现，因为这时候event还不存在。为解决这个问题，可以调用`event_self_cbarg()`。

### 例子

```c
#include <event2/event.h>

static int n_calls = 0;

void cb_func(evutil_socket_t fd, short what, void *arg)
{
    struct event *me = arg;

    printf("cb_func called %d times so far.\n", ++n_calls);

    if (n_calls > 100)
       event_del(me);
}

void run(struct event_base *base)
{
    struct timeval one_sec = { 1, 0 };
    struct event *ev;
    /* We're going to set up a repeating timer to get called called 100
       times. */
    ev = event_new(base, -1, EV_PERSIST, cb_func, event_self_cbarg());
    event_add(ev, &one_sec);//可以看到，超时参数的设置并不受event_new是否有EV_TIMEOUT标记的影响
    event_base_dispatch(base);
}
```

这个函数可以用于 `event_new()`, `evtimer_new()`, `evsignal_new()`, `event_assign()`, `evtimer_assign()`,  `evsignal_assign()`。但是对于非event回调函数，不能将其作为一个回调函数的参数使用。

## Timeout-only events

```c
#define evtimer_new(base, callback, arg) \
    event_new((base), -1, 0, (callback), (arg))
#define evtimer_add(ev, tv) \
    event_add((ev),(tv))
#define evtimer_del(ev) \
    event_del(ev)
#define evtimer_pending(ev, tv_out) \
    event_pending((ev), EV_TIMEOUT, (tv_out))
```

这些宏只是提供了一些写代码上的便利，此外就没有了。

## Constructing signal events

```c
#define evsignal_new(base, signum, cb, arg) \
    event_new(base, signum, EV_SIGNAL|EV_PERSIST, cb, arg)
#define evsignal_add(ev, tv) \
    event_add((ev),(tv))
#define evsignal_del(ev) \
    event_del(ev)
#define evsignal_pending(ev, what, tv_out) \
    event_pending((ev), (what), (tv_out))
```

这里是针对的是posix风格的信号。

### 例子

```c
struct event *hup_event;
struct event_base *base = event_base_new();

/* call sighup_function on a HUP signal */
hup_event = evsignal_new(base, SIGHUP, sighup_function, NULL);
```

要注意信号回调函数是在信号发生后才在事件循环中调度运行的，所以，只有确实无法使用常规的POSIX信号处理时才使用这种方式。另外，不要在信号事件中设置超时，可能会不支持。

### Caveats when working with signals

在当前版本的Libevent，大多数后端中，每个进程同时只能有一个event_base监听信号。如果一次给两个event_base添加了信号事件，即使两个信号是不同的，也只有一个event_base可以接收到信号。

kqueue没有这个限制。

## Setting up events without heap-allocation

### event_assign

```c
int event_assign(struct event *event, struct event_base *base,
    evutil_socket_t fd, short what,
    void (*callback)(evutil_socket_t, short, void *), void *arg);
```

出于性能或者其他一些原因，有时可能会把event作为一个更大的数据结构的一部分来进行分配，这样就可以节省：

- 内存分配器在堆上分配小对象的开销
- 对event结构指针进行解引用的时间开销
- 由于事件不在缓存中造成的缓存未命中而产生的时间开销

这个方法对不同版本Libevent会有二进制不兼容的风险，因为不同版本的event结构可能大小会不一样，换句话来说，如果将Libevent版本升级了，那么已有的程序就需要重新编译，并且需要保持版本的一致性。

实际上对大多数程序而言，前面所说的代价是很小并且没有什么关系的。所以，尽可能的使用`event_new() `，除非确信在堆上分配event时会有显著的性能惩罚。另外使用`event_assign()`可能会照成难以诊断的错误，因为谁也说不准将来Libevent的event结构会变成啥样。

`event_assign()`的参数除了event外，与`event_new()`是一致的，event参数必须指向一个未初始化的event结构，这里所谓的未初始化，指的是还没有与一个event_base关联，调用event_assign本身就是在做这个关联的工作。

也可以用event_assign来初始化在栈上或者静态分配的event。

要注意的是，绝对不要对一个已经在某个event_base上挂起的事件调用event_assign。如果一个event已经初始化并且已经挂起，那么就必须在调用event_assign之前调用event_del将其从原event_base上移除。

成功返回0，失败返回-1。

### 例子

```c
#include <event2/event.h>
/* Watch out!  Including event_struct.h means that your code will not
 * be binary-compatible with future versions of Libevent. */
#include <event2/event_struct.h>
#include <stdlib.h>
/*一个较大的结构*/
struct event_pair {
         evutil_socket_t fd;
         struct event read_event;
         struct event write_event;
};
void readcb(evutil_socket_t, short, void *);
void writecb(evutil_socket_t, short, void *);
struct event_pair *event_pair_new(struct event_base *base, evutil_socket_t fd)
{
        struct event_pair *p = malloc(sizeof(struct event_pair));
        if (!p) return NULL;
        p->fd = fd;
        event_assign(&p->read_event, base, fd, EV_READ|EV_PERSIST, readcb, p);
        event_assign(&p->write_event, base, fd, EV_WRITE|EV_PERSIST, writecb, p);
        return p;
}
```

### 超时和信号宏

```c
#define evtimer_assign(event, base, callback, arg) \
    event_assign(event, base, -1, 0, callback, arg)
#define evsignal_assign(event, base, signum, callback, arg) \
    event_assign(event, base, signum, EV_SIGNAL|EV_PERSIST, callback, arg)
```

与前面的是类似的。

## event_get_struct_event_size

```c
size_t event_get_struct_event_size(void);
```

如果一定想要使用event_assign并且保持二进制兼容性，可以使用这个函数来获取当前版本的event结构的大小。

要注意的是，这个函数的大小可能会小于sizeof返回的大小，如果这样的话，说明该版本的Libevent对event结构有一些填充字节，以保留使用。

### 例子

```c
#include <event2/event.h>
#include <stdlib.h>

/* When we allocate an event_pair in memory, we'll actually allocate
 * more space at the end of the structure.  We define some macros
 * to make accessing those events less error-prone. */
struct event_pair {
         evutil_socket_t fd;
};
/*下面的宏严重依赖于结构的布局，所以二进制不兼容几乎是必然的*/
/* Macro: yield the struct event 'offset' bytes from the start of 'p' */
#define EVENT_AT_OFFSET(p, offset) \
            ((struct event*) ( ((char*)(p)) + (offset) ))
/* Macro: yield the read event of an event_pair */
#define READEV_PTR(pair) \
            EVENT_AT_OFFSET((pair), sizeof(struct event_pair))
/* Macro: yield the write event of an event_pair */
#define WRITEEV_PTR(pair) \
            EVENT_AT_OFFSET((pair), \
                sizeof(struct event_pair)+event_get_struct_event_size())

/* Macro: yield the actual size to allocate for an event_pair */
#define EVENT_PAIR_SIZE() \
            (sizeof(struct event_pair)+2*event_get_struct_event_size())

void readcb(evutil_socket_t, short, void *);
void writecb(evutil_socket_t, short, void *);
struct event_pair *event_pair_new(struct event_base *base, evutil_socket_t fd)
{
        struct event_pair *p = malloc(EVENT_PAIR_SIZE());
        if (!p) return NULL;
        p->fd = fd;
        event_assign(READEV_PTR(p), base, fd, EV_READ|EV_PERSIST, readcb, p);
        event_assign(WRITEEV_PTR(p), base, fd, EV_WRITE|EV_PERSIST, writecb, p);
        return p;
}
```

## Making events pending and non-pending

### event_add

```c
int event_add(struct event *ev, const struct timeval *tv);
```

对一个非挂起的event调用event_add会使其在所配置的base上挂起，也就是调用event_new时传入的base上。成功返回0，失败返回-1。如果tv为NULL，则event没有超时，否则就以tv所设置的时间设置超时。

如果对一个已经挂起的事件调用event_add，则其保持挂起状态，并且重新调度超时，但如果此时tv设为NULL是不起作用的。

注意不要把*tv* 设置为你想要超时发生的时刻，比方说如果`tv->tv_sec = time(NULL)+10;`，那么超时时间就会是当前时间距epoch的秒数再加上10秒，而不仅仅是10秒。

### event_del

```c
int event_del(struct event *ev);
```

对一个已初始化的event调用event_del会使其变成非挂起和非活动的。如果event不处于挂起或者活动状态，调用这个函数无效。也就是说，这个函数不是要销毁event，只是使其与base脱离关联。

要注意如果在一个事件变成活动的但其回调还没有机会执行的时候调用这个函数，其回调将不会被执行。

成功返回0，失败返回-1。

### event_remove_timer

```c
int event_remove_timer(struct event *ev);
```

这个函数用来完全移除一个挂起事件的超时，而不会删除其IO和信号部分。如果事件本身没有超时设置，这个函数不会起效。如果一个事件只有超时而没有IO或者信号，这个函数的效果等同于event_del。

成功返回0，失败返回-1。

## Events with priorities

### event_priority_set

```c
int event_priority_set(struct event *event, int priority);
```

当多个事件同时触发时，Libevent并没有定义以什么顺序来执行回调。可以通过优先级来确定一些事件的执行顺序。

如前面讨论过的，每一个event_base都关联有一个或多个优先级层数，在把一个事件添加到event_base之前，但必须在初始化之后，可以设置其优先级。

事件的优先级从0开始，到event_base的优先级层数-1,也就是event_base_priority_init函数的参数n_priorities再减1。

函数成功返回0，失败返回-1。

当多个不同优先级的事件变成活动时，低优先权的事件不会运行，如前面event_loop的伪代码所示，Libevent只会运行最高优先权的事件，然后再检测事件。只有没有更高优先权的活动事件时才会运行低优先权的事件。

### 例子

```c
#include <event2/event.h>

void read_cb(evutil_socket_t, short, void *);
void write_cb(evutil_socket_t, short, void *);

void main_loop(evutil_socket_t fd)
{
  struct event *important, *unimportant;
  struct event_base *base;

  base = event_base_new();
  event_base_priority_init(base, 2);
  /* Now base has priority 0, and priority 1 */
  important = event_new(base, fd, EV_WRITE|EV_PERSIST, write_cb, NULL);
  unimportant = event_new(base, fd, EV_READ|EV_PERSIST, read_cb, NULL);
  event_priority_set(important, 0);
  event_priority_set(unimportant, 1);

  /* Now, whenever the fd is ready for writing, the write callback will
     happen before the read callback.  The read callback won't happen at
     all until the write callback is no longer active. */
}
```

在创建事件时，会缺省设置事件优先级为event_base中各优先级队列的个数除以2，也就是把一个事的优先级件置为中间优先级，这里需要明确一点，各优先级队列的总数是等于n_priorities的，因此与前面说到event_base_priority_init时的设置本质上是一样的。

## Inspecting event status
### event_pending
```c
int event_pending(const struct event *ev, short what, struct timeval *tv_out);
```

这个函数用来确定一个给定事件是否挂起或者活动。如果是，并且what参数设置为任意的EV_READ、EV_WRITE、EV_SIGNAL或者EV_TIMEOUT，那么函数返回event当前所挂起或者活动的所有标志。如果提供了tv_out参数，并且在what参数中设置了EV_TIMEOUT，以及事件当前挂起或者活动于一个超时，那么tv_out被设置为事件的超时时间。

这里的返回值需要参照源码说明一下，返回0首先表示ev对应的event_base为空，也就是说这个event并没有与event_base关联，从而也就不可能是挂起的也更不可能是活动的。如果有关联的event_base，则获取event的当前状态，这个状态暂时不重要，接下来如果传入的what为0，那么返回值最后是0，如果what不为0，就是上面所说的那样，如果event没有任何状态，当然返回的也是0。

所以，这个函数的逻辑上是有问题的。不应该用一个返回值来代表几层涵义，毕竟这里所做的所有判定都有专门的函数列在下面。

### event_get_signal

```c
#define event_get_signal(ev) /* ... */
```

### event_get_fd
```c
evutil_socket_t event_get_fd(const struct event *ev);
```

这两个函数返回一个事件对应的文件描述符或者信号编号。

### event_get_base

```c
struct event_base *event_get_base(const struct event *ev);
```

 返回event所配置的event_base。

### event_get_events

```c
short event_get_events(const struct event *ev);
```

返回event的标记(EV_READ、EV_WRITE等)。

### event_get_callback

```c
event_callback_fn event_get_callback(const struct event *ev);
```

### event_get_callback_arg
```c
void *event_get_callback_arg(const struct event *ev);
```

返回回调函数及其参数的指针。

### event_get_priority

```c
int event_get_priority(const struct event *ev);
```

返回event当前赋予的优先级。

### event_get_assignment

```c
void event_get_assignment(const struct event *event,
        struct event_base **base_out,
        evutil_socket_t *fd_out,
        short *events_out,
        event_callback_fn *callback_out,
        void **arg_out);
```

把event所有域的值复制到参数中对应的指针上，如果某个参数指针为NULL，则忽略对应的域。参数的后缀_out实际上提示了这些参数是所谓传出参数。

### 例子

```c
#include <event2/event.h>
#include <stdio.h>

/* Change the callback and callback_arg of 'ev', which must not be
 * pending. */
int replace_callback(struct event *ev, event_callback_fn new_callback,
    void *new_callback_arg)
{
    struct event_base *base;
    evutil_socket_t fd;
    short events;

    int pending;

    pending = event_pending(ev, EV_READ|EV_WRITE|EV_SIGNAL|EV_TIMEOUT,
                            NULL);
    if (pending) {
        /* We want to catch this here so that we do not re-assign a
         * pending event.  That would be very very bad. */
        fprintf(stderr,
                "Error! replace_callback called on a pending event!\n");
        return -1;
    }

    event_get_assignment(ev, &base, &fd, &events,
                         NULL /* ignore old callback */ ,
                         NULL /* ignore old callback argument */);
	/*如果这里ev的base为空，下面的函数毫无意义吧*/
    event_assign(ev, base, fd, events, new_callback, new_callback_arg);
    return 0;
}
```

## Finding the currently running event

### event_base_get_running_event

```c
struct event *event_base_get_running_event(struct event_base *base);
```

出于调试或者其它某种需要时，使用函数获取当前正在运行的event。这个函数的行为只有在指定的event_base的循环里调用才是有意义的。在其它线程中调用是不支持的，其行为也是未定义的。这里的意思是，调用这个函数的线程必须与event_base的事件循环位于同一个线程，或者说只能获取同一个线程中的event_base事件循环中当前正在运行的event。

## Configuring one-off events
### event_base_once
```c
int event_base_once(struct event_base *, evutil_socket_t, short,  void (*)(evutil_socket_t, short, void *), void *, const struct timeval *);
```

如果一个事件不需要多次添加，或者添加完运行之后马上删除，或者不需要是持久的，可以使用这个函数。

这个函数的接口与event_new相同，除了不支持 EV_SIGNAL和EV_PERSIST。事件被插入并且以缺省优先级运行。当回调结束，Libevent就会释放这个event的内部结构本身。

成功返回0，失败返回-1.

用这个函数插入的事件不能手工删除或者激活，因此如果想要常规操作一个事件，就不要使用这个函数。

2.0版本之前，如果事件没有被触发过，其内部存储是不会被释放的。到了2.1.2-alpha，当event_base被释放时，即使没有被激活的也可以释放这些事件。但是仍然要注意，如果有一些存储空间与其回调函数的参数相关联，这些空间只能程序自行释放，Libevent是追踪不到的。

## Manually activating an event

### event_active

```c
void event_active(struct event *ev, int what, short ncalls);
```

即使触发条件不满足，也可以强制启动一个事件。

这个函数就是以what标记(EV_READ、EV_WRITE或者EV_TIMEOUT的组合)来强制激活一个event，这个event不需要事先出于挂起状态，激活也不会使其挂起。这里所谓的强制激活，可以理解为手工发送一个事件触发标记，事件被激活还是要等到事件循环开始之后。

注意，对同一个事件递归调用这个函数会造成资源耗尽。

### 不好的例子：making an infinite loop with event_active()

```c
struct event *ev;

static void cb(int sock, short which, void *arg) {
        /* Whoops: Calling event_active on the same event unconditionally
           from within its callback means that no other events might not get
           run! */

        event_active(ev, EV_WRITE, 0);
}

int main(int argc, char **argv) {
        struct event_base *base = event_base_new();

        ev = event_new(base, -1, EV_PERSIST | EV_READ, cb, NULL);

        event_add(ev, NULL);

        event_active(ev, EV_WRITE, 0);

        event_base_loop(base, 0);

        return 0;
}
```

于是事件循环只运行了一次，之后就在无限调用cb。这里只是说明问题，其它的两个改进例子没什么意义，就略过了。

## Optimizing common timeouts

### event_base_init_common_timeout

```c
const struct timeval *event_base_init_common_timeout(
    struct event_base *base, const struct timeval *duration);
```

当前版本Libevent使用一个二叉树堆算法来跟踪挂起事件的超时。一个二叉树堆对于添加和删除事件超时操作的性能O(lg n)，如果超时值是随机分布的，其性能就还不错，如果大量的事件有相同的超时值，那么就比较糟糕了。

比如，假定有1万个事件，每一个都会在其添加5秒钟后触发超时，在这种情况下，如果使用双向队列，则每次超时可以得到O(1)的性能。但是，队列只是对于不变的超时值才会有较快的速度，如果这个超时值是多多少少随机分布的，那么在队列中插入这些值就会是O(n)的时间，明显是不如二叉树堆的。

Libevent的解决方法是把一部分超时放在队列中，一部分放在二叉树堆中。为此，可以向Libevent申请一个公共超时时间，然后使用这个值去添加事件。如果有大量的事件具有同一个公共超时事件，那么就可以优化超时处理。

这个函数的base参数就不说明了。duration参数是传递给函数，表示用其值来初始化公共超时。返回值是一个指针，指向一个特殊的timeval结构，指示一个事件是插入到O(1)的队列而不是O(lg n)堆中，这个特殊timeval可以自由的复制或者赋值，但只对base参数所指的event_base有效。不要依赖其真实的内容，Libevent只是用其来提示自己使用哪一个队列。

这个函数没有特别的需要，不要使用。

### 例子

```c
#include <event2/event.h>
#include <string.h>

/* We're going to create a very large number of events on a given base,
 * nearly all of which have a ten-second timeout.  If initialize_timeout
 * is called, we'll tell Libevent to add the ten-second ones to an O(1)
 * queue. */
struct timeval ten_seconds = { 10, 0 };

void initialize_timeout(struct event_base *base)
{
    struct timeval tv_in = { 10, 0 };
    const struct timeval *tv_out;
    tv_out = event_base_init_common_timeout(base, &tv_in);
    /*注意这里覆盖了原来ten_seconds的值,接下来使用的也是ten_seconds*/
    memcpy(&ten_seconds, tv_out, sizeof(struct timeval));
}

int my_event_add(struct event *ev, const struct timeval *tv)
{
    /* Note that ev must have the same event_base that we passed to
       initialize_timeout */
    /*不要依赖于ten_seconds的具体值，根据传入的tv的值来判断插入哪个队列*/
    if (tv && tv->tv_sec == 10 && tv->tv_usec == 0)
        return event_add(ev, &ten_seconds);
    else
        return event_add(ev, tv);
}
```

## Telling a good event apart from cleared memory

### event_initialized

```c
int event_initialized(const struct event *ev);

#define evsignal_initialized(ev) event_initialized(ev)
#define evtimer_initialized(ev) event_initialized(ev)
```

使用这个函数可以分辨一个已初始化的event是通过设置为0来清空的(比方说用`calloc()`来分配)，还是通过memset或者bzero来清空的。

但要注意，这个函数并不能可靠的辨别出一个初始化的event和一块未初始化的内存块。除非你知道需要处理的内存确实是给一个event分配然后清空或者初始化的。

通常来说，并不需要使用这些函数，除非有特别的需要。由event_new返回的event总是初始化了的，也就是清空了的。

### 例子

```c
#include <event2/event.h>
#include <stdlib.h>

struct reader {
    evutil_socket_t fd;
};

#define READER_ACTUAL_SIZE() \
    (sizeof(struct reader) + \
     event_get_struct_event_size())

#define READER_EVENT_PTR(r) \
    ((struct event *) (((char*)(r))+sizeof(struct reader)))

struct reader *allocate_reader(evutil_socket_t fd)
{
    struct reader *r = calloc(1, READER_ACTUAL_SIZE());
    if (r)
        r->fd = fd;
    return r;
}

void readcb(evutil_socket_t, short, void *);
int add_reader(struct reader *r, struct event_base *b)
{
    struct event *ev = READER_EVENT_PTR(r);
    if (!event_initialized(ev))
        event_assign(ev, b, r->fd, EV_READ, readcb, r);
    return event_add(ev, NULL);
}
```

