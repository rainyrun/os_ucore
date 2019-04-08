# lab6_report
## 练习1：使用 Round Robin 调度算法
lab5和lab6的差异
1. lab6/kern/init/init.c：多了sched_init();
2. lab6/kern/process/proc.c：多了lab6_set_priority(uint32_t priority)函数
3. lab6/kern/process/proc.h：proc_struct结构多了调度相关的变量
4. lab6/kern/schedule/default_sched.c：多了RR调度算法
5. lab6/kern/schedule/default_sched_stride_c：新增
6. lab6/libs/skew_heap.h：新增

问：请理解并分析sched_calss中各个函数指针的用法，并接合Round Robin调度算法描ucore的调度执行过程
答：sched_class中的函数指针如下
void (*init)(struct run_queue *rq);//初始化运行队列。运行队列中存放着等待被调度的进程
void (*enqueue)(struct run_queue *rq, struct proc_struct *proc);//进程入队
void (*dequeue)(struct run_queue *rq, struct proc_struct *proc);//进程出队
struct proc_struct *(*pick_next)(struct run_queue *rq);//选择下一个被调度的进程
void (*proc_tick)(struct run_queue *rq, struct proc_struct *proc);//
ucore调度执行过程
1. 通过schedule函数进入调度算法，默认调度算法是RR
2. 将当前正在运行的进程入队，添加到run_list末尾
3. 根据RR算法选出下一个被调度的进程next，将该进程出队
4. proc_run切换到next运行

问：请在实验报告中简要说明如何设计实现”多级反馈队列调度算法“，给出概要设计，鼓励给出详细设计
答：

一、需求分析
1. 进程有反馈队列号，表示属于哪一级反馈队列。若在该级反馈队列的最大时间片用完后，还没有结束运行，则队列号+1，进入下一级队列。当队列号达到最大时，不再增加，留在该级。
2. 各级反馈队列按照FIFO进行调度，共有RUN_MAXNUM个反馈队列，反馈队列的最大时间片递增。
二、概要设计
1. 进程抽象数据结构设计
在proc_struct的基础上，增加反馈队列号
2. 多级反馈队列调度数据结构设计
ADT 多级反馈队列调度{
	数据对象：proc
	数据关系：多级队列
	基本操作：
	init
		初始条件：
		操作结果：初始化
	enqueue(rq, proc, fnum)
		初始条件：
		操作结果：
	dequeue(rq, proc, fnum)
		初始条件：
		操作结果：
	pick_next(rq, proc, fnum)
		初始条件：
		操作结果：
}

1. 设计run_list结构
struct run_list{
	list_entry_t rlist;
	unsigned int proc_num;
}
2. 改造run_queue结构
#define RUN_MAXNUM 4
struct run_queue{
	struct run_list feedback_list[RUNL_NUM];//四级反馈队列
	int max_time_slice;
}

## 练习2: 实现 Stride Scheduling 调度算法
一、需求分析
1. 为每个进程创建stride结构，保存进程的pass，stride信息
2. 进程运行队列采用优先队列的结构，使用二叉树做存储结构
3. 进程运行固定时间，每次选择运行队列中stride值最小的进程作为下一个被调度的进程
4. 进程出队后，要修改stride信息，为下次调度做准备。
二、详细设计

