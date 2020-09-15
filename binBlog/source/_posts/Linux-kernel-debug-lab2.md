---
title: 操作系统课程设计 实验二 代码详解
date: 2020-03-22 22:17:37
tags: Linux
top_img: https://images.ryanhor.com/OS%20lab2.png
cover: https://images.ryanhor.com/OS%20lab2.png
---

# **实验二**
## **预备知识:Makefile以及模块化编程**

## **模块一 列出内核线程的程序名，pid号、进程状态以及进程优先级**

### **原理** 
在上个实验中已经对task_struct结构体有了初步的认识，知道在task_strcut中有关于`进程地址空间`、`进程状态`、`进程优先级`、`pid`的说明以及定义。
#### 进程地址空间说明
```c
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
本次实验应用到的特性是当结构体`mm`内容为空时表示的是内核线程。借此可以循环遍历进程链表，筛选出其中`task->mm == NULL `的进程直接打印即可。
#### 进程状态说明
```c
#define TASK_RUNNING		0
#define TASK_INTERRUPTIBLE	1
#define TASK_UNINTERRUPTIBLE	2
#define __TASK_STOPPED		4
#define __TASK_TRACED		8
/* in tsk->exit_state */
#define EXIT_ZOMBIE		16
#define EXIT_DEAD		32
/* in tsk->state again */
#define TASK_DEAD		64
#define TASK_WAKEKILL		128
#define TASK_WAKING		256
```
在实际测验中，多数进程的进程状态为1，即`阻塞态`，对应OS理论课中三态转换中的阻塞态：直到某个条件变为真（资源得到）。条件一旦达成，进程的状态就被设置为TASK_RUNNING。

#### 进程优先级说明和PID
此处在lab1已经有详细介绍，在此不做多述。\
打印出`task->pid`，`task -> prio`即可。
### **流程**
![module1](https://ae01.alicdn.com/kf/U8803ef28bce4408c8e99255014abcb25B.png)
### **代码实现**
```c
#include <linux/module.h>
#include <linux/init.h>
#include <linux/kernel.h>
#include <linux/sched.h>
#include <linux/init_task.h>


static int kprocprnt_init(void)
{
    struct task_struct *task;
    printk(KERN_ALERT "Hello This function was make by Ryan.\n");
    printk(KERN_ALERT "名称\t进程号\t状态\t优先级\t");


    for (task = &init_task; (task = next_task(task)) != &init_task;)
    {
        // kernel thread
        if (task->mm == NULL)
        {
            printk(KERN_ALERT  "%s\t%d\t%ld\t%d\n",
            task->comm, task->pid, task->state, task->prio);
        }
    }
    return 0;
}

static void kprocprnt_exit(void)
{
    printk(KERN_ALERT "Have a good time!\n");
}

MODULE_LICENSE("GPL");
MODULE_AUTHOR("Ryan");
MODULE_DESCRIPTION("Kernel Process Print");
module_init(kprocprnt_init);
module_exit(kprocprnt_exit);
```
### **代码详解**
#### 头文件
`
#include <linux/module.h> `\
这个头文件包含了许多符号与函数的定义，这些符号与函数多与加载模块有关,其中包括描述性的声明
`MODULE_LICENSE("GPL");`和`MODULE_AUTHOR`、`MODULE_DESCRIPTION`等等 \
`
#include <linux/init.h>`\
`#include <linux/init_task.h>`\
顾名思义，init&exit相关的头文件。\
`#include <linux/sched.h>`\
task_struct头文件。\
``
#### **for_each_process(循环遍历)**
```c
for (task = &init_task; (task = next_task(task)) != &init_task;)
```
此处实际上是for_each_process的宏替换内容，其作用就是从0号进程向下遍历所有进程。（定义在include/linux/sched.h中）
### **注意问题**
#### **MODULE_LICENSE("GPL")**
此处为模块的许可证声明，是**不可缺少的**，因为从2.4.10版本内核开始，模块必须通过MODULE_LICENSE宏声明此模块的许可证，否则在加载此模块时，会收到内核被污染“kernel tainted” 的警告。而AUTHOR和DESCRIPE都是可以省略不写的，签名也是可以忽略的.(但是仍然建议在模块编写是要加上作者信息，用以署名。)
#### **KERN_ALERT**
KERN_ALERT代表log的级别。在/include/linux/kern_levels.h中
```c
#define KERN_SOH	"\001"		/* ASCII Start Of Header */
#define KERN_SOH_ASCII	'\001'

#define KERN_EMERG	KERN_SOH "0"	/* system is unusable */
#define KERN_ALERT	KERN_SOH "1"	/* action must be taken immediately */
#define KERN_CRIT	KERN_SOH "2"	/* critical conditions */
#define KERN_ERR	KERN_SOH "3"	/* error conditions */
#define KERN_WARNING	KERN_SOH "4"	/* warning conditions */
#define KERN_NOTICE	KERN_SOH "5"	/* normal but significant condition */
#define KERN_INFO	KERN_SOH "6"	/* informational */
#define KERN_DEBUG	KERN_SOH "7"	/* debug-level messages */

#define KERN_DEFAULT	""		/* the default kernel loglevel */
```
这也是在dmesg时候出现红色底的原因。
### **Query**
留给提问更新的区间。
</br>
## **模块二 设计带参数为PID号的模块，列出进程的家族信息以及PID和程序名**
### **原理** 
利用task_struct结构体，以传入PID为参数找到对应的task_struct结构体，结合list_entry进入不同的家属链表头进行遍历和打印。其中进程信息的PID和程序名在模块一已有陈述。
#### **task_struct家庭成员**

task_struct结构体中有关于家庭成员的介绍。
```c
struct task_struct *real_parent; /* real parent process */
struct task_struct *parent; /* recipient of SIGCHLD, wait4() reports */
struct list_head children;    /* list of my children */
struct list_head sibling;    /* linkage in my parent's children list */
struct task_struct *group_leader;    /* threadgroup leader */
```
不做重述。
### **流程**
![modules2](https://ae01.alicdn.com/kf/Uf93c0bdf22ce46f2854893dcc480f7c0b.png)
### **代码实现**
```c
#include <linux/module.h>
#include <linux/init.h>
#include <linux/kernel.h>
#include <linux/sched.h>
#include <linux/init_task.h>
#include <linux/moduleparam.h>
#include <linux/pid.h>
#include <linux/list.h>


static int pid;
module_param(pid, int, 0755);

static int procfaminfo_init(void)
{
    struct task_struct *task = pid_task(find_vpid(pid), PIDTYPE_PID);
    printk(KERN_ALERT "Here are Process family infomation echo:\n");
    if(task == NULL){
        printk(KERN_ALERT "Sorry,PID seems to be invaid\n");
        return -EFAULT;
    }
    struct task_struct *parent = task->parent;
    struct task_struct *p = NULL;//attention

    struct list_head *sibling = NULL;
    struct list_head *child = NULL;

    printk(KERN_ALERT "父进程:%s,pid:%d\n", parent->comm, parent->pid);
    printk(KERN_ALERT "该进程:%s,pid:%d\n", task->comm, task->pid);
    list_for_each(sibling, &task->sibling)
    {
        p = list_entry(sibling, struct task_struct, sibling);
        if (p->pid != pid)//attention
        {
            printk(KERN_ALERT "兄弟进程:%s,pid:%d\n", p->comm, p->pid);
        }
    }

    list_for_each(child, &task->children)
    {
        p = list_entry(child, struct task_struct, sibling);//attention
        printk(KERN_ALERT "子进程:%s,pid:%d\n", p->comm, p->pid);
    }

    return 0;
}

static void procfaminfo_exit(void)
{
    printk(KERN_ALERT "All family members done.Have a good day then.\n");
}

MODULE_LICENSE("GPL");
MODULE_AUTHOR("Ryan");
MODULE_DESCRIPTION("Process Family Info Print");
module_init(procfaminfo_init);
module_exit(procfaminfo_exit);
```
### **代码详解**
#### **头文件**
新增加的
```c
#include <linux/moduleparam.h>
#include <linux/pid.h>
#include <linux/list.h>
```
这三个头文件，分别对应传递参数、getpid(putpid)、list_for_each和list_entry的依赖。
#### **module_param**
`module_param(pid, int, 0755);`\
在源码中可以看见
```c
 #define module_param(name, type, perm)               
 module_param_named(name, name, type, perm)
```
需要注意的是，`name `是参数名，也是模块内接受参数的变量，在此模块内是进程的PID。`type`是参数的数据类型，可以是int，uint，short，ushort,long，ulong，bool，byte，charp，invboll之一。\
(or XXX if you define param_get_XXX)
`perm`字段是权限赋予（权限掩码），用来做一个辅助的sysfs入口。本例为755.即`rwx-rx-rx`。
#### **-EFAULT**
- -EFAULT It happen if the memory address of some argument passed to sendto (or more generally to any system call) is invalid. \
**在用户传入错误的PID会提示Bad Address.**
#### **list_head**
```c
struct list_head {            //list_head不是拿来单独用的，它一般被嵌到其它结构中
	struct list_head *next, *prev;
};
```
需要注意，list_head这个结构看起来怪怪的，它没有数据域，
list_head不是拿来单独用的，它一般被嵌到其它结构中，如：
```c
struct file_node{

　　char c;

　　struct list_head node;

};
```
此时list_head就作为它的父结构中的一个成员了，当我们知道`list_head的地址（指针）`时，我们可以通过list.c提供的宏 `list_entry `来获得它的父结构的地址。
#### **list_entry**
list_entry的源码解读涉及三个宏的实现，在此不做多述，如果有需要我会再开一篇具体解释,或者参考[list_head](https://www.cnblogs.com/zhuyp1015/archive/2012/06/02/2532240.html)。简单来说：\
list_entry(ptr,type,member)宏的功能就是，由结构体成员地址求结构体地址。其中`ptr`是所求结构体中list_head成员指针，`type`是所求结构体类型，`member`是结构体list_head成员名。
list_entry表示在找出ptr指向的链表节点所在的type类型的结构体首地址，member时type类型结构体成员。
#### **list_for_each**
list_for_each的源码
```c
#define list_for_each(pos, head) \
	for (pos = (head)->next; prefetch(pos->next), pos != (head); \
        	pos = pos->next)
* @pos:	the &struct list_head to use as a loop cursor.
* @head: the head for your list.
```
`list_for_each`实际上是一个 for 循环，利用传入的pos 作为循环变量，从表头 head开始，逐项向后（next方向）移动 pos ，直至又回到 head （prefetch() 可以不考虑，用于预取以提高遍历速度）。
**注意：此宏必要把list_head放在数据结构第一项成员，至此，它的地址也就是结构变量的地址。**

### **注意问题**
#### *struct task_struct \*p = NULL;//attention*
此结构体p用于存放list_head所找到的兄弟进程或者是子进程，用以遍历兄弟进程/子进程链表。
#### *if (p->pid != pid)//attention*
当寻找兄弟进程时，可以看出list_head并没有将task本身的进程去除，所以打印时需要加判断条件`siblings(p)->pid!=pid`,用以保证打印的兄弟节点不包含本身。
#### *p = list_entry(child, struct task_struct, sibling);//attention*
在上方的`list_for_each(child,&task->children)`中已经将list_head更新为当前task的子进程链表头，从此处开始遍历打印即当前进程的子进程，主要注意的是，此处不需要判断pid的情况，因为task本身的pid不可能是子进程的pid。\
此外：为什么在遍历时要传入sibling成员，是因为，sibling成员在源码的解释为`linkage in the parents' children`。这是指当当前节点为task->children的时候，要遍历其父节点的所有子节点就要给定一个子节点链表的链表头，而利用list_entry()找到子进程链表头，sibling承担着找到子节点链表项的功能。
### **Query**
留给提问更新的区间。
</br>