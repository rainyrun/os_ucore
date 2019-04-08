# lab4_report
## 练习1:完成proc_struct结构的初始化
```
static struct proc_struct *
alloc_proc(void) {
    struct proc_struct *proc = kmalloc(sizeof(struct proc_struct));
    if (proc != NULL) {
      proc->state = PROC_UNINIT;
      proc->pid = -1;
      proc->runs = 0;
      proc->kstack = 0;
      proc->need_resched = 0;
      proc->parent = NULL;
      proc->mm = NULL;
      memset(&(proc->context), 0, sizeof(struct context));
      proc->tf = NULL;
      proc->cr3 = boot_cr3;
      proc->flags = 0;
      memset(proc->name, 0, PROC_NAME_LEN);
    }
    return proc;
}
```

问：请说明proc_struct中 struct context context 和 struct trapframe *tf 成员变量含义和在本实验中的作用是啥?  
答：context保持程序切换时的上下文，tf保存程序被中断时的现场。

## 练习2:完成do_fork函数
```
int
do_fork(uint32_t clone_flags, uintptr_t stack, struct trapframe *tf) {
    int ret = -E_NO_FREE_PROC;
    struct proc_struct *proc;
    if (nr_process >= MAX_PROCESS) {
        goto fork_out;
    }
    ret = -E_NO_MEM;

    proc = alloc_proc();//分配进程控制块

    if(setup_kstack(proc) != 0){//分配线程栈
      cprintf("setup_kstack failed.\n");
      goto bad_fork_cleanup_proc;
    }

    if(copy_mm(clone_flags, proc) != 0){//复制or共享父进程的内存管理信息
      cprintf("copy_mm failed.\n");
      goto bad_fork_cleanup_kstack;
    }

    copy_thread(proc, stack, tf);//复制上下文context和tf

    proc->pid = get_pid();//获取进程号

    hash_proc(proc);//插入hash_list

    list_entry_t *le = &proc_list;
    struct proc_struct *p = NULL;
    while((le = list_next(le)) != &proc_list){//在proc_list中寻找合适的插入位置（按pid升序排）
      p = le2proc(le, list_link);
      if(proc->pid < p->pid)
        break;
    }
    list_add_before(le, &(proc->list_link));//插入proc_list

    wakeup_proc(proc);//设置程序状态为可执行
    ret = proc->pid;//返回新程序的pid

fork_out:
    return ret;

bad_fork_cleanup_kstack:
    put_kstack(proc);
bad_fork_cleanup_proc:
    kfree(proc);
    goto fork_out;
}
```
问：请说明ucore是否做到给每个新fork的线程一个唯一的id？  
答：可以，通过get_pid()函数可获得唯一的id。因proc_list中的程序控制块，按pid升序排序。get_pid函数从序号[1, MAX_PID - 1]到pid从左到右扫描proc_list列表，找到未被使用的pid进行分配。get_pid()函数中的last_pid指向了预计要分配的pid。当last_pid确定后，从last_pid到next_safe之间的pid是未分配的pid。

## 练习3：对proc_run函数进行分析  

问：在本实验的执行过程中，创建且运行了几个内核线程?  
答：2个：idleproc，initproc

问：语句 local_intr_save(intr_flag);....local_intr_restore(intr_flag); 在这里有何作用?请说明理由  
答：分别是开中断和关中断，保证在更新进程控制块的信息时，避免中断嵌套而导致数据错误