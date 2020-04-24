# lab1

## 练习1：理解通过make生成执行文件的过程。（要求在报告中写出对下述问题的回答）

1. 操作系统镜像文件ucore.img是如何一步一步生成的？(需要比较详细地解释Makefile中每一条相关命令和命令参数的含义，以及说明命令导致的结果)

   * 编译

   ```makefile
   + cc kern/init/init.c
   gcc -Ikern/init/ -fno-builtin -fno-PIC -Wall -ggdb -m32 -gstabs -nostdinc  -fno-stack-protector -Ilibs/ -Ikern/debug/ -Ikern/driver/ -Ikern/trap/ -Ikern/mm/ -c kern/init/init.c -o obj/kern/init/init.o
   .....
   gcc -g -Wall -O2 obj/sign/tools/sign.o -o bin/sign
   ```

   首先通过`gcc`对`c`程序进行编译，下面对使用的参数进行说明

   ```makefile
   -I: 指定头文件目录
   -fno-builtin: 不使用c语言的内建函数
   -fno-PIC: 不使用位置无关代码
   -Wall: 启用所有的警告
   -ggdb: 尽可能的生成gdb所需要的信息
   -m32: 编译为32位程序
   -nostdinc: 编译器不再系统指目录查找头文件，使用-I指定的目录
   -fno-stack-protector:禁用堆栈保护
   ```

   * 链接

   之后使用链接器`ld`对编译产生的目标文件进行链接产生可执行elf程序，以下面指令为例子

   ```makefile
   ld -m    elf_i386 -nostdlib -N -e start -Ttext 0x7C00 obj/boot/bootasm.o obj/boot/bootmain.o -o obj/bootblock.o
   'obj/bootblock.out' size: 500 bytes
   
   ```

   下面将对使用的参数进行说明

   ```M
   -m: 创建32位的程序
   -e: 指定start为程序的入口
   -Ttext: 设置0x7c00为程序的起始地址，也就是start函数的起始地址，操作系统从这里开始执行
   -nostdlib: 只使用指定的库
   -N：关闭段的页对齐
   ```

   * 写入硬盘

   ```
   # 10000 个扇区全部清零
   dd if=/dev/zero of=bin/ucore.img count=10000
   10000+0 records in
   10000+0 records out
   5120000 bytes (5.1 MB, 4.9 MiB) copied, 0.0288719 s, 177 MB/s
   # 第一个扇区写入bootblock
   dd if=bin/bootblock of=bin/ucore.img conv=notrunc
   1+0 records in
   1+0 records out
   512 bytes copied, 0.000103618 s, 4.9 MB/s
   # 第二个扇区及以后写入kernel
   dd if=bin/kernel of=bin/ucore.img seek=1 conv=notrunc
   146+1 records in
   146+1 records out
   74868 bytes (75 kB, 73 KiB) copied, 0.000350067 s, 214 MB/s
   ```

2. 一个被系统认为是符合规范的硬盘主引导扇区的特征是什么？

   最后两个字节是两个魔数，分别为0x55和0xaa

## 练习2：使用qemu执行并调试lab1中的软件。（要求在报告中简要写出练习过程）
为了熟悉使用qemu和gdb进行的调试工作，我们进行如下的小练习：

1. 从CPU加电后执行的第一条指令开始，单步跟踪BIOS的执行。

   ![image-20200423141508902](C:\Users\zhj\AppData\Roaming\Typora\typora-user-images\image-20200423141508902.png) 

   途中显示了开机后pc的值和第一条指令，跳转指令，且处于关中断状态

   ![image-20200423143848396](C:\Users\zhj\AppData\Roaming\Typora\typora-user-images\image-20200423143848396.png) 

2. 在初始化位置0x7c00设置实地址断点,测试断点正常。

   ![image-20200423144306231](C:\Users\zhj\AppData\Roaming\Typora\typora-user-images\image-20200423144306231.png) 

   在gdb中通过`b *address`在程序中打断点的方式，使得程序在0x7c00处停下，此时处于开中断状态

   ![image-20200423144538876](C:\Users\zhj\AppData\Roaming\Typora\typora-user-images\image-20200423144538876.png) 

3. 从0x7c00开始跟踪代码运行,将单步跟踪反汇编得到的代码与bootasm.S和 bootblock.asm进行比较。

   反汇编的结果和原始代码相同。

4. 自己找一个bootloader或内核中的代码位置，设置断点并进行测试。

## 练习3：分析bootloader进入保护模式的过程。（要求在报告中写出分析） 

BIOS将通过读取硬盘主引导扇区到内存，并转跳到对应内存中的位置执行bootloader。请分析bootloader是如何完成从实模式进入保护模式的。

1. 使用键盘控制器8042打开a20
2. 设置三个gdt描述符，其中第零个不可用，第二个为可读可执行的代码段，第三个为可写的数据段，采用了"平坦模式"的设置方法。即base为0，limit为最大值。
3. 使用lgdt指令加载内存中gdt段描述符表首地址和长度到GDTR寄存器
4. 打开cr0中的PE位
5. 使用长跳转指令刷新流水线，更新cs, eip寄存器，进入32位保护模式
6. 设置ds、es、fs、gs、ss为数据段选择子的值
7. 更新ebp为0， esp为0x7c00用于堆栈后跳转到bootmain函数
8. bootmain中复制硬盘上的kernel的elf文件到内存`0x10000` 处，并进行解析
9. 跳转到内核处执行，内核的entry point 为 `0x100000` 

## 练习4：分析bootloader加载ELF格式的OS的过程。（要求在报告中写出分析）

通过阅读bootmain.c，了解bootloader如何加载ELF文件。通过分析源代码和通过qemu来运行并调试bootloader&OS，

- bootloader如何读取硬盘扇区的？

  1. bootloader中使用LBA编址方式，通过cpu忙等测试硬盘端口状态，读取硬盘中的扇区。

- bootloader是如何加载ELF格式的OS？

  首先读取了8个扇区，这个是kernel的elf文件，通过解析elf文件，把elf中的三个段的内容复制到其对应的地址处，之后跳转并执行。

## 扩展练习 Challenge 1（需要编程）

扩展proj4,增加syscall功能，即增加一用户态函数（可执行一特定系统调用：获得时钟计数值），当内核初始完毕后，可从内核态返回到用户态的函数，而用户态的函数又通过系统调用得到内核态的服务（通过网络查询所需信息，可找老师咨询。如果完成，且有兴趣做代替考试的实验，可找老师商量）。需写出详细的设计和分析报告。完成出色的可获得适当加分。

提示： 规范一下 challenge 的流程。

kern_init 调用 switch_test，该函数如下：

```
    static void
    switch_test(void) {
        print_cur_status();          // print 当前 cs/ss/ds 等寄存器状态
        cprintf("+++ switch to  user  mode +++\n");
        switch_to_user();            // switch to user mode
        print_cur_status();
        cprintf("+++ switch to kernel mode +++\n");
        switch_to_kernel();         // switch to kernel mode
        print_cur_status();
    }
```

switch*to** 函数建议通过 中断处理的方式实现。主要要完成的代码是在 trap 里面处理 T_SWITCH_TO* 中断，并设置好返回的状态。

在 lab1 里面完成代码以后，执行 make grade 应该能够评测结果是否正确。

```c
static void
lab1_switch_to_user(void) {
    //LAB1 CHALLENGE 1 : TODO
    asm volatile (
        "movl %%esp, %%eax \n\t"    // 保存栈顶地址
	    "pushl $0 \n\t"				// 构造ss寄存器值，在中断处理程序中进行修改
	    "pushl %%eax \n\t"			// 构造esp的值，中断返回时cpu自动加载到esp中
	    "int $120 \n\t"
	    :
	    :
	    : "eax"
	);
}
```

在切换到user模式之前，系统处于内核态，因此我通过`int 0x120` 中断内核，由于中断返回后，还要继续执行`witch_to_user` 后面的代码，因此不能切换栈。故在中断之前 在`eax` 寄存器中保存当前栈顶地址。因为中断前位于内核态，所以`ss` 和`esp` 不会被入栈，但是通过修改cpu自动保存的中断现场，使得中断返回到用户态，这是由于特权级变化，cpu会弹出栈中的`ss`和`esp`,故需要提前构造数据。

```c
    case T_SWITCH_TOU:
        tf->tf_cs = USER_CS;
        tf->tf_ds = USER_DS;
        tf->tf_es = USER_DS;
        tf->tf_fs = USER_DS;
        tf->tf_gs = USER_DS;
        tf->tf_ss = USER_DS;
        //tf->tf_regs.reg_ebp = 0;
        tf->tf_eflags = (FL_IOPL_0 | FL_IOPL_MASK | FL_IF);
        //tf->tf_esp = tf->tf_regs.reg_oesp;
        break;
```

段寄存器全部设为用户态，并对`eflags`特权级进行设置。中断返回后便到了用户态。

下面来谈谈用户态到内核态的转换。

```c
static void
lab1_switch_to_kernel(void) {
    //LAB1 CHALLENGE 1 :  TODO
    asm volatile (
 		"movl %%esp, %%ebx \n\t" // 保存返回的栈顶到ebx中
        "movl $121, %%eax \n\t"	 // 传参数
        "int $0x80 \n\t"		 // 系统调用
		"movl %%ebx, %%esp \n\t" // 恢复栈顶
        :
        :
        : "eax", "ebx"
    );
}
```

在用户态唯一能使用的就是`0x80`系统调用，但是用户态向内核态转化时，cpu会自动的切换到tss中保存的`ss0` 栈，并且中断处理程序是内核态，中断退出后也是内核态，这意味着中断退出的过程中不会切换栈，也就是说如果不做处理中断退出后的栈和中断处理程序使用的是同一个栈，与当前用户态的栈不同，因此需要进行处理。我的做法是在进入中断前保存当前栈顶作为参数，中断退出后手动换栈。`eax` 做为子调用号。

```c
    case T_SWITCH_TOK:
        tf->tf_cs = KERNEL_CS;
        tf->tf_eip = tf->tf_regs.reg_eax;
        tf->tf_ds = KERNEL_DS;
        tf->tf_es = KERNEL_DS;
        tf->tf_fs = KERNEL_DS;
        tf->tf_ss = KERNEL_DS;
        //tf->tf_regs.reg_ebp = 0;
        tf->tf_eflags = (FL_IOPL_3 | FL_IOPL_MASK | FL_IF);
        break;
    case T_SYSCALL:
        if(tf->tf_regs.reg_eax == T_SWITCH_TOK) {
            asm volatile (
                "movl %0, %%eax \n\t"
				"movl %1, %%ebx \n\t"
                "int $121 \n\t"
                :
                : "r"(tf->tf_eip), "r"(tf->tf_regs.reg_ebx)
                : "eax","ebx"
                );
        }
        break;
```

进入中断后，进入了系统调用处理程序。根据参数eax的值进入第一个分支，在这里首先需要保存`eip` 的值作为参数，此处`eip`中的值是`int $0x80` 下一条指令的地址。这个系统调用不返回。为“真正的”中断退出做准备，之后再保存之前的`ebx` ，进行真正的中断。中断嵌套。这时进入到`T_SWITCH_TOK` 分支，在这里设置`eip` 的值，返回到内核态。