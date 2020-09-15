---
title: 操作系统课程实践设计 实验一 代码详解
date: 2020-03-21 11:17:16
tags:  Linux
top_img: https://images.ryanhor.com/OS%20lab1.png
cover: https://images.ryanhor.com/OS%20lab1.png
---

# 获取与修改优先级和nice

**实验一的内容是nice值的修改与获取，我将flag提取出来做出了两个getnice和modinice的功能**

## **getnicesyscall**

声明和添加调用号的过程不做重述，以下仅将实现部分的代码进行讲解。
</br>

```c
SYSCALL_DEFINE4(getnicesyscall, pid_t , pid, int, nicevalue,void __user * ,prio,void __user * ,nice)
{
 
   struct pid * kpid;
   struct task_struct * task;
 
   kpid = find_get_pid(pid);// Retuern Pid value
   task = pid_task(kpid, PIDTYPE_PID);// Return task_struct
   int nice_before;
   int prio_before;
   nice_before = task_nice(task);// Return current nice value
   
   prio_before = task_prio(task);//return current prio value

//}
//Get info
 
   copy_to_user(nice,(const void*)&nice_before ,sizeof(nice_before ));//Copy nice
   copy_to_user(prio,(const void*)&prio_before,sizeof(prio_before));//Copy prio
   printk("This process: nice value :%d and prio value%d\n",nice_before,prio_before);
 
   return 0;
 
}


```

###  **pid_t** 
:pid_t的源码在/usr/include/sys/types.h中

``` bash
#include <bits/types.h>
     ......
   
#ifndef __pid_t_defined
typedef __pid_t pid_t;
# define __pid_t_defined
#endif
```
第一个替换：pid_t -> __pid_t

:__pid_t的源码/usr/include/bits/types.h中
``` bash
#include <bits/typesizes.h>

#if __WORDSIZE == 32
        ......
# define __STD_TYPE        __extension__ typedef
＃elif __WORDSIZE == 64
          ......
#endif
        ......
__STD_TYPE __PID_T_TYPE __pid_t;    /* Type of process identifications.  */
```
第二个替换：__pid_t ->  __extension__ typedef __PID_T_TYPE

:__PID_T_TYPE 在/usr/include/bits/typesizes.h
```bash

#define __PID_T_TYPE        __S32_TYPE


```
第三个替换 __PID_T_TYPE ->__S32_TYPE

:__S32_TYPE
/usr/include/bits/types.h
```bash
#define    __S32_TYPE        int
```
pid_  ---->  int  

我他妈...
绕这么大一圈 为啥不直接就是int呢？？


### **struct pid**


struct pid 是进程描述符，在include/linux/pid.h 中:

```bash

struct pid
{
        atomic_t count;
        unsigned int level;
        /* lists of tasks that use this pid */
        struct hlist_head tasks[PIDTYPE_MAX];
        struct rcu_head rcu;
        struct upid numbers[1];
};

```
### **task_struct**
task strcut 无论是在哪个实验中都是十分重要的一个存在，在这里详细讲一下task_struct结构体。
其定义在include/linux/sched.h中。
需要注意的是：
>task_struct包含很多内容：
1标示符： 描述本进程的唯一标识符，用来区别其他进程。
状态 ：任务状态，退出代码，退出信号等。\
优先级 ：相对于其他进程的优先级。\
程序计数器：程序中即将被执行的下一条指令的地址。\
内存指针：包括程序代码和进程相关数据的指针，还有和其他进程共享的内存块的指针。\
上下文数据：进程执行时处理器的寄存器中的数据。\
I/O状态信息：包括显示的I/O请求,分配给进程的I/O设备和被进程使用的文件列表。\
记账信息：可能包括处理器时间总和，使用的时钟数总和，时间限制，记账号等。 

另外，task_strcut是以链表的形式存在内核中的。\
本文仅略微描述功能和代码，具体`task_strcut` 我会另外新一篇博客中详细写。 
#### 1.状态
`volatile long state` 进程状态中可以看到在操作系统理论课中有讲到的zombie进程，DEAD进程等等。
```c
#define TASK_RUNNING        0//进程要么正在执行，要么准备执行

#define EXIT_ZOMBIE     16 //僵尸状态的进程，表示进程被终止，但是父进程还没有获取它的终止信息，比如进程有没有执行完等信息。                     
#define EXIT_DEAD       32 //进程的最终状态，进程死亡
/* in tsk->state again */ 
#define TASK_DEAD       64 //死亡
#define TASK_WAKEKILL       128 //唤醒并杀死的进程
#define TASK_WAKING     256 //唤醒进程
```
仅展示一部分 。

#### 2.进程标识（pid）

`pid_t pid` 
`pid_t tpid` \
pid是进程的唯一标识，tpid是领头线程的pid成员的值。

*领头线程* 是指：\
一个线程组中的所有线程使用和该线程组的领头线程（该组中的第一个轻量级进程）相同的PID，并被存放在tgid成员中。只有线程组的领头线程的pid成员才会被设置为与tgid相同的值。\
getpid()系统调用返回的是当前进程的tgid值而不是pid值。（线程是程序运行的最小单位，进程是程序运行的基本单位。）*此部分在后文详述。*

#### 3.进程标记

`unsigned int flags` \
此部分不进行详述，实验中涉及很少，在以后的task_struct详细解析。

#### 4.进程间的亲属关系

```c
struct task_struct *real_parent; /* real parent process */
struct task_struct *parent; /* recipient of SIGCHLD, wait4() reports */
struct list_head children;    /* list of my children */
struct list_head sibling;    /* linkage in my parent's children list */
struct task_struct *group_leader;    /* threadgroup leader */
```
**在Linux系统中，所有进程之间都有着直接或间接地联系，每个进程都有其父进程，也可能有零个或多个子进程。拥有同一父进程的所有进程具有兄弟关系。**

>real_parent指向其父进程，如果创建它的父进程不再存在，则指向PID为1的init进程。\
parent指向其父进程，当它终止时，必须向它的父进程发送信号。它的值通常与** real_parent**相同。\
children表示链表的头部，链表中的所有元素都是它的子进程（进程的子进程链表）。\
sibling用于把当前进程插入到兄弟链表中（进程的兄弟链表）。
group_leader指向其所在进程组的领头进程。

|成员|描述|
|:---:|:---:|
|real_parent|指向当前操作系统执行进程的父进程，如果父进程不存在，指向pid为1的init进程|
|parent|指向当前进程的父进程，当当前进程终止时，需要向它发送wait4()的信号|
|children|表示链表的头部，链表中的所有元素都是它的子进程（进程的子进程链表）。|
|sibling|用于把当前进程插入到兄弟链表中（进程的兄弟链表）|
|group_leader|指向其所在进程组的领头进程。|

在后续的遍历进程的实验中，task_struct中此部分应用比较广泛。

#### 5.进程的调度信息

```c
 int prio, static_prio, normal_prio;
 unsigned int rt_priority;
 const struct sched_class *sched_class;
 struct sched_entity se;
 struct sched_rt_entity rt;
 unsigned int policy; 
```
实时优先级范围是`0到MAX_RT_PRIO-1（即99）`，而普通进程的静态优先级范围是从`MAX_RT_PRIO到MAX_PRIO-1（即100到139）`。值越大静态优先级越`低`。

`static_prio`保存的是静态优先级。
`rt_priority`保存实时优先级
`normal_prio`值的大小取决于`静态优先级`和`调度策略`(进程的调度策略有：先来先服务，短作业优先、时间片轮转、高响应比优先等等的调度算法。
`prio`保存动态优先级。我们通过nice值修改的也是这个优先级。算法是`prio+=nice`。所以**nice值修改进程优先级的时候，一般赋给负值来提高优先级**。
`policy`保存进程的调度策略。
>调度策略：
```c
#define SCHED_NORMAL        0//按照优先级进行调度（有些地方也说是CFS调度器）
#define SCHED_FIFO        1//先进先出的调度算法
#define SCHED_RR        2//时间片轮转的调度算法
#define SCHED_BATCH        3//用于非交互的处理机消耗型的进程
#define SCHED_IDLE        5//系统负载很低时的调度算法
#define SCHED_RESET_ON_FORK     0x40000000
```
#### 6.ptrace系统调用
跟踪进程时应用，此处不做详述。
#### 7.时间数据成员
```c
cputime_t utime, stime, utimescaled, stimescaled;
    cputime_t gtime;
    cputime_t prev_utime, prev_stime;//记录当前的运行时间（用户态和内核态）
    unsigned long nvcsw, nivcsw; //自愿/非自愿上下文切换计数
    struct timespec start_time;  //进程的开始执行时间    
    struct timespec real_start_time;  //进程真正的开始执行时间
    unsigned long min_flt, maj_flt;
    struct task_cputime cputime_expires;//cpu执行的有效时间
    struct list_head cpu_timers[3];//用来统计进程或进程组被处理器追踪的时间
    struct list_head run_list;
    unsigned long timeout;//当前已使用的时间（与开始时间的差值）
    unsigned int time_slice;//进程的时间片的大小
    int nr_cpus_allowed;
```
#### 8.进程地址空间
```bash
struct mm_struct *mm, *active_mm;
/* per-thread vma caching */
u32 vmacache_seqnum;
struct vm_area_struct *vmacache[VMACACHE_SIZE];
#if defined(SPLIT_RSS_COUNTING)
struct task_rss_stat    rss_stat;
#endif

#ifdef CONFIG_COMPAT_BRK
unsigned brk_randomized:1;
#endif
```
在打印内核线程时候，用到的就是mm==NULL的条件判断的。

|成员|描述|
|:---:|:---:|
|mm|进程所拥有的内存空间描述符，对于内核线程的mm为NULL|
|active_mm|指进程运行时所使用的进程描述符|
|rss_stat|被用来记录缓冲信息|


**注意：如果当前内核线程被调度之前运行的也是另外一个内核线程时候，那么其mm和avtive_mm都是NULL。** 

### **find_get_pid**

有了中间很长一部分的task_struct的解析，一下部分应该会容易理解很多。

`find_get_pid`
```c
struct pid *find_get_pid(pid_t nr)  
{  
    struct pid *pid; 
    rcu_read_lock();  
    pid = get_pid(find_vpid(nr));  
    rcu_read_unlock();  
    return pid;  
} 
```
**find_get_pid(pid_t nr)** 是通过进程号pid_t nr得到进程描述符，并将结构体中的count加1.

在这里，`find_vpid`返回的是进程描述符。`get_pid`是让count+1。

总之，kpid返回的是一个pid类型的结构体，其定义在task_struct中。

### **pid_task**

`pid_task(pid* pid,PIDTYPE_PID)`可以通过进程描述符找到task_struct* task,简而言之，就是返回当前进程标识符的task_struct结构体。

### **task_nice**
```c
static inline int task_nice(const struct task_struct *p)
{
return PRIO_TO_NICE((p)->static_prio);
}
```
**通过当前task的静态优先级的值减去DEFAULT_PRIO。DEFAULT_PRIO在前面的博文中已经分析过，其值是120.**

### **task_prio**
同理。

### **copy_to_user**
由于内核空间与用户空间的内存不能直接互访，所以用此函数。
```c
// ./include/linux/uaccess.h
copy_to_user(void __user *to, const void *from, unsigned long n)
{
	if (likely(check_copy_size(from, n, true)))
		n = _copy_to_user(to, from, n);
	return n;
}
/*
 to	目标地址	用户空间的地址；
 from	源地址	内核空间的地址；
 n	将要拷贝的数据的字节数。
*/
```

## **modinicesyscall**

前半部分和get基本相同，就是后边加上了set的函数。
```c
//Modify nice
 
   printk("Before: nice value :%d and prio value%d\n",nice_before,prio_before);
   set_user_nice(task, nicevalue);//Modify nicevalue
   printk("After : nice value : %d\n ",nicevalue);
 
 
 
   return 0;  
 
}

```

### **set_user_nice**
set_user_nice源码解释起来比较复杂，简言之，根据 task_struct 确定一个进程，并改变进程 nice 值。


```c
void set_user_nice(struct task_struct *p, long nice)
{
    bool queued, running;
    int old_prio, delta;
    struct rq_flags rf;
    struct rq *rq;
    if (task_nice(p) == nice || nice < MIN_NICE || nice > MAX_NICE) //要修改的Nice值与当前进程的nice值相同或超出了范围，则直接返回
        return;
    rq = task_rq_lock(p, &rf);       //将队列上锁
    update_rq_clock(rq);           //更新cpu clock
    if (task_has_dl_policy(p) || task_has_rt_policy(p)) //如果该进程是实时进程，那就开始改变该进程nice值
	{ 
        p->static_prio = NICE_TO_PRIO(nice);  //根据nice值计算优先级
        goto out_unlock;            //去锁
    }
    queued = task_on_rq_queued(p);  //判断该进程是否在rq队列中
    running = task_current(rq, p);     //判断p进程是否在运行，如果在队列中，取出该进程
    if (queued)                    //如果该进程在队列中
        dequeue_task(rq, p, DEQUEUE_SAVE | DEQUEUE_NOCLOCK);  //取出该进程
    if (running)                   //如果该进程正在运行
        put_prev_task(rq, p);

    p->static_prio = NICE_TO_PRIO(nice);  //根据Nice值计算进程优先级
    set_load_weight(p);                  //计算权重
    
    /*
*以下三句是将优先级做差
*/
    old_prio = p->prio;           
    p->prio = effective_prio(p);       
    delta = p->prio - old_prio;
    if (queued) {
        enqueue_task(rq, p, ENQUEUE_RESTORE | ENQUEUE_NOCLOCK);
        if (delta < 0 || (delta > 0 && task_running(rq, p)))  //如果差值小于0（优先级提高）或，重新调度rq队列
            resched_curr(rq);
    }
    if (running)
        set_curr_task(rq, p);
		out_unlock:
    	task_rq_unlock(rq, p, &rf);
}
```

## **myapi**

说来惭愧..
当初想打印计算机信息结果能力超出范围，于是就变成了一个math库里的函数
return m^2

太惭愧了。

## Query

### nicevalue的作用：

在设计这个get和modify的时候，本意是想着两个放到一起，用flag的形式装上去，然后听完课之后突然发现功能模块化也是很不错的，然后就在原来flag的基础上改了一下，两个系统调用用的都是传进来的`nicevalue`。

**实际上在getnicesyscall中的nicevalue没有任何作用**

这是我在写代码时犯下的错误，不建议模块化的同学们这样写。

此外，在传递参数时候一定考虑到，分模块的功能就是为了在每个子函数能够使用尽量少的参数，所以一定要精简。

### 第二个花括号：
现在已经注释掉了，原来在从报告向md文档粘贴时候少了后半段代码，我就直接给后面加了一个花括号，后来在检查的时候发现get的后半段没粘贴上就手动又重新粘贴了后半段，中间的`}`就没删..\
太惭愧了。

### 传递的pid不存在：

对于程序健全性来说，`pid`不为空的检查是必要的，但是很多代码都没这么写，包括我在内。\
后来在各类资料中查找才知道，如果用户给出的pid没有与之对应的进程，find_get_pid会返回一个NULL指针。

### 关于copy_to_user的再理解
内核和用户空间应用程序具有不同的地址空间，因此复制到用户空间需要更改地址空间。每个进程都有自己的（用户）地址空间。
另外，复制到用户空间时，内核不应该崩溃，所以`copy_to_user`函数可能会检查目标地址是否有效（也许该地址应该从分区空间中分页）。

### getpid和putpid

getpid的作用就是获得当前进程的进程标识符（pid）\
putpid的源码：

```c
void put_pid(struct pid *pid)
{
	struct pid_namespace *ns;

	if (!pid)
		return;

	ns = pid->numbers[pid->level].ns;
	if ((atomic_read(&pid->count) == 1) ||
	     atomic_dec_and_test(&pid->count)) {
		kmem_cache_free(ns->pid_cachep, pid);
		put_pid_ns(ns);
	}
}
```

在每次get_pid时候，最后最好都要put_pid一下，因为由get_pid的含义可以知道，应用`atomic_inc(&pid->count);`每次都会给task中count字段自增1，而put_pid则是`atomic_dec_and_test(&pid->count)`用来减1来达到释放资源的目的（**引用计数，当被引用一次就自增1，每次调用pud_pid用来给count字段减1，最后字段为0时候释放资源**）