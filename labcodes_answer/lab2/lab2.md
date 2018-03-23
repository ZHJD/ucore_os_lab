# 实验二：物理内存管理

## 练习一

### default_init_memmap的实现

* 对提供的内存块进行初始化操作
* 将代表内存块第一个帧的`struct Page`插入合适位置
    * 若链表为空，则直接加入链表
    * 否则从前往后搜索
        * 若链表中当前块的地址比较高，则插入到它的前方
        * 若已到链表结尾(表头前的一个`page_link`)，则在该位置插入

按这种方式初始化可以保证内存块按起始地址从低到高在链表中顺序排列

### default_alloc_pages的实现

* 若申请空间大于可用总空间，直接返回NULL
* 从表头开始向后搜索，知道再次遇到表头位置
* 对每块内存，检查可用空间
    * 若有可用空间，则从链表中删除这块内存
        * 若可用空间大于申请空间，则从末端切除多余的空间放回链表中原来的位置
    * 若无可用空间，继续搜索下一块内存
* 返回page指针

这种分配方式符合"first fit"的要求

### default_free_pages实现

* 用default_init_memmap中的方式将返还的Page插入链表
* 检查前方的内存块是否可以和该内存块合并，若能则进行合并，并将指针指向合并后的Page
* 检查后方的内存块是否可以和该内存块合并，若能则进行合并

注意插入后最多是三块内存合并，若有超过三块内存需要合并，说明之前有连续的内存块未进行合并，在正确
初始化后这是不可能发生的

### 改进空间

内存块可以改用线段树管理，这样可以保证在log(n)时间内找到满足要求的空闲内存块

## 练习二

### get_pte函数的实现

* 将页目录表的逻辑地址加上线性地址前十位得到页目录项的逻辑地址
* 检查页目录项中的Present Bit
    * 若该bit为1
        * 则将页目录项的后12项置0，得到页表所在页帧的物理地址
        * 将页帧物理地址加上线性地址中间10位得到页表的物理地址
        * 用KADDR将物理地址转换为逻辑地址
    * 若该bit为0，且参数create为true
        * 用alloc_page为页表分配一个页帧，对`struct Page`进行设置
        * 令页目录项等于页帧的物理地址，然后设置权限位
        * 用page2kva宏得到页的逻辑地址
* 返回页目录项的逻辑地址

### 目录项和页表项的含义

页目录项结构和页表项一致，页表项的结构如下

```
       31                                  12 11                      0
      +--------------------------------------+-------+---+-+-+---+-+-+-+
      |                                      |       |   | | |   |U|R| |
      |      PAGE FRAME ADDRESS 31..12       | AVAIL |0 0|D|A|0 0|/|/|P|
      |                                      |       |   | | |   |S|W| |
      +--------------------------------------+-------+---+-+-+---+-+-+-+

                P      - PRESENT
                R/W    - READ/WRITE
                U/S    - USER/SUPERVISOR
                D      - DIRTY
                AVAIL  - AVAILABLE FOR SYSTEMS PROGRAMMER USE

                NOTE: 0 INDICATES INTEL RESERVED.
```

* D为Dirty Flag，代表该页是否被写过
* A为Accessed Flag，代表该页是否被访问(读/写)过
* U/S为'User/Supervisor' bit，代表该页的访问权限
* R/W为'Read/Write' permissions flag，代表读写权限
* P为'Present' bit，表示该页是否实际在物理内存中

低12位可以为系统提供有用的信息，如U/S就可用于对应用程序做访问权限控制

### 页访问异常

若发生页访问异常，则硬件应当引发一个`page fault`中断，交由中断处理例程解决(如将swap中的内存放
回物理内存等)

## 练习三

### page_remove_pte实现

* 若页目录项的Present Bit为1
    * 使用pte2page找到对应的`struct Page`
    * 将该页的引用数减1，若引用数为0，则释放该页
    * 将页表项置0
    * 使TLB无效

### Page与页表项的关系

每个Page表示一个4K的页帧，对应一个页表项

### 使逻辑地址等于物理地址

* 在链接内核时不加入0xC0000000的偏移即可实现逻辑地址等于物理地址
* 或者在分配页时对页表项内的基址加-0xC0000的偏移

## 总结

### 实现与参考答案的区别

参考答案中`default_init_memmap`的实现会将完整的大块内存拆分成4k的块，我的实现中会一次产生一整个内存块

### 知识点

* 分页机制：如下图，通过页目录索引和页表项索引找到相应的页表项，找到对应的页基址，加上偏移量得到物理地址

```
+--------10------+-------10-------+---------12----------+
| Page Directory |   Page Table   | Offset within Page  |
|      Index     |     Index      |                     |
+----------------+----------------+---------------------+
```

### 没有对应的知识点

ucore的逻辑地址放在0xC0000000上方，属于Higher Half Kernel的设计，好处有：

* 0~1MB空间可用，方便Virtual 8086 Mode下程序的执行
* 若系统是64位的，则32位程序能使用全部32位地址空间