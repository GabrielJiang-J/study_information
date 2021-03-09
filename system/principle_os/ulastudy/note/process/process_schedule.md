## 进程调度

### 远古调度方式

### O(1)调度方式

### CFS调度类
#### CFS基本原理
> 设定一个调度周期（sched_latency_ns），目标是让每个进程在这个周期内至少有机会运行一次，换一种说法就是每个进程等待CPU的时间最长不超过这个调度周期；然后根据进程的数量，大家平分这个调度周期内的CPU使用权，由于进程的优先级即nice值不同，分割调度周期的时候要加权；每个进程的累计运行时间保存在自己的vruntime字段里，哪个进程的vruntime最小就获得本轮运行的权利。
**vruntime相当于对实际运行时间根据进程权重进行修正。vruntime短的进程说明此进程对CPU的使用已经不公平，因此下次调度时就优先选择此进程运行，以弥补不公平带来的损失。进程运行的实际时间没有办法体现每个进程重要程度，或者是对时间的敏感程度，或者是进程的优先级，所以使用虚拟运行对实际运行时间就行修正，使用进程权重对实际运行时间加权，使虚拟运行时间既能体现进程的运行时间，又能体现进程的重要程度。**
所以CFS算法的机制在于对vruntime的管理，从而实现整体的相对公平。

举例来说：假如，某系统总共5个进程，调度粒度为5，
时刻T1，CPU0的就绪队列为：P5(3)->P1(6)->P4(7)->P2(10)->P3(11)
因为P5的vruntime为3，所以下次调度到的进程为P5，P5运行一段时间后，vruntime增加5，之后重新入队，则P5当前的vruntime为3+5+1=9，
时刻T2，CPU0的就绪队列为：P1(7)->P4(8)->P5(9)->P2(11)->P3(12)

> 与进程调度相关的系统文件：
> /proc/sched_debug: 此文件包含了调度器当前状态所有的信息。编译时启用CONFIG_SCHED_DEBUG，可以生成此文件。
> /proc/sys/kernel/sched_min_granularity_ns: 默认值为：4 msec * (1 + ilog(ncpus))，单位为纳秒。这个文件可以用来控制进程在CPU上最小的调度粒度，即如果当前进程运行时间不足sched_min_granularity_ns，则不能被调度出去，防止进程频繁切换，上下文频繁切换带来的性能损失。
> /proc/sys/kernel/sched_latency_ns: 默认值为20ms * (1 + ilog(ncpus))，单位为纳秒。这个文件用来控制系统的调度周期，目标是让每个进程在这个周期内至少有机会运行一次，也就是说每个进程等待CPU的时间最长不超过这个调度周期。如果系统进程数过多，每个进程的时间片就会太小，如果小于sched_min_granularity_ns，就以sched_min_granularity_ns为准，并且调度周期也不再为sched_latency_ns，而是以sched_min_granularity_ns * 进程数量为准。
> /proc/sys/kernel/sched_wakeup_granularity_ns: 默认值为：10 msec * (1 + ilog(ncpus))，单位为纳秒。这个文件可以用来控制唤醒的进程能够抢占当前进程之前必须满足的条件。只有当该唤醒进程的vruntime比当前进程的vruntime小、并且两者差距(vdiff)大于sched_wakeup_granularity_ns的情况下，才可以抢占，否则不可以。这个参数越大，发生唤醒抢占就越不容易。

CFS调度类的实现主要在于使用虚拟运行时间vruntime来模拟实际时间。vruntime用来度量调度实体可以得到的CPU时间。
vruntime的计算是通过实际时间和进程的负荷权重进行的。内核将进程的优先级通过一张转换表转化为对应的权重，
再根据权重将实际时间转换为虚拟运行时间，根据CPU队列中虚拟运行时间，来挑选最应该运行的实体进行调度。

此公式表示虚拟运行时间vruntime的增长方式；
$$
delta = delta \times \frac{NICE\_0\_LOAD}{cfs\_rq->curr->load->inv\_weight}
$$
