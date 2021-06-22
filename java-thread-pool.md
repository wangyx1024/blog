### 线程池运行机制

1. 当前线程数小于核心线程数时，新建线程，并让这个线程去执行任务
2. 否则看看队列有没有满（当然无界队列可以认为不会满），如果没有满，就把任务加入队列
3. 如果队列也满了，再比较当前线程数是否小于最大线程数，如果小于就创建非核心线程
4. 如果非核心线程数也满了，说明线程池已经完全饱和，只好去执行拒绝策略

### 简介addWorker()

+ 两个参数：1.Runnable firstTask创建出来的线程要执行的任务，2.boolean core是否核心线程
+ 这两种参数的4种组合代表的含义如下：
    1. addWorker(command, true)，创建核心线程，如果线程数>=核心线程数则return false
    2. addWorker(command, false)，创建非核心线程，如果线程数>=最大线程数则return false
    3. addWorker(null, true)，只创建核心线程，不立刻执行任务
    4. addWorker(null, false)，只创建非核心线程，不立刻执行任务
+ 另外4种情况遇到线程池状态非运行中都会return false

### execute()

#### 源码



```java
public void execute(Runnable command) {
    if (command == null)
        throw new NullPointerException();
    /*
     * Proceed in 3 steps:
     *
     * 1. If fewer than corePoolSize threads are running, try to
     * start a new thread with the given command as its first
     * task.  The call to addWorker atomically checks runState and
     * workerCount, and so prevents false alarms that would add
     * threads when it shouldn't, by returning false.
     *
     * 2. If a task can be successfully queued, then we still need
     * to double-check whether we should have added a thread
     * (because existing ones died since last checking) or that
     * the pool shut down since entry into this method. So we
     * recheck state and if necessary roll back the enqueuing if
     * stopped, or start a new thread if there are none.
     *
     * 3. If we cannot queue task, then we try to add a new
     * thread.  If it fails, we know we are shut down or saturated
     * and so reject the task.
     */

    // 如果线程数小于核心线程数，就新建线程（不判断线程池状态吗？不判断，在addWorker中统一判断），并让这个新建的线程去执行任务。如果新建线程成功了直接返回，非常顺利且简短的一种情况，对应线程池机制的case1
    // 但是！！！新建线程是有可能会失败的，啥情况会失败，下面说
    int c = ctl.get();
    if (workerCountOf(c) < corePoolSize) {
        // 增加核心线程并立即执行任务
        if (addWorker(command, true))
            return;
        // 增加线程失败时更新一下线程池状态的参数
        c = ctl.get();
    }

    // 啥情况会新建线程失败：
    // 1.判断线程数量时，确实没超过核心线程数，但是在判断后到新建前这段时间，被别的线程抢了先，也就是出现了并发提交任务并且竞争创建线程失败了
    // 2.新建线程前没有判断线程池状态，在addWorker中判断线程池状态时发现线程池已经关闭了

    // 整合一下上面的信息，走到这里时一共有2种情况：
    // 1.线程数大于核心线程数，无法新建核心线程
    // 2.线程数小于核心线程数，新建核心线程失败，也许是线程池关闭了或者被别的线程抢先新建导致超过了核心线程数
    // 总之一句话：无法新建核心线程去处理当前任务

    // 查看线程池是否在运行，如果在运行的话，把当前任务加入队列（添加队列可能失败？是的，比如队列满了）
    if (isRunning(c) && workQueue.offer(command)) {
        // 再提醒一下，走到这里时，线程池在运行，无法新建核心线程，任务在队列里
      
        // 添加队列成功后，重新检查一下线程池状态（为啥要做这步？）
        // 如果线程池已经停止运行了，队列移除任务，并执行拒绝策略（为啥拒绝？因为期望线程池执行任务，但线程池无法执行），结束
        // 总结一下这条分支，无法新建核心线程，于是改为添加到队列。添加成功后再次检查线程池的状态，哦豁，线程池已经关闭了，刚才白添加队列了，于是再尝试把任务从队列中移除，并执行拒绝策略，结束
        // 这条分支并不属于线程池运行机制的分支，是容错（任务加入队列后线程池关闭）的处理
        int recheck = ctl.get();
        if (! isRunning(recheck) && remove(command))
            reject(command);

        // 走到这里是什么情况？
        // 1.线程池在运行（真的在运行，recheck过了），无法新建核心线程，任务在队列中
        // 2.无法新建核心线程，任务在队列中，但添加后发现线程池已经停止了，所以企图移除队列but移除失败了（看起来不是主分支，回头先不考虑）
        // 总结一下就是，无法新建核心线程，任务在队列中，线程池运行与否都有可能


        // 下面要执行的：判断当前线程数量，如果是0就新建非核心线程（为啥是0？）。注意看addWorker()的参数，新建非核心线程，并且不立即处理任务。也就是说单纯给线程池增加一个非核心线程而已
        // 结合一下上面的情况
        // 1.无法新建核心线程，添加队列成功，线程池在运行，当前线程数量是0，新建非核心线程（仅新建，不立即处理当前任务），新建成功对应运行机制case3
        // 2.无法新建核心线程，添加队列成功，线程池没在运行，当前线程数量是0，去addWorker()。因为线程池没在运行，所以一定是失败，这里为啥没去执行拒绝策略？还是说上面的分支如果线程池没在运行了remove一定会成功？
        else if (workerCountOf(recheck) == 0)
            addWorker(null, false);


        // 走到这里说明什么？
        // 无法新建核心线程，添加队列成功，线程池运行中，线程数量>0，对应线程池机制的case2
    }

    // 走到这里是什么情况？
    // 1.线程池没在运行了
    // 2.线程池在运行，不过添加队列失败，因为线程池满了

    // 下面的代码比较清晰，新建一个非核心线程去执行任务（线程池没在运行了还要增加线程？addWorker中会处理），如果失败了就拒绝任务
    // 结合上面的情况分析一下
    // 1.如果线程池没在运行，addWorker一定失败，所以直接去执行拒绝策略没毛病，同样是容错机制
    // 2.如果线程池在运行，但是添加任务到队列失败了（队列满了），这时尝试新建非核心线程去处理任务，创建成功对应线程池机制的case3，创建失败（非核心线程数也满了）对应case4
    else if (!addWorker(command, false))
        reject(command);
}
```

#### 总结

+ 简单理解对应线程池运行机制4条

### addWorker()

#### 源码

```java
/**
 * Checks if a new worker can be added with respect to current
 * pool state and the given bound (either core or maximum). If so,
 * the worker count is adjusted accordingly, and, if possible, a
 * new worker is created and started, running firstTask as its
 * first task. This method returns false if the pool is stopped or
 * eligible to shut down. It also returns false if the thread
 * factory fails to create a thread when asked.  If the thread
 * creation fails, either due to the thread factory returning
 * null, or due to an exception (typically OutOfMemoryError in
 * Thread.start()), we roll back cleanly.
 *
 * @param firstTask the task the new thread should run first (or
 * null if none). Workers are created with an initial first task
 * (in method execute()) to bypass queuing when there are fewer
 * than corePoolSize threads (in which case we always start one),
 * or when the queue is full (in which case we must bypass queue).
 * Initially idle threads are usually created via
 * prestartCoreThread or to replace other dying workers.
 *
 * @param core if true use corePoolSize as bound, else
 * maximumPoolSize. (A boolean indicator is used here rather than a
 * value to ensure reads of fresh values after checking other pool
 * state).
 * @return true if successful
 */

// 主要两部分
// 1.校验能否增加Worker
// 2.增加Worker
private boolean addWorker(Runnable firstTask, boolean core) {
    // 这块for循环是干啥的？
    // 校验线程池状态、线程数量是否ok，并cas增加线程数量
    retry:
    for (;;) {
        int c = ctl.get();
        int rs = runStateOf(c);

        // Check if queue empty only if necessary.
        // 线程池状态：
        // -1 RUNNING，运行中
        //  0 SHUTDOWN，执行shutdown()命令。停止运行，拒绝接受新任务（不过如果有任务队列没完成好像可以创建新线程？），等待任务队列执行完
        //  1 STOP，执行shutdownNow()命令。停止运行，不仅拒绝新任务，任务队列未完成的任务也要终止，任务队列还没清理完
        //  2 TYDING，SHUTDOWN、STOP后，任务队列也清空了
        //  3 TERMINATED，TYDING的后一个状态
        
        // 这条if条件换个写法：rs >= SHUTDOWN && (rs != SHUTDOWN || firstTask != null || workQueue.isEmpty())
        // 其中rs >= SHUTDOWN，表示SHUTDOWN、STOP、TYDING、TERMINATED，共4种状态
        // 当下列两个条件同时满足时return false
        // 1.如果状态是4者之一：SHUTDOWN、STOP、TYDING、TERMINATED
        // 2.状态是STOP、TYDING、TERMINATED之一  或  firstTask不为null  或  队列为空
        
        // 再翻译一下
        // 线程池状态是STOP、TYDING、TERMINATED时，不允许添加任务，也不允许添加任何线程，包括核心非核心，所以return false
        // 线程池状态是SHUTDOWN时，1.不允许添加任务（firstTask!=null）return false，如果有任务没执行完允许添加线程，2.任务已经执行完了，线程也不允许添加了
        // 简单说就是线程池的状态是否允许进行当前操作
        if (rs >= SHUTDOWN &&
            ! (rs == SHUTDOWN &&
               firstTask == null &&
               ! workQueue.isEmpty()))
            return false;
        
        // 这个for循环又是干啥的？
        // 如果线程池状态没变就一直cas尝试增加线程数量，直到
        // 1.添加成功，往下走
        // 2.线程池状态变化，跳到外层循环重新来过
        for (;;) {
            // 判断线程数量是否超限，超过了return false
            // 1.线程数量是否达到线程池容量上线（32-3=29位，5亿多）
            // 2.根据添加的是核心\非核心线程，判断是否超过coreSize\maxSize
            int wc = workerCountOf(c);
            if (wc >= CAPACITY ||
                wc >= (core ? corePoolSize : maximumPoolSize))
                return false;
            
            // cas增加线程数量，增加成功跳出外层循环（retry处）进行下一步
            // 否则更新一下线程池状态、线程数量，再判断线程池状态是否变化，如果没变就继续cas增加线程，否则重新执行外层循环（retry处）的步骤
            if (compareAndIncrementWorkerCount(c))
                break retry;
            c = ctl.get();  // Re-read ctl
            if (runStateOf(c) != rs)
                continue retry;
            // else CAS failed due to workerCount change; retry inner loop
        }
    }
    
    // 走到这里说明上面的校验通过了：线程池状态、线程数量
    // 下面开始创建Worker去执行任务
    boolean workerStarted = false;
    boolean workerAdded = false;
    Worker w = null;
    try {
        // 初始化Worker对象，从线程工厂new一个线程作为Worker的线程（Worker的thread和command有点混😷）
        w = new Worker(firstTask);
        final Thread t = w.thread;
        // 啥情况会是null，ThreadFactory是一个接口，也许是为了兼容第三方实现？
        if (t != null) {
            
            final ReentrantLock mainLock = this.mainLock;
            mainLock.lock();
            try {
                // Recheck while holding lock.
                // Back out on ThreadFactory failure or if
                // shut down before lock acquired.
                int rs = runStateOf(ctl.get());
                
                // RUNNING 或 虽然SHUTDOWN但是没有task
                if (rs < SHUTDOWN ||
                    (rs == SHUTDOWN && firstTask == null)) {
                    // 没懂，如果活着就报错？
                    if (t.isAlive()) // precheck that t is startable
                        throw new IllegalThreadStateException();
                    // 加入到worker队列
                    workers.add(w);
                    // 更新一下worker最大数量并标记workerAdded
                    int s = workers.size();
                    if (s > largestPoolSize)
                        largestPoolSize = s;
                    workerAdded = true;
                }
            } finally {
                mainLock.unlock();
            }
          
            // worker添加成功就启动线程，并标记workerStarted
            if (workerAdded) {
                t.start();
                workerStarted = true;
            }
        }
    } finally {
        // worker创建失败的逻辑，擦屁股，比如刚才cas增加的worker数量要减回来
        if (! workerStarted)
            addWorkerFailed(w);
    }
    return workerStarted;
}
```

#### 总结

+ 根据线程池状态，判断是否允许添加线程
  + STOP之后的状态，不允许添加任务也不允许添加线程
  + SHUTDOWN状态，不允许添加任务，但如果有任务未完成，允许添加线程执行剩余任务
+ 根据当前线程数量和要添加的是否核心线程（core参数），判断是否允许添加线程
  + 核心\非线程的上限数量分别是coreSize和maxSize
  + 当然也必须小于线程池最大容量（5亿+）
+ 创建Worker对象，调用runWorker()执行任务，firstTask优先执行，之后执行getTask()获取任务

### getTask()

#### 源码

```java
/**
 * Performs blocking or timed wait for a task, depending on
 * current configuration settings, or returns null if this worker
 * must exit because of any of:
 * 1. There are more than maximumPoolSize workers (due to
 *    a call to setMaximumPoolSize).
 * 2. The pool is stopped.
 * 3. The pool is shutdown and the queue is empty.
 * 4. This worker timed out waiting for a task, and timed-out
 *    workers are subject to termination (that is,
 *    {@code allowCoreThreadTimeOut || workerCount > corePoolSize})
 *    both before and after the timed wait, and if the queue is
 *    non-empty, this worker is not the last thread in the pool.
 *
 * @return task, or null if the worker must exit, in which case
 *         workerCount is decremented
 */
private Runnable getTask() {
    boolean timedOut = false; // Did the last poll() time out?

    for (;;) {
        int c = ctl.get();
        int rs = runStateOf(c);

        // 整理一下：(rs >= SHUTDOWN && rs >= STOP) || (rs >= SHUTDOWN && workQueue.isEmpty())
        // 再化简：rs>=STOP || (rs==SHUTDOWN && workQueue.isEmpty())
        // 两种情况分别是：
        // 1.线程池关闭，且不执行任务队列时，不需要执行剩余任务，返回null
        // 2.线程池关闭，允许任务队列继续执行，但任务队列已经执行完了，无剩余任务可执行，返回null
        // 总结一下，线程池处于关闭状态，且没有要执行的任务时，减少线程数量，返回null告知外层，可以干掉本worker了
        // Check if queue empty only if necessary.
        if (rs >= SHUTDOWN && (rs >= STOP || workQueue.isEmpty())) {
            decrementWorkerCount();
            return null;
        }
      
        // 是否需要回收线程，下面两种情况需要回收（timed这个名字有些怪）
        // 1.当前线程数>核心线程数，这个很好理解
        // 2.允许回收核心线程参数（允许回收核心线程，和coreSize=0只使用maxSize有啥区别）
        // 
        int wc = workerCountOf(c);
        // Are workers subject to culling?
        boolean timed = allowCoreThreadTimeOut || wc > corePoolSize;

        // 下面2种情况需要回收线程
        // 1.线程数超过最大线程数
        // 2.需要回收线程且确实超时了
        // 但是并不是满足2个之一就立马回收，还需要满足一个条件：线程数>1 或 任务队列为空。也就是说有任务且只剩一个线程时，是不允许回收它的，不然岂不是回收完又要创建线程去执行这个任务
        // P.S 为啥会有当前线程>最大线程这种情况？调用了setCorePoolSize(int corePoolSize)
        if ((wc > maximumPoolSize || (timed && timedOut))
            && (wc > 1 || workQueue.isEmpty())) {
            if (compareAndDecrementWorkerCount(c))
                return null;
            continue;
        }

        try {
            // 如果需要回收线程，就限时拉取任务，到时间了还拉不到就设置超时，下一轮循环就干掉了
            // 如果不需要回收，就一直阻塞获取
            Runnable r = timed ?
                workQueue.poll(keepAliveTime, TimeUnit.NANOSECONDS) :
                workQueue.take();
            if (r != null)
                return r;
            timedOut = true;
        } catch (InterruptedException retry) {
            timedOut = false;
        }
    }
}
```

#### 总结

+ 先校验线程池状态，处于关闭状态且没有要执行的任务时回收线程
+ 再校验线程数量，判断是否需要回收线程
  + 首先，下面两种情况需要回收
    1. 超过了最大线程数
    2. 需要回收线程且确实超时了
  + 其次，如果只剩一个线程且还有任务要执行（workQueue不为空）也不允许回收
+ 从任务队列阻塞获取任务
  + 如果需要回收，等待有限时长（poll(keepAliveTime)），失败后设置超时状态，下一轮循环可能回收
  + 不需要回收一直阻塞获取（take()）



