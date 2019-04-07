# lab1实验报告
## 练习1-1:ucore.img的生成过程
答：在lab1文件夹下，执行make命令，可生成ucore.img文件  
1. makefile中第一个目标文件是UCOREIMG，make后自动生成
```
UCOREIMG	:= $(call totarget,ucore.img)
$(UCOREIMG): $(kernel) $(bootblock)
//为了生成UCOREIMG，需要$(kernel) $(bootblock)
	$(V)dd if=/dev/zero of=$@ count=10000
	$(V)dd if=$(bootblock) of=$@ conv=notrunc
	$(V)dd if=$(kernel) of=$@ seek=1 conv=notrunc
	//在ucore.img中写入需要的信息
$(call create_target,ucore.img)
//生成目标文件ucore.img
```
2. 生成$(kernel)
```
kernel = $(call totarget,kernel)
//生成的kernel在bin/kernel
$(kernel): tools/kernel.ld
//需要目标文件kernel.ld
$(kernel): $(KOBJS)
//需要目标文件$(KOBJS), KOBJS	= $(call read_packet,kernel libs)
//即kernel,libs中的所有.o文件
	@echo + ld $@
	$(V)$(LD) $(LDFLAGS) -T tools/kernel.ld -o $@ $(KOBJS)
	@$(OBJDUMP) -S $@ > $(call asmfile,kernel)
	@$(OBJDUMP) -t $@ | $(SED) '1,/SYMBOL TABLE/d; s/ .* / /; /^$$/d' > $(call symfile,kernel)
	//然后编译成汇编文件，符号文件？
$(call create_target,kernel)
//生成目标文件kernel
   //1.生成kernel,libs中的所有.o文件
	$(call add_files_cc,$(call listf_cc,$(KSRCDIR)),kernel,$(KCFLAGS))
   //2.kernel.ld（已存在，无需生成）
```
3. 生成$(bootblock)
```
bootblock = $(call totarget,bootblock)
//生成的kernel在bin/bootblock
$(bootblock): $(call toobj,$(bootfiles)) | $(call totarget,sign)
//需要bootasm.o, bootmain.o, sign
	@echo + ld $@
	$(V)$(LD) $(LDFLAGS) -N -e start -Ttext 0x7C00 $^ -o $(call toobj,bootblock)
	@$(OBJDUMP) -S $(call objfile,bootblock) > $(call asmfile,bootblock)
	@$(OBJCOPY) -S -O binary $(call objfile,bootblock) $(call outfile,bootblock)
	@$(call totarget,sign) $(call outfile,bootblock) $(bootblock)
	//汇编
$(call create_target,bootblock)
//生成目标文件bootblock
//1.生成bootasm.o, bootmain.o
	bootfiles = $(call listf_cc,boot)
	//bootfiles = boot目录下的所有.c .S文件
	$(foreach f,$(bootfiles),$(call cc_compile,$(f),$(CC),$(CFLAGS) -Os -nostdinc))
	//依次取出bootfiles中的文件并编译（生成bootasm.o, bootmain.o）
//2.生成sign
		$(call add_files_host,tools/sign.c,sign,sign)
		$(call create_target_host,sign,sign)
```
## 练习1-2
答：从sign.c的代码来看，一个磁盘主引导扇区只有512字节。且  
第510个（倒数第二个）字节是0x55，  
第511个（倒数第一个）字节是0xAA。  

## 练习2
执行步骤如下：  
1. 使用qemu模拟器，让ucore在qemu模拟的x86硬件环境中执行，并与gdb配合进行源码调试。
	命令： qemu -S -s -hda bin/ucore.img -monitor stdio  
	-S -s让qemu进入等待gdb调试器的接入并且还不能让qemu中的CPU执行  
2. 启动gdb，在gdb界面连接到qemu。为了让gdb获知符号信息，需要指定调试目标文件，gdb中使用file命令。
	命令：(gdb)  file ./bin/kernel  
	命令：(gdb)  target remote 127.0.0.1:1234  
	使用si命令可单步一条机器指令，x /2i 0xffff0可查看0xffff0处的汇编代码  
3. 在0x7c00处设置断点。使用命令c（continue）可执行到断点
	命令：b *0x7c00  
	命令：c  
4. 在调用qemu时增加`-d in_asm -D q.log`参数，便可以将运行的汇编指令保存在q.log中。
备注：可将编译命令写在makefile内。如make debug。  

## 练习3
### 问：为何开启A20？
答：早期的8086cpu寻址能力只有1MB，32位的80386为了向下兼容，A20初始被置为0，保证兼容8086的回卷特性（地址超过1MB时，重新返回到低地址）。
在保护模式下，为了使能所有地址位的寻址能力，需要打开A20地址线控制，即需要通过向键盘控制器8042发送一个命令来完成。  
### 问：如何开启A20？
在bootasm.S中有开启A20的过程。  
```
首先，关中断，寄存器清0
	cli
	cld
	xorw %ax, %ax
	movw %ax, %ds
	movw %ax, %es
	movw %ax, %ss
然后，seta20.1部分
	1. 等待8042Inputbuffer为空;
	2. 发送Write8042OutputPort(P2)命令到8042Inputbuffer;
再然后，seta20.2部分
	1. 等待8042Inputbuffer为空;
	2. 将8042OutputPort(P2)得到字节的第2位置1，然后写入8042Inputbuffer;
A20就开启了。
```
### 问：如何初始化GDT表？
答：
1. 填写gdt表项，内容包括：段基址，段界限，段属性。第一个gdt表项约定为空。
```
gdt:
    SEG_NULLASM                                     # 空表项
    SEG_ASM(STA_X|STA_R, 0x0, 0xffffffff)           # 代码段
    SEG_ASM(STA_W, 0x0, 0xffffffff)                 # 数据段
```
2. 加载gdt表，将gdt表的基址填入gdtr寄存器
`lgdt gdtdesc`
注：在bootasm.S中实现。  

### 问：如何使能和进入保护模式？
答：
1. 设置好GDT表后，将cr0寄存器的最低位（PE位）置1。
```
	movl %cr0, %eax
    orl $CR0_PE_ON, %eax
    movl %eax, %cr0
```
2. 长跳转更新cs寄存器
	`ljmp $PROT_MODE_CSEG, $protcseg`
3. 更新保护模式下的寄存器
```
	movw $PROT_MODE_DSEG, %ax
    movw %ax, %ds
    movw %ax, %es
    movw %ax, %fs
    movw %ax, %gs
    movw %ax, %ss
```
4. 跳转到bootmain开始执行程序
```
	movl $0x0, %ebp
    movl $start, %esp
    call bootmain
```
## 练习4
### 问：bootloader如何读取硬盘扇区的?
答：bootloader访问硬盘是LBA模式的PIO(Program IO)方式，即所有的IO操作通过CPU访问硬盘的IO地址寄存器完成。一般主板有2个IDE通道，每个通道可以接2个IDE硬盘。一般第一个IDE通道通过访问IO地址0x1f0-0x1f7来实现，第二个IDE通道通过访问0x170-0x17f实现。   
在bootmain.c中，readsect()函数实现读取磁盘扇区。
```
static void
readsect(void *dst, uint32_t secno) {
    //等待磁盘准备好
    waitdisk();

    outb(0x1F2, 1);	//读一个扇区
    //填写LBA参数（secno）
    outb(0x1F3, secno & 0xFF);
    outb(0x1F4, (secno >> 8) & 0xFF);
    outb(0x1F5, (secno >> 16) & 0xFF);
    outb(0x1F6, ((secno >> 24) & 0xF) | 0xE0);
    outb(0x1F7, 0x20);	//读扇区命令

    //等待磁盘准备好
    waitdisk();

    //从端口0x1F0读扇区内容到地址dst的位置
    insl(0x1F0, dst, SECTSIZE / 4);
}
```
### 问：bootloader是如何加载ELF格式的OS?
答：在bootmain.c中的bootmain函数中实现
```
1.读取ELF文件的文件头ELF header，获得文件信息。并判断是否为符合条件的elf文件
	readseg((uintptr_t)ELFHDR, SECTSIZE * 8, 0);//读ELF header
	if (ELFHDR->e_magic != ELF_MAGIC) {
        goto bad;
    }
2.找到program header表的起始地址和个数，并依次读取到制定的位置。
	ph = (struct proghdr *)((uintptr_t)ELFHDR + ELFHDR->e_phoff);
    eph = ph + ELFHDR->e_phnum;
    for (; ph < eph; ph ++) {
        readseg(ph->p_va & 0xFFFFFF, ph->p_memsz, ph->p_offset);
    }
3.跳转到程序入口处执行。
	((void (*)(void))(ELFHDR->e_entry & 0xFFFFFF))();
```
## 练习5
函数print_stackframe的实现如下：
```
void
print_stackframe(void) {
    uint32_t ebp = read_ebp();  //取得当前ebp的值
    uint32_t eip = read_eip();  //取得调用函数的返回地址
    int i, j;
    for(i = 0; ebp != 0 && i < STACKFRAME_DEPTH; i++){  //ebp为0（初始值）说明没有函数运行
        cprintf("ebp: 0x%08x, eip: 0x%08x, args: ", ebp, eip);
        uint32_t * args = (uint32_t *) ebp + 2; //参数1的地址
        for(j = 0; j < 4; j++)
            cprintf("0x%08x ", args[j]);
        cprintf("\n");
        print_debuginfo(eip - 1);   //打印debug信息
        eip = ((uint32_t *)ebp)[1];
        ebp = ((uint32_t *)ebp)[0];
    }
}
```
运行结果如下：
```
(THU.CST) os is loading ...

Special kernel symbols:
  entry  0x00100000 (phys)
  etext  0x001032c3 (phys)
  edata  0x0010ea16 (phys)
  end    0x0010fd20 (phys)
Kernel executable memory footprint: 64KB
ebp: 0x00007b08, eip: 0x001009a6, args: 0x00010094 0x00000000 0x00007b38 0x00100092 
    kern/debug/kdebug.c:306: print_stackframe+21
ebp: 0x00007b18, eip: 0x00100c95, args: 0x00000000 0x00000000 0x00000000 0x00007b88 
    kern/debug/kmonitor.c:125: mon_backtrace+10
ebp: 0x00007b38, eip: 0x00100092, args: 0x00000000 0x00007b60 0xffff0000 0x00007b64 
    kern/init/init.c:48: grade_backtrace2+33
ebp: 0x00007b58, eip: 0x001000bb, args: 0x00000000 0xffff0000 0x00007b84 0x00000029 
    kern/init/init.c:53: grade_backtrace1+38
ebp: 0x00007b78, eip: 0x001000d9, args: 0x00000000 0x00100000 0xffff0000 0x0000001d 
    kern/init/init.c:58: grade_backtrace0+23
ebp: 0x00007b98, eip: 0x001000fe, args: 0x001032fc 0x001032e0 0x0000130a 0x00000000 
    kern/init/init.c:63: grade_backtrace+34
ebp: 0x00007bc8, eip: 0x00100055, args: 0x00000000 0x00000000 0x00000000 0x00010094 
    kern/init/init.c:28: kern_init+84
ebp: 0x00007bf8, eip: 0x00007d68, args: 0xc031fcfa 0xc08ed88e 0x64e4d08e 0xfa7502a8 
    <unknow>: -- 0x00007d67 --
++ setup timer interrupts
```
## 练习6
1.问：中断描述符表(也可简称为保护模式下的中断向量表)中一个表项占多少字节?其中哪几位代表中断处理代码的入口?   
答：占8字节。选择子（第2、3字节）+ 偏移量（0-1字节和6-7字节）构成中断处理代码的入口.   

2.请编程完善kern/trap/trap.c中对中断向量表进行初始化的函数idt_init.   
idt_init函数如下：
```
void
idt_init(void) {
    extern uintptr_t __vectors[];
    int i;
    for(i = 0; i < sizeof(idt) / sizeof(struct pseudodesc); i++){//初始化中断描述符
        SETGATE(idt[i], 0, GD_KTEXT, __vectors[i], DPL_KERNEL);
    }
    SETGATE(idt[T_SWITCH_TOK], 0, GD_KTEXT, __vectors[T_SWITCH_TOK], DPL_USER);//初始化系统调用中断(T_SYSCALL)的中断描述符
    lidt(&idt_pd);//加载中断描述符表基址到idtr
}
```
3.请编程完善trap.c中的中断处理函数trap，填写trap函数中处理时钟中断的部分，使操作系统每遇到100次时钟中断后，调用print_ticks子程序，向屏幕上打印一行文字”100 ticks”。  
```
trap函数中处理时钟中断的部分，代码如下：
	ticks++;
    if(ticks % TICK_NUM == 0){
        print_ticks();
    }
    break;
```

解惑：


challenge1






# 附录

## 注意事项
1.调试时，先关闭qemu再退出gdb
2.栈向低地址方向增长


## 疑问解答
问：散落在makefile里的零碎命令会被执行吗？ 
答：不会，只有被需要时，才会执行。  

问：需要加一个中断服务例程的段吗？还是直接用代码段即可？  
答：直接使用内核代码段. 

问：系统调用中断(T_SYSCALL)，对应中断向量中的偏移量是多少？  
答：第T_SWITCH_TOK个中断服务例程，即__vectors[T_SWITCH_TOK]. 

问：trap_dispatch中，case: T_SWITCH_TOU，为什么没有iret？  

问：tf的作用是什么？在内存中如何存储的？

问：内核栈和用户栈的地址在哪？是自己定义的吗？

## make工具

$@：目标文件.  
$^：所有的依赖文件.  
$<：第一个依赖文件.  

名称：加前缀函数——addprefix.  
语法：$(addprefix <prefix>,<names>).  
功能：把前缀<prefix>加到<names>中的每个单词前面。   
示例：$(addprefix src/,foo bar)返回值是“src/foo src/bar”。   
返回：返回加过前缀的文件名序列。

名称：if函数.  
语法：$(if <condition>,<then-part>).   
或是，$(if <condition>,<then-part>,<else-part> ).  
说明：if函数可以包含“else”部分，或是不含。即if函数的参数可以是两个，也可以是三个。if函数的返回值是，如果<condition>为真（非空字符串），那个<then- part>会是整个函数的返回值，如果<condition>为假（空字符串），那么<else-part>会是整个函数的返回值，此时如果<else-part>没有被定义，那么，整个函数返回空字串。   

$(foreach <var>,<list>,<text>).  
名称：foreach 函数.  
功能：把参数<list>中的单词逐一取出放到参数<var>所指定的变量中，然后再执行<text>所包含的表达式。每一次<text>会返回一个字符串，循环过程中，<text>的所返回的每个字符串会以空格分隔，最后当整个循环结束时，<text>所返回的每个字符串所组成的整个字符串（以空格分隔）将会是foreach函数的返回值。  
所以，<var>最好是一个变量名，<list>可以是一个表达式，而<text>中一般会使用<var>.  
这个参数来依次枚举<list>中的单词。
例子：  
names := a b c d  
files := $(foreach n,$(names),$(n).o)    
上面的例子中，$(name)中的单词会被挨个取出，并存到变量“n”中，“$(n).o”每次根据“$(n)”计算出一个值，这些值以空格分隔，最后作为foreach函数的返回，所以，$(f
iles)的值是“a.o b.o c.o d.o”。注意，foreach中的<var>参数是一个临时的局部变量，foreach函数执行完后，参数<var>的变量将不在作用，其作用域只在foreach函数当中。  

$(patsubst <pattern>,<replacement>,<text>)  
名称：模式字符串替换函数——patsubst。  
功能：查找<text>中的单词（单词以“空格”、“Tab”或“回车”“换行”分隔）是否符合模式<pattern>，如果匹配的话，则以<replacement>替换。这里，<pattern>可以包括通配符“%”，表示任意长度的字串。如果<replacement>中也包含“%”，那么，<replacement>中的这个“%”将是<pattern>中的那个“%”所代表的字串。（可以用“\”来转义，以“\%”来表示真实含义的“%”字符）返回：函数返回被替换过后的字符串。  
示例：$(patsubst %.c,%.o,x.c.c bar.c)  
把字串“x.c.c bar.c”符合模式[%.c]的单词替换成[%.o]，返回结果是“x.c.o bar.o”.  

$(basename <names...> )  
名称：取前缀函数——basename。  
功能：从文件名序列<names>中取出各个文件名的前缀部分。  
返回：返回文件名序列<names>的前缀序列，如果文件没有前缀，则返回空字串。  
示例：$(basename src/foo.c src-1.0/bar.c hacks)返回值是“src/foo src-1.0/bar hacks”。  

$(addsuffix <suffix>,<names...> ).  
名称：加后缀函数——addsuffix。  
功能：把后缀<suffix>加到<names>中的每个单词后面。  
返回：返回加过后缀的文件名序列。  
示例：$(addsuffix .c,foo bar)返回值是“foo.c bar.c”。  

$(filter <pattern...>,<text> ).  
名称：过滤函数——filter。  
功能：以<pattern>模式过滤<text>字符串中的单词，保留符合模式<pattern>的单词。可以有多个模式。
返回：返回符合模式<pattern>的字串。  
示例：
```
sources := foo.c bar.c baz.s ugh.h
foo: $(sources)
cc $(filter %.c %.s,$(sources)) -o foo
$(filter %.c %.s,$(sources))返回的值是“foo.c bar.c baz.s”。
```
## linux命令工具
```
dd：用指定大小的块拷贝一个文件，并在拷贝的同时进行指定的转换。
	conv=notrunc：不截短输出文件
	seek=blocks：从输出文件开头跳过blocks个块后再开始复制。
	count=blocks：仅拷贝blocks个块，块大小等于ibs指定的字节数。
```
## gcc编译选项
```
-Wall 
打开所有 cc 的作者认为值得注意的警告。不要只看这个选项的名字，它并没有打开所有 cc 能够注意到的所有警告。
-g 
产生一个可调试的可执行文件。编译器会在可执行文件中植入一些信息，这些信息能够把源文件中的行数和被调用的函数联系起来。在你一步一步调试程序的时候，调试器能够使用这些信息来显示源代码。
```

## 操作系统加载顺序
1. 加电自启，到0xffff0处，执行长跳转指令
2. 跳转到bios处，bios的功能：读入bootloader（处于硬盘镜像的第一个扇区）
3. 跳转到bootloader处（0x7c00），bootloader的功能
	1. 切换到保护模式，启动分段机制（对等映射）
	2. 读ucore操作系统到内存（0x100000)
	3. 显示字符串信息
	4. 控制权交给ucore
4. ucore开始执行，从入口开始。ucore的功能
	1. （lab2）kern_entry:临时段映射，分页机制做准备
	2. 初始化终端（bss段，串口，键盘，时钟中断）;
	3. 显示堆栈中的多层函数调用关系;
	4. (lab1)pmm_init:启用分段机制;(lab2)pmm_init:
		1.  初始化物理内存页管理器框架pmm_manager;
		2. 建立空闲的page链表，这样就可以分配以页(4KB)为单位的空闲内存了;
		3. 检查物理内存页分配算法;
		4. 为确保切换到分页机制后，代码能够正常执行，先建立一个临时二级页表;
		5. 建立一一映射关系的二级页表;
		6. 使能分页机制;
		7. 从新设置全局段描述符表;
		8. 取消临时二级页表;
		9. 检查页表建立是否正确;
		10. 通过自映射机制完成页表的打印输出
	5. 初始化中断控制器8259A，设置中断描述符表，初始化时钟中断，使能整个系统的中断机制; 执行while(1)死循环。