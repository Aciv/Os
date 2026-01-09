## 1.分析copy_page_tables（）函数的代码，叙述父进程如何为子进程复制页表。
```c
// memory.c

/**
 * @brief 复制进程的页目录页表
 * 
 * @param from 源地址的内存偏移
 * @param to 目的地址的内存偏移
 * @param size 需要复制的内存大小
 * @return int 
 */
// 就是二级页表的复制~~~
int copy_page_tables(unsigned long from,unsigned long to,long size)   
{
	// 进程1创建时 from = 0, to = 64M，  size = 640k或160个页面
	unsigned long * from_page_table;
	unsigned long * to_page_table;
	unsigned long this_page;
	unsigned long * from_dir, * to_dir;
	unsigned long nr;

	if ((from&0x3fffff) || (to&0x3fffff))    // 源地址和目的地址都需要是 4Mb 的倍数。否则出错，死机
		panic("copy_page_tables called with wrong alignment");
	// 取得源地址和目的地址的目录项(from_dir 和 to_dir)	 分别是0和64
	// 页目录表项的前10位，再加上long long 数组的4b偏移左移两位

	// 以4B为单位移动
	// 4B为页目录项大小
	from_dir = (unsigned long *) ((from>>20) & 0xffc); /* _pg_dir = 0 */
	to_dir = (unsigned long *) ((to>>20) & 0xffc);
	// 单位为4M 取整上界
	size = ((unsigned) (size+0x3fffff)) >> 22;     // 计算要复制的内存块占用的页目录表数  4M（1个页目录项管理的页面大小 = 1024*4K）的数量
	for( ; size-->0 ; from_dir++,to_dir++) {
		if (1 & *to_dir)                 // 如果目的目录项指定的页表已经存在，死机
			panic("copy_page_tables: already exist");
		if (!(1 & *from_dir))            // 如果源目录项未被使用，不用复制，跳过
			continue;

		// get the page's address
		from_page_table = (unsigned long *) (0xfffff000 & *from_dir);
		if (!(to_page_table = (unsigned long *) get_free_page()))     // // 为目的页表取一页空闲内存. 关键！！是目的页表存储的地址
			return -1;	/* Out of memory, see freeing */
		// 前12位是地址，后四位是标记位
		*to_dir = ((unsigned long) to_page_table) | 7;    // 设置目的目录项信息。7 是标志信息，表示(Usr, R/W, Present)
		// 多少个页表
		nr = (from==0)?0xA0:1024;              // 如果是进程0复制给进程1，则复制160个页面；否则将1024个页面全部复制
		for ( ; nr-- > 0 ; from_page_table++,to_page_table++) { 
			// 复制页的地址和标记位   
			this_page = *from_page_table;     // 复制！
			if (!(1 & this_page))
				continue;
			this_page &= ~2;         // 010， 代表用户，只读，存在
			*to_page_table = this_page;       // 复制！
			if (this_page > LOW_MEM) {    // 如果该页表项所指页面的地址在 1M 以上，则需要设置内存页面映射数组 mem_map[]
				*from_page_table = this_page;
				this_page -= LOW_MEM;
				this_page >>= 12;
				mem_map[this_page]++;
			}
		}
	}
	invalidate();     // 刷新页变换高速缓冲 TLB
	return 0;
}
```
子进程复制父进程的所有页，并修改页标记为为010，表示只读，实现写时复制

## 2.进程0创建进程1时，为进程1建立了task_struct及内核栈，第一个页表，分别位于物理内存16MB顶端倒数第一页、第二页。请问，这两个页究竟占用的是谁的线性地址空间，内核、进程0、进程1、还是没有占用任何线性地址空间？说明理由（可以图示）并给出代码证据。
占用的是内核线性地址:
```c
// memory.c
/*
 * Get physical address of first (actually last :-) free page, and mark it
 * used. If no free pages left, return 0.
 */
//// 取空闲页面。如果已经没有内存了，则返回 0。 
 // 输入：%1(ax=0) - 0；%2(LOW_MEM)；%3(cx=PAGING PAGES)；%4(di=mem_map+PAGING_PAGES-1)。 
 // 输出：返回%0(ax=页面号)。 
 // 从内存映像末端开始向前扫描所有页面标志（页面总数为 PAGING_PAGES），如果有页面空闲（对应 
 // 内存映像位为 0）则返回页面地址。
unsigned long get_free_page(void)
{
	// 在mem_map 中反向扫描得到空闲页面，故从16M高地址开始分配页面
    register unsigned long __res asm("ax");

    /*
    scasb: 
        if (AL == *EDI) {
            ZF = 1;  // 设置零标志
        } else {
            ZF = 0;  // 清除零标志
        }
        EDI = EDI ± 1;  // 根据 DF 标志决定增减
    scasb:
        repeat until ECX = 0 or ZF = 1
    init :
    AL : 0
    ecx : PAGING_PAGES
    edx: mem_map+PAGING_PAGES-1
    */

    __asm__("std ; repne ; scasb\n\t"
        "jne 1f\n\t"
        "movb $1,1(%%edi)\n\t"		// edi is free page's last position
        "sall $12,%%ecx\n\t"		// page actually location
        "addl %2,%%ecx\n\t"			// add LOW_MEM
        "movl %%ecx,%%edx\n\t"
        "movl $1024,%%ecx\n\t"
        "leal 4092(%%edx),%%edi\n\t"
        "rep ; stosl\n\t"
        "movl %%edx,%%eax\n"
        "1:"
        :"=a" (__res)
        :"0" (0),"i" (LOW_MEM),"c" (PAGING_PAGES),
        "D" (mem_map+PAGING_PAGES-1)		// sign the memory
        :"di","cx","dx");
    return __res;
}

// fork.c
	p = (struct task_struct *) get_free_page();

```

## 3. 代码中的"ljmp %0\n\t" 很奇怪，按理说jmp指令跳转到得位置应该是一条指令的地址，可是这行代码却跳到了"m" (*&__tmp.a)，这明明是一个数据的地址，更奇怪的，这行代码竟然能正确执行。请论述其中的道理。
```c
#define switch_to(n) {\
struct {long a,b;} __tmp; \
__asm__("cmpl %%ecx,_current\n\t" \
    "je 1f\n\t" \
    "movw %%dx,%1\n\t" \
    "xchgl %%ecx,_current\n\t" \
    "ljmp %0\n\t" \
    "cmpl %%ecx,_last_task_used_math\n\t" \
    "jne 1f\n\t" \
    "clts\n" \
    "1:" \
    ::"m" (*&__tmp.a),"m" (*&__tmp.b), \
    "d" (_TSS(n)),"c" ((long) task[n])); \
}
```
ljmp to TSS(n)

从GDT中读取对应的TSS描述符
保存当前任务状态到当前TSS
加载新任务的TSS（包括寄存器、CR3等）
切换到新任务的上下文


## 4.进程0开始创建进程1，调用fork（），跟踪代码时我们发现，fork代码执行了两次，第一次，执行fork代码后，跳过init（）直接执行了for(;;) pause()，第二次执行fork代码后，执行了init（）。奇怪的是，我们在代码中并没有看到向转向fork的goto语句，也没有看到循环语句，是什么原因导致fork反复执行？请说明理由（可以图示），并给出代码证据。
```c
// unisted.h
#define _syscall0(type,name) \
type name(void) \
{ \
    long __res; \
    __asm__ volatile ("int $0x80" \
        : "=a" (__res) \
        : "0" (__NR_##name)); \				 
    if (__res >= 0) \
        return (type) __res; \
    errno = -__res; \
    return -1; \
}


// system_call.s
// int 80 program
_system_call:
	cmpl $nr_system_calls-1,%eax
	ja bad_sys_call
	push %ds
	push %es
	push %fs
	pushl %edx
	pushl %ecx		# push %ebx,%ecx,%edx as parameters
	pushl %ebx		# to the system call
	movl $0x10,%edx		# set up ds,es to kernel space
	mov %dx,%ds
	mov %dx,%es
	movl $0x17,%edx		# fs points to local data space
	mov %dx,%fs
	// linux/sys.h
	call _sys_call_table(,%eax,4)
    // get's new progress's eax
	// new progress's eax is set to 0 initial
	pushl %eax			# last_pid
	movl _current,%eax
	cmpl $0,state(%eax)		# state
	jne reschedule
	cmpl $0,counter(%eax)		# counter
	je reschedule
ret_from_sys_call:
	movl _current,%eax		# task[0] cannot have signals
	cmpl _task,%eax			# process 0
	je 3f
	cmpw $0x0f,CS(%esp)		# was old code segment supervisor ?
	jne 3f
	cmpw $0x17,OLDSS(%esp)		# was stack segment = 0x17 ?
	jne 3f
	movl signal(%eax),%ebx
	movl blocked(%eax),%ecx
	notl %ecx
	andl %ebx,%ecx
	bsfl %ecx,%ecx
	je 3f
	btrl %ecx,%ebx
	movl %ebx,signal(%eax)
	incl %ecx
	pushl %ecx
	call _do_signal
	popl %eax
3:	popl %eax			# eax = last_pid
	popl %ebx
	popl %ecx
	popl %edx
	pop %fs
	pop %es
	pop %ds
	iret

// fork.c
// copy_process
	p->tss.eax = 0;

// main.c
static inline _syscall0(int,fork) 

// with the value of ax
if (!fork()) {		/* we count on this going ok */       // 进程1
    init();                                               //child process
}

```

进程0（父进程）              进程1（子进程）
↓                            ↓
sys_fork()                  （被创建）
↓                            ↓
copy_process()              （初始化完成）
↓                            ↓
返回子进程PID               返回0
↓                            ↓
执行if(!fork())的else部分    执行if(!fork())的if部分
↓                            ↓
for(;;) pause()             init()


## 5、打开保护模式、分页后，线性地址到物理地址是如何转换的？
线性地址 = 段基址 + 偏移量
转换步骤：

a. CR3寄存器存放当前页目录物理地址

b. 页目录索引 × 4 → 页目录项物理地址

c. 从页目录项获取页表物理地址

d. 页表索引 × 4 → 页表项物理地址

e. 从页表项获取物理页基地址

f. 物理页基地址 + 页内偏移 = 物理地址


## 6、getblk函数中，申请空闲缓冲块的标准就是b_count为0，而申请到之后，为什么在wait_on_buffer(bh)后又执行if（bh->b_count）来判断b_count是否为0？
```c
struct buffer_head * getblk(int dev,int block)
{
	struct buffer_head * tmp, * bh;

repeat:
    // 搜索 hash 表，如果指定块已经在高速缓冲中，则返回对应缓冲区头指针，退出
	if (bh = get_hash_table(dev,block))
		return bh;
	// 扫描空闲数据块链表，寻找空闲缓冲区	 
	tmp = free_list;
	do {
		if (tmp->b_count)        // 如果该缓冲区正被使用（引用计数不等于 0），则继续扫描下一项
			continue;

		// 如果缓冲区头指针 bh 为空，或者 tmp 所指缓冲区头的标志(修改、锁定)少于(小于)bh 头的标志， 
 		// 则让 bh 指向该 tmp 缓冲区头。如果该 tmp 缓冲区头表明缓冲区既没有修改也没有锁定标志置位， 
 		// 则说明已为指定设备上的块取得对应的高速缓冲区，则退出循环	
		// 空闲的块可能被修改或者被其他调用占用
		if (!bh || BADNESS(tmp)<BADNESS(bh)) {
			bh = tmp;
			if (!BADNESS(tmp))
				break;
		}
/* and repeat until we find something good */
	} while ((tmp = tmp->b_next_free) != free_list);
	if (!bh) {
		// 如果所有缓冲区都正被使用（所有缓冲区的头部引用计数都>0），则睡眠，等待有空闲的缓冲区可用
		sleep_on(&buffer_wait);
		goto repeat;
	}
	// 等待该缓冲区解锁（如果已被上锁的话）
	wait_on_buffer(bh);
	if (bh->b_count)
		goto repeat;
	while (bh->b_dirt) {        // 如果该缓冲区已被修改，则将数据写盘，并再次等待缓冲区解锁
		sync_dev(bh->b_dev);
		wait_on_buffer(bh);
		if (bh->b_count)
			goto repeat;
	}
/* NOTE!! While we slept waiting for this block, somebody else might */
/* already have added "this" block to the cache. check it */
/* 注意！！当进程为了等待该缓冲块而睡眠时，其它进程可能已经将该缓冲块 */ 
/*  加入进高速缓冲中，所以要对此进行检查。*/ 
 // 在高速缓冲 hash 表中检查指定设备和块的缓冲区是否已经被加入进去。如果是的话，就再次重复 
 // 上述过程。
	if (find_buffer(dev,block))
		goto repeat;
/* OK, FINALLY we know that this buffer is the only one of it's kind, */
/* and that it's unused (b_count=0), unlocked (b_lock=0), and clean */
/* OK，最终我们知道该缓冲区是指定参数的唯一一块，*/ 
 /* 而且还没有被使用(b_count=0)，未被上锁(b_lock=0)，并且是干净的（未被修改的）*/ 
 // 于是让我们占用此缓冲区。置引用计数为 1，复位修改标志和有效(更新)标志。
	bh->b_count=1;
	bh->b_dirt=0;
	bh->b_uptodate=0;
	// 从 hash 队列和空闲块链表中移出该缓冲区头，让该缓冲区用于指定设备和其上的指定块。
	remove_from_queues(bh);
	// 由于要先从hash与free_list中移除，所以后更新dev和block
	bh->b_dev=dev;
	bh->b_blocknr=block;
	// 然后根据此新的设备号和块号重新插入空闲链表和 hash 队列新位置处。并最终返回缓冲头指针
	insert_into_queues(bh);
	return bh;
}
```

wait_on_buffer(bh)内包含睡眠函数，虽然此时已经找到比较合适的空闲缓冲块，但是可能在睡眠阶段该缓冲区被其他任务所占用，因此必须重新搜索，判断是否被修改，修改则写盘等待解锁

第一次判断： 检查缓冲块当前是否被占用

wait_on_buffer期间： 其他进程可能已占用该缓冲块

第二次判断： 确认等待后缓冲块是否仍然空闲，因为在wait_on_buffer完成之后也有可能其他程序占用该块。



## 7、b_dirt已经被置为1的缓冲块，同步前能够被进程继续读、写？给出代码证据。
同步前能够被进程继续读、写。但不能挪为它用（即关联其它物理块）。
b_uptodate设置为1后，内核就可以支持进程共享该缓冲块的数据了，读写都可以，读操作不会改变缓冲块的内容，所以不影响数据，而执行写操作后，就改变了缓冲块的内容，就要将b_dirt标志设置为1。
由于此前缓冲块中的数据已经用硬盘数据块更新了，所以后续的同步未被改写的部分不受影响，同步是不更改缓冲块中数据的，所以b_uptodate仍为1。即进程在b_dirt置为1时，仍能对缓冲区数据进行读写。

```c
// fs/buffer.c bread函数
// 与b_dirt无关
struct buffer_head * bread(int dev,int block)
{
    struct buffer_head * bh;
    
    bh = getblk(dev,block);
    if (bh->b_uptodate)
        return bh;
    
    ll_rw_block(READ,bh);
    wait_on_buffer(bh);
    if (bh->b_uptodate)
        return bh;
    return NULL;
}

// fs/buffer.c bwrite函数
void bwrite(struct buffer_head * bh)
{
    bh->b_dirt = 1;  // 只是标记脏，不立即写
    // 实际写操作在sync或其他时机执行
}

// 进程可以继续读写bh指向的数据
memcpy(bh->b_data, data, BLOCK_SIZE);  // 即使b_dirt=1也可以写
```

## 8、分析panic函数的源代码，根据你学过的操作系统知识，完整、准确的判断panic函数所起的作用。假如操作系统设计为支持内核进程（始终运行在0特权级的进程），你将如何改进panic函数？
panic()函数是当系统发现无法继续运行下去的故障时将调用它，会导致程序终止，然后由系统显示错误号。如果出现错误的函数不是进程0，那么就要进行数据同步，把缓冲区中的数据尽量同步到硬盘上。遵循了Linux尽量简明的原则。
```c
// kernel/panic.c

// 该函数用来显示内核中出现的重大错误信息，并运行文件系统同步函数，然后进入死循环 -- 死机。 
 // 如果当前进程是任务 0 的话，还说明是交换任务出错，并且还没有运行文件系统同步函数
volatile void panic(const char * s)
{
	printk("Kernel panic: %s\n\r",s);
	if (current == task[0])
		printk("In swapper task - not syncing\n\r");
	else
		sys_sync();
	for(;;);
}


```
改进：

将死循环for(;;)改进为跳转到内核进程（始终运行在0特权级的进程），让内核继续执行
## 9、详细分析进程调度的全过程。考虑所有可能（signal、alarm除外）
```c
void schedule(void)
{
	int i,next,c;
	struct task_struct ** p;

/* check alarm, wake up any interruptible tasks that have got a signal */
// ========== 根据信号唤醒进程 ===========
	for(p = &LAST_TASK ; p > &FIRST_TASK ; --p)
		if (*p) {                  // 如果任务的 alarm 时间已经过期(alarm<jiffies),在信号位图中置 SIGALRM 信号，然后清 alarm
			if ((*p)->alarm && (*p)->alarm < jiffies) {
					(*p)->signal |= (1<<(SIGALRM-1));
					(*p)->alarm = 0;
				}
			if (((*p)->signal & ~(_BLOCKABLE & (*p)->blocked)) &&        //则置任务为就绪状态
			(*p)->state==TASK_INTERRUPTIBLE)
				(*p)->state=TASK_RUNNING;
		}

/* this is the scheduler proper: */

	while (1) {
		c = -1;
		next = 0;
		i = NR_TASKS;
		p = &task[NR_TASKS];
		// back to find running state progress
		// set will the max counter
		while (--i) {
			if (!*--p)
				continue;
			if ((*p)->state == TASK_RUNNING && (*p)->counter > c)   //get counter_max 值
				c = (*p)->counter, next = i;
		}
		if (c) break;
		// renew the counter because all the progress's conuter is zero                       
		for(p = &LAST_TASK ; p > &FIRST_TASK ; --p)                //calculate counter
			if (*p)
				(*p)->counter = ((*p)->counter >> 1) +
						(*p)->priority;
	}
	switch_to(next);
}

```
所有可能情况：

正常调度： 时间片用完

进程阻塞： 调用sleep_on等函数

进程退出： do_exit()后调用schedule()

中断返回： 某些中断处理中可能调用schedule()

主动让出： 调用pause()或schedule()

进程中有就绪进程，且时间片没有用完
    正常情况下，schedule()函数首先扫描任务数组。遍历就绪（TASK_RUNNING）任务数组，选择运行状态且剩余时间片最多的任务，最后调用switch_to()执行实际的进程切换操作。

进程中有就绪进程，但所有就绪进程时间片都用完（c=0）
    如果此时所有处于TASK_RUNNING 状态进程的时间片都已经用完，系统就会根据每个进程的优先权值重新分配时间片。
    然后 schdeule()函数重新扫描任务数组中所有处于TASK_RUNNING 状态，重复上述过程，直到选择出一个进程为止。最后调用switch_to()执行实际的进程切换操作。

所有进程都不是就绪的c=-1
    此时代码中的c=-1，next=0，跳出循环后，执行switch_to(0)，切换到进程0执行，因此所有进程都不是就绪的时候进程0执行。


## 10、wait_on_buffer函数中为什么不用if（）而是用while（）？

```c
// 等待指定缓冲区解锁
static inline void wait_on_buffer(struct buffer_head * bh)
{
	cli();
	// make_request will lock the buffer block
	while (bh->b_lock)
		sleep_on(&bh->b_wait);
	sti();
}
```
多个等待者： 多个进程可能等待同一缓冲块

唤醒所有： wake_up()唤醒所有等待进程

竞争条件： 只有一个进程能获得锁，其他需要继续等待

## 11、操作系统如何利用b_uptodate保证缓冲块数据的正确性？new_block (int dev)函数新申请一个缓冲块后，并没有读盘，b_uptodate却被置1，是否会引起数据混乱？详细分析理由。
b_uptodate = 1: 缓冲块数据与磁盘一致（有效）

b_uptodate = 0: 缓冲块数据无效或未同步
```c
// fs/inode.c
int new_block(int dev)
{
    struct buffer_head * bh;
    int b;
    
    // 1. 申请空闲磁盘块
    if (!(b = alloc_block(dev)))
        return 0;
    
    // 2. 获取缓冲块
    bh = getblk(dev,b);
    
    // 3. 清空缓冲块
    lock_buffer(bh);
    memset(bh->b_data, 0, BLOCK_SIZE);
    bh->b_uptodate = 1;  // 标记为最新
    bh->b_dirt = 1;      // 标记为脏，需要写回
    unlock_buffer(bh);
    brelse(bh);
    
    return b;
}
```
新分配的块： 磁盘上该块原内容无意义

已清空： memset清零了缓冲块内容

立即写回： b_dirt=1确保数据会写回磁盘

后续使用： 文件系统会重新读写这个块

## 12、add_request（）函数中有下列代码
```c
    if (!(tmp = dev->current_request)) {
        dev->current_request = req;
        sti();
        (dev->request_fn)();
        return;
    }
```
## 其中的
```c
    if (!(tmp = dev->current_request)) {
        dev->current_request = req;
```
## 是什么意思？

如果 dev 的当前请求(current_request)子段为空，则表示目前该设备没有请求项，本次是第 1 个请求项

## 13、do_hd_request()函数中dev的含义始终一样吗？
```c
void do_hd_request(void)
{
    unsigned int dev = MINOR(CURRENT->dev);  // 设备号
    // ...
    block = CURRENT->sector;  // 扇区号
    // ...
    if (dev >= 5*NR_HD) {  // 分区检查
        // ...
    }
    // dev在这里表示分区号
    // 0-3: 主分区，4+: 逻辑分区
}
```
初始： 从CURRENT请求中获取设备号

计算： 用于确定分区和物理扇区

变化： 在处理过程中可能被重新计算

## 14、read_intr（）函数中，下列代码是什么意思？为什么这样做？
```c
    if (--CURRENT->nr_sectors) {
        do_hd = &read_intr;
        return;
    }
```
--CURRENT->nr_sectors：剩余扇区数减1

如果还有扇区要读，设置do_hd = &read_intr，即继续读

中断返回，等待下一个中断继续读

## 15、bread（）函数代码中为什么要做第二次if (bh->b_uptodate)判断？
```c
    if (bh->b_uptodate)
        return bh;
    ll_rw_block(READ,bh);
    wait_on_buffer(bh);
    if (bh->b_uptodate)
        return bh;
```
理由同 #6， 在睡眠的时候被块其他进程修改

## 16、getblk（）函数中，两次调用wait_on_buffer（）函数，两次的意思一样吗？
第一次： 等待原始缓冲块解锁

第二次： 等待新找到的缓冲块解锁

## 17、getblk（）函数中
```c
    do {
        if (tmp->b_count)
            continue;
        if (!bh || BADNESS(tmp)<BADNESS(bh)) {
            bh = tmp;
            if (!BADNESS(tmp))
                break;
        }
/* and repeat until we find something good */
    } while ((tmp = tmp->b_next_free) != free_list);
```
说明什么情况下执行continue、break。

continue执行： 当tmp->b_count != 0时，缓冲块正在使用

break执行： 当找到BADNESS(tmp) == 0的缓冲块时

BADNESS=0表示缓冲块既干净(b_dirt=0)又未锁定(b_lock=0)

## 18、make_request（）函数    
```c   
    if (req < request) {
        if (rw_ahead) {
            unlock_buffer(bh);
            return;
        }
        sleep_on(&wait_for_request);
        goto repeat;
``` 
其中的```sleep_on(&wait_for_request)```是谁在等？等什么？

```c
// 将当前任务置为不可中断的等待状态,并让睡眠对象的任务指针指向当前任务，只有明确地唤醒时才会返回
void sleep_on(struct task_struct **p)
{
	struct task_struct *tmp;
	
	if (!p)
		return;
	if (current == &(init_task.task))
		panic("task[0] trying to sleep");
	// set b_wati to current progress 
	tmp = *p; 
	*p = current;
	current->state = TASK_UNINTERRUPTIBLE;
	schedule();
	
	if (tmp)
		tmp->state=0;

}
```
谁在等： 当前进程（发起IO请求的进程）

等什么： 等待空闲的request结构体

等待-唤醒机制：

进程A：请求队列满，sleep_on(&wait_for_request)

进程B：完成请求，wake_up(&wait_for_request)

进程A：被唤醒，重新尝试获取请求结构体