# 函数解析

`void* threadpool_thread(void* threadpool);` 线程池中工作线程要做的事情

`void* adjust_thread(void *threadpool);` 管理线程要做的事情

`int threadpool_free(threadpool_t *pool);` 释放线程

`threadpool_t *threadpool_create(int min_thr_num, int max_thr_num, int queue_max_size);` 创建线程池

`int threadpool_add(threadpool_t *pool, void *(*function)(void *arg), void *arg);` 向任务队列增加一个任务

`int threadpool_destroy(threadpool_t *pool);` 销毁线程池

线程池结构体

```c
typedef struct threadpool_t{
    pthread_mutex_t lock;               /* 用于锁住本结构体 ，和条件变量一起使用 */
    pthread_mutex_t thread_counter;     /* 记录忙状态线程个数的锁 -- busy_thr_num */
    pthread_cond_t queue_not_full;      /* 当任务队列满时，添加任务的线程阻塞，等待此条件变量 */
    pthread_cond_t queue_not_empty;     /* 任务队列里不为空时，通知线程池中等待任务的线程 */

    pthread_t *threads;                 /* 存放线程池中每个线程的tid。数组 */
    pthread_t adjust_tid;               /* 存管理线程tid */
    threadpool_task_t *task_queue;      /* 任务队列 */

    int min_thr_num;                    /* 线程池最小线程数 */
    int max_thr_num;                    /* 线程池最大线程数 */
    int live_thr_num;                   /* 当前存活线程个数 */
    int busy_thr_num;                   /* 忙状态线程个数 */
    int wait_exit_thr_num;              /* 要销毁的线程个数 */

    int queue_front;                    /* task_queue队头下标 */
    int queue_rear;                     /* task_queue队尾下标 */
    int queue_size;                     /* task_queue队中实际任务数 */
    int queue_max_size;                 /* task_queue队列可容纳任务数上限 */

    int shutdown;                       /* 标志位，线程池使用状态，true或false */
} threadpool_t;
```

# 互斥量

```c
int pthread_detach(pthread_t thread);    //成功：0；失败：错误号

// if 0
 
        pthread_attr_t attr;            /*通过线程属性来设置游离态（分离态）*/
        pthread_attr_init(&attr);
        pthread_attr_setdetachstate(&attr, PTHREAD_CREATE_DETACHED);
        pthread_create(&tid, &attr, tfn, NULL);
 
// else
 
        pthread_create(&tid, NULL, tfn, NULL);
        pthread_detach(tid);         //让线程分离  ----自动退出,无系统残留资源

```

作用：从状态上实现线程分离，注意不是指该线程独自占用地址空间。

**线程分离状态：**指定该状态，线程主动与主控线程断开关系。线程结束后（不会产生僵尸线程），其退出状态不由其他线程获取，而直接自己自动释放（自己清理掉PCB的残留资源）。网络、多线程服务器常用。



```c
pthread_cond_wait();
```

用于阻塞当前线程，等待别的线程使用`pthread_cond_signal()`或`pthread_cond_broadcast`来唤醒它 `pthread_cond_wait()` 必须与`pthread_mutex`配套使用。`pthread_cond_wait()`函数一进入`wait`状态就会自动`release mutex`。当其他线程通过`pthread_cond_signal()`或`pthread_cond_broadcast`，把该线程唤醒，使`pthread_cond_wait()`通过（返回）时，该线程又自动获得该`mutex`。



```c
pthread_cond_signal();
```

函数的作用是发送一个信号给另外一个正在处于阻塞等待状态的线程,使其脱离阻塞状态,继续执行.如果没有线程处在阻塞等待状态,`pthread_cond_signal`也会成功返回。
使用`pthread_cond_signal`一般不会有“惊群现象”产生，他最多只给一个线程发信号。假如有多个线程正在阻塞等待着这个条件变量的话，那么是根据各等待线程优先级的高低确定哪个线程接收到信号开始继续执行。如果各线程优先级相同，则根据等待时间的长短来确定哪个线程获得信号。但无论如何一个`pthread_cond_signal`调用最多发信一次。

但是`pthread_cond_signal`在多处理器上可能同时唤醒多个线程，当你只能让一个线程处理某个任务时，其它被唤醒的线程就需要继续
`wait`，而且规范要求`pthread_cond_signal`至少唤醒一个`pthread_cond_wait`上的线程，其实有些实现为了简单在单处理器上也会唤醒多个线程. 另外，某些应用，如线程池，`pthread_cond_broadcast`唤醒全部线程，但我们通常只需要一部分线程去做执行任务，所以其它的线程需要继续wait.所以强烈推荐对`pthread_cond_wait()`使用`while`循环来做条件判断。