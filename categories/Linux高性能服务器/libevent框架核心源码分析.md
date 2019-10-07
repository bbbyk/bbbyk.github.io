# libevent框架源码分析
## event结构体
```
struct event {
	//TAILQ_ENTRY是尾队列中的节点类型
	TAILQ_ENTRY(event) ev_active_next; //被激活的事件处理器串成一个队列 活动事件队列不止一个 按优先级划分
	TAILQ_ENTRY(event) ev_next;	//注册事件尾队列 
	/* for managing timeouts
	   使用时间堆和链表来管理 分为两种时间定时器类型
	*/
	union {
		TAILQ_ENTRY(event) ev_next_with_common_timeout; //该定时器在通用定时队列中的位置
		int min_heap_idx; //定时器在时间堆上的位置
	} ev_timeout_pos; 
	evutil_socket_t ev_fd; // fd or signal

	struct event_base *ev_base; //该事件处理器从属的event_base实例
	//所有具有相同文件描述符的IO事件处理器通过ev.ev_io.ev_io_next成员创建成一个尾队列
	//具有相同信号值的信号事件串成一个队列
	union {

		/* used for io events */
		struct {
			TAILQ_ENTRY(event) ev_io_next;
			struct timeval ev_timeout;
		} ev_io;// 相同fd IO事件队列

		/* used by signal events */
		struct {
			TAILQ_ENTRY(event) ev_signal_next;
			short ev_ncalls;//回调函数的执行次数
			/* Allows deletes in callback */
			short *ev_pncalls;//NULL or &ev_ncalls
		} ev_signal;// 相同signal 信号事件队列
	} _ev;

	short ev_events; // 事件类型
	short ev_res;		/* result passed to event callback 当前激活事件的类型*/
	short ev_flags; // 事件标志
	ev_uint8_t ev_pri;	/* smaller numbers are higher priority */
	ev_uint8_t ev_closure; //规定执行回调函数的行为
	struct timeval ev_timeout;  // 指定定时器的超时值

	/* allows us to adopt for different types of events */
	void (*ev_callback)(evutil_socket_t, short, void *arg); //回调函数
	void *ev_arg; // 回调函数的参数
};
```
值得注意的是，首先所有的激活事件和注册事件都分别由尾队列串成，TAILQ_ENTRY是尾队列的节点类型，定义在compat/sys/queue.h中。对于定时器的数据结构，ev_timeout_pos提供两个类型支持，一个是链表维护一个是时间堆维护，event.c中的is_common_timeout函数来判定一个定时器的类型。_ev联合体中，所有相同fd的IO事件串成一个队列，相同信号值的信号处理器串成一个队列是因为**我们可能针对同一个socket文件描述符上的读写事件创建多个事件处理器（有不同的回调函数）**，在状态激活时我们需要同时处理这些事件处理器，维护这样的链表可以大大的提高效率。

## 往注册队列中添加事件处理器
```
//将event对象添加到注册事件队列中并将对应的事件注册到事件多路分发器上
int
event_add(struct event *ev, const struct timeval *tv)
{
	int res;

	if (EVUTIL_FAILURE_CHECK(!ev->ev_base)) {
		event_warnx("%s: event has no event_base set.", __func__);
		return -1;
	}

	EVBASE_ACQUIRE_LOCK(ev->ev_base, th_base_lock);
	//主要调用另一个内部函数，is_absolut_tv = 0表示
	res = event_add_internal(ev, tv, 0);

	EVBASE_RELEASE_LOCK(ev->ev_base, th_base_lock);

	return (res);
}
```
调用的实际上是event_add_internal函数，具体的实现在其中，第三个参数is_absolut_tv == 0  

```
static inline int
event_add_internal(struct event *ev, const struct timeval *tv,
    int tv_is_absolute)
{
	struct event_base *base = ev->ev_base;
	int res = 0;
	int notify = 0;

	EVENT_BASE_ASSERT_LOCKED(base);
	_event_debug_assert_is_setup(ev);

	event_debug((
		 "event_add: event: %p (fd %d), %s%s%scall %p",
		 ev,
		 (int)ev->ev_fd,
		 ev->ev_events & EV_READ ? "EV_READ " : " ",
		 ev->ev_events & EV_WRITE ? "EV_WRITE " : " ",
		 tv ? "EV_TIMEOUT " : " ",
		 ev->ev_callback));

	EVUTIL_ASSERT(!(ev->ev_flags & ~EVLIST_ALL));

	/*
	 * prepare for timeout insertion further below, if we get a
	 * failure on any step, we should not change any state.
	 */
	//为定时处理器在通用事件队列或时间堆上预留位置

	if (tv != NULL && !(ev->ev_flags & EVLIST_TIMEOUT)) {
		if (min_heap_reserve(&base->timeheap,
			1 + min_heap_size(&base->timeheap)) == -1)
			return (-1);  /* ENOMEM == errno */
	}

	/* If the main thread is currently executing a signal event's
	 * callback, and we are not the main thread, then we want to wait
	 * until the callback is done before we mess with the event, or else
	 * we can race on ev_ncalls and ev_pncalls below. */
	/*
	   
	*/
#ifndef _EVENT_DISABLE_THREAD_SUPPORT
	//不是主线程调用添加，则等待主线程执行回调函数处理完，否则将引起竞争（event中的ev_naclls和ev_pncalss）
	if (base->current_event == ev && (ev->ev_events & EV_SIGNAL)
	    && !EVBASE_IN_THREAD(base)) { 
		++base->current_event_waiters;
		EVTHREAD_COND_WAIT(base->current_event_cond, base->th_base_lock);
	}
#endif
	
	if ((ev->ev_events & (EV_READ|EV_WRITE|EV_SIGNAL)) &&
	    !(ev->ev_flags & (EVLIST_INSERTED|EVLIST_ACTIVE))) {
			//添加io事件和io事件处理器的映射关系
		if (ev->ev_events & (EV_READ|EV_WRITE))
			res = evmap_io_add(base, ev->ev_fd, ev);
		else if (ev->ev_events & EV_SIGNAL)
			//添加信号事件和信号事件处理器的映射关系
			res = evmap_signal_add(base, (int)ev->ev_fd, ev);

		if (res != -1)
			//插入注册事件队列
			event_queue_insert(base, ev, EVLIST_INSERTED);
		if (res == 1) {
			// 事件多路分发器中添加了新的事件所以要通知主线程
			/* evmap says we need to notify the main thread. */
			notify = 1;
			res = 0;
		}
	}

	/*
	 * we should change the timeout state only if the previous event
	 * addition succeeded.
	 */
	if (res != -1 && tv != NULL) {
		struct timeval now;
		int common_timeout; //

		/*
		 * for persistent timeout events, we remember the
		 * timeout value and re-add the event.
		 *
		 * If tv_is_absolute, this was already set.
		 */
		/*
		   永久性事件处理器，其超时时间不是绝对时间则超时时间tv记录在ev->ev_io_timeout表示
		*/
		if (ev->ev_closure == EV_CLOSURE_PERSIST && !tv_is_absolute)
			ev->ev_io_timeout = *tv;

		/*
		 * we already reserved memory above for the case where we
		 * are not replacing an existing timeout.
		 */
		if (ev->ev_flags & EVLIST_TIMEOUT) { // 已经被插入,则先删除它
			/* XXX I believe this is needless. */
			if (min_heap_elt_is_top(ev))
				notify = 1;
			event_queue_remove(base, ev, EVLIST_TIMEOUT);
		}
	
		/* Check if it is active due to a timeout.  Rescheduling
		 * this timeout before the callback can be executed
		 * removes it from the active list. */
		//如果待添加的事件已经被激活且激活原因是超时，应该从活动事件队列中删除它
		if ((ev->ev_flags & EVLIST_ACTIVE) &&
		    (ev->ev_res & EV_TIMEOUT)) {
			if (ev->ev_events & EV_SIGNAL) {
				/* See if we are just active executing
				 * this event in a loop
				 */
				if (ev->ev_ncalls && ev->ev_pncalls) {
					/* Abort loop */
					*ev->ev_pncalls = 0;
				}
			}

			event_queue_remove(base, ev, EVLIST_ACTIVE);
		}

		gettime(base, &now);

		common_timeout = is_common_timeout(tv, base); // 判断是不是属于通用时间表
		
		if (tv_is_absolute) {
			ev->ev_timeout = *tv;

		} else if (common_timeout) {
			struct timeval tmp = *tv;
			tmp.tv_usec &= MICROSECONDS_MASK;
			evutil_timeradd(&now, &tmp, &ev->ev_timeout);
			ev->ev_timeout.tv_usec |=
			    (tv->tv_usec & ~MICROSECONDS_MASK);
		} else {
			evutil_timeradd(&now, tv, &ev->ev_timeout);
		}

		event_debug((
			 "event_add: timeout in %d seconds, call %p",
			 (int)tv->tv_sec, ev->ev_callback));
		// 最后插入定时器
		event_queue_insert(base, ev, EVLIST_TIMEOUT);
		// ? 为什么是定时器链表的第一个元素就要转换成时间堆加入？？？
		if (common_timeout) {
			struct common_timeout_list *ctl =
			    get_common_timeout_list(base, &ev->ev_timeout);
			if (ev == TAILQ_FIRST(&ctl->events)) {
				common_timeout_schedule(ctl, &now, ev);
			}
		} else {
			/* See if the earliest timeout is now earlier than it
			 * was before: if so, we will need to tell the main
			 * thread to wake up earlier than it would
			 * otherwise. */
			if (min_heap_elt_is_top(ev))
				notify = 1;
		}
	}

	/* if we are not in the right thread, we need to wake up the loop */
	if (res != -1 && notify && EVBASE_NEED_NOTIFY(base))
		evthread_notify_base(base);

	_event_debug_note_add(ev);

	return (res);
}
```
梳理一下主体思路和方法，由于添加的事件只可能是IO事件、信号处理事件和定时器，所以要考虑三种不同类型的事件来处理添加注册。  
对于**定时器**事件，首先在时间堆和通用事件表上预留位置但不填充，在处理完一些潜在的情况之后，代码的最后用event_queue_insert(base, ev, EVLIST_TIMEOUT);插入到定时器队列或时间堆，之所以不在开始就插入是因为IO事件和信号事件也带有超时时间，同样有情况需要加入到定时队列或时间堆中。  
添加IO事件和信号事件使用evmap_io_add和evmap_signal_add函数并同时建立事件处理器的映射关系，并且事件多路分发器中成功添加了新的事件所以要通知主线程。  
  
  
  
**event_queue_insert**将事件处理器添加到各种事件队列中  

## 往事件多路分发器中注册事件
对于添加的事件处理器，需要让事件多路分发器来监听其对应的事件，同事建立文件描述符、信号值与事件处理器之间的映射关系。其由evmap_io_add和evmap_signal_add两个函数来完成，在evmap.c文件实现。  
在Linux环境下，由于文件描述符和信号值是连续的正整数，可以通过数组索引来直接建立映射。而在win下，映射关系采用哈希表实现event_io_map定义。
```
struct event_signal_map {
        void **entries;// 信号值和信号处理器之间的关系
	/* The number of entries available in entries */
	int nentries;
};

struct evmap_io {
	struct event_list events; //IO事件队列 
	ev_uint16_t nread;
	ev_uint16_t nwrite;
};

struct evmap_signal {
	struct event_list events; //信号事件队列
};
```
IO实现和信号相同，这里以添加IO事件为例子。往事件多路分发中的添加是使用后端统一封装结构中的add函数添加。在后面会分析后端统一复用IO eventop结构的内容。

```
/* return -1 on error, 0 on success if nothing changed in the event backend,
 * and 1 on success if something did. 
 事件添加到evmap_io队列中并且建立事件与事件处理器之间的映射关系 */
int
evmap_io_add(struct event_base *base, evutil_socket_t fd, struct event *ev)
{
	// 获得event_base的后端IO复用机制
	const struct eventop *evsel = base->evsel;
	struct event_io_map *io = &base->io;
	// fd参数对应的IO事件队列
	struct evmap_io *ctx = NULL;
	int nread, nwrite, retval = 0;
	short res = 0, old = 0;
	struct event *old_ev;
	
	EVUTIL_ASSERT(fd == ev->ev_fd);

	if (fd < 0)
		return 0;

#ifndef EVMAP_USE_HT
	if (fd >= io->nentries) { //如果fd大于nentries则需要增大数组的容量
		if (evmap_make_space(io, fd, sizeof(struct evmap_io *)) == -1)
			return (-1);
	}
#endif
	//创建ctx，在映射表io中为fd和ctx添加映射关系
	GET_IO_SLOT_AND_CTOR(ctx, io, fd, evmap_io, evmap_io_init,
						 evsel->fdinfo_len);

	nread = ctx->nread;
	nwrite = ctx->nwrite;

	if (nread)
		old |= EV_READ;
	if (nwrite)
		old |= EV_WRITE;

	if (ev->ev_events & EV_READ) {
		if (++nread == 1)
			res |= EV_READ;
	}
	if (ev->ev_events & EV_WRITE) {
		if (++nwrite == 1)
			res |= EV_WRITE;
	}
	if (EVUTIL_UNLIKELY(nread > 0xffff || nwrite > 0xffff)) {
		event_warnx("Too many events reading or writing on fd %d",
		    (int)fd);
		return -1;
	}
	if (EVENT_DEBUG_MODE_IS_ON() &&
	    (old_ev = TAILQ_FIRST(&ctx->events)) &&
	    (old_ev->ev_events&EV_ET) != (ev->ev_events&EV_ET)) {
		event_warnx("Tried to mix edge-triggered and non-edge-triggered"
		    " events on fd %d", (int)fd);
		return -1;
	}

	if (res) {
		void *extra = ((char*)ctx) + sizeof(struct evmap_io);
		/* XXX(niels): we cannot mix edge-triggered and
		 * level-triggered, we should probably assert on
		 * this. */
		//往事件多路分发器中注册事件，add是事件多路分发器的接口之一
		if (evsel->add(base, ev->ev_fd,
			old, (ev->ev_events & EV_ET) | res, extra) == -1)
			return (-1);
		retval = 1; 
	}

	ctx->nread = (ev_uint16_t) nread;
	ctx->nwrite = (ev_uint16_t) nwrite;
	// 将ev插入到IO事件队列ctx的尾部
	TAILQ_INSERT_TAIL(&ctx->events, ev, ev_io_next);

	return (retval);
}
```

## eventop结构体
eventop封装了IO复用机制必要的一些操作，如注册事件、等待事件。为evnet_base支持的所有后端IO复用机制提供了一个统一的接口。在实际实现的时候会根据环境的不同用到select、epoll、kqueue等不同的IO多路复用系统调用。eventop的功能就是像C++类中多态一样，为上层提供一个统一的接口函数。  
```
/*eventop结构 */
struct eventop {
    const char *name;
    void *(*init)(struct event_base *); // 初始化
    int (*add)(void *, struct event *); // 注册事件
    int (*del)(void *, struct event *); // 删除事件
    int (*dispatch)(struct event_base *, void *, struct timeval *); // 事件分发
    void (*dealloc)(struct event_base *, void *); // 注销，释放资源
    /* set if we need to reinitialize the event base */
    int need_reinit;
    //IO复用技术支持的一些特性选项
	enum event_method_feature features;
	//每个IO、信号队列的额外内存长度
	size_t fdinfo_len;
};
```
在C语言中实现接口的统一，libevent中是这么操作的。  
* 首先extern声明外部实例，即每个最后用到的eventop是一个实例，但在外部定义（如在epoll.c select.c中定义)。  
```
// 全局实例对象
// Libevent支持的后端IO复用技术以及它们的优先级
#ifdef _EVENT_HAVE_EVENT_PORTS
extern const struct eventop evportops;
#endif
#ifdef _EVENT_HAVE_SELECT
extern const struct eventop selectops;
#endif
#ifdef _EVENT_HAVE_POLL
extern const struct eventop pollops;
#endif
#ifdef _EVENT_HAVE_EPOLL
extern const struct eventop epollops;
#endif
#ifdef _EVENT_HAVE_WORKING_KQUEUE
extern const struct eventop kqops;
#endif
#ifdef _EVENT_HAVE_DEVPOLL
extern const struct eventop devpollops;
#endif
#ifdef WIN32
extern const struct eventop win32ops;
#endif

```
* 设定使用IO复用种类的优先级，存放在eventops[]数组中。  
*
```
static const struct eventop *eventops[] = {
#ifdef _EVENT_HAVE_EVENT_PORTS
	&evportops,
#endif
#ifdef _EVENT_HAVE_WORKING_KQUEUE
	&kqops, 
#endif
#ifdef _EVENT_HAVE_EPOLL
	&epollops,
#endif
#ifdef _EVENT_HAVE_DEVPOLL
	&devpollops,
#endif
#ifdef _EVENT_HAVE_POLL
	&pollops,
#endif
#ifdef _EVENT_HAVE_SELECT
	&selectops,
#endif
#ifdef WIN32
	&win32ops,
#endif
	NULL
};
```
* event_base_new()函数中，根据eventops数组来确定选用哪一种IO复用。注意这是在编译时完成的，也就是说运行时是不能够更改IO复用方式的。
```
 base->evbase = NULL;
     for (i = 0; eventops[i] && !base->evbase; i++) {
         base->evsel = eventops[i];
         base->evbase = base->evsel->init(base);
    }
 ```
 * 在各个复用封装源文件中，需要对struct eventop中的函数指针进行定向。
 以epoll.c为例子，实例epollops被定义如下
 ```
 const struct eventop epollops = {
	"epoll",
	epoll_init,
	epoll_nochangelist_add,
	epoll_nochangelist_del,
	epoll_dispatch,
	epoll_dealloc,
	1, /* need reinit */
	EV_FEATURE_ET|EV_FEATURE_O1,
	0
};
 ```
 其中的具体方法是epoll函数的封装使用。在epoll.c源文件中被定义。  

## event_base结构体
一个event_base实际上是一个Reactor实例，定义了整个Reactor执行所要求的各种参数。  
```
struct event_base {
	/** Function pointers and other data to describe this event_base's
	 * backend. 选择一种IO后端复用机制*/
	const struct eventop *evsel;
	/** Pointer to backend-specific data. 指向IO复用机制真正存储的数据*/
	void *evbase;

	/** List of changes to tell backend about at next dispatch.  Only used
	 * by the O(1) backends. */
	struct event_changelist changelist; // 缓冲队列

	/** Function pointers used to describe the backend that this event_base
	 * uses for signals */
	const struct eventop *evsigsel;
	// 信号事件处理器使用的数据结构,封装了一个sockpair创建的管道，回想信号统一事件源的设计
	/** Data to implement the common signal handelr code. */
	struct evsig_info sig;

	// 添加到该event_base的虚拟事件、所有事件和激活事件的数量
	/** Number of virtual events */
	int virtual_event_count;
	/** Number of total events added to this event_base */
	int event_count;
	/** Number of total events active in this event_base */
	int event_count_active;

	/** Set if we should terminate the loop once we're done processing
	 * events. */
	int event_gotterm;
	/** Set if we should terminate the loop immediately */
	int event_break;
	/** Set if we should start a new instance of the loop immediately. */
	int event_continue;

	/** The currently running priority of events */
	int event_running_priority;

	/** Set if we're running the event_base_loop function, to prevent
	 * reentrant invocation. */
	int running_loop;

	/* Active event management. */
	/** An array of nactivequeues queues for active events (ones that
	 * have triggered, and whose callbacks need to be called).  Low
	 * priority numbers are more important, and stall higher ones.
	 * 活动事件队列数组，索引越小优先级越高
	 */
	struct event_list *activequeues;
	/** The length of the activequeues array */// 组数优先级
	int nactivequeues;

	/* common timeout logic */

	/** An array of common_timeout_list* for all of the common timeout
	 * values we know. */
	struct common_timeout_list **common_timeout_queues;
	/** The number of entries used in common_timeout_queues */
	int n_common_timeouts;
	/** The total size of common_timeout_queues. */
	int n_common_timeouts_allocated;

	/** List of defered_cb that are active.  We run these after the active
	 * events. */
	struct deferred_cb_queue defer_queue; // 回调函数链表

	/** Mapping from file descriptors to enabled (added) events */
	struct event_io_map io;

	/** Mapping from signal numbers to enabled (added) events. */
	struct event_signal_map sigmap;

	/** All events that have been enabled (added) in this event_base
	 * 注册事件队列
	 */
	struct event_list eventqueue;

	/** Stored timeval; used to detect when time is running backwards. */
	struct timeval event_tv;

	/** Priority queue of events with timeouts. */
	struct min_heap timeheap;

	/** Stored timeval: used to avoid calling gettimeofday/clock_gettime
	 * too often. */
	struct timeval tv_cache;

#if defined(_EVENT_HAVE_CLOCK_GETTIME) && defined(CLOCK_MONOTONIC)
	/** Difference between internal time (maybe from clock_gettime) and
	 * gettimeofday. */
	struct timeval tv_clock_diff;
	/** Second in which we last updated tv_clock_diff, in monotonic time. */
	time_t last_updated_clock_diff;
#endif

/* 多线程支持 */
#ifndef _EVENT_DISABLE_THREAD_SUPPORT
	/* threading support */
	/** The thread currently running the event_loop for this base */
	unsigned long th_owner_id;// 当前运行事件循环的线程
	/** A lock to prevent conflicting accesses to this event_base */
	void *th_base_lock; // 对event_base的独占锁
	/** The event whose callback is executing right now */
	struct event *current_event;
	/** A condition that gets signalled when we're done processing an
	 * event with waiters on it. */
	void *current_event_cond;// 条件变量
	/** Number of threads blocking on current_event_cond. */
	int current_event_waiters; // 等待event的线程数
#endif

#ifdef WIN32
	/** IOCP support structure, if IOCP is enabled. */
	struct event_iocp_port *iocp;
#endif

	/** Flags that this base was configured with */
	enum event_base_config_flag flags;

	/* Notify main thread to wake up break, etc. */
	/** True if the base already has a pending notify, and we don't need
	 * to add any more. */
	int is_notify_pending;
	/** A socketpair used by some th_notify functions to wake up the main
	 * thread. */
	evutil_socket_t th_notify_fd[2];
	/** An event used by some th_notify functions to wake up the main
	 * thread. */
	struct event th_notify;
	/** A function used to wake up the main thread from another thread. */
	int (*th_notify_fn)(struct event_base *base);
};
```

## 事件循环
Libevent事件循环主要是通过event_base_loop()函数完成，流程图如下。  
![](_v_images/20190423135113688_1985233078.png)
```
int
event_base_loop(struct event_base *base, int flags)
{
	const struct eventop *evsel = base->evsel;
	struct timeval tv;
	struct timeval *tv_p;
	int res, done, retval = 0;

	/* Grab the lock.  We will release it inside evsel.dispatch, and again
	 * as we invoke user callbacks. */
	EVBASE_ACQUIRE_LOCK(base, th_base_lock);

	//有其他线程运行事件循环
	if (base->running_loop) {
		event_warnx("%s: reentrant invocation.  Only one event_base_loop"
		    " can run on each event_base at once.", __func__);
		EVBASE_RELEASE_LOCK(base, th_base_lock);
		return -1;
	}

	base->running_loop = 1;

	clear_time_cache(base);
	//设置信号事件的event_base实例
	if (base->sig.ev_signal_added && base->sig.ev_n_signals_added)
		evsig_set_base(base);

	done = 0;

#ifndef _EVENT_DISABLE_THREAD_SUPPORT
	base->th_owner_id = EVTHREAD_GET_ID();
#endif

	base->event_gotterm = base->event_break = 0;

	while (!done) {
		base->event_continue = 0;

		/* Terminate the loop if we have been asked to */
		if (base->event_gotterm) {
			break;
		}

		if (base->event_break) {
			break;
		}

		timeout_correct(base, &tv); //更新系统时间

		tv_p = &tv;
		if (!N_ACTIVE_CALLBACKS(base) && !(flags & EVLOOP_NONBLOCK)) {//没有激活事件且不阻塞
			timeout_next(base, &tv_p); //获取时间堆上堆顶元素的超时值，IO复用系统调用本次应设置的超时值
		} else {
			/*
			 * if we have active events, we just poll new events
			 * without waiting.
			 */
			evutil_timerclear(&tv);// IO系统调用超时值设置成0 立即返回
		}

		/* If we have no events, we just exit */
		if (!event_haveevents(base) && !N_ACTIVE_CALLBACKS(base)) {
			event_debug(("%s: no events registered.", __func__));
			retval = 1;
			goto done;
		}

		//更新系统时间
		/* update last old time */
		gettime(base, &base->event_tv);

		clear_time_cache(base);

		//调用事件多路分发器的dispatch方法等待事件，将就绪事件插入活动事件队列
		res = evsel->dispatch(base, tv_p);

		if (res == -1) {
			event_debug(("%s: dispatch returned unsuccessfully.",
				__func__));
			retval = -1;
			goto done;
		}

		update_time_cache(base);
		// 检查时间堆上的到期事件并依次执行
		timeout_process(base);

		if (N_ACTIVE_CALLBACKS(base)) {
			// 依次处理就绪信号事件和IO事件
			int n = event_process_active(base);
			if ((flags & EVLOOP_ONCE)
			    && N_ACTIVE_CALLBACKS(base) == 0
			    && n != 0)
				done = 1;
		} else if (flags & EVLOOP_NONBLOCK)
			done = 1;
	}
	event_debug(("%s: asked to terminate loop.", __func__));

done:
	//事件循环结束，清空时间缓存，设置停止循环标志
	clear_time_cache(base);
	base->running_loop = 0;

	EVBASE_RELEASE_LOCK(base, th_base_lock);

	return (retval);
}
```

[参考：libevent源码深度剖析](https://www.cnblogs.com/lfsblack/p/5498556.html)