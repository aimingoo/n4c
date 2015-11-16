## N4C

N4C是一个标准架构，它提供一套指导规范、案例和高度可靠的模块，用于在“可控、可计算、可通信的集群”上实现复杂的、稳定的业务逻辑。

> N4C is a Controllable & Computable Communication Cluster architecture.

N4C最初提出的背景是面向实时计算的，它对实时计算提出了三点原则性意见，并作为N4C架构下的实时计算的基本原则。

> ``` 
> - 没有全量：瞬态观察下，没有一致性与有效性的要求；在单次计算中的数据完整性是不必要的。
> - 就近计算：数据越早规格化则收益越高；总是存在数据合并的中心点，但可以是动态的；层次决定并行效率。
> - 精度逼近：不存在一次精确，应通过足够多的正确来逼近精确。
> ```

实时计算并非N4C唯一考虑的环境因素。

本项目(aimingoo/n4c)只提供N4C的标准化文档和相关的索引。

### Table of Contents

* [N4C specifications](#n4c-specifications)
  * [1. N4C architecture](#1-n4c-architecture)
  * [2. PEDT specifications](#2-pedt-specifications)
* [N4C projects](#n4c-projects)
  * [PEDT implements](#pedt-implements)
    * [redpoll](#redpoll)
    * [harpseal](#harpseal)
    * [tundrawolf](#tundrawolf)
  * [N4C implements](#n4c-implements)
    * [ngx_4c](#ngx_4c)
    * [sandpiper](#sandpiper)
* [N4C documents](#n4c-documents)
* [others](#others)
* [history](#history)

# N4C specifications

## 1. N4C architecture

(TODO)

## 2. PEDT specifications

> @see $(n4c)/PEDT/*

并行的可交换分布式任务（PEDT, Parallel Exchangeable Distribution Task）是以可计算集群为对象的、以实时处理为目标的并行任务规范。该规范包括对任务数据、任务处理和任务调用接口三个部分的定义。

PEDT旨在为可计算集群提供一种轻量、高效和跨平台的可靠任务处理机制。

# N4C projects

本节列出有关N4C的主要开源项目。

## PEDT implements

* [redpoll for nodejs](https://github.com/aimingoo/redpoll)
* [harpseal for lua](https://github.com/aimingoo/harpseal)
* [tundrawolf for nginx_lua](https://github.com/aimingoo/tundrawolf)

### redpoll

redpoll（红顶雀）是一个nodejs上的项目，它完整地实现了PEDT 1.1。它可以作为一个npm的模块安装。

### harpseal

harpseal（竖琴海豹）是一个lua上的项目，它在原生lua上实现了PEDT 1.1。它需要LuaScoket模块来实现http client，并且需要copas模块来实现并行的http requests。

harpseal实现了一个完整的distributionScope解析函数(prefixParse in infra/taskhelper.lua)，这是最符合PEDT规范的三段标记(three parts token)解析算法。这与其它（例如redpoll for nodejs）很不相同，他们（后者）通常采用简单的前缀匹配来处理三段标记。参见：

> ``` text
> $(harpseal)/testcase/t_pedt_prefix.lua
> ```

harpseal还实现了一个递归注册taskDef的工具，可以将多层次的taskDef一次性地注册到register_center，参见：

> ``` text
> $(harpseal)/testcase/t_loadTask.lua
> $(harpseal)/tools/taskloader.lua
> ```

### tundrawolf

tundrawolf（冻原狼）是一个nginx_lua上的项目，它与harpseal（竖琴海豹）项目是基本相同的，但直接使用nginx lua中内置的http client、MD5、BASE64等功能/模块，是专门为nginx lua环境定制的。

tundrawolf实现了一种独特的（独立的）机制：路由发现（system_route discoveries）。用于管理当前结点中注册到系统路由(system_route)的对象。这些对象可以是任何（除lua false/nil值之外的）东西。一旦这些对象注册到系统路由，那么它就可以在Promise框架中作为并行对象来使用了。

tundrawolf通常是在nginx lua中实现N4C的基础组件。

## N4C implements

* [ngx_4c in nginx](https://github.com/aimingoo/ngx_4c)
* [sandpiper in nodejs](https://github.com/aimingoo/sandpiper)

### ngx_4c

ngx_4c是一个在nginx上的编程框架，它基于如下项目来实现了N4C架构：

> * tundrawolf
>   
>   实现PEDT distribution tasks规范。
>   
> * ngx_cc
>   
>   实现nginx集群内通信。
>   
> * Promise
>   
>   实现并行处理。
>   
> * Events
>   
>   实现了在nginx下实现的，基于多投事件的标准编程架构；并基于Events实现了N4C中的资源管理和任务管理服务。
>   
> * etcd
>   
>   使用etcd来做N4C中集群构建层的存储和心跳通知。

你需要安装一个nginx来构建你的测试环境。由于ngx_4c可选集成ngx_cc，而后者需要在nginx上打一个特定的补丁以便为每个nginx工作进程分配一个通信端口，因此建议你先参考如下文档：

> ``` text
> https://github.com/aimingoo/ngx_4c#install--usage
> ```

但是，如果不集成ngx_cc，那么ngx_4c本身并不要求nginx打上述补丁。

### sandpiper

sandpiper（Baird's sandpiper, 黑腰滨鹬）是一个在nodejs上的N4C实现，它基于如下项目：

> * redpoll
>   
>   实现PEDT distribution tasks规范。
>   
> * node-etcd-promise
>   
>   使用promise并行处理实现的etcd客户端。
>   
> * request
>   
>   简单的http客户端。
>   
> * crc-32
>   
>   在注册任务时使用crc-32对taskDef进行二次验证。
>   
> * etcd
>   
>   使用etcd来做N4C中集群构建层的存储和心跳通知。

sandpiper使用nodejs内建的标准Promise来实现并行处理。与ngx_4c类似，它实现了完整的资源中心与任务中心，并发布了它们的标准客户端界面。基于PEDT规范，它完整地实现了N4C集群中执行者(exeutor)、发布者(publisher)、分发者(dispatcher)等角色。

sandpiper提供一些小工具，包括一个资源服务（用于快速地检测/测试其它的N4C项目）：

``` bash
> git clone 'http://github.com/aimingoo/sandpiper'
> npm install; npm start
```

# N4C documents

本节列出有关N4C的主要一些公开文档或讨论。

# others

其它未确定内容。



# history

``` text
	2015.11		N4C opensource and github hosted.
```