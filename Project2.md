# A Writeup of Project 2
```
Contributor: Cheng-Kai Wang, Ryan Wang, Yung-Peng Hsu
```
## Problem
[Link](https://staff.csie.ncu.edu.tw/hsufh/COURSES/FALL2021/linux_project_2.html)
### Kernel Code
```c=
#include <linux/init.h>
#include <linux/module.h>
#include <linux/kernel.h>
#include <linux/string.h>
#include <linux/uaccess.h>
#include <linux/init_task.h>
#include <linux/syscalls.h>



static unsigned long vaddr2paddr(unsigned long vaddr)
{
    pgd_t *pgd;
    pud_t *pud;
    pmd_t *pmd;
    pte_t *pte;
    p4d_t *p4d;

    unsigned long paddr = 0;
    unsigned long page_addr = 0;
    unsigned long page_offset = 0;
    struct page* page;
    struct task_struct *task;

    pgd = pgd_offset(current->mm, vaddr);
    printk("pgd_val = 0x%lx\n", pgd_val(*pgd));
    printk("pgd_index = %lu\n", pgd_index(vaddr));
    if (pgd_none(*pgd)) {
        printk("not mapped in pgd\n");
        return -1;
    }

    p4d = p4d_offset(pgd, vaddr);
    printk("p4d_val = 0x%lx\n", p4d_val(*p4d));
    printk("p4d_index = %lu\n", p4d_index(vaddr));
    if (p4d_none(*p4d)) {
        printk("not mapped in p4d\n");
        return -1;
    }


    pud = pud_offset(p4d, vaddr);
    printk("pud_val = 0x%lx\n", pud_val(*pud));
    if (pud_none(*pud)) {
        printk("not mapped in pud\n");
        return -1;
    }

    pmd = pmd_offset(pud, vaddr);
    printk("pmd_val = 0x%lx\n", pmd_val(*pmd));
    printk("pmd_index = %lu\n", pmd_index(vaddr));
    if (pmd_none(*pmd)) {
        printk("not mapped in pmd\n");
        return -1;
    }

    pte = pte_offset_kernel(pmd, vaddr);
    printk("pte_val = 0x%lx\n", pte_val(*pte));
    printk("pte_index = %lu\n", pte_index(vaddr));
    if (pte_none(*pte)) {
        printk("not mapped in pte\n");
        return -1;
    }

    /* Page frame physical address mechanism | offset */
    page = pte_page(*pte);
    paddr = page_to_phys(page) + (vaddr & (~PAGE_MASK));
    printk("page_addr = %lx, page_offset = %lx\n", page_addr, page_offset);
        printk("vaddr = %lx, paddr = %lx\n", vaddr, paddr);

    return paddr;
}

SYSCALL_DEFINE2(getpaddr, unsigned long, vaddr, unsigned long*, result)
{
    unsigned long addr;
    addr = vaddr2paddr(vaddr);
    copy_to_user(result, &addr, sizeof(unsigned long));
    return 1;
}
```

* SYSCALL_DEFINEn(syscall_name ), n需與後面傳入參數量一致，若不相同會出現"error: macro "__MAP2" requires 4 arguments, but only 2 given" 的錯誤


## user space code
* User space的code須以以下指令編譯：gcc hello.c -lpthread -o hello
```c=
#include<syscall.h>
#include<sys/types.h>
#include<stdio.h>
#include<unistd.h>
#include<time.h>
#include<fcntl.h>
#include<stdint.h>
#include<stdio.h>
#include<stdlib.h>


int globalVar = 0;
int bss;
unsigned long heap_addr;

void* thread1(void* args);
void* thread2(void* args);

int main()
{
	int *heap = malloc(sizeof(int));
	*heap = 888;
	heap_addr = (unsigned long)heap;
	printf("=========main==============\n");
	int n = 0;
	unsigned long result = 0;
	printf("==========virtual address===========\n");
	printf("code: 0x%-16lx, data: 0x%-16lx, bss: 0x%-16lx, heap: 0x%-16lx\n", (unsigned long)main, (unsigned long)&globalVar, (unsigned long)&bss, (unsigned long)heap_addr);
	printf("share_lib: 0x%-16lx, stack: 0x%-16lx\n", (unsigned long)printf, (unsigned long)&n);
	printf("=========physical address===========\n");
	int a = syscall(334, (unsigned long)main, &result);
	printf("code: 0x%-16lx, ", result);
        a = syscall(334, (unsigned long)&globalVar, &result);
	printf("data: 0x%-16lx, ", result);
        a = syscall(334, (unsigned long)&bss, &result);
        printf("bss: 0x%-16lx, ", result);
	a = syscall(334, (unsigned long)heap_addr, &result);
        printf("heap: 0x%-16lx\n", result);
	a = syscall(334, (unsigned long)printf, &result);
        printf("share_lib: 0x%-16lx, ", result);
	a = syscall(334, (unsigned long)&n, &result);
        printf("stack: 0x%-16lx\n", result);

	pthread_t p1, p2;
	pthread_create(&p1, NULL, thread1, 0);
	pthread_create(&p2, NULL, thread2, 0);

	pthread_join(p1, NULL);
	pthread_join(p2, NULL);

	return 0;
}

void* thread1(void* args)
{
	int n = 0;
	int *heap = malloc(sizeof(int));
        *heap = 888;
        heap_addr = (unsigned long)heap;
        unsigned long result = 0;
	printf("==========Thread 1===============\n");
        printf("==========virtual address===========\n");
        printf("code: 0x%-16lx, data: 0x%-16lx, bss: 0x%-16lx, heap: 0x%-16lx\n", (unsigned long)main, (unsigned long)&globalVar, (unsigned long)&bss, (unsigned long)heap_addr);
        printf("share_lib: 0x%-16lx, stack: 0x%-16lx\n", (unsigned long)printf, (unsigned long)&n);
        printf("=========physical address===========\n");
        int a = syscall(334, (unsigned long)main, &result);
        printf("code: 0x%-16lx, ", result);
        a = syscall(334, (unsigned long)&globalVar, &result);
        printf("data: 0x%-16lx, ", result);
        a = syscall(334, (unsigned long)&bss, &result);
        printf("bss: 0x%-16lx, ", result);
        a = syscall(334, (unsigned long)heap_addr, &result);
        printf("heap: 0x%-16lx\n", result);
        a = syscall(334, (unsigned long)printf, &result);
        printf("share_lib: 0x%-16lx, ", result);
        a = syscall(334, (unsigned long)&n, &result);
        printf("stack: 0x%-16lx\n", result);
	//while(1){};

	return 0;
}

void* thread2(void* args)
{
	sleep(3);
	int n = 0;
	int *heap = malloc(sizeof(int));
        *heap = 888;
        heap_addr = (unsigned long)heap;
        unsigned long result = 0;
	printf("=========thread2=============\n");
        printf("==========virtual address===========\n");
        printf("code: 0x%-16lx, data: 0x%-16lx, bss: 0x%-16lx, heap: 0x%-16lx\n", (unsigned long)main, (unsigned long)&globalVar, (unsigned long)&bss, (unsigned long)heap_addr);
        printf("share_lib: 0x%-16lx, stack: 0x%-16lx\n", (unsigned long)printf, (unsigned long)&n);
        printf("=========physical address===========\n");
        int a = syscall(334, (unsigned long)main, &result);
        printf("code: 0x%-16lx, ", result);
        a = syscall(334, (unsigned long)&globalVar, &result);
        printf("data: 0x%-16lx, ", result);
        a = syscall(334, (unsigned long)&bss, &result);
        printf("bss: 0x%-16lx, ", result);
        a = syscall(334, (unsigned long)heap_addr, &result);
        printf("heap: 0x%-16lx\n", result);
        a = syscall(334, (unsigned long)printf, &result);
        printf("share_lib: 0x%-16lx, ", result);
        a = syscall(334, (unsigned long)&n, &result);
        printf("stack: 0x%-16lx\n", result);
	//while(1){};
	return 0;
}
```

執行結果
```
=========main==============
==========virtual address===========
code: 0x5643594ba89a
data: 0x5643596bc014
bss: 0x5643596bc020
heap: 0x56435ab3c260    
share_lib: 0x7fee46acaf70
stack: 0x7ffd75c03090    
=========physical address===========
code: 0x112b3a89a
data: 0x1115cd014
bss: 0x1115cd020
heap: 0x111631260       
share_lib: 0x13fbbcf70
stack: 0x1115cc090       
==========Thread 1===============
==========virtual address===========
code: 0x5643594ba89a
data: 0x5643596bc014
bss: 0x5643596bc020
heap: 0x7fee40000b20    
share_lib: 0x7fee46acaf70
stack: 0x7fee46a64ed0    
=========physical address===========
code: 0x112b3a89a 
data: 0x1115cd014
bss: 0x1115cd020
heap: 0x1344a3b20       
share_lib: 0x13fbbcf70
stack: 0x111632ed0       
=========thread2=============
==========virtual address===========
code: 0x5643594ba89a
data: 0x5643596bc014
bss: 0x5643596bc020
heap: 0x7fee40000b40    
share_lib: 0x7fee46acaf70
stack: 0x7fee46263ed0    
=========physical address===========
code: 0x112b3a89a
data: 0x1115cd014 
bss: 0x1115cd020 
heap: 0x1344a3b40       
share_lib: 0x13fbbcf70
stack: 0x11296ced0       
```
### User Space Code
```c=
#include<syscall.h>
#include<sys/types.h>
#include<stdio.h>
#include<unistd.h>
#include<time.h>
#include<fcntl.h>
#include<stdint.h>
#include<stdio.h>
#include<stdlib.h>


int globalVar = 0;
int bss;
unsigned long heap_addr;

//void* thread1(void* args);
//void* thread2(void* args);

int main()
{
	
        int *heap = malloc(sizeof(int));
        *heap = 888;
        heap_addr = (unsigned long)heap;
        printf("=========main==============\n");
        int n = 0;
        unsigned long result = 0;
        printf("==========virtual address===========\n");
        printf("code: 0x%-16lx, data: 0x%-16lx, bss: 0x%-16lx, heap: 0x%-16lx\n", (unsigned long)main, (unsigned long)&globalVar, (unsigned long)&bss, (unsigned long)heap_addr);
        printf("share_lib: 0x%-16lx, stack: 0x%-16lx\n", (unsigned long)printf, (unsigned long)&n);
        printf("=========physical address===========\n");
        int a = syscall(334, (unsigned long)main, &result);
        printf("code: 0x%-16lx, ", result);
	a = syscall(334, (unsigned long)&globalVar, &result);
        printf("data: 0x%-16lx, ", result);
        a = syscall(334, (unsigned long)&bss, &result);
        printf("bss: 0x%-16lx, ", result);
        a = syscall(334, (unsigned long)&heap_addr, &result);
        printf("heap: 0x%-16lx\n", result);
        a = syscall(334, (unsigned long)printf, &result);
        printf("share_lib: 0x%-16lx, ", result);
        a = syscall(334, (unsigned long)&n, &result);
        printf("stack: 0x%-16lx\n", result);

        //pthread_t p1, p2;
        //pthread_create(&p1, NULL, thread1, 0);
        //pthread_create(&p2, NULL, thread2, 0);

        //pthread_join(p1, NULL);
        //pthread_join(p2, NULL);
        char s;
	scanf("%c", &s);
        return 0;
}


```

執行結果
```
process1

=========main==============
==========virtual address===========
code:     0x55e707cfe7ea
data:     0x55e707eff014
bss:      0x55e707eff020
heap:     0x55e709cc7260    
share_lib:0x7f27daf3df70
stack:    0x7ffeecb21f10    
=========physical address===========
code:     0x92d8a7ea
data:     0x1dd47014
bss:      0x1dd47020
heap:     0x1dd47018        
share_lib:0x13fbbcf70
stack:    0x93d5ff10 

process2
=========main==============
==========virtual address===========
code:     0x5630027a57ea 
data:     0x5630029a6014 
bss:      0x5630029a6020 
heap:     0x563004445260    
share_lib:0x7f67ee8fbf70
stack:    0x7ffc7dfc3340    
=========physical address===========
code:     0x92d8a7ea        
data:     0x92e28014
bss:      0x92e28020
heap:     0x92e28018        
share_lib:0x13fbbcf70
stack:    0x944d8340  
```
* current in linux是include/linux/sched.h
```
extern struct task_struct *current_set[NR_CPUS];
/*
 *	On a single processor system this comes out as current_set[0] when cpp
 *	has finished with it, which gcc will optimise away.
 */
#define current (0+current_set[smp_processor_id()])	/* Current on this processor */
```







## Q&A
1. 當程式調用memory allocation的方法，如：malloc、new時，OS會如何處理？
答: Process分配memory有兩種方式：分別由brk和mmap完成。brk是將data段的最高地址指針_edata往高地址推，mmap是在process的虛擬地址空間中，找一塊空閒的虛擬空間
2. 說明multithread和multiprocess記憶體共用情形並畫圖。
答: multithread除了stack與heap段不共用外，其餘均共用。 multiprocess除了code與share library段共用外，其餘均不共用。
![](https://i.imgur.com/DX8u3Up.jpg)


3. 說明當程式執行pthread_create時，system call的執行流程。
答: 
* 使用strace看使用到的system call(mmap, mprotect, clone)
* 接著看clone的code，呼叫到_do_fork
```
SYSCALL_DEFINE5(clone, unsigned long, clone_flags, unsigned long, newsp,
                int __user *, parent_tidptr,
                int __user *, child_tidptr,                                                                                          
                unsigned long, tls)         
{                
       return _do_fork(clone_flags, newsp, 0, parent_tidptr, child_tidptr, tls);
}      
```
* 接著看_do_fork，呼叫到copy_process來複製一個新的process
```
/*
 *  Ok, this is the main fork-routine.
 *
 * It copies the process, and if successful kick-starts
 * it and waits for it to finish using the VM if required.
 */
long do_fork(unsigned long clone_flags,
	      unsigned long stack_start,
	      struct pt_regs *regs,
	      unsigned long stack_size,
	      int __user *parent_tidptr,
	      int __user *child_tidptr)
{
	struct task_struct *p; // 宣告一個 process descriptor
	int trace = 0;
	struct pid *pid = alloc_pid(); // 要求一個 PID 給新的 process 使用
	long nr;

	if (!pid)
		return -EAGAIN;
	nr = pid->nr;
	if (unlikely(current->ptrace)) {
		trace = fork_traceflag (clone_flags);
		if (trace)
			clone_flags |= CLONE_PTRACE;
	}
	// 呼叫 copy_process()，以複制出新的 process
	p = copy_process(clone_flags, stack_start, regs, stack_size, parent_tidptr, child_tidptr, nr);
	/*
	 * Do this prior waking up the new thread - the thread pointer
	 * might get invalid after that point, if the thread exits quickly.
	 */
	if (!IS_ERR(p)) {
		struct completion vfork;

		if (clone_flags & CLONE_VFORK) {
			p->vfork_done = &vfork;
			init_completion(&vfork);
		}

		if ((p->ptrace & PT_PTRACED) || (clone_flags & CLONE_STOPPED)) {
			/*
			 * We'll start up with an immediate SIGSTOP.
			 */
			sigaddset(&p->pending.signal, SIGSTOP);
			set_tsk_thread_flag(p, TIF_SIGPENDING);
		}

		if (!(clone_flags & CLONE_STOPPED))
			wake_up_new_task(p, clone_flags);
		else
			p->state = TASK_STOPPED;

		if (unlikely (trace)) {
			current->ptrace_message = nr;
			ptrace_notify ((trace << 8) | SIGTRAP);
		}

		if (clone_flags & CLONE_VFORK) {
			wait_for_completion(&vfork);
			if (unlikely (current->ptrace & PT_TRACE_VFORK_DONE))
				ptrace_notify ((PTRACE_EVENT_VFORK_DONE << 8) | SIGTRAP);
		}
	} else {
		free_pid(pid);
		nr = PTR_ERR(p);
	}
	return nr;
}
```
* copy_process開始複製current給新的process：
```
	if ((retval = copy_semundo(clone_flags, p)))
		goto bad_fork_cleanup_audit;
	if ((retval = copy_files(clone_flags, p)))
		goto bad_fork_cleanup_semundo;
	if ((retval = copy_fs(clone_flags, p)))
		goto bad_fork_cleanup_files;
	if ((retval = copy_sighand(clone_flags, p)))
		goto bad_fork_cleanup_fs;
	if ((retval = copy_signal(clone_flags, p)))
		goto bad_fork_cleanup_sighand;
	if ((retval = copy_mm(clone_flags, p)))
		goto bad_fork_cleanup_signal;
	if ((retval = copy_keys(clone_flags, p)))
		goto bad_fork_cleanup_mm;
	if ((retval = copy_namespace(clone_flags, p)))
		goto bad_fork_cleanup_keys;
	retval = copy_thread(0, clone_flags, stack_start, stack_size, p, regs);
	if (retval)
		goto bad_fork_cleanup_namespace;
```
    流程依序為：clone->_do_fork->copy_process->copy_mm
4. virtual address 什麼時候連上 physical address ?
答：在第一次訪問到資料時，對應的PTE的valid bit是沒有被設置的，此時為page fault。當遇到page fault時，MMU會將控制權還給OS，這時交由Page fault handler來處理，handler會選擇某個在DRAM中的page來替換。(page fault handler)

5. current macro 在不同thread用到時是同個task struct?
答: 不同thread皆有自己的pid，因此不同thread的task struct不會一樣。



## Reference

1. [Stack Overflow](https://stackoverflow.com/questions/41090469/linux-kernel-how-to-get-physical-address-memory-management?fbclid=IwAR1hAHO4eZy7BhUIFCfVxtTsBkP5njKV31jj7kRU1p10Y3mqefzmfLFiOic)
2. [Module教學](https://jerrynest.io/how-to-write-a-linux-kernel-module/)
3. [Page Table Management](https://www.kernel.org/doc/gorman/html/understand/understand006.html?fbclid=IwAR3gGfrQmiSeGSsTO2b_0cHaneq1TVs-lnjimZLG2MaTojJM-25CCHQG8ZU)
4. [copy_process()](https://www.jollen.org/blog/2007/01/process_creation_5_copy_process.html?fbclid=IwAR0Hrlti9YLluxAwbRxj9DMzpytCN_GSAPQB2YHMKH1EtGLWrsEFjH6vGeA)
5. [內存的分配與管理](https://codertw.com/%E7%A8%8B%E5%BC%8F%E8%AA%9E%E8%A8%80/676150/)
