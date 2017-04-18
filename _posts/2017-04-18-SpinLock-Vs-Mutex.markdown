## SpinLock VS Mutex

SpinLock与Mutex最大的区别，在于加锁的时候，如果该锁被其他线程占用，那么若是Mutex，那么当前线程阻塞。若是SpinLock，那么当前线程会不断进行重试。

从[锁的开销](http://xbay.github.io/2015/12/31/%E9%94%81%E7%9A%84%E5%BC%80%E9%94%80/ "锁的开销")这篇blog提到锁的开销大致来自于三个部分：
1. *纯上下文切换开销。*
2. *调度器开销（把线程从睡眠变成就绪或者反过来），在多核系统上，还存在跨处理器调度的开销，那部分开销很大。*
3. *上下文切换带来的cache不命中和TLB不命中的开销。*

*具体开销大小及测试环境见链接*

### 测试代码
下面给出一个测试程序，对比下mutex和Spinlock的性能：
```
#include <stdio.h>
#include <stdlib.h>
#include <pthread.h>
#include <sys/syscall.h>
#include <unistd.h>

#define THREAD_NUM 2

pthread_t g_thread[THREAD_NUM];
#ifdef USE_SPINLOCK
pthread_spinlock_t g_spin;
#else
pthread_mutex_t g_mutex;
#endif

__uint64_t g_count;

pid_t gettid()
{
	return syscall(SYS_gettid);
}

void *run_amuck(void*arg)
{
	int i, j;

	printf("Thread %lu started.\n", (unsigned long)gettid());

	for (i = 0; i < 10000; i++) {
#ifdef USE_SPINLOCK
		pthread_spin_lock(&g_spin);
#else
		pthread_mutex_lock(&g_mutex);
#endif
		for (j = 0; j < 100000; j++) {
			if (g_count++ == 123456789)
				printf("Thread %lu wins!\n", (unsigned long)gettid());
		}

#ifdef USE_SPINLOCK
		pthread_spin_unlock(&g_spin);
#else
		pthread_mutex_unlock(&g_mutex);
#endif
	}

	printf("Thread %lu finished!\n", (unsigned long)gettid());

	return (NULL);
}

int main(int argc, char *argv[])
{
	int i, threads = THREAD_NUM;

	printf("Creating %d threads...\n", threads);
#ifdef USE_SPINLOCK
	pthread_spin_init(&g_spin, 0);
#else
	pthread_mutex_init(&g_mutex, NULL);
#endif
	for (i = 0; i < threads; i++)
		pthread_create(&g_thread[i], NULL, run_amuck, (void *) i);

	for (i = 0; i < threads; i++)
		pthread_join(g_thread[i], NULL);

	printf("Done.\n");

	return (0);
}
```

测试环境及结果：
```
Test Env:
OS: CentOS release 6.3 (Final)
Kernal: Linux 2.6.32_1-16-0-0
CpuInfo: Intel(R) Xeon(R) CPU  E5620  @ 2.40GHz (Processor Num=8)
MemInfo: 32G

Compile command:
g++ -o mutex_version  spin_vs_mutex.cpp -lpthread
g++ -o spin_version -DUSE_SPINLOCK spin_vs_mutex.cpp -lpthread

Output:
youwuwei@cq02-giano-dev02 ~/workspace/common/test$ time ./mutex_version
Creating 2 threads...
Thread 22975 started.
Thread 22976 started.
Thread 22976 wins!
Thread 22976 finished!
Thread 22975 finished!
Done.

real 0m5.241s
user 0m5.224s
sys  0m0.034s
youwuwei@cq02-giano-dev02 ~/workspace/common/test$ time ./spin_version
Creating 2 threads...
Thread 25726 started.
Thread 25727 started.
Thread 25726 wins!
Thread 25726 finished!
Thread 25727 finished!
Done.

real 0m13.923s
user 0m26.257s
sys  0m0.005s
```

### 结果分析

1. 在该测试中,临界区较大,导致锁竞争非常激烈, 此时使用mutex优于spinlock.
在临界区较小的时候, 采用spinlock的性能会优于mutex.

2. spinlock消耗更多user time, mutex消耗更多sys time.
因为两个线程分别运行在两个核上, 大部分时间只有一个线程能拿到锁,所以另一个线程就一直在它运行的core上进行忙等待, CPU占用率一直是100%; 而mutex则不同,当对锁的请求失败后上下文切换就会发生, 这样就能空出一个核来进行别的运算任务了.其实这种上下文切换对已经拿着锁的那个线程性能也是有影响的,因为当该线程释放该锁时它需要通知操作系统去唤醒那些被阻塞的线程，这也是额外的开销.

3. 思考为什么要存在spinlock?
spinlock比mutex在临界区比较小的时候优于mutex，我个人认为这个不是结果，而是SpinLock的设计目标。spinlock更像是一种特定优化后的mutex，假设临界区的操作耗时远小于锁冲突的开销,那么根本就没必要进入内核加锁，直接在那边自旋等待还好点。glibc的mutex实际上也是这么实现的,pthread_mutex_lock的源码中在没有contention的情况下，就是一条CAS指令，内核都没有陷入;在contention发生的情况下，(此时也可能会像spinlock一样try有限次数)，然后就选择进入内核睡觉，等待某个线程unlock后唤醒。


### 注意事项
1.**被自旋锁保护的临界区代码执行时不能睡眠**。单核处理器下，获取到锁的线程睡眠，若恰好此时CPU调度的另一个执行线程也需要获取这个锁，则会造成死锁；多核处理器下，若想获取锁的线程在同一个处理器下，同样会造成死锁，若位于另外的处理器，则会长时间占用CPU等待睡眠的线程释放锁，从而浪费CPU资源。
2.**被自旋锁保护的临界区代码执行时不能被其他中断打断。**原因同上类似。
3.被自旋锁保护的临界区代码在执行时，内核不能被抢占，亦同上类似。
