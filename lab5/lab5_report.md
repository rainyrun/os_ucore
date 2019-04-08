# lab5_report
## 练习1: 加载应用程序并执行
### 设置trapframe中的内容
答：因为程序刚被创建，trapframe中的内容实际是程序最初的状态，即.  
```
tf->tf_cs = USER_CS;
tf->tf_ds = tf->tf_es = tf->tf_ss = USER_DS;
tf->tf_esp = USTACKTOP;
tf->tf_eip = elf->e_entry;//程序入口
tf->tf_eflags = FL_IF;//使能中断
```
### 用户态进程被ucore选择占用CPU执行(RUNNING态)到具体执行应用程序第一条指令的整个经过
创建用户进程：  
1. proc_init创建idle内核线程，之后再创建init内核线程(运行init_main)
2. 通过cpu_idle --> schedule函数 --> proc_run函数调用init内核线程(init_main)。init内核线程运行init_main函数，创建了内核线程user_main
3. 经过schedule函数后，运行user_main函数，默认调用KERNEL_EXECVE(exit)-->kernel_execve函数。该函数传递参数后，产生T_SYSCALL中断。
4. T_SYSCALL中断调用syscall()函数，根据传递的参数-->sys_exec函数-->do_execve函数，将exit函数的内容加载到对应的用户内存空间，使user_main内核线程变成了用户进程。当中断返回后，将从exit函数开始执行。
5. exit创建了一个子进程

idle-->init_main-->do_wait
				-->user_main-->kernel_execve-->int T_syscall-->syscall-->sys_exec-->do_execve-->exit

## 练习2: 父进程复制自己的内存空间给子进程
### 补充copy_range的实现
补充的代码如下：
```
uintptr_t src_kvaddr = page2kva(page);//A程序该页的虚拟地址
uintptr_t dst_kvaddr = page2kva(npage);//B程序该页的虚拟地址
memcpy(dst_kvaddr, src_kvaddr, PGSIZE);//A页复制到B页
page_insert(to, npage, start, perm);//B页与线性地址建立映射
```
### 简要说明如何设计实现“Copy on Write 机制”


## 练习3：阅读分析源代码
问：请分析fork/exec/wait/exit在实现中是如何影响进程的执行状态的?   
答：

请给出ucore中一个用户态进程的执行状态生命周期图(包执行状态，执行状态之间的变换关系，以及产生变换的事件或函数调用。(字符方式画即可)。 




# 附录
疑问解答
问：子程序如何从“复制的父程序”变为“全新的子程序”？

思考
用户态进程
把虚拟地址分类，用户态程序使用"用户态虚拟内存"空间，内核态程序使用“内核态虚拟内存”空间。通过一个程序的虚拟地址，能够判断这是一个运行在内核态还是用户态的程序？
为什么要这么划分？判断内核态还是用户态，不是可以根据cs的优先级域吗？
怎样区分内核态和用户态，根据使用的虚拟地址吗？如果是的话，虚拟地址对应的实际物理地址是可以改变的，通过改变页表的映射关系。那么能保证用户态虚拟内存，就一定不能访问内核态虚拟内存使用的物理内存了吗？是因为物理内存一旦分配给内核，就不会被再次分配了吗？还是说，内核虚拟内存建立的页表中对应的物理内存，就是内核。用户态虚拟内存建立的页表中对应的物理内存，就是用户态。物理内存反而不重要了，只是一种有限的资源，供内核态和用户态使用。

进程管理
进程控制块中，内存管理相关部分：页表，可访问空间，进程状态

内存管理
增加用户态虚拟内存的管理，对页表内容进行扩展（？）

idle(k) --> init_main(k) --> user_main(k) --> user_main(u)

内核态到用户态的切换：将内核线程 复制到 用户空间 吗？

用户进程虽然不能访问内核虚拟内存，但是有内核虚拟内存的布局？
用户进程，从用户态进入核心态时，如何使用内核程序？

问：user_mem_check检查的目的是什么？
答：检查地址、权限等是否合法，通过检查返回1

问：do_execve释放了父进程的mm，父进程的mm保存了吗？父进程自己运行不需要吗？为什么要mm_count_dec呢？

问：load_icode函数中memcpy(page2kva(page) + off, from, size)是说明，用户进程的页表，程序代码是保存在内核空间吗？
答：不是。ph->va制定了程序被复制到的虚拟地址，属于用户空间。memcpy函数中的参数page2kva(page) + off只是指明了被复制到的页的地址。使用内核虚拟地址，是因为在内核中进行的复制？实际该页与用户空间相互映射。这说明了使用不同的映射方式，可以找到同一个物理页。

问：(process/proc.c)user_main函数中的TEST是什么，在哪儿定义，作用是什么？
答：

问：(process/proc.c)宏KERNEL_EXECVE(x)中_binary_obj___user_是什么？
答：


问：如果内核需要空间的时候，发现对应的物理页已被分配出去，该如何处理？

问：物理内存并不是完全连续的，但内核空间的线性地址和物理地址是一一映射的，那是不是意味内核空间有部分线性地址也是不能使用的？

问：hello程序在哪儿运行？有没有运行？
答：在user内，默认运行的程序是exit。

问：init_main的父亲是idle吗？在哪建立父子关系？
答：是，在kernel_thread调用do_fork时，set_links设置了父子关系。

问：kernel_thread只是复制了父进程，子进程如何执行自己的程序呢？
答：内核子进程的tf中的reg_ebx存储了程序地址，reg_edx存储了程序参数等。且do_fork中的copy_tread已存储了子程序的上下文，当切换到子程序时，会从上下文恢复运行环境。到时候就能运行子程序的程序了。

