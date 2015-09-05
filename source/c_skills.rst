.. highlight:: rst

C编程技巧
=========


__builtin_return_address和_RET_IP
----------------------------------
GCC内建函数 ``__builtin_return_address`` 返回当前函数或者调用者的返回地址，参数 ``LEVEL`` 指定在栈上搜索框的个数。0表示
当前函数的返回地址，1表示当前函数的调用者的返回地址，依此类推。例如：

.. code-block:: c

        struct vm_struct *__get_vm_area(unsigned long size, unsigned long flags,
                                unsigned long start, unsigned long end)
        {
                return __get_vm_area_node(size, 1, flags, start, end, NUMA_NO_NODE,
                                          GFP_KERNEL, __builtin_return_address(0));
        }
        EXPORT_SYMBOL_GPL(__get_vm_area);

为了简便，内核中将 ``__builtin_return_address(0)`` 进行了封装：

.. code-block:: c

        #define  _RET_IP   (unsigned long)__builtin_return_address(0)

所以，内核中也存在如下的用法：

.. code-block:: c

 609 static int __init_memblock memblock_add_region(phys_addr_t base,
 610                                                 phys_addr_t size,
 611                                                 int nid,
 612                                                 unsigned long flags)
 613 {
 614         struct memblock_type *_rgn = &memblock.memory;
 615 
 616         memblock_dbg("memblock_add: [%#016llx-%#016llx] flags %#02lx %pF\n",
 617                      (unsigned long long)base,
 618                      (unsigned long long)base + size - 1,
 619                      flags, (void *)_RET_IP_);
 620 
 621         return memblock_add_range(_rgn, base, size, nid, flags);
 622 }


代码中类似typeof((foo) + 1) 是什么意思呢？
------------------------------------------

在内核的代码中，有如下的代码（文件：include/linux/kfifo.h）：

.. code-block::  c

        /**
         * kfifo_len - returns the number of used elements in the fifo
         * @fifo: address of the fifo to be used
         */
        #define kfifo_len(fifo) \
        ({ \
                typeof((fifo) + 1) __tmpl = (fifo); \
                __tmpl->kfifo.in - __tmpl->kfifo.out; \
        })
        
        /**
         * kfifo_is_empty - returns true if the fifo is empty
         * @fifo: address of the fifo to be used
         */
        #define kfifo_is_empty(fifo) \
        ({ \
                typeof((fifo) + 1) __tmpq = (fifo); \
                __tmpq->kfifo.in == __tmpq->kfifo.out; \
        })
        
        /**
         * kfifo_is_full - returns true if the fifo is full
         * @fifo: address of the fifo to be used
         */
        #define kfifo_is_full(fifo) \
        ({ \
                typeof((fifo) + 1) __tmpq = (fifo); \
                kfifo_len(__tmpq) > __tmpq->kfifo.mask; \
        })
        

从上面的代码中， ``typeof((fifo) + 1)``  是什么意思？为什么不适用 ``typeof(fifo) __tmpq = (fifo)；`` ?


通过google，原因如下：

kfifo_is_empty and other kfifo related macros needs the 1st argument as pointer to kfifo struct. if users of this macro accidentally used plain kfifo struct to 1st argument, this will make compilation error because of +1 in typeof.

This construct is used to check if a pointer was used as argument. When a plain struct is given, the expression will raise a compiler error because a struct is used in a binary + with an int. When a pointer is given, the binary + will add 1 to it which will still be a pointer to the same type and the expression is syntactically right.


总结
^^^^

这样使用的的原因主要是：防止用户给宏传递了一个错误的参数。该宏期望传递一个kfifo结构体的指针，如果用户传递不是结构体指针，而是结构体，就会导致编译出错。

参考文档
^^^^^^^^^


- http://stackoverflow.com/questions/16196440/what-is-typeoffifo-1-means-from-linux-kfifo-h-file
- http://gcc.gnu.org/onlinedocs/gcc/Typeof.html
- http://stackoverflow.com/questions/4436889/what-is-typeofc-1-in-c



volatile关键字的用法
--------------------


理解 ``volatile`` 的关键是知道它的目的是用来消除优化。

在内核中，为了防止意外的并发访问破坏共享的数据结构，在这种情况下，如果可以正确的使用内核原语（自旋锁、互斥量、内存屏障等等），那么就没有必要再使用 ``volatile`` 。

在内核中，有一些稀少的情况下应该使用 ``volatile`` :

1. 在一些体系架构的系统上，在允许直接的IO内存访问，那么访问函数可以使用 ``volatile`` 。
2. 某些会改变内存的内联汇编代码：有些改变内存的内联汇编代码，可能会被GCC删除，在汇编声明中加上 ``volatile`` 关键字可以防止这种删除操作。

.. code-block:: c

        asm volatile ("mrc p15, 1, %0, c15, c0, 0\n" : "=r" (ctrl)); 
        asm volatile ("mcr p15, 1, %0, c15, c0, 0\nisb\n" : : "r" (ctrl));

3. ``jiffies`` 变量是一种特殊情况。

.. code-block:: c

        #ifdef CONFIG_X86_64
        __visible volatile unsigned long jiffies __cacheline_aligned = INITIAL_JIFFIES;
        #endif

4. 由于某些IO设备可能会修改连续一致的内存，所以指向连续一致的内存的数据结构的指针需要正确的使用 ``volatile`` 。

.. code-block:: c

         volatile void __iomem *sysctl_base;

