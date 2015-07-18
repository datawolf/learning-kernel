.. highlight:: rst

BootLoader
===========

uboot header 分析
------------------

一般情况下，可以使用如下命令对一个内核添加uboot头 ::

        mkimage -A arm -O linux -T kernel -C none -a 80008000 -e 80008000 \
         -n linux-3.13.0  -d zImage  uImage

其实就是向zImage的前面添加了一个 **64字节** 的uboot头 ::

        wanglong@wanglong-Lenovo-Product:~/kernel-dev/linux-3.10$ xxd < uImage | more
        0000000: 2705 1956| 533a 4b45| 536c ec37| 0019 d072  '..VS:KESl.7...r
        0000010: 8000 8000| 8000 8000| 555c 7591 |05| 02| 02| 00  ........U\u.....
        0000020: 6c69 6e75 782d 332e 3133 2e30 0000 0000  linux-3.13.0....
        0000030: 0000 0000 0000 0000 0000 0000 0000 0000  ................

uboot头的结构体定义如下
^^^^^^^^^^^^^^^^^^^^^^^

.. code-block:: c

        typedef struct image_header {
                uint32_t        ih_magic;       /* Image Header Magic Number    */
                uint32_t        ih_hcrc;        /* Image Header CRC Checksum    */
                uint32_t        ih_time;        /* Image Creation Timestamp     */
                uint32_t        ih_size;        /* Image Data Size              */
                uint32_t        ih_load;        /* Data  Load  Address          */
                uint32_t        ih_ep;          /* Entry Point Address          */
                uint32_t        ih_dcrc;        /* Image Data CRC Checksum      */           
                uint8_t         ih_os;          /* Operating System             */
                uint8_t         ih_arch;        /* CPU architecture             */
                uint8_t         ih_type;        /* Image Type                   */
                uint8_t         ih_comp;        /* Compression Type             */
                uint8_t         ih_name[IH_NMLEN];      /* Image Name           */
        } image_header_t;


针对arm linux的具体点宏定义如下
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

.. code-block:: c

        #define IH_MAGIC        0x27051956      /* Image Magic Number           */
        #define IH_OS_LINUX             5       /* Linux        */
        #define IH_ARCH_ARM             2       /* ARM          */
        #define IH_TYPE_KERNEL          2       /* OS Kernel Image              */ 
        #define IH_COMP_NONE            0       /*  No   Compression Used       */


uboot头中的压缩类型字段是针对uboot头后面的内容，与内核中的压缩方式没有关系。

**注意：** ``mkimage`` 只是简单地给数据文件添加一个uboot头，它不会进行任何压缩处理。
