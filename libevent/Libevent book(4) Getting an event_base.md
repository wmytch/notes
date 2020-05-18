# Getting an event_base

[TOC]

## Creating an event_base

在使用Libevent函数前，必须先分配一个或者多个event_base结构。每一个event_base结构都持有一个事件集并通过轮训来检查活动的事件。

如果一个event_base被设置为使用锁，那么在多个线程中对其进行访问是安全的，但是其循环只能在单独的一个线程中进行，如果需要多个线程轮询IO，就需要每个线程一个event_base。

每一个event_base都有一个“方法”，或者说后端，用来检测就绪事件。已知的方法有：

- select
- poll
- epoll
- kqueue
- devpoll
- evport
- win32

用户可以通过环境变量来禁用指定的后端。比方说如果要关掉kqueue后端，就设置EVENT_NOKQUEUE环境变量，等等。如果要在程序中关掉某个后端，参见下面`event_config_avoid_method() `的说明。

## Setting up a default event_base

### event_base_new

```c
struct event_base *event_base_new(void);
```

`event_base_new()`函数分配并返回一个新的event base，其设置为缺省设置。该函数会检查环境变量并返回指向一个新的event_base的指针，如果出错，则返回NULL。

当有可选方法时，会选择操作系统支持的最快的那个(指上面所说的方法)。

对大多数程序，这就够了。

这个函数在` <event2/event.h>`中声明。

## Setting up a complicated event_base

如果希望对event_base有更多的控制，就需要使用event_config，这个结构用来保存一些关于event_base的偏好设置，将其传递给`event_base_new_with_config()`就可以得到需要event_base。

### event_config

```c
struct event_config *event_config_new(void);
struct event_base *event_base_new_with_config(const struct event_config *cfg);
void event_config_free(struct event_config *cfg);
```

调用`event_config_new()`得到一个新的 event_config，然后调用下面将要描述的函数来设置event_config，最后，调用`event_base_new_with_config() `得到一个新的event_base。所有工作完成之后，可以用`event_config_free()`释放event_config，至于什么时候可以释放event_config，按照一般道理，如果将来不再使用的话，event_base_new_with_config返回之后就可以释放了，但是谁又说得准程序中什么时候又要用呢，所以，我觉得要么不管它，等程序退出时自动清理，如果一些工具提示资源泄露，那再说。

需要注意的是，如果一个配置Libevent无法满足，`event_base_new_with_config()`将返回NULL。  For example, as of Libevent 2.0.1-alpha, there is no O(1) backend for Windows, and no backend on Linux that provides both EV_FEATURE_FDS and EV_FEATURE_O1。比如，在Libevent 2.0.1-alpha，对于Windows没有O(1)的后端，对于Linux不存在同时满足EV_FEATURE_FDS和EV_FEATURE_O1的后端。

### event_config_avoid_method

```c
int event_config_avoid_method(struct event_config *cfg, const char *method);
```
 调用event_config_avoid_method可以告知Libevent避免使用一个特定名字的可用后端，注意这的参数是后端的名字。

### event_config_require_features
```c
enum event_method_feature {
    EV_FEATURE_ET = 0x01,
    EV_FEATURE_O1 = 0x02,
    EV_FEATURE_FDS = 0x04,
};
int event_config_require_features(struct event_config *cfg,
                                  enum event_method_feature feature);
```
调用`event_config_require_feature()`告知Libevent不使用那些不支持指定特性集的后端。已知的特性值有：

- EV_FEATURE_ET

    后端要支持边缘触发IO。

- EV_FEATURE_O1

    要求对于增加或者移除一个单独的事件，或者使得一个单独事件变成活动的这些操作是O(1)的。

- EV_FEATURE_FDS

    要求后端能够支持任意类型的文件描述符，而不仅仅是socket。
### event_config_set_flag
```c
enum event_base_config_flag {
    EVENT_BASE_FLAG_NOLOCK = 0x01,
    EVENT_BASE_FLAG_IGNORE_ENV = 0x02,
    EVENT_BASE_FLAG_STARTUP_IOCP = 0x04,
    EVENT_BASE_FLAG_NO_CACHE_TIME = 0x08,
    EVENT_BASE_FLAG_EPOLL_USE_CHANGELIST = 0x10,
    EVENT_BASE_FLAG_PRECISE_TIMER = 0x20
};
int event_config_set_flag(struct event_config *cfg,
    enum event_base_config_flag flag);
```
调用`event_config_set_flag()`告知Libevent在构造event base时设置一个或者多个运行时标志。这些已知的标志的有：

- EVENT_BASE_FLAG_NOLOCK

    不为event_base分配锁。设置这个参数可以节省一点对event_base加锁和释放锁的时间，但是在多线程下是不安全并且无法操作的。

- EVENT_BASE_FLAG_IGNORE_ENV

    挑选后端方法时不检查` EVENT_*`环境变量。在使用这个标记前要考虑清楚：这可能会使得用户难以调试程序与Libevent之间的互动。

- EVENT_BASE_FLAG_STARTUP_IOCP

    只适用于Windows，这个标记使得Libevent在启动时就启用所有必需的IOCP派遣逻辑，而不是需要时才启用。

- EVENT_BASE_FLAG_NO_CACHE_TIME

    不是每当事件循环即将运行超时回调函数时检查当前时间，而是在超时回调完成后检查。这可能会大量耗费CPU，所以千万注意。

- EVENT_BASE_FLAG_EPOLL_USE_CHANGELIST

    告知Libevent，如果要使用epoll后端，使用更快的基于”changelist”的后端是安全的。当同一个fd在调用后端派遣函数期间自身状态改变不止一次时，epoll-changelist后端可以避免一些不必要的系统调用。因为fd状态发生变化就意味着事件发生，如果在调用派遣函数期间fd发生了多次状态变化，当轮到处理这个fd的事件时，是依次处理发生的事件呢，还是处理最后一次事件？这里就是要解决这个问题。但是如果提供给Libevent的是用`dup()`或者其它类似的函数复制的fd，那么就会造成错误的结果，这是由于一个系统bug。如果不使用epoll，这个标志是无效的。也可以使用EVENT_EPOLL_USE_CHANGELIST环境变量来启用epoll-changelist。

- EVENT_BASE_FLAG_PRECISE_TIMER

    缺省情况下，Libevent总是要使用操作系统提供的最快的可用计时机制。如果有慢一些的计时机制可以提供精度更高的计时，这个标记就告知Libevent使用这个慢一些但精度更高的计时机制。如果操作系统没有提供这样的更慢但是更精确的机制，这个标记无效。

上面所有处理event_config的函数成功都返回0，失败返回-1。

### event_config_set_num_cpus_hint

```c
int event_config_set_num_cpus_hint(struct event_config *cfg, int cpus)
```

这个函数只在Windows下使用IOCP时有效，将来也许对其他平台也会有效。表示通过event_config生成的event_base在多线程时要充分利用CPU数量。注意这只是一个提示，event base不一定会遵从这个值，可能会多也可能会少。

### event_config_set_max_dispatch_interval

```
int event_config_set_max_dispatch_interval(struct event_config *cfg,
    const struct timeval *max_interval, int max_callbacks,
    int min_priority);
```

这个函数用来避免优先权倒转，在对更高优先权的事件进行检测前，限制调用低优先权事件回调。如果max_interval非null，事件循环在每次回调之后检查时间，如果max_interval已经过去，则重新扫描高优先权的事件。如果max_callbacks非负，事件循环也会在max_callbacks这么多个回调调用之后检查更多的事件。这些规则适用于min_priority或者更高优先权的事件。实际上，这个函数设置的是在一定条件下优先扫描高于或等于指定优先权的事件，如果不存在或者处理完了，才会去扫描低于指定优先权的事件。

### 例子： Preferring edge-triggered backends

```c
struct event_config *cfg;
struct event_base *base;
int i;

/* My program wants to use edge-triggered events if at all possible.  So
   I'll try to get a base twice: Once insisting on edge-triggered IO, and
   once not. */
for (i=0; i<2; ++i) {
    cfg = event_config_new();

    /* I don't like select. */
    event_config_avoid_method(cfg, "select");

    if (i == 0)
        event_config_require_features(cfg, EV_FEATURE_ET);

    base = event_base_new_with_config(cfg);
    event_config_free(cfg);
    if (base)
        break;

    /* If we get here, event_base_new_with_config() returned NULL.  If
       this is the first time around the loop, we'll try again without
       setting EV_FEATURE_ET.  If this is the second time around the
       loop, we'll give up. */
}
```

### 例子: Avoiding priority-inversion

```c
struct event_config *cfg;
struct event_base *base;

cfg = event_config_new();
if (!cfg)
   /* Handle error */;

/* I'm going to have events running at two priorities.  I expect that
   some of my priority-1 events are going to have pretty slow callbacks,
   so I don't want more than 100 msec to elapse (or 5 callbacks) before
   checking for priority-0 events. */
struct timeval msec_100 = { 0, 100*1000 };
event_config_set_max_dispatch_interval(cfg, &msec_100, 5, 1);

base = event_base_new_with_config(cfg);
if (!base)
   /* Handle error */;

event_base_priority_init(base, 2);
```

以上函数和类型定义在<event2/event.h>中声明。

## Examining an event_base’s backend method

### event_get_supported_methods

```c
const char **event_get_supported_methods(void);
```

返回一个当前版本Libevent所支持方法的名称的数组，这个数组的最后一个元素是NULL。注意，这个函数返回的是编译Libevent时所指定支持的方法，而不是操作系统确实支持的方法。

#### 例子

```c
int i;
const char **methods = event_get_supported_methods();
printf("Starting Libevent %s.  Available methods are:\n",
    event_get_version());
for (i=0; methods[i] != NULL; ++i) {
    printf("    %s\n", methods[i]);
}
```

### event_base_get_method/event_base_get_features

```c
const char *event_base_get_method(const struct event_base *base);
enum event_method_feature event_base_get_features(const struct event_base *base);
```

 `event_base_get_method()`返回event_base确实在使用的方法名称，`event_base_get_features()`返回所支持的所有特性的bitmask。这些函数在`<event2/event.h>`中声明。

#### 例子

```c
struct event_base *base;
enum event_method_feature f;

base = event_base_new();
if (!base) {
    puts("Couldn't get an event_base!");
} else {
    printf("Using Libevent with backend method %s.",
        event_base_get_method(base));
    f = event_base_get_features(base);
    if ((f & EV_FEATURE_ET))
        printf("  Edge-triggered events are supported.");
    if ((f & EV_FEATURE_O1))
        printf("  O(1) event notification is supported.");
    if ((f & EV_FEATURE_FDS))
        printf("  All FD types are supported.");
    puts("");
}
```

## Deallocating an event_base

### event_base_free

```c
void event_base_free(struct event_base *base);
```

注意这个函数并不释放当前与该event_base关联的事件，也不关闭socket，也不释放它们的指针。这个函数在` <event2/event.h>`中定义。

## Setting priorities on an event_base

### event_base_priority_init

```ｃ
int event_base_priority_init(struct event_base *base, int n_priorities);
```

缺省情况下，base_event只支持一个优先级。通过这个函数可以设置所支持的优先级的层数。调用成功返回0，失败返回-1。base参数是要改变的event_base，n_priorities参数是要支持的优先级层数，必须至少为1。对于新事件的有效优先级数从0(最重要的，或者说优先级最高的)到n_priorities-1(最低的)。

常量EVENT_MAX_PRIORITIES设定了n_priorities的值的上界，如果n_priorities的值高于这个值，就会出错。

必须在任意事件有效前调用这个函数，最好是在创建了event_base之后马上调用。缺省情况下，所有与这个base关联的事件的优先权都等于n_priorities / 2。

### event_base_get_npriorities

```c
int event_base_get_npriorities(struct event_base *base);
```

返回值是对base所配置的优先权的层数，如果函数返回3，则允许的优先权值是0，1，2。

以上函数都在`<event2/event.h>`中定义。

## Reinitializing an event_base after fork()

### event_reinit

```c
int event_reinit(struct event_base *base);
```

并非所有的事件后端在调用`fork()`后会清理，所以，如果程序中使用fork或者其它类似调用来启动一个新进程，并且希望在fork之后继续使用event_base，那么就需要重新初始化。

成功返回0，失败返回-1。在`<event2/event.h>`中定义。

#### 例子

```c
struct event_base *base = event_base_new();

/* ... add some events to the event_base ... */

if (fork()) {
    /* In parent */
    continue_running_parent(base); /*...*/
} else {
    /* In child */
    event_reinit(base);
    continue_running_child(base); /*...*/
}
```

##　Obsolete event_base functions

旧版本的Libevent非常着重于"当前"event_base的概念。“当前”event_base是一个全局设定，由所有的线程共享。如果忘记了指定所需要的event_base，就会得到当前这一个。由于event_base不是线程安全的，这就会相当大容易出错。

### event_init

```c
struct event_base *event_init(void);
```

这个函数与`event_base_new()`一样创建新的event_base，但是会把所分配的base设定为当前的base。没有其它方式可以改变当前base。

本节的一些event_base函数有一些变型用来操作当前base。这些函数的行为与当前版本的函数一致，除了它们不需要一个base参数外。

| Current function           | Obsolete current-base version |
| -------------------------- | ----------------------------- |
| event_base_priority_init() | event_priority_init()         |
| event_base_get_method()    | event_get_method()            |