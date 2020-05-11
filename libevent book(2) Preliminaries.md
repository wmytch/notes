# The Libevent Reference Manual: Preliminaries

[TOC]

## Libevent from 10,000 feet

libevent由以下这些组件或者说部分组成：

- evutil

    ​    对不同平台的网络实现的一个通用的抽象。或者说，提供了一个通用的接口，封装了底层不同平台的网络实现。主要这里指的是平台的网络模块的抽象或者封装。比方说recv或者send函数在不同平台可能有不同的调用接口，而libevent则对此作了封装，并提供了一套一致的读写接口。

- event and event_base

       这是libevent的中心，提供了一套抽象的api，以适应不同平台的基于事件的非堵塞IO后端，注意，这里着重的是**事件**，包括socket什么时候可读可写，基本的计时器操作，以及检测操作系统信号。

- bufferevent

    ​    针对libevent基于事件的内核，提供了一套基于buffer事件的更为便利的封装，而不是基于socket本身的事件。bufferevent本身也是支持多种后端的，因此对不同的平台，是可以充分利用平台提供的高效非堵塞IO机制的。

- evbuffer

     这个模块实现了bufferevent底层的buferr，以及提供了对其高效便利的访问操作。

- evhttp

    一个简单的HTTP客户端和服务端的实现

- evdns

     一个简单的DNS客户端和服务端的实现

- evrpc

    一个简单的RPC实现

## The Libraries

构建libevent时，会缺省安装下面这些库：

- libevent_core

    ​    所有核心事件和缓冲的操作函数，包括所有的event_base，evbuffer，bufferevent，以及一些应用函数。

- libevent_extra

       定义了一些协议特定的函数，包括HTTP，DNS，RPC。

- libevent

    ​    由于一些历史原因提供的这个库，包含了前面提到的两个库，将来可能会取消的。

下面这些库可能只在某些平台上安装：

- libevent_pthreads

    ​    基于pthreads库的线程和锁的实现。如果不是确实要在多线程中使用libevent，那么就不需要链接这个库。

- libevent_openssl

    ​    支持使用bufferevent和OpenSSL库进行加密通信，同样，如果不需要，也是可以不用链接的。

## The Headers

所有公共的libevent的头文件都安装在`event2`目录下，头文件有三类：

- API headers

       API头文件定义了libevent当前版本的公共接口，这些头文件没有特别的后缀。

- Compatibility headers

      兼容性头文件包含了一些过时的定义，并不需要包含这些头文件，除非要移植使用libevent旧版本的程序。

- Structure headers

    ​    这些头文件定义的是内存布局相对易变的结构。其中一些是为了更快的访问结构体，一些是由于历史原因。直接依赖于这些头文件中的结构定义可能会影响程序的二进制兼容性或者难以调试。这些头文件有后缀“_struct.h”。

(有一些旧版本的Libevent没有 *event2* 目录.  参见下面的 "If you have to work with an old version of Libevent" )

## If you have to work with an old version of Libevent

事实上，libevent有差不多10年没有更新了，所以这里看看就行，没有必要过于关注，相信现在使用libevent的不会使用更老版本的libevent了。

| OLD HEADER… | …REPLACED BY CURRENT HEADERS                                 |
| ----------- | ------------------------------------------------------------ |
| event.h     | event2/event*.h, event2/buffer*.h event2/bufferevent*.h event2/tag*.h |
| evdns.h     | event2/dns*.h                                                |
| evhttp.h    | event2/http*.h                                               |
| evrpc.h     | event2/rpc*.h                                                |
| evutil.h    | event2/util*.h                                               |

