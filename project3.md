# Linux共筆 project3
> Contributor: Yung-peng Hsu, Cheng-Kai Wang
## Problem
[Link](https://staff.csie.ncu.edu.tw/hsufh/COURSES/FALL2021/linux_project_3.html)

## Question 1
### kernel space code
```c=
#include <linux/init.h>
#include <linux/module.h>
#include <linux/kernel.h>
#include <linux/string.h>
#include <linux/uaccess.h>
#include <linux/init_task.h>
#include <linux/syscalls.h>
#include <asm/errno.h>

SYSCALL_DEFINE1(context_switches, unsigned int*, w)
{
    unsigned int switch_counts;
    switch_counts = current->nivcsw+current->nvcsw;
    if(copy_to_user(w, &switch_counts, sizeof(unsigned int))){
        return -EFAULT;
    }

    return 0;
}
```
在include/linux/sched.h的task_struct中用兩個counter紀錄context switch發生的次數:
1. nvcsw: Number of Voluntary Context Switches
2. nivcsw: Number of InVoluntary Context Switches
```c=
switch_count = &prev->nivcsw;
if (!preempt && prev->state) {
    ...
    // 若非preempt 則為Voluntary Context Switches
    switch_count = &prev->nvcsw;
}
rq->nr_switches++;
rq->curr = next;
++*switch_count;

trace_sched_switch(preempt, prev, next);

rq = context_switch(rq, prev, next, &rf);
```
scheduler會決定目前是要由哪個process是被CPU執行。(在kernel/sched/core.c中的[schedule()](https://elixir.bootlin.com/linux/v4.15.1/source/kernel/sched/core.c#L3427))
trace路徑:schedule()->__schedule()->context_switch()->[switch_to](https://elixir.free-electrons.com/linux/v3.9/source/arch/x86/include/asm/switch_to.h#L31)

switch_to是一個macro，傳入的參數有三個(prev, next, last)，分別是process自己本身、要轉交控制權給下一個的process、最後將控制權還給一開始的process。

當中
```c=
movl %%esp, %[prev_sp]\n\t
```
把esp指到拿到CPU使用權的process的kernel mode stack頂端的位置
```c=
movl %[next_sp],%%esp\n\t
```
這時將控制權交還給原process
### User space code
```c=
#include <stdio.h>
#include <syscall.h>
#include<unistd.h>
#define  NUMBER_OF_ITERATIONS     99999999
int main ()
{
        int i,t=2,u=3,v;
        unsigned int w;

        for(i=0; i<NUMBER_OF_ITERATIONS; i++)
            v=(++t)*(u++);

        if(syscall(335, &w) != 0)
            printf("Error!\n");
        else
            printf("This process encounters %u times context switches.\n", w);
}       
```

### 執行結果
![](https://i.imgur.com/3dGnznq.png)


Process狀態圖:
![](https://i.imgur.com/Laxqhn2.png)
Linux系統中是由forK()建立新的process，process建立後，狀態會是TASK_RUNNING，指的是被放在task queue/run queue（OS叫 ready queue）中的process。
需經過context switch後，才會變成是正在執行中的process(current)。當current正在執行時，可能會被priority更高的process搶奪執行權，此時process會回到ready queue。





## Question 2
### kernel space code
```c=
#include <linux/init.h>
#include <linux/module.h>
#include <linux/kernel.h>
#include <linux/string.h>
#include <linux/uaccess.h>
#include <linux/init_task.h>
#include <linux/syscalls.h>
#include <asm/errno.h>

SYSCALL_DEFINE1(enter_queue, unsigned int*, w)
{
    unsigned int enter_queue_counts;
    enter_queue_counts = current->enter_queue_counter;
    if(copy_to_user(w, &enter_queue_counts, sizeof(unsigned int))){
        return -EFAULT;
    }

    return 0;
}

```
### user space code
```c=
#include <stdio.h>
#include <syscall.h>
#include <unistd.h>

#define  NUMBER_OF_IO_ITERATIONS     6
#define  NUMBER_OF_ITERATIONS        99999999

int main ()
{
    char c;
    int i,t=2,u=3,v;
    unsigned int w;

    for(i=0; i<NUMBER_OF_IO_ITERATIONS; i++)
    {
        v=1;
        c = getchar();
    }

    for(i=0; i<NUMBER_OF_ITERATIONS; i++)
        v=(++t)*(u++);

    if(syscall(335, &w)!=0)
        printf("Error (1)!\n");
    else
        printf("This process encounters %u times context switches.\n", w);

    if(syscall(336, &w)!=0)
        printf("Error (2)!\n");
    else
        printf("This process enters a wait queue %u times.\n", w);


    for(i=0; i<NUMBER_OF_IO_ITERATIONS; i++)
    {
        v=1;
        printf("I love my home.\n");
    }


    if(syscall(336, &w)!=0)
        printf("Error (3)!\n");
    else
	printf("This process enters a wait queue %u times.\n", w);
}
```

在include/linux中的task_struct新增一個變數enter_queue_counter，用來計算總共進入waiting queue幾次。
:::info
如果將變數加到task_struct中的任意位置，會產生問題。因為context switch是靠固定的位移記憶體大小來得到task_struct其中的變數，交換process的
:::

```c=
...
#ifdef CONFIG_SECURITY
        /* Used by LSM modules for access restriction: */
        void                            *security;
#endif
        unsigned int enter_queue_counter; //here

        /*
         * New fields for task_struct should be added above here, so that
         * they are included in the randomized portion of task_struct.
         */
        randomized_struct_fields_end
...
```

在/kernel/fork.c的copy_mm中進行enter_queue_counter的初始化
```c=
static int copy_mm(unsigned long clone_flags, struct task_struct *tsk)
{
        struct mm_struct *mm, *oldmm;
        int retval;

        tsk->enter_queue_counter = 0; //here
        tsk->min_flt = tsk->maj_flt = 0;
        tsk->nvcsw = tsk->nivcsw = 0;
#ifdef CONFIG_DETECT_HUNG_TASK
        tsk->last_switch_count = tsk->nvcsw + tsk->nivcsw;
#endif
...

```
欲計算進入waiting queue的次數，等同於計算有多少個process(task)執行wake up。
trace 路徑:  [default_wake_function](https://elixir.free-electrons.com/linux/v3.9/source/kernel/sched/core.c#L3109)->try_to_wake_up->ttwu_queue->ttwu_do_activate->[ttwu_do_wakeup](https://elixir.free-electrons.com/linux/v3.9/source/kernel/sched/core.c#L1289)。
每次執行wake up時，counter+1。
```c=
/*
 * Mark the task runnable and perform wakeup-preemption.
 */
static void ttwu_do_wakeup(struct rq *rq, struct task_struct *p, int wake_flags,
                           struct rq_flags *rf)
{
        check_preempt_curr(rq, p, wake_flags);
        p->enter_queue_counter++; //here
        p->state = TASK_RUNNING;
        trace_sched_wakeup(p);

#ifdef CONFIG_SMP
...
```
### 輸出結果
![](https://i.imgur.com/YSBl5LW.png)

![](https://i.imgur.com/pYPUWdx.png)

## Reference
[Context Switch](http://blake31113.github.io/2014/12/30/%5BLinux-Kernel%5D%5B紀錄Process%20Context%20Switch次數和idle%20time%5D/)
