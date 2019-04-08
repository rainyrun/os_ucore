lab2实验报告
练习1:实现 first-fit 连续物理内存分配算法
一、需求分析
1.需要探测物理内存的分布，将连续的页组织成空闲内存块，再把空闲内存块组成链表，以物理地址递增的顺序排列。
2.分配内存。给出需要空闲内存的大小（以页数表示），从空闲块链表中，选出第一个符合要求的空闲块，并分配。
3.回收内存。给出需回收的内存起始地址和大小，将该内存置为空闲后，在空闲块链表中找到合适的位置插入。前后空闲块若相邻，则需要合并。
4.程序执行的命令为：1）探测内存。2）建空闲块链表。3）分配。4）回收
5.测试数据：
二、概要设计
1.设定空闲块链表的抽象数据类型定义：
ADT free_area_t{
	数据对象：空闲块
	数据关系：以起始物理地址递增的顺序排列
	基本操作：
	page_init(void)
		初始条件：已探测到的内存分布情况
		操作结果：形成空闲块链表
}
2.设定物理内存管理的抽象数据类型定义：
ADT pmm{
	init()
		初始条件：空
		操作结果：空的内存管理器
	alloc_pages(n)
		初始条件：n > 0
		操作结果：分配有n个页的物理内存
	init_memmap(base, n)
		初始条件：base指向空闲物理内存块的起始地址，n为内存块的页数
		操作结果：将base指向的内存块设为空，并插入到空闲块链表中
	free_pages(base, n)
		初始条件：base指向待回收的内存块，n为内存块的页数
		操作结果：将base指向的内存块回收到空闲块链表中
}
三、详细设计
1.页类型、空闲块链表、双向链表
//页结构
//位置：lab2/kern/mm/memlayout.h
struct Page {
    int ref;                        // 该页被引用的次数（可重入代码可能被引用多次，仅当引用次数=0时才可被释放）
    uint32_t flags;                 // 属性，如是否被占用等
    unsigned int property;          // 空闲块中的页数，只有空闲块的第一个页才有值
    list_entry_t page_link;         // 对应的链表节点
};

//空闲块链表
//位置：lab2/kern/mm/memlayout.h
typedef struct {
    list_entry_t free_list;         // 空闲块链表的头节点
    unsigned int nr_free;           // 空闲页总数
} free_area_t;

//双向链表（通用）
//位置：lab2/libs/list.h
struct list_entry {
    struct list_entry *prev, *next;
};

/*
page_init(void)
		初始条件：已探测到的内存分布情况
		操作结果：形成空闲块链表
*/
位置：lab2/kern/mm/pmm.c
static void
page_init(void) {
    struct e820map *memmap = (struct e820map *)(0x8000 + KERNBASE);//内存分布的探测结果，存储在0x8000内
    uint64_t maxpa = 0; //最高物理内存地址

    cprintf("e820map:\n");
    int i;
    for (i = 0; i < memmap->nr_map; i ++) {//打印内存探测结果
        uint64_t begin = memmap->map[i].addr, end = begin + memmap->map[i].size;
        cprintf("  memory: %08llx, [%08llx, %08llx], type = %d.\n",
                memmap->map[i].size, begin, end - 1, memmap->map[i].type);//%llx输出64位16进制数
        if (memmap->map[i].type == E820_ARM) {
            if (maxpa < end && begin < KMEMSIZE) {
                maxpa = end;
            }
        }
    }
    if (maxpa > KMEMSIZE) {//maxpa限制在KMEMSIZE内
        maxpa = KMEMSIZE;
    }

    extern char end[];

    npage = maxpa / PGSIZE;//物理内存对应的总页数
    pages = (struct Page *)ROUNDUP((void *)end, PGSIZE);//end：ucore的结束地址，ucore结束后就是pages数据结构

    for (i = 0; i < npage; i ++) {//初始化，所有内存页设为reserved
        SetPageReserved(pages + i);
    }

    uintptr_t freemem = PADDR((uintptr_t)pages + sizeof(struct Page) * npage);//空闲内存的起始物理地址

    for (i = 0; i < memmap->nr_map; i ++) {
        uint64_t begin = memmap->map[i].addr, end = begin + memmap->map[i].size;
        if (memmap->map[i].type == E820_ARM) {//空闲内存，begin，end指向物理地址
            if (begin < freemem) {
                begin = freemem;
            }
            if (end > KMEMSIZE) {
                end = KMEMSIZE;
            }
            if (begin < end) {
                begin = ROUNDUP(begin, PGSIZE);
                end = ROUNDDOWN(end, PGSIZE);
                if (begin < end) {//空闲内存大小 > 1页
                    init_memmap(pa2page(begin), (end - begin) / PGSIZE);
                }
            }
        }
    }
}

2.物理内存管理器结构
//物理内存管理器（通用，与c++中的抽象类概念类似）
//位置：lab2/kern/mm/pmm.h
struct pmm_manager {
    const char *name;                                 // XXX_pmm_manager的名字
    void (*init)(void);                               // 初始化页结构和空闲块链表 
    void (*init_memmap)(struct Page *base, size_t n); // 置数据块为空闲块
    struct Page *(*alloc_pages)(size_t n);            // 分配n个页
    void (*free_pages)(struct Page *base, size_t n);  // 释放从base开始的n个页即某空闲块
    size_t (*nr_free_pages)(void);                    // 返回空闲页总数
    void (*check)(void);                              // 测试函数
};

first-fit物理内存管理器实例：defalut_pmm
/*
init_memmap(base, n)
	初始条件：base指向空闲物理内存块的起始地址，n为内存块的页数
	操作结果：将base指向的内存块设为空，并插入到空闲块链表中
*/
位置：lab2/kern/mm/default_pmm.c
static void
default_init_memmap(struct Page *base, size_t n) {


/*
alloc_pages(n)
	初始条件：n > 0
	操作结果：分配有n个页的物理内存
*/
位置：lab2/kern/mm/default_pmm.c
static struct Page *
default_alloc_pages(size_t n) 


/*
free_pages(base, n)
	初始条件：base指向待回收的内存块，n为内存块的页数
	操作结果：将base指向的内存块回收到空闲块链表中
*/
位置：lab2/kern/mm/default_pmm.c
static void
default_free_pages(struct Page *base, size_t n) {
    assert(n > 0);
    struct Page *p = base;
    for (; p != base + n; p ++) {
        // assert(!PageReserved(p));
        // assert(!PageProperty(p));
        assert(!PageReserved(p) && !PageProperty(p));//非保留页，且被占用
        p->flags = 0;
        set_page_ref(p, 0);
    }
    SetPageProperty(base);//设为空闲页
    base->property = n;
    list_entry_t *temp = free_list.next;
    struct Page *first = le2page(free_list.next, page_link);//第一个空闲块
    struct Page *end = le2page(free_list.prev, page_link);//最后一个空闲块
    // print_free_list();
    if(base + n <= first){//插在链头
        temp = free_list.next;
    }
    else if(base >= end + end->property){//插在链尾
        temp = &free_list;
    }
    else{//插在链中
        struct Page *ptemp = le2page(temp, page_link);
        struct Page *pntemp = le2page(temp->next, page_link);
        while(temp != &free_list){
            if(base >= ptemp + ptemp->property && base + n <= pntemp)
                break;
            temp = temp->next;
            ptemp = le2page(temp, page_link);
            pntemp = le2page(temp->next, page_link);
        }
        if(temp == &free_list)
            panic("default_free_pages: base or n error\n");
        temp = temp->next;
    }
    struct Page *left = le2page(temp->prev, page_link);//插入位置左侧的空闲块
    struct Page *right = le2page(temp, page_link);//插入位置的空闲块
    if((temp->prev != &free_list) && (base == left + left->property)){//有左空闲块，且和左空闲块相邻，向左合并
        ClearPageProperty(base);
        base = left;
        list_del(temp->prev);
        base->property += n;
    }
    if((temp != &free_list) && (base + base->property) == right){//有右空闲块，且和右空闲块相邻，向右合并
        ClearPageProperty(right);
        list_del(temp);
        base->property += right->property;
        temp = temp->next;
    }
    list_add_before(temp, &(base->page_link));
    nr_free += n;
}

注：如非特殊说明，输入输出内的地址都是指虚拟地址

准备内容：
1.探测物理内存，检测空闲页，得到空闲内存的起始地址和结束地址
数据
//保存探测到的内存布局。
//位置：lab2/kern/mm/memlayout.h
struct e820map {
    int nr_map;	//内存块个数
    struct {
        uint64_t addr;	//内存块起始地址
        uint64_t size;	//大小
        uint32_t type;	//类型
    } __attribute__((packed)) map[E820MAX];	//上限为E820MAX个
};

功能
位置：lab2/boot/bootasm.S
probe_memory:
    movl $0, 0x8000			;内存0x8000位置清0。(e820map的变量存放在0x8000处，即，nr_map = 0)
    xorl %ebx, %ebx			;ebx置0
    movw $0x8004, %di		;map[E820MAX]从0x8004开始，es:di:指向保存地址范围描述符结构的缓冲区，BIOS把信息写入这个结构的起始地址。
start_probe:
    movl $0xE820, %eax 		;INT 15的中断调用参数
    movl $20, %ecx			;保存地址范围描述符的内存大小
    movl $SMAP, %edx		;534D4150h (即4个ASCII字符“SMAP”) ，只是一个签名
    int $0x15
    jnc cont
    movw $12345, 0x8000
    jmp finish_probe
cont:
    addw $20, %di
    incl 0x8000				;即nr_map++
    cmpl $0, %ebx 			;ebx为0，探测结束
    jnz start_probe
finish_probe:


练习2

2.根据线性地址la，找到对应的物理地址pa
一、需求分析
//1.程序存储在磁盘上，有对应的虚拟地址va。将程序按页装入内存，需要建立页表。虚拟地址va通过段表映射到线性地址la，线性地址la通过页表映射到物理地址pa。
2.虚拟地址pa、线性地址la、物理地址的最终映射关系为：va = la = pa + 0xC0000000
3.在初次为程序设置页表时，需要为页目录表分配空间。
4.页目录项前20位为页框号，可找到一级页表；后12为页偏移，可找到页表中的页表项。页表项前20为页框号，可找到la所在到页；后12位为页偏移，可找到la对应的pa
5.程序执行命令：1）分配页表。2）将映射关系写入页表
6.测试：

二、概要设计
1.页表抽象数据类型的定义：
ADT pgdir{
	数据对象：页
	数据关系：索引结构
	基本操作：
	lpdt()
		初始条件：二级页表（页目录表）存在
		操作结果：将页目录表的基址加载到寄存器cr3
	get_pte	(pgdir, la, create)
		初始条件：无
		操作结果：返回la对应的页表项。若没有页目录项，creat = 1则分配。
	page_insert(pgdir, la, page)
		初始条件：无
		操作结果：将la和page建立起映射关系
	page_remove_pte(pgdir, la, ptep)
		初始条件：pgdir，pte存在
		操作结果：释放la所在的页，并清除la与pte的关系
	boot_map_segment
		初始条件：(pgdir, la, size, pa, perm)
		操作结果：建立页表。la和pa的映射关系为：la = pa + 0xC0000000。
}

三、详细设计
/*
get_pte	(pgdir, la, creat)
	初始条件：无
	操作结果：返回la对应的页表项。若没有页目录项，creat = 1则分配。
*/
位置：位置：lab2/kern/mm/pmm.c
pte_t *
get_pte(pde_t *pgdir, uintptr_t la, bool create) {
    pde_t *pdep = &pgdir[PDX(la)];  //pdep指向对应的pde
    if(!(*pdep & PTE_P)){    //pde不存在
        struct Page *page;
        if(create == 0 || (page = alloc_page()) == NULL){
            return NULL;
        }
        set_page_ref(page, 1);
        uintptr_t pa = page2pa(page);
        memset(KADDR(pa), 0 ,PGSIZE);//将PT初始化为全0
        pgdir[PDX(la)] = pa | PTE_P | PTE_W | PTE_U;//填充pde的内容
    }
    return &((pte_t *)KADDR(PDE_ADDR(*pdep)))[PTX(la)];//PDE_ADDR(*pdep) = pa
}

问：请描述页目录项(Page Directory Entry)和页表项(Page Table Entry)中每个组成部分的含义以及对ucore而言的潜在用处。
答：页目录项内容pde = (页表起始物理地址 & ~0x0FFF) | PTE_U | PTE_W | PTE_P
   页表项内容pte = (pa & ~0x0FFF) | PTE_P | PTE_W
其中:
PTE_U：位3，表示用户态的软件可以读取对应地址的物理内存页内容
PTE_W：位2，表示物理内存页内容可写
PTE_P：位1，表示物理内存页存在
给出la，PDX(la)为对应的pde，pde前20位+PTX(la)找到对应的pte，PTE_ADDR(pte)+offset(la)找到pa
属性位（后三位）可以控制访问权限，如，避免对只读页进行页操作。

问：如果ucore执行过程中访问内存，出现了页访问异常，请问硬件要做哪些事情?
答：页访问异常分为缺页、权限不足。
	缺页：为程序分配页，将缺失的内容，从磁盘读入该页，并在页表中建立映射（调用page_insert函数）。若页不足，则启动换入换出机制。
	权限不足：报错。

练习3
/*
page_remove_pte(pgdir, la, ptep)
		初始条件：pgdir，pte存在
		操作结果：释放la所在的页，并清除la与pte的关系
*/
位置：位置：lab2/kern/mm/pmm.c
static inline void
page_remove_pte(pde_t *pgdir, uintptr_t la, pte_t *ptep) {
    if(*ptep & PTE_P){
        // struct Page *page = pte2page(*ptep);
        struct Page *page = get_page(pgdir, la, NULL);  //取得la对应的页
        if(page_ref_dec(page) == 0)
            free_page(page);
        *ptep = 0;  //清除pte内容
        tlb_invalidate(pgdir, la);//清除tlb缓存
    }
}

问：数据结构Page的全局变量(其实是一个数组)的每一项与页表中的页目录项和页表项有无对应关系?如果有，其对应关系是啥?
答：页目录项(or页表项) & (~0xfff)得到的地址是某页的起始地址，该页属于Page中的某一项。

问：如果希望虚拟地址与物理地址相等，则需要如何修改lab2，完成此事?鼓励通过编程来具体完成这个问题
答：将pmm_init中的boot_map_segment函数，参数改为(boot_pgdir, 0, KMEMSIZE, 0, PTE_W)。段映射从最初就是对等映射即可。


问：将pte的PTE_P位置为0，和清除pte的区别？
答：将pte的PTE_P位置为0，则pte的其他位还可以表示对应的la在磁盘中的位置，映射关系没有改变。而清除pte则将映射关系清除了。

问：需要限制能使用的虚拟地址范围？意义何在？物理地址和虚拟地址可以以任意关系映射啊

问：pmm_init中，boot_pgdir[PDX(VPT)] = PADDR(boot_pgdir) | PTE_P | PTE_W的作用？
答：

疑问解答
问：boot_pgdir指向哪里？
答：boot_pgdir是ucore操作系统的页表。pde_t *boot_pgdir = &__boot_pgdir; __boot_pgdir在entry.S中，已分配好内存空间。

问：实验指导里的enable_paging，enable_page函数在哪？

问：pgdir在程序最开始的时候，就有分配内存吗？在哪里实现该功能？

附录
函数功能说明
PGOFF(la)
	取得线性地址la里的偏移量off
get_page(pgdir, la, ptep_store)
	根据页目录表地址pgdir，返回线性地址la对应的页，pte_store != NULL时，把la对应的pte存储在ptep_store里。
page_insert(pgdir, page, la, perm)
	根据页目录表地址pgdir，将page和la对应起来，属性为perm
get_pte(pgdir, la, create)
	根据页目录表地址pgdir，返回la对应的pte的虚拟地址，若包含pte的PT不存在，则create=1时创造。
PADDR(kva)
	把内核虚拟地址，转化为对应的物理地址
KADDR(pa)
	把物理地址，转化为对应的内核虚拟地址
boot_map_segment(pde_t *pgdir, uintptr_t la, size_t size, uintptr_t pa, uint32_t perm)
	进行地址映射，[la, la + size] <--> [pa, pa + size]
SEG(type, base, lim, dpl)
	设置段选择子。type：段描述符的不同类型；base：段基址；lim：段界限；dpl：特权级