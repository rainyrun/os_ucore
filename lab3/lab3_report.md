lab3实验报告
练习1：给未被映射的地址映射上物理页
一、需求分析
1.程序需要对自己的虚拟地址空间进行管理，以判断某地址是否在自己的空间内，访问的权限是否足够。
2.程序可能有多个连续的虚拟地址空间，每个空间的属性不同。需要一个数据结构vma进行管理。管理内容有：起始地址、结束地址、空间属性等。
3.某地址，如果不越界，不越权，且对应的页不在内存，则需要把该页加载到内存，并建立映射关系。

二、概要设计
1.mm_struct的抽象数据类型定义
ADT mm_struct{
	数据对象：mm_struct
	数据关系：双向链表，以起始地址递增排序
	基本操作：
	mm_create()
		初始条件：无
		操作结果：建一个某程序的mm变量
	mm_destroy(mm)
		初始条件：mm变量已存在
		操作结果：释放mm变量占用的空间
}
1.vma_struct的抽象数据类型定义
ADT vma_struct{
	数据对象：vma_struct
	数据关系：集合
	基本操作：
	vma_create(start, end, flag)
		初始条件：输入虚拟地址空间的起始和结束地址，和空间属性
		操作结果：创建vma
	insert_vma_struct(mm, vma)
		初始条件：vma, mm已存在
		操作结果：在mm变量的合适位置插入vma变量
	find_vma(mm, addr)
		初始条件：mm存在
		操作结果：返回addr所在的vma
}
以上函数都在
位置：lab3\kern\mm\vmm.c
三、详细设计
1.mm_struct结构、vma_struct结构
位置：lab3\kern\mm\vmm.h
struct mm_struct {
    list_entry_t mmap_list;        // vma结构链成的双向链表的表头
    struct vma_struct *mmap_cache; // 当前正在使用的vma结构（加快检索速度）
    pde_t *pgdir;                  // 程序的页目录表基址
    int map_count;                 // 程序的vma的个数
    void *sm_priv;                 // 指向待交换的页的链表，从第一个开始交换
};

struct vma_struct {
    struct mm_struct *vm_mm; // 对应的mm结构 
    uintptr_t vm_start;      // vma的起始地址     
    uintptr_t vm_end;        // vma的结束地址, 不包括vm_end
    uint32_t vm_flags;       // 属性（只读、只写、可执行）
    list_entry_t list_link;  // vma双向链表中的节点
};

访问未被映射的地址时，会产生页异常，在异常处理中，会跳转到do_pgfault函数处理页异常。函数处理如下
int
do_pgfault(struct mm_struct *mm, uint32_t error_code, uintptr_t addr) {
    int ret = -E_INVAL;//函数错误码：无效
    //try to find a vma which include addr
    struct vma_struct *vma = find_vma(mm, addr);//找到addr对应的vma

    pgfault_num++;//记录页异常发生的总数
    //If the addr is in the range of a mm's vma?
    if (vma == NULL || vma->vm_start > addr) {
        cprintf("not valid addr %x, and  can not find it in vma\n", addr);
        goto failed;
    }
    //check the error_code。
    switch (error_code & 3) {
    default:
            /* error code flag : default is 3 ( W/R=1, P=1): write, present */
    case 2: /* error code flag : (W/R=1, P=0): write, not present */
        if (!(vma->vm_flags & VM_WRITE)) {//写一个不能写的页面
            cprintf("do_pgfault failed: error code flag = write AND not present, but the addr's vma cannot write\n");
            goto failed;
        }
        break;
    case 1: /* error code flag : (W/R=0, P=1): read, present 读一个不存在的页面*/
        cprintf("do_pgfault failed: error code flag = read AND present\n");
        goto failed;
    case 0: /* error code flag : (W/R=0, P=0): read, not present */
        if (!(vma->vm_flags & (VM_READ | VM_EXEC))) {//读一个不能读or执行一个不能执行的页面
            cprintf("do_pgfault failed: error code flag = read AND not present, but the addr's vma cannot read or exec\n");
            goto failed;
        }
    }
    if ((ptep = get_pte(mm->pgdir, addr, 1)) == NULL){
        goto failed;
    }
    struct Page *page = NULL;
    if(*ptep == 0){//   页表项内容为0，即未分配页
        if((page = pgdir_alloc_page(&mm->pgdir, addr, perm)) == NULL)//分配页并建立映射
            goto failed;
    }
    else{
        if(swap_init_ok){//swap_manager初始化已完成
            swap_in(mm, addr, &page);//分配page，并将addr指示的磁盘内容读入page中
            page_insert(mm->pgdir, page, addr, perm);//建立page和addr的映射
            swap_map_swappable(mm, addr, page, 1);//将page按照可被替换的顺序，插入待替换页链表(sm_priv)
            page->pra_vaddr = addr;//设置该页对应的虚拟地址
        }
        else{
            cprintf("no swap_init_ok but ptep is %x, failed\n",*ptep);
            goto failed;
        }
    }
    
   ret = 0;
failed:
    return ret;
}

问：请描述页目录项(Page Directory Entry)和页表项(Page Table Entry)中组成部分对ucore实现页替换算法的潜在用处。
答：页表项指向的页，如果被换出，则该页表项pte可转化成swap_entry，前24位存储被换出到的磁盘位置，存在位（最后一位）设为0。pde同pte。

问：如果ucore的缺页服务例程在执行过程中访问内存，出现了页访问异常，请问硬件要做哪些事情?
答：如果支持中断嵌套，则产生trap，保存现场后跳转到缺页服务历程处理。如果不支持中断嵌套，则调用失败，无法处理。


练习2:补充完成基于FIFO的页面替换算法
一、需求分析
1.当空闲页=0时，需要将程序最早占用的内存页的内容换出到磁盘，对应的pte的存在位清0，并将磁盘位置记录在pte中。对应的内存页置为空闲。因此，需要按照使用时间递增的顺序，排列已使用的页面。pte存在位清0后，对应的pte变为swap_entry，前24b表示对应的页在磁盘上的位置。
2.将需要的磁盘内容加载到空闲页内，并建立page和pte的映射关系。
3.为适应不同的页面替换算法，可建立一个通用的页面替换管理器。
二、概要设计
1.定义页面替换管理器的数据结构
ADT swap_manager{
	数据对象：页
	数据关系：
	基本操作：
	init()
		初始条件：
		操作结果：建立一个空页面替换管理器
	map_swappable(mm, page)
		初始条件：mm, page
		操作结果：将使用的页按照可被替换的顺序组织起来，mm->sm_priv为链表的表头
	swap_out_victim()
		初始条件：
		操作结果：选出适合交换的内存页，并从可被替换链表中删除
}
三、详细设计
struct swap_manager
{
	 const char *name;//名字
     int (*init)            (void);//名字
     int (*init_mm)         (struct mm_struct *mm);//初始化mm中的sm_priv成员
     int (*tick_event)      (struct mm_struct *mm);//？
     int (*map_swappable)   (struct mm_struct *mm, uintptr_t addr, struct Page *page, int swap_in);//将分配的页按可被替换的顺序插入到sm_priv中
     int (*set_unswappable) (struct mm_struct *mm, uintptr_t addr);//设置为不可替换
     int (*swap_out_victim) (struct mm_struct *mm, struct Page **ptr_page, int in_tick);//选出被替换的页
     int (*check_swap)(void);//测试函数
};

位置：lab3\kern\mm\swap_fifo.c
/*
map_swappable(mm, page)
	初始条件：mm, page
	操作结果：将使用的页按照可被替换的顺序组织起来，mm->sm_priv为链表的表头
*/
static int
_fifo_map_swappable(struct mm_struct *mm, uintptr_t addr, struct Page *page, int swap_in)
{
    list_entry_t *head=(list_entry_t*) mm->sm_priv;//待替换页链表的表头
    list_entry_t *entry=&(page->pra_page_link);//该页在待替换页链表的节点
    assert(entry != NULL && head != NULL);
    if(head == NULL)
        head = entry;
    else
        list_add_before(head, entry);
    return 0;
}

/*
swap_out_victim()
	初始条件：
	操作结果：选出适合交换的内存页，并从可被替换链表中删除
*/
static int
_fifo_swap_out_victim(struct mm_struct *mm, struct Page ** ptr_page, int in_tick)
{
     list_entry_t *head=(list_entry_t*) mm->sm_priv;
     assert(head != NULL);
     assert(in_tick==0);
     *ptr_page = le2page(head, page_link);
     mm->sm_priv = list_next(head);
     list_del(head);
     return 0;
}

请在实验报告中回答如下问题:
如果要在ucore上实现"extended clock页替换算法"请给你的设计方案，现有的swap_manager框架是否足以支持在ucore中实现此算法?如果是，请给你的设计方案。如果不是，请给出你的新的扩展和基此扩展的设计方案。并需要回答如下问题
问：需要被换出的页的特征是什么? 在ucore中如何判断具有这样特征的页? 何时进行换入和换出操作?
答：支持。需要修改swap_out_victim函数。
在pte中，设置引用位，脏位(dirty bit)。在swap_out_victim函数中，增加对引用位，脏位(dirty bit)的判断。sm_priv依然按访问顺序组织。



疑问解答
问：程序占用的虚拟地址空间是连续的吗？
答：因为程序可能有多个模块、共享库等组成，所以占用的虚拟地址空间可能是不连续的。但一个模块占用的空间应该是连续的。而且即使是连续的，不同模块的属性可能是不同的，比如读、写、执行属性。vma_struct结构描述的是程序中一个连续的虚拟地址空间。mm_struct结构描述的是一个程序所有的vma，按照地址递增链起来。

问：在缺页处理中，如果某个线性地址的映射关系不存在（pte = 0），如何处理？
答：首先，给pte分配一个页，把la和该页映射起来。此时可以在该页写入内容，但是读取内容是不合理的。

问：boot_pgdir[0] = 0在哪里设置的？


swap_check调用记录
check_content_set --> 14号trap --> trap_dispatch --> pgfault_handler --> do_pgfault

函数功能
swap_map_swappable(check_mm_struct, la, page, 0)

swapfs_init()
	检查交换磁盘有效性，并设置max_swap_offset的值

实验问题
调用check_vma_struct时，一调用就开始出现page fault