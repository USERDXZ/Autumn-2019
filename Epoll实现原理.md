# epoll的实现原理（1）

本文是学习epoll过程中的笔记，方便自己理解，基本翻译自下面的文章：

[The Implementation of epoll(1)](https://idndx.com/2014/09/01/the-implementation-of-epoll-1/)

[The Implementation of epoll(2)](https://idndx.com/2014/09/01/the-implementation-of-epoll-2/)

## 概述

epoll与传统的I/O多路复用技术之间的最大差别在于，用户只需要获取一个epoll实例，然后将文件描述**注册**到它上面（一次性），而不是每次将大量文件描述符传递到**内核**当中。

## epoll实例的创建

通过调用`epoll_create(2)`或者`epoll_create1(2)`来请求实例，返回的是**文件描述符**。所以，epoll也是可以轮询的，用于一些高级用法。

epoll实例实际的重要部分是内核数据`struct eventpoll`，这个数据结构几乎维护了epoll实例正常工作所需要的所有内容。它的创建方法如下：

```C
/*
 * Create the internal data structure ("struct eventpoll").
 */
error = ep_alloc(&ep);
```

`ep_alloc()`所做的，只是从内核堆分配足够的空间来保存`eventpoll`  (ep)，然后`epoll_create()`去尝试在进程中获取未使用的描述符。如果能够获取到描述符，它会尝试从系统获取一个匿名的inode （[什么是匿名inode？](https://stackoverflow.com/questions/4508998/what-is-an-anonymous-inode-in-linux)）。 这里注意，`epoll_create()`将`eventpoll`的指针保存在了文件的`private_data` 中，这样访问`eventpoll`非常方便。

然后，`epoll_create()`将匿名inode与文件描述符绑定起来，并向用户返回文件描述符fd。

## epoll实例如何保存它监控的文件描述符

`epoll`使用非常常用的内核数据结构 - [红黑树](https://en.wikipedia.org/wiki/Red–black_tree)（缩写为RB-Tree），以跟踪当前由特定`epoll`实例监视的所有文件描述符。RB-Tree的根是其`struct eventpoll`的`rbr`成员，在函数`ep_alloc()`内初始化。

对于epoll所监控的文件描述符，epoll使用一个与之关联的`struct epitem` 保存在红黑树当中。`struct epitem` 中同时还保存了一些其他的重要数据供epoll使用。

### ep_find()

当我们使用`epoll_ctl(2)` 向epoll实例加入一个文件描述符的时候，首先要调用`ep_find()`去定位`epitem`在红黑树中的位置。红黑树的键值为存储在`epitem`中的`struct epoll_filefd` 

```C
struct epoll_filefd {
	struct file *file; // pointer to the target file struct corresponding to the fd
	int fd; // target file descriptor number
} __packed;
```

键值比较方式如下，`struct file`地址大就更大，相同就比较描述符号

```C
/* Compare RB tree keys */
static inline int ep_cmp_ffd(struct epoll_filefd *p1,
                            struct epoll_filefd *p2)
{
	return (p1->file > p2->file ? +1:
		   (p1->file < p2->file ? -1 : p1->fd - p2->fd));
}
```

当添加fd的时候，需要找到一个空位置，如果没找到合适的位置会返回错误`EEXIST`。找到后，开始调用`ep_insert() `。

### ep_insert() 

目的是将epoll的回调函数注册到文件描述符，让fd在事件发生时通知epoll。

重点操作的对象：`struct epitem`（最后需要插入到epoll的RB-tree中的struct），监控的文件描述符

##### 1.确保用户监控的fd总数不超，然后从slab分配struct epitem的内存

```C
/* Item initialization follow here ... */
INIT_LIST_HEAD(&epi->rdllink);
INIT_LIST_HEAD(&epi->fllink);
INIT_LIST_HEAD(&epi->pwqlist);    //保存struct eppoll_entry *pwq的链表，后面有讲
epi->ep = ep;                     //epoll实例的eventpoll
ep_set_ffd(&epi->ffd, tfile, fd); //设置fd
epi->event = *event; //监控的事件
epi->nwait = 0; //就是epi->pwqlist链表的长度，后面有奖
epi->next = EP_UNACTIVE_PTR;
```



##### 2.尝试将epoll的回调函数注册到**文件描述符**中

介绍一些需要的struct 

```C
struct ep_pqueue {         //包装epitem与poll_table
	poll_table pt;
    struct epitem *epi;
};

typedef void (*poll_queue_proc)(struct file *, wait_queue_head_t *, struct poll_table_struct *);//回调函数

typedef struct poll_table_struct { //poll_table
	poll_queue_proc _qproc;        //回调函数,待会让poll_wait执行！
	unsigned long _key;            //设置为~0，表示任何事件都要通知epoll，然后在epoll实现中过滤
} poll_table;
```

初始化`struct ep_pqueue`（设置`epitem`指针epi，设置`poll_table`的回调函数和事件）



##### 3.epoll调用`ep_item_poll(epi, &epq.pt)`，这将调用相应文件fd的`poll()`来实现。

`epi：epitem`， `epq.pt：poll_tabel`(包含回调函数)

```C
//例如TCP的poll函数
unsigned int tcp_poll(struct file *file, struct socket *sock, poll_table *wait)
{
	unsigned int mask;
	struct sock *sk = sock->sk;
	const struct tcp_sock *tp = tcp_sk(sk);

	sock_rps_record_flow(sk);
    
	sock_poll_wait(file, sk_sleep(sk), wait);//参数：文件，套接字，wait就是poll_tabel
    //sk_sleep(sk) 返回特定sock的等待队列！
	// code omitted
}
```

`sock_poll_wait()`只是进行一些检查，然后用相同的参数调用`poll_wait()`

```C
static inline void poll_wait(struct file * filp, wait_queue_head_t * wait_address, poll_table *p)
{
	if (p && p->_qproc && wait_address)      //回调函数 && 等待队列 存在
		p->_qproc(filp, wait_address, p);   //_qproc函数保存poll_table中，定义见下
}
```

```C
/*
* This is the callback that is used to add our wait queue to the
* target file wakeup lists.           _qproc函数
*/
static void ep_ptable_queue_proc(struct file *file, wait_queue_head_t *whead,
			 poll_table *pt)
{
	struct epitem *epi = ep_item_from_epqueue(pt);  //恢复指向的epitem
	struct eppoll_entry *pwq;                       //创建一个epitem与fd的关联（glue）
													//后添加到epitem
	if (epi->nwait >= 0 && (pwq = kmem_cache_alloc(pwq_cache, GFP_KERNEL))) {
        //用ep_poll_callback初始化pwq->wait
		init_waitqueue_func_entry(&pwq->wait, ep_poll_callback);
		pwq->whead = whead;       //保存等待队列头，方便epoll取消在队列中的注册
		pwq->base = epi;
        //把pwq->wait（ep_poll_callback）加入到(特定sock的)等待队列
		add_wait_queue(whead, &pwq->wait);
		list_add_tail(&pwq->llink, &epi->pwqlist);   //把eppoll_entry的加到epi->pwqlist链表
		epi->nwait++;                                //代表上面链表的长度，通常为1，不知道有啥用
	} else {
		/* We have to signal that an error occurred */
		epi->nwait = -1;
	}
}
```

**pwq->wait（ep_poll_callback）的作用：**

1. **监视在该特定受监视文件上发生的事件**
2. **必要时唤醒其他进程**

最后通过`ep_rbtree_insert(ep, epi)`将`struct epitem`添加到红黑树中

##### 4.疑点解释

- 加入到`wait_queue_head_t`中的`wait_queue_t（ep_poll_callback）`通常是基于机器的唤醒机制，是一个包含函数指针struct。在函数中，epoll可以选择如何处理这个唤醒信号（不监控的事件就不唤醒）。
- 对于上面提到的`poll()`，完全取决于它是怎么实现的。例如对于TCP套接字的fd来说，等待队列在它的`strcut sock`中，可以见得，不同的文件实现会将他的等待队列头放在完全不同的位置（是不确定的），所以我们只能将`ep_ptable_queue_proc()`交给`poll()`以回调的方式执行！
- `sk_wp`内部的`sock`（等待队列）唤醒的时机？

```C
//struct sock defined the following hooks in net/core/sock.c, line 2312
void sock_init_data(struct socket *sock, struct sock *sk)
{
	// code omitted...
	 sk->sk_data_ready  =   sock_def_readable;
	sk->sk_write_space =  sock_def_write_space;
    // code omitted...
}
```

在` sock_def_readable() `和`sock_def_write_space()`函数内, `(struct sock)->sk_wq `在执行唤醒回调时调用了 `wake_up_interruptible_sync_poll()`

对于tcp来说，`sk->sk_data_ready()`只要TCP连接完成三次握手，或者已经收到特定TCP套接字的缓冲区，就会在下面的中断处理程序调用。`sk->sk_write_space()`只要从该套接字发生从*full-> available*的缓冲区状态转换，就会调用它。



# epoll的实现原理（2）

参考自：

[The Implementation of epoll(3)](https://idndx.com/2014/09/01/the-implementation-of-epoll-3/)

[The Implementation of epoll(4)](https://idndx.com/2014/09/01/the-implementation-of-epoll-4/)

### 回调函数 ep_poll_callback()

前面提到的`ep_insert()`函数将epoll实例附加到监视文件描述符fd的等待队列，注册`ep_poll_callback()`为队列唤醒的回调函数。下面剖析一下这个回调函数：

```c
static int ep_poll_callback(wait_queue_t *wait, unsigned mode, int sync, void *key)
{                           //↑ pwq->wait
	int pwake = 0;
	unsigned long flags;
	struct epitem *epi = ep_item_from_wait(wait);  //通过strcut eppoll_entry找到epitem
	struct eventpoll *ep = epi->ep;                //进一步找到eventpoll
```

**接下来，使用自旋锁锁定`eventpoll`**

```c
spin_lock_irqsave(&ep->lock, flags);
```

**然后检查事件是否是用户让epoll监视的。**前面`ep_insert()`将注册事件为`~0`，有两个原因：

1.用户可能频繁改变需要监视的事件，但是重新注册`poll`回调效率不高。

2.其次，并非所有的事件都遵循系统的事件掩码设置，因此完全依靠系统的设置不太可靠。

```c
if (key && !((unsigned long) key & epi->event.events))
	goto out_unlock;
```

如果没有监控任何事件，跳到解锁`out_unlock`



**接下来检查epoll实例是否正尝试将事件传输到用户态（也就是在调用`ep_send_events_proc()`时）。**

如果是的话，将当前`epitem`加入到一个链表头在 `struct eventpoll`中的单链表里。代码如下：

```c
if (unlikely(ep->ovflist != EP_UNACTIVE_PTR)) { //正在传输为NULL（默认状态是EP_UNACTIVE_PTR）
	if (epi->next == EP_UNACTIVE_PTR) {				
		epi->next = ep->ovflist;                 //前面已经取得了ep的锁
		ep->ovflist = epi;
		if (epi->ws) {
			__pm_stay_awake(ep->ws);
		}
	}
	goto out_unlock;                              //完成后跳到解锁
}
```



**不是的话，接下来`ep_poll_callback()` 检查当前的`struct epitem` 是否已经在就绪队列中。**

在用户没有没有机会调用`epoll_wait()`时可能发生这种情况（以前加入过了，用户还没机会处理）。

如果不在的话，函数会将`struct epitem` 加入到就绪队列（`struct eventpoll`的成员`rdllist` ）中。

```c
if (!ep_is_linked(&epi->rdllink)) {
	list_add_tail(&epi->rdllink, &ep->rdllist);
	ep_pm_stay_awake_rcu(epi);
}
```



**然后 `ep_poll_callback()`唤醒等待`wq`和`poll_wait`的进程。**

`wq`用于**在超时时间(timeout)没到时**，用户正在通过`epoll_wait()`等待events的时候（最多唤醒一个进程）。 

`poll_wait`是epoll实现的文件系统的`poll()`，请记住epoll同样是一个可轮询的文件描述符。

```c
if (waitqueue_active(&ep->wq))
	wake_up_locked(&ep->wq);                    //加锁wq（正在等待的wait_qunue）
	if (waitqueue_active(&ep->poll_wait))       //
		pwake++;								//poll_wait类比前文Tcp文件描述符的poll_wait
```



**最后的out_unlock部分，`ep_poll_callback()`释放自旋锁，并且唤醒(eventpoll的)`poll_wait`（`poll()`就绪队列`&ep->rdllist`）**

我们不能在保持自旋锁的时候唤醒，因为epoll可以将自身fd加入到受监视文件中（这种情况下有锁唤醒会导致死锁）。

```c
out_unlock:
	spin_unlock_irqrestore(&ep->lock, flags);
	if (pwake)
		ep_poll_safewake(&ep->poll_wait);
return 1;
```



###  `struct eventpoll`的成员`rdllink`

在epoll中，`rdllink`是存储就绪文件描述符的双链表的头结点，链表中的节点就是有事件发生的`struct epitem`。



### 函数`epoll_wait()` 和 `ep_poll()`

该部分讨论用户程序调用`epoll_wait()` 时，是如何将文件描述符传输到用户态的。

`epoll_wait()` 很简单，它做了一些简单的错误检查，从epoll的文件描述符取出 `struct eventpoll` 然后调用 `ep_poll()` 函数去执行真正的拷贝工作。

```c
static int ep_poll(struct eventpoll *ep, struct epoll_event __user *events, int maxevents, long timeout)
{
	int res = 0, eavail, timed_out = 0;
	unsigned long flags;
	long slack = 0;
	wait_queue_t wait;                     
	ktime_t expires, *to = NULL;

	if (timeout > 0) {                     //阻塞
		struct timespec end_time = ep_set_mstimeout(timeout);
		slack = select_estimate_accuracy(&end_time);
		to = &expires;
		*to = timespec_to_ktime(end_time);
	} else if (timeout == 0) {             //非阻塞
		timed_out = 1;
		spin_lock_irqsave(&ep->lock, flags);
		goto check_events;
	}
```

`ep_poll()` 函数根据timeout确定是否阻塞来执行不同的调用方法。

阻塞时，根据timeout来计算到期时间；非阻塞则直接加锁->执行`check_events`

#### 阻塞时执行：

```c
fetch_events:
	spin_lock_irqsave(&ep->lock, flags);           //获取锁

	if (!ep_events_available(ep)) {                //检查是否有新事件，没有的话执行下面的

		init_waitqueue_entry(&wait, current);      //用当前任务初始化wait_queue_t wait
		__add_wait_queue_exclusive(&ep->wq, &wait);//加入等待队列

		for (;;) {
			set_current_state(TASK_INTERRUPTIBLE);       //信号或者wake_up()唤醒
			if (ep_events_available(ep) || timed_out)    //如果有新事件或者time_out = 1
				break;
			if (signal_pending(current)) {               //或者信号
				res = -EINTR;
				break;
			}

			spin_unlock_irqrestore(&ep->lock, flags);    //解锁
			if (!schedule_hrtimeout_range(to, slack, HRTIMER_MODE_ABS))
				timed_out = 1; /* resumed from sleep */  //如果到期，置位1，break,唤醒

			spin_lock_irqsave(&ep->lock, flags);
		}
		__remove_wait_queue(&ep->wq, &wait);      //唤醒后移出自身

		set_current_state(TASK_RUNNING);          //任务设回TASK_RUNNING
	}
```

检查是否有新事件，如果没有的话，将当前进程加入到epoll的等待队列，函数将当前任务设定为`TASK_INTERRUPTIBLE`，**解锁自旋锁并告诉程序重新调度**，还会设置内核定时器在指定超时到期或者收到任何信号时重新调度。当唤醒时（超时、新事件或者信号中断），将任务设置回`TASK_RUNNING`，然后进行`check_events`检查是否有监控事件发生。                                             [这一部分有些难以理解，后面会补充说明]

[TASK_INTERRUPTIBLE](https://www.cnblogs.com/yfceshi/p/6800069.html)：处于等待队伍中，等待资源有效时唤醒（比方等待键盘输入、socket连接、信号等等），但能够被中断唤醒.普通情况下，进程列表中的绝大多数进程都处于TASK_INTERRUPTIBLE状态.（单个CPU时），假设不是绝大多数进程都在睡眠，CPU又怎么响应得过来.

##### `check_events:`

 首先，在持有锁的情况下确认是否有监控的事件，然后才解锁。

如果没有事件并且超时未到期（time_out = 0），唤醒过早会发生这种情况。会返回到`fetch_events`并再次进行等待。 

```c
check_events:
	eavail = ep_events_available(ep);
	spin_unlock_irqrestore(&ep->lock, flags);

if (!res && eavail && !(res = ep_send_events(ep, events, maxevents)) && !timed_out)
	goto fetch_events;
return res;
```

#### 非阻塞时：

直接进入了`check_events`，而不是等待事件。（timed_out = 1）



### 用户空间交互

`ep_poll()` 函数的调用 

```c
error = ep_poll(ep, events, maxevents, timeout); //ep:epollevent
```

如果指定了timeout并没有可用事件，`ep_poll()`则将自己添加到`ep->wq`(waitqueue)，这是因为，在前一段中提到的，`ep_poll_callback()`将**唤醒**`ep->wq`在执行期间等待的任何进程。（实现了新事件到达后唤醒！）

然后调用了`schedule_hrtimeout_range()`进行休眠（高精度的内核定时器，时间到后唤醒）。

所以，加上前面设定任务为`TASK_INTERRUPTIBLE`，有4种唤醒：

1. 任务超时(time_out = 1)
2. 该任务收到了一个信号（TASK_INTERRUPTIBLE）
3. 有一个新的事件发生，（`ep_poll_callback()`将唤醒`ep->wq`）
4. 什么都没发生，只是唤醒了（TASK_INTERRUPTIBLE，wake_up()）

对于方案1,2和3，该函数设置适当的标志并退出等待循环。对于最后一个，回到睡眠状态。



接下来是`check_events:` 首先确保实际上有可用的事件，然后它调用了

```c
ep_send_events(ep, events, maxevents)
    
sys_epoll_wait() > ep_send_evnets() > ep_scan_ready_list() > ep_send_events_proc()
```

此函数依次调用`ep_scan_ready_list()`，并且传递`ep_send_events_proc()`作为回调函数。`ep_scan_ready_list()`**遍历就绪列表**并对它找到的每个就绪事件调用`ep_send_events_proc()`。



`ep_send_events`首先将`eventpoll` 中的**就绪队列(ready list)拼接到了(spliced away)函数的局部变量**中，然后将它的`ovflist` 设定为`NULL` (相对的，它的默认值为`EP_UNACTIVE_PTR`)。

下面来解释这是为什么：(因为效率！)就绪队列从`eventpoll` 拼接以后， `ep_scan_ready_list()` 设置 `ovflist` 为 `NULL` ，在回调 `ep_poll_callback()` 发生时，回调函数不会把正在传输到用户态的事件链接到`ep->rdllist`(如果这么做了就是灾难，仔细思考一下原因)，复制内存的消耗是非常昂贵的。通过 `ovflist` ，`ep_scan_ready_list()`不需要持有锁。



**下面是`ep_send_events_proc()`的执行过程**
`ep_send_events_proc()`对就绪事件再次调用文件描述符的`poll()`以**确保事件确实被触发**。这是为了确保用户注册的事件确实触发了。例如，`EPOLLOUT`在用户程序写入文件描述符时将文件描述符添加到就绪列表中。用户程序完成写入后，文件描述符可能不能写了，如果epoll不处理这种情况，那么用户将在不能写的情况下收到`EPOLLOUT`（可写）。

提醒：尽管尽力确保用户空间程序获得正确的通知，但是用户程序仍然是有可能收到不再存在的事件通知的（）。这就是为什么在使用epoll时，**总是使用非阻塞的套接字**是一个好习惯。

在检查完后，将` event struct`拷贝到用户态。



### ET/LT实现区别

```c
else if (!(epi->event.events & EPOLLET)) {           //在ep_send_events_proc()中
    list_add_tail(&epi->rdllink, &ep->rdllist);
}
```

在**LT模式**下，`ep_send_events_proc()`最后会把事件重新添加到就绪队列中，这样在下一次调用`ep_poll()`时，程序会再次检查这些事件是否可用。

由于在事件返回到用户态空间之前，`ep_send_events_proc()`每次都要通过`poll()`检查这些事件，所以这些事件如果再也不可用（没发生监控事件）的话，会略微增加开销（相对于ET模式）。但是保证不报告任何不可用事件更加重要。

到这里，`ep_send_events_proc()`函数返回了， `ep_scan_ready_list()` 继续执行一些清理工作。它首先将一些**没被 `ep_send_events_proc()` 消耗（处理）的事件**拼接回就绪队列（用户缓冲区事件已满时会发生）。

`ep_send_events_proc()`还可以快速的把所有`ovflist`中的事件添加到就绪，这样新的事件会被添加到主就绪队列，在这之后把 `ovflist`设回 `EP_UNACTIVE_PTR` 。如果仍然还有其他事件可用，函数会唤醒其他进程并结束。

```c
static int ep_send_events_proc(struct eventpoll *ep, struct list_head *head,
                   void *priv)
{
    struct ep_send_events_data *esed = priv;
    int eventcnt;
    unsigned int revents;
    struct epitem *epi;
    struct epoll_event __user *uevent;
    struct wakeup_source *ws;
    poll_table pt;
    
	init_poll_funcptr(&pt, NULL);

/*
 * We can loop without lock because we are passed a task private list.
 * Items cannot vanish during the loop because ep_scan_ready_list() is
 * holding "mtx" during this call.
 */
for (eventcnt = 0, uevent = esed->events;
     !list_empty(head) && eventcnt < esed->maxevents;) {
    epi = list_first_entry(head, struct epitem, rdllink);

    /*
     * Activate ep->ws before deactivating epi->ws to prevent
     * triggering auto-suspend here (in case we reactive epi->ws
     * below).
     *
     * This could be rearranged to delay the deactivation of epi->ws
     * instead, but then epi->ws would temporarily be out of sync
     * with ep_is_linked().
     */
    ws = ep_wakeup_source(epi);
    if (ws) {
        if (ws->active)
            __pm_stay_awake(ep->ws);
        __pm_relax(ws);
    }

    list_del_init(&epi->rdllink);

    revents = ep_item_poll(epi, &pt);　　　　// 调用f_op->poll，但不是sys_poll

    /*
     * If the event mask intersect the caller-requested one,
     * deliver the event to userspace. Again, ep_scan_ready_list()
     * is holding "mtx", so no operations coming from userspace
     * can change the item.
     */
    if (revents) {
        if (__put_user(revents, &uevent->events) ||
            __put_user(epi->event.data, &uevent->data)) {
            list_add(&epi->rdllink, head);
            ep_pm_stay_awake(epi);
            return eventcnt ? eventcnt : -EFAULT;
        }
        eventcnt++;
        uevent++;
        if (epi->event.events & EPOLLONESHOT)
            epi->event.events &= EP_PRIVATE_BITS;
        else if (!(epi->event.events & EPOLLET)) {
            /*
             * If this file has been added with Level
             * Trigger mode, we need to insert back inside
             * the ready list, so that the next call to
             * epoll_wait() will check again the events
             * availability. At this point, no one can insert
             * into ep->rdllist besides us. The epoll_ctl()
             * callers are locked out by
             * ep_scan_ready_list() holding "mtx" and the
             * poll callback will queue them in ep->ovflist.
             */
            list_add_tail(&epi->rdllink, &ep->rdllist);
            ep_pm_stay_awake(epi);
        }
    }
}

return eventcnt;
}
```
