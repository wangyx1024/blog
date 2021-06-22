### 回顾一下线程池运行机制

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

### execute()源码

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
    // 2.上面新建线程前其实没有判断线程池状态，在addWorker中判断线程池状态时发现线程池已经关闭了

    // 整合一下上面的信息，走到这里时一共有2种情况：
    // 1.线程数大于核心线程数，无法新建核心线程
    // 2.线程数小于核心线程数，新建核心线程失败，也许是线程池关闭了或者并发抢新建失败
    // 总之一句话：无法新建新核心线程去处理当前任务

    // 查看线程池是否在运行，如果在运行的话，把当前任务加入队列（添加队列可能失败？是的，比如队列满了）
    if (isRunning(c) && workQueue.offer(command)) {
        // 再提醒一下，走到这里时，线程池在运行，无法新建核心线程，任务在队列里
        // 添加队列成功后，重新检查一下线程池状态（为啥要做这步？）
        // 如果线程池已经停止运行了，从队列中把任务移除，并且拒绝当前任务（为啥拒绝，因为线程池停止了吗，不拒绝会怎样），结束
        // 总结一下这条分支，无法新建核心线程，于是改为添加到队列。添加成功后再次检查线程池的状态，哦豁，线程池已经关闭了，刚才白添加队列了，于是再尝试把任务从队列中移除，并执行拒绝策略，结束
        // 这条分支并不属于线程池运行机制的分支，是容错（任务加入队列后线程池关闭）的处理
        int recheck = ctl.get();
        if (! isRunning(recheck) && remove(command))
            reject(command);

        // 走到这里是什么情况？
        // 1.线程池在运行（真的在运行，recheck过了），无法新建核心线程，添加队列成功
        // 2.无法新建核心线程，添加队列成功，但添加后发现线程池已经停止了，所以企图移除队列but移除失败了（看起来不是主分支，回头先不考虑）
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



