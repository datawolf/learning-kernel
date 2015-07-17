.. highlight:: rst

C编程技巧
=========

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


