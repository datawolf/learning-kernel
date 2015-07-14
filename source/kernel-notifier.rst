.. highlight:: rst

linux内核通知链
===============

概述
----

在Linux内核中，各个子系统之间有很强的相互关系，某些子系统可能对其它子系统产生的事件感兴趣。为了让某个子系统在发生某个事件时通知感兴趣的子系统，Linux内核引入了通知链技术。通知链只能够在内核的子系统之间使用，而不能够在内核和用户空间进行事件的通知。

组成内核的核心系统代码均位于kernel目录下，通知链表就位于其中，它位于 *kernel/notifier.c* 中，对应的头文件为 *include/linux/notifier.h* 。

从技术上来讲，这并不是一个多么复杂、高深、难懂的部分，说白了就是一个单向链表的插入、删除和遍历等操作。实现她的代码不超过1000行。

数据结构
--------

所有通知链的核心数据结构都位于notifier.h中。通知链的核心结构是notifier_block 。

	struct notifier_block {
	        notifier_fn_t notifier_call;
	        struct notifier_block __rcu *next;
	        int priority;
	};

其中 notifier_call 是通知链要执行的函数指针， next 用来连接其它的通知结构， priority 是这个通知的优先级，同一条链上的 notifier_block 是按优先级排列的,数字越大，优先级越高，越会被先执行。

内核代码中一般把通知链命名为xxx_chain, xxx_nofitier_chain这种形式的变量名。围绕核心数据结构notifier_block，内核定义了四种通知链类型，它们的主要区别就是在执行通知链上的回调函数时是否有安全保护措施：

1. 原子通知链（ Atomic notifier chains ）：原子通知链采用的自旋锁，通知链元素的回调函数（当事件发生时要执行的函数）在中断或原子操作上下文中运行，不允许阻塞。对应的链表头结构：

		struct atomic_notifier_head {
	        spinlock_t lock;
	        struct notifier_block __rcu *head;
		};

2. 可阻塞通知链（ Blocking notifier chains ）：可阻塞通知链使用信号量实现回调函数的加锁，通知链元素的回调函数在进程上下文中运行，允许阻塞。对应的链表头：

		struct blocking_notifier_head {
		        struct rw_semaphore rwsem;
		        struct notifier_block __rcu *head;
		};

3. 原始通知链（ Raw notifier chains ）：对通知链元素的回调函数没有任何限制，所有锁和保护机制都由调用者维护。对应的链表头：

		struct raw_notifier_head {
		        struct notifier_block __rcu *head;
		};

4. SRCU 通知链（ SRCU notifier chains ）：可阻塞通知链的一种变体，采用互斥锁和叫做“可睡眠的读拷贝更新机制”(Sleepable Read-Copy UpdateSleepable Read-Copy Update)。对应的链表头：

		struct srcu_notifier_head {
		        struct mutex mutex;
		        struct srcu_struct srcu;
		        struct notifier_block __rcu *head;
		};

如何使用通知链
--------------

这四类通知链，我们该怎么用这才是我需要关心的问题。在定义自己的通知链的时候，心里必须明确，自己需要一个什么样类型的通知链，是原子的、可阻塞的还是一个原始通知链。内核中用于定义并初始化不同类通知链的函数分别是：

	#define ATOMIC_NOTIFIER_HEAD(name)                              \
	        struct atomic_notifier_head name =                      \
	                ATOMIC_NOTIFIER_INIT(name)
	#define BLOCKING_NOTIFIER_HEAD(name)                            \
	        struct blocking_notifier_head name =                    \
	                BLOCKING_NOTIFIER_INIT(name)
	#define RAW_NOTIFIER_HEAD(name)                                 \
	        struct raw_notifier_head name =                         \
	                RAW_NOTIFIER_INIT(name)

其实， ATOMIC_NOTIFIER_HEAD(mynofifierlist)和下面的代码是等价的，展开之后如下：

	struct atomic_notifier_head mynofifierlist = 
	{
		.lock = __SPIN_LOCK_UNLOCKED(mynofifierlist.lock),
		.head = NULL
	}

另外两个接口也类似。如果我们已经有一个通知链的对象，Linux还提供了一组用于初始化一个通知链对象的API：
	
	#define ATOMIC_INIT_NOTIFIER_HEAD(name) do {    \
	                spin_lock_init(&(name)->lock);  \
	                (name)->head = NULL;            \
	        } while (0)
	#define BLOCKING_INIT_NOTIFIER_HEAD(name) do {  \
	                init_rwsem(&(name)->rwsem);     \
	                (name)->head = NULL;            \
	        } while (0)
	#define RAW_INIT_NOTIFIER_HEAD(name) do {       \
	                (name)->head = NULL;            \
	        } while (0)

这一组接口一般在下列格式的代码里见到的会比较多一点：

	static struct atomic_notifier_head dock_notifier_list;
	ATOMIC_INIT_NOTIFIER_HEAD(&dock_notifier_list);

有了通知链只是第一步，接下来我们还需要提供往通知链上注册通知块、卸载通知块、已经遍历执行通知链上每个通知块里回调函数的基本接口，说白了就是单向链表的插入、删除和遍历，这样理解就可以了。

内核提供最基本的通知链的常用接口为如下：

	static int notifier_chain_register(struct notifier_block **nl,
	                struct notifier_block *n)
	static int notifier_chain_cond_register(struct notifier_block **nl,
	                struct notifier_block *n)
	static int notifier_chain_unregister(struct notifier_block **nl,
	                struct notifier_block *n)
	static int notifier_call_chain(struct notifier_block **nl,
	                               unsigned long val, void *v,
	                               int nr_to_call, int *nr_calls)

这最基本的三个接口分别实现了对通知链上通知块的注册、卸载和遍历操作，可以想象，原子通知链、可阻塞通知链和原始通知链一定会对基本通知链的操作函数再进行一次包装的，事实也确实如此：

	// 注册函数
	extern int atomic_notifier_chain_register(struct atomic_notifier_head *nh,
	                struct notifier_block *nb);
	extern int blocking_notifier_chain_register(struct blocking_notifier_head *nh,
	                struct notifier_block *nb);
	extern int raw_notifier_chain_register(struct raw_notifier_head *nh,
	                struct notifier_block *nb);
	extern int srcu_notifier_chain_register(struct srcu_notifier_head *nh,
	                struct notifier_block *nb);
	
	extern int blocking_notifier_chain_cond_register(
	                struct blocking_notifier_head *nh,
	                struct notifier_block *nb);
	
	// 卸载函数
	extern int atomic_notifier_chain_unregister(struct atomic_notifier_head *nh,
	                struct notifier_block *nb);
	extern int blocking_notifier_chain_unregister(struct blocking_notifier_head *nh,
	                struct notifier_block *nb);
	extern int raw_notifier_chain_unregister(struct raw_notifier_head *nh,
	                struct notifier_block *nb);
	extern int srcu_notifier_chain_unregister(struct srcu_notifier_head *nh,
	                struct notifier_block *nb);
	
	// 遍历操作
	extern int atomic_notifier_call_chain(struct atomic_notifier_head *nh,
	                unsigned long val, void *v);
	extern int __atomic_notifier_call_chain(struct atomic_notifier_head *nh,
	        unsigned long val, void *v, int nr_to_call, int *nr_calls);
	extern int blocking_notifier_call_chain(struct blocking_notifier_head *nh,
	                unsigned long val, void *v);
	extern int __blocking_notifier_call_chain(struct blocking_notifier_head *nh,
	        unsigned long val, void *v, int nr_to_call, int *nr_calls);
	extern int raw_notifier_call_chain(struct raw_notifier_head *nh,
	                unsigned long val, void *v); 
	extern int __raw_notifier_call_chain(struct raw_notifier_head *nh,
	        unsigned long val, void *v, int nr_to_call, int *nr_calls);
	extern int srcu_notifier_call_chain(struct srcu_notifier_head *nh,
	                unsigned long val, void *v);
	extern int __srcu_notifier_call_chain(struct srcu_notifier_head *nh,
	        unsigned long val, void *v, int nr_to_call, int *nr_calls);

上述这四类通知链的基本API又构成了内核中其他子系统定义、操作自己通知链的基础。例如，Netlink定义了一个原子通知链，所以，它对原子通知链的基本API又封装了一层，以形成自己的特色：

	static ATOMIC_NOTIFIER_HEAD(netlink_chain);

	int netlink_register_notifier(struct notifier_block *nb) 
	{
	        return atomic_notifier_chain_register(&netlink_chain, nb); 
	}
	EXPORT_SYMBOL(netlink_register_notifier);
	
	int netlink_unregister_notifier(struct notifier_block *nb) 
	{
	        return atomic_notifier_chain_unregister(&netlink_chain, nb); 
	}
	EXPORT_SYMBOL(netlink_unregister_notifier);

网络事件也有一个原子通知链（net/core/netevent.c）：

	/*   
	 *      Network event notifiers
	 *
	 *      Authors:
	 *      Tom Tucker             <tom@opengridcomputing.com>
	 *      Steve Wise             <swise@opengridcomputing.com>
	 *
	 *      This program is free software; you can redistribute it and/or
	 *      modify it under the terms of the GNU General Public License
	 *      as published by the Free Software Foundation; either version
	 *      2 of the License, or (at your option) any later version.
	 *
	 *      Fixes:
	 */
	
	#include <linux/rtnetlink.h>
	#include <linux/notifier.h>
	#include <linux/export.h>
	#include <net/netevent.h>

	static ATOMIC_NOTIFIER_HEAD(netevent_notif_chain);
	
	/**
	 *      register_netevent_notifier - register a netevent notifier block
	 *      @nb: notifier
	 *
	 *      Register a notifier to be called when a netevent occurs.
	 *      The notifier passed is linked into the kernel structures and must
	 *      not be reused until it has been unregistered. A negative errno code
	 *      is returned on a failure.
	 */
	int register_netevent_notifier(struct notifier_block *nb)
	{
	        int err;
	
	        err = atomic_notifier_chain_register(&netevent_notif_chain, nb);
	        return err;
	}
	EXPORT_SYMBOL_GPL(register_netevent_notifier);
	
	/**
	 *      netevent_unregister_notifier - unregister a netevent notifier block
	 *      @nb: notifier
	 *
	 *      Unregister a notifier previously registered by
	 *      register_neigh_notifier(). The notifier is unlinked into the
	 *      kernel structures and may then be reused. A negative errno code
	 *      is returned on a failure.
	 */
	
	int unregister_netevent_notifier(struct notifier_block *nb)
	{
	        return atomic_notifier_chain_unregister(&netevent_notif_chain, nb);
	}
	EXPORT_SYMBOL_GPL(unregister_netevent_notifier);
	
	/**
	 *      call_netevent_notifiers - call all netevent notifier blocks
	 *      @val: value passed unmodified to notifier function
	 *      @v:   pointer passed unmodified to notifier function
	 *
	 *      Call all neighbour notifier blocks.  Parameters and return value
	 *      are as for notifier_call_chain().
	 */
	
	int call_netevent_notifiers(unsigned long val, void *v)
	{
	        return atomic_notifier_call_chain(&netevent_notif_chain, val, v);
	}
	EXPORT_SYMBOL_GPL(call_netevent_notifiers); 

运作机制
--------


通知链的运作机制包括两个角色：

* **被通知者**：对某一事件感兴趣一方。定义了当事件发生时，相应的处理函数，即回调函数，被通知者将其注册到通知链中（被通知者注册的动作就是在通知链中增加一项）。
* **通知者**：事件的通知者。当检测到某事件，或者本身产生事件时，通知所有对该事件感兴趣的一方事件发生。它定义了一个通知链，其中保存了每一个被通知者对事件的回调函数。通知这个过程实际上就是遍历通知链中的每一项，然后调用相应的回调函数。


包括以下过程：

* 通知者定义通知链。
* 被通知者向通知链中注册回调函数。
* 当事件发生时，通知者发出通知（执行通知链中所有元素的回调函数）。

其他注意事项
------------

1. 如果一个子系统A在运行过程中会产生一个实时事件，而这些事件对其他子系统来说非常重要，那么子系统A可以定义一个自己的通知链对象，根据需求可以选择原子通知链、非阻塞通知链和原始通知链，并向外提供向这个通知链里注册、卸载、执行事件的回调函数的接口。

2. 如果子系统B对子系统A中的某些事件感兴趣，或者说强依赖，就是说子系统B需要根据子系统A中某些事件来执行自己特定的操作，那么此时系统B需要实例化一个通知块`struct notifier_block xxx{}`，然后编写通知块里的回调处理函数来相应系统A中的事件就可以了。

3. 通知块`struct notifier_block xxx{}`里有一个优先级的特性，起始在标准内核里每个实例化的通知块都没有使用优先级。不用优先级字段的结果就是：先注册的通知块里的回调函数在事件发生时会先执行。注意这里说的后注册指的是模块被动态加载到内核的先后顺序，和哪个模块代码先写没有关系。

	注意区分。意思就是说，如果子系统B和C都对子系统A的up事件感兴趣，B和C在向A注册up事件的回调函数时并没有指定函数的优先级。无论是通过`insmod`手动加载模块B和C，还是系统`boot`时自动加载B和C，哪个模块先被加载，它的回调函数在A系统的up事件发生时会先被执行

4. 关于通知链的回调函数，正常情况下都需要返回`NOTIFY_OK`或者`NOTIFY_DONE`，这样通知链上后面挂载的其他函数可以继续执行。如果返回`NOTIFY_STOP`，则会使得通知链上后续挂载的函数无法得到执行，除非特别想这么做，否则编写通知链回调函数时，最好不要返回这个值。

5. 通知链上的回调函数的原型为
	
			typedef int (*notifier_fn_t)(struct notifier_block *nb,
		                        unsigned long action, void *data);
		
			struct notifier_block {
			        notifier_fn_t notifier_call;   
					struct notifier_block __rcu *next;
			        int priority;
			};

	其中第二个参数一般用于指明事件的类型。通知都是一个整数；而第三个参数是一个`void`类型的内存地址，在不同的子系统中表示不同的信息。我们在设计自己的通知链系统可以用第三个入参实现在通知系统和被通知系统之间数据的传递，以便被通知系统的工作可以更加紧凑、高效。

6.如果以后在看到内核代码中某个子系统在调用通知链注册函数时，做到以下几点就没事了：

* 心里首先要明确，这个注册通知链回调函数的系统一定和提供通知链的系统有某种联系，且本系统需要那个系统对某些重要事件进行响应。
* 看本系统注册的通知链回调函数的实现，具体看它对哪些事件感兴趣，并且是怎么处理的。
* 看看提供通知链对象的系统有哪些事件；

最后，也就明白了这个子系统为什么要用通知链来感知别的系统的变化了，这样一来，对这两个子系统从宏观到微观的层面上都有一个总体的认识和把握，后续研究起来就顺风顺水了。





