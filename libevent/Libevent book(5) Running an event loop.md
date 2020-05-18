# Running an event loop

[TOC]

## Running the loop

### event_base_loop

```c
#define EVLOOP_ONCE             0x01
#define EVLOOP_NONBLOCK         0x02
#define EVLOOP_NO_EXIT_ON_EMPTY 0x04

int event_base_loop(struct event_base *base, int flags);
```

缺省情况下，`event_base_loop()`会运行一个event_base直到没有事件注册进来，在循环中，会不断的检测是否有注册事件触发，比如，一个读事件的文件描述符读就绪，或者一个超时事件即将到期，当出现这种情况时，就会把所有触发的事件标记为“活动的“，然后开始运行这些事件(注册的回调函数)。

可以通过flags参数设置一些标记来改变`event_base_loop()`的行为。如果设置了EVLOOP_ONCE，则循环会等待直到一些事件变成活动的，然后运行这些事件，直到没有更多的事件可以运行，于是从函数返回。如果设置了EVLOOP_NONBLOCK，循环不会等待事件触发，而只是检测是否有事件会马上触发，如果有，则运行其回调函数。

通常来说，如果没有挂起或者活动的事件循环就会退出。也可以传递EVLOOP_NO_EXIT_ON_EMPT标记来改变这种行为，比方说如果需要从其它的线程中添加事件，设置了这个标记之后，循环就会一直运行，直到调用`event_base_loopbreak()`,或者 `event_base_loopexit()`，或者有错误发生。

正常退出函数返回0，如果后端发生了某些未处理的错误，返回-1，如果没有挂起或者活动的事件则返回1。

为帮助理解，下面是说明event_base_loop算法的伪代码：

```c++
while (any events are registered with the loop,
        or EVLOOP_NO_EXIT_ON_EMPTY was set) {

    if (EVLOOP_NONBLOCK was set, or any events are already active)
        If any registered events have triggered, mark them active.
    else
        Wait until at least one event has triggered, and mark it active.

    for (p = 0; p < n_priorities; ++p) {
        /*for循环只对某个优先级的事件进行处理，处理完成之后就退出循环了。
        只有在下一次while循环到了这里的时候，如果不存在更高优先级的活动事件，
        才会运行低优先级的事件*/
       if (any event with priority of p is active) {
          Run all active events with priority of p.
          break; /* Do not run any events of a less important priority */
       }
    }
	/*如果没有事件是活动的，或者当前最高优先级的活动事件处理完了，函数就会返回，
	这也意味着需要用户自己维护一个事件循环，而不是在函数里由函数维护*/
    if (EVLOOP_ONCE was set or EVLOOP_NONBLOCK was set)
       break;
}
```

### event_base_dispatch

```c
int event_base_dispatch(struct event_base *base)
```

这个函数等同于没有设置标记的event_base_loop。也就是说它会一直运行，直到没有注册的事件或者调用`event_base_loopbreak()`/`event_base_loopexit()`。

这些函数在 <event2/event.h>中定义。

## Stopping the loop

### 接口

```c
int event_base_loopexit(struct event_base *base,
                        const struct timeval *tv);
int event_base_loopbreak(struct event_base *base);
int event_base_got_exit(struct event_base *base);
int event_base_got_break(struct event_base *base);
```

前面两个函数用于结束事件循环，两者有一些区别：

- `event_base_loopexit()` 函数告诉一个event_base在一个给定的时间后停止循环。如果tv参数为NULL，则event_base马上停止循环，当然如果event_base正在运行事件的回调，会等待直到所有回调处理完才退出。如果以前面event_base_loop的伪代码来说明，这里就是等for循环退出后才退出while循环。

- `event_base_loopbreak()` 函数告诉event_base马上退出循环，与`event_base_loopexit(base, NULL)`不同的是，如果event_base正在运行一个活动事件的回调，则会等待该回调完成后马上退出。同样以前面伪代码说明的话，就是不等待for循环正常结束，而是处理完一个事件回调后就退出while循环。

注意当没有事件循环运行时，也就是说在循环时没有事件发生的情况下， `event_base_loopexit(base,NULL)` 和`event_base_loopbreak(base) `行为是不一样的：如前所述，不管怎么样，loopexit都会在下一个轮次的回调函数运行之后停止下一个事件循环也就是while循环的运行，就等于使用了EVLOOP_ONCE标志，就是说无论如何都要进到循环里边跑一遍，而loopbreak只是停止当前的循环运行，如果没有事件循环，则不产生任何影响而直接退出。

两个函数都是成功返回0，失败返回-1.

### 例子: Shut down immediately

```c
#include <event2/event.h>

/* Here's a callback function that calls loopbreak */
void cb(int sock, short what, void *arg)
{
    struct event_base *base = arg;
    event_base_loopbreak(base);
}

void main_loop(struct event_base *base, evutil_socket_t watchdog_fd)
{
    struct event *watchdog_event;

    /* Construct a new event to trigger whenever there are any bytes to
       read from a watchdog socket.  When that happens, we'll call the
       cb function, which will make the loop exit immediately without
       running any other active events at all.
     */
    watchdog_event = event_new(base, watchdog_fd, EV_READ, cb, base);

    event_add(watchdog_event, NULL);

    event_base_dispatch(base);
}
```

### 例子: Run an event loop for 10 seconds, then exit.

```c
#include <event2/event.h>

void run_base_with_ticks(struct event_base *base)
{
  struct timeval ten_sec;

  ten_sec.tv_sec = 10;
  ten_sec.tv_usec = 0;

  /* Now we run the event_base for a series of 10-second intervals, printing
     "Tick" after each.  For a much better way to implement a 10-second
     timer, see the section below about persistent timer events. */
  while (1) {
     /* This schedules an exit ten seconds from now. */
     event_base_loopexit(base, &ten_sec);

     event_base_dispatch(base);//挂起，10秒后退出
     puts("Tick");
  }
}
```
有时候可能需要知道` event_base_dispatch()` 或者 `event_base_loop() `是正常退出，还是因为调用了`event_base_loopexit()` 或者`event_base_break()`退出的，这时候可以使用下面两个函数：

```c
int event_base_got_exit(struct event_base *base);
int event_base_got_break(struct event_base *base);
```

对应两种情况，这两个函数分别会返回真或假，也就是非0或者0，当然，如果是正常退出，两个函数都会返回假，也就是0。它们的返回值在启动下一次事件循环后会重置。

上面函数都在`<event2/event.h>`中定义。

## Re-checking for events

### event_base_loopcontinue

```c
int event_base_loopcontinue(struct event_base *);
```

通常，Libevent会坚持事件，然后运行最高优先权的活动事件，然后再检查事件，等等。但有时候可能需要在当前回调结束之后停止Libevent，然后再开始扫描。对应于`event_base_loopbreak()`，这种情况可以调用函数`event_base_loopcontinue()`。

如果当前没有正在运行的回调，则调用这个函数没有任何影响。

## Checking the internal time cache

### event_base_gettimeofday_cached

```c
int event_base_gettimeofday_cached(struct event_base *base,
    struct timeval *tv_out);
```

有时候可能需要在一个事件回调中获取当前时间的近似视图，但是又想使用系统调用`gettimeofday()`，这时候可以使用event_base_gettimeofday_cached来获取这个视图，得到本次回调轮转开始的时间。

`event_base_gettimeofday_cached()`函数设置tv_out参数的值为一个缓存的时间，如果event_base当前正在执行回调的话，否则就调用`evutil_gettimeofday() `来获取真实的当前时间。成功返回0，失败返回负数。

### event_base_update_cache_time

```c
int event_base_update_cache_time(struct event_base *base);
```

由于前面的timeval缓存的是Libevent开始运行回调的时间，所以不是那么的准确，如果回调会执行比较长的时间，就会非常的不准确，这时候可以用event_base_update_cache_time来强制刷新这个缓存。

成功返回0，失败返回-1，如果没有事件循环在运行则不产生任何效果。

## Dumping the event_base status

### event_base_dump_events

```c
void event_base_dump_events(struct event_base *base, FILE *f);
```

为了帮助调试程序或者Libevent，有时候可能需要一个完整的列表，包含所有添加到event_base中的事件及其状态。这时候可以调用event_base_dump_events把这个列表写到文件中去。

这个列表是用来给人看的，所以其格式可能会随Libevent版本改变而变。

## Running a function over every event in an event_base

### event_base_foreach_event

```c
typedef int (*event_base_foreach_event_cb)(const struct event_base *,
    const struct event *, void *);

int event_base_foreach_event(struct event_base *base,
                             event_base_foreach_event_cb fn,
                             void *arg);
```

可以使用`event_base_foreach_event()`来迭代与一个event_base关联的所有当前活动的或者挂起的事件。每个事件的回调会被确实地调用一次，其调用顺序是未指定的。`event_base_foreach_event()`的第三个参数arg会被作为每一个回调函数的第三个参数传入。

回调函数返回0则继续迭代，其它整数则会停止迭代。回调函数最终返回的值会作为`event_base_foreach_function()`的返回值返回。

回调函数**不能**改变其所收到的任何事件，或者增加和移除event base中的任何事件，或者改变任何与event base关联的任何事件，否则就会发生未定义行为，包括程序崩溃或者heap-smashing。

在调用` event_base_foreach_event()`时会锁住event_base，这样可以避免其它线程对event_base的影响，所以确保回调不会执行太长的时间。

