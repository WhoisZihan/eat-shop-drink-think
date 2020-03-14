---
title: "Paper Reading - User-Level Implementations of Read-Copy Update"
date: 2018-08-17T17:26:13+08:00
tags: [Multithread, Lock, RCU]
draft: false
---

## 1. Abstraction & Motivation

RCU在内核中使用非常广泛, 但在用户态很少有实现, 但用户态应用对于RCU的需求却是真实存在的。这篇论文的作者分析了QSBR-based RCU, general purpose RCU, 最后提出了他们自己的方案, signal-based RCU, 来将RCU实现在user space。

## 2. RCU overview

RCU最适合read-most操作, 而且能忍受stale/inconsistent数据。但是对于需要consistent数据的read-most操作也能工作的很好。[URCU的作者在lwn发了一篇文章][1], 还写了一个库叫[liburcu.org][2]

RCU的主要思想就是读操作完全不需要重量级的锁, 需要读的时候调用`rcu_read_lock`, 完成后调用`rcu_read_unlock`. 这里假定代码没有bug, 不会在critical section中挂掉。

而updater的操作就更复杂一些, 它不能影响读者, 但又不想像rwlock一样完全等所有读者完成, 所以它采取的是一种COW的方法, 首先复制对象, 修改操作在新对象上进行, 然后使用原子操作把旧指针改为指向新对象, 从这时候起, 所有后续的reader看到的都是新的object, 不会再引用旧的对象。

但是这里有个问题, 旧的对象并不能马上删除, 因为这时候可能还有reader在读它, 所以updater要回收这些对象会分为三个period, 分别为*removal period*, *grace period* and *reclamation period*, 通过调用`synchronize_rcu`进入grace period等待所有旧的reader都完成, 然后在最后一个period会做实际的删除操作。

注意进入到grace period的时候对象指针已经指向了新的对象, 所以在grace period开始后的reader都会读到新的数据, 不必担心。另外updater可能会需要spinlock等锁来控制和其他updater的并发访问。

grace period什么时候结束? 实际上就是所有旧的reader不再引用旧的对象的时候, 然后在reclamation阶段就回收这个旧的对象, 那么什么时候可以确定不会有reader再引用呢？这个和具体的回收策略有关, 经典的回收策略有QSBR和EBR等

<!-- more -->

### 2.1 Quiescent-State-Based Reclamation RCU

The QSBR RCU is an extremely reader-friendly RCU, but it could hurt performance of writer badly. The main idea is that the reader is responsible to tell others that it does not need any resources anymore. The writer can do nothing but wait until the readers have left the *Quiescent State*.

The *Quiescent State* actually ends when the reader does not need to access any data in critical section, but the waiter does not know until the reader tells it(usually through some kind of call, but there are also other methods). Jeff Preshing wrote a really [good article about the QSBR on his website][3].  A chinese version can be found [at the CSDN][4]

QSBR的总体思想就是, 写者要删除的时候, 向垃圾回收器注册一个回调函数, 然后等待所有的reader完成, 然后每个reader做完后就会调用一个quiescent callback(一般是会在结束critical section后调用, 但理论上可以在任何时候调用), 来表示我这个reader已经做完了, 进入了quiescent state, 没有任何引用到目标object的地方了。然后所有的reader都进入到了quiescent state之后, 就会触发writer注册的回调, 代表grace period已经结束, 现在可以安全删除这些object了。

可以看到, QSBR是一种对reader非常友好的回收策略, 因为reader进入critical section的时候不需要拿任何锁, 可以想做多久就做多久, 只需要做完后进入quiescent state, 然后需要再使用RCU的时候调用一下`rcu_thread_online`即可, 也不需要去主动的通知写者什么, 由内核/垃圾回收器来负责回收。

**Extra:**

对于对于有的应用, 可能只是偶尔用一下RCU, 但是一旦用起来会疯狂的用, 如果频繁的在quiescent state和其他状态之间进行切换, 会比较损耗性能(`rcu_quiescent_state()`开头和末尾各有一个`smp_mb()`, 而memory barrier是比较损耗性能的)。这些应用可以调用`rcu_thread_online`来表明我开始使用RCU了, 离开quiescent state(为什是离开? 因为quiescent state表明我没有任何对该对象的引用了, 但这里开始使用就产生了引用, 所以是从quiescent state离开), 然后调用`rcu_thread_offline`来更新状态(计数器), 但是不会离开quiescent state, 这样在`rcu_thread_offline`和下一个`rcu_thread_online`之间, reader不会进入quiescent state, 减少了额外切换操作的开销。

这两个方法的代码如下

```c
static inline void rcu_thread_offline(void)
{
    smp_mb();
    __get_thread_var(rcu_reader_qs_gp) =
    ACCESS_ONCE(rcu_gp_ctr);
}

static inline void rcu_thread_online(void)
{
    __get_thread_var(rcu_reader_qs_gp) =
    ACCESS_ONCE(rcu_gp_ctr) + 1;
    smp_mb();
}
```

不过值得注意的是, 这个只是作者的一种实现, 也可以有其他的实现, 只要功能一样就行, 虽然效率上可能会差别很大。

### 2.2 Epoch-based Reclamation

不过值得注意的是, 上面的CSDN的文章也说了, QSBR只是一种回收策略, 还有其他的回收策略, 比如 *Epoch Based Reclamation(EBR)*, 关于EBR的文章可以看[concurrentkit的这篇presentation][5]。如果想看非常具体的回收策略技术细节的话, 可以看[Tom Hart的2005年在toronto的thesis][6]

EBR的两外两篇文章可以看[剑客的这一篇][7], 也可以看[这篇讲无锁结构的][8](感觉中文版翻译的有点诡异, 建议直接看[英文版的][9])。个人感觉英文版的比较透彻, 但是剑客的那一篇最容易理解。大致思想如下, 假设有时间段$T=[t_1, t_2]$, 如果在t1之前进行逻辑删除的节点, 都可以在t2后进行实际删除(物理删除), 那么就称时间段T为一段grace period。

(插嘴: 关于QSBR和EBR的粗略对比可以看[Seiichi的stackoverflow的回答][11])

文中给出的实现代码如下

```c
#define N_THREADS 4 //一共4个线程
bool active[N_THREADS] = {false};
int epoches[N_THREADS] = {0};
int globla_epoch = 0;
vector<int> retire_lists[3];

void read(int thread_id)
{
    active[thread_id] = true;
    memory_barrier(); //防止写读乱序
    epoches[thread_id] = globla_epoch;
    acquire_memory_barrier(); //acquire语义
    //进入临界区了。可以安全的读取*p，比如cout << *p << endl;
    //......   //读取完毕，离开临界区
    active[thread_id] = false;
    release_memory_barrier();
    //release语义
}

void logical_deletion(int thread_id)
{
    active[thread_id] = true;
    memory_barrier(); //防止写读乱序
    epoches[thread_id] = globla_epoch;
    acquire_memory_barrier();//acquire语义
    //进入临界区了，这里，我们可以安全的读取*p，比如cout << *p << endl;
    //好了，假如说我们现在要删除它了。先逻辑删除。
    int *new_p = new int(*p + 1);
    int *tmp = p;
    p = new_p;
    //tmp对应的内存还不能马上被回收，加入到对应的retire list
    retire_lists[epoches[thread_id]].push_back(tmp);
    //离开临界区
    active[thread_id] = false;
    //看看能不能物理删除
    try_gc();
}

bool try_gc()
{
    e = global_epoch;
    for (int i = 0; i < N_THREADS; i++) {
        if (active[i] && epochs[i] != e) {
            //还有部分线程没有更新到最新的全局的epoch值
            //这时候可以回收(e + 1) % 3对应的retire list。
            free((e + 1) % 3); //不是free(e)，也不是free(e-1)。参看下面
            return false;
        }
    }
    //更新epoch
    globla_epoch = (e + 1) % 3;
    //更新之后，那些active线程中，部分线程的epoch值可能还是e - 1（模3）
    //那些inactive的线程，之后将读到最新的值，也就是e。
    //不管如何，(e + 1) % 3对应的retire list的那些内存，不会有人再访问到了，可以回收它们了
    //因此epoch的取值需要有三种，仅仅两种是不够的。
    free((globla_epoch + 1) % 3);
}

bool free(int epoch)
{
    for each pointer in retire_lists[epoch]
        if (pointer is not NULL)
            delete pointer;
} 
```

#### reader (with EBR)

对于EBR的读者来说, 首先需要在active数组中标识自己的部分为true, 代笔自己现在要试图去读数据, 然后会把自己的epoch更新为全局的epoch, 然后就进入了read critical section了。

然后reader执行完critical section后, 需要把active数组中自己的部分设置为false, 表示自己读完了。注意这里的两个memory barrier是不可少的, 它可以防止编译器的乱序优化。

#### writer (with EBR)

采用EBT的写者的开头和reader很类似, 都是需要首先标注active数组中自己的部分并且获取global_epoch。然后这时候writer并不是直接修改原始数据, 而是创建了一个数据的副本, 然后把旧值给放到对应的retire_list里面，索引是当前线程的epoch值。这里一共有3个epoch, 可以看到之前的大部分操作都是`(e+1) % 3`, 事实上至少需要3个epoch, 一般3个也就够了。

然后放完后重新设置了active数组, 然后就调用了`try_gc`。

这个try_gc内部实现和网上其他的一些实现提到的略有不同。

其他的实现大部分都是检查`if ( all threads are in the epoch which m_nGlobalEpoch )`, 也就是必须所有的thread的epoch值都等于global_epoch的时候, 然后才释放(e + 1) % 3的元素(注意这里`(e + 1) % 3 == (e - 2) % 3`, 所以实质上就是释放epoch是$e - 2$的元素, 后面遇到`(e + 1) % 3`可以直接脑海里替换成`(e - 2) % 3`)。

但是这篇文章给的实现为了尽可能减小内存开销, 即使遇到遇到一个active的线程或者有线程的epoch值还没有更新到全局的epoch, 仍然释放当前`(e + 1) % 3`的元素, **注意这里的e还没有加1**, 也就是可能的释放现有的元素。那么为什么(e - 2)的元素现在可以释放了呢, 真的不会再有线程引用它了吗? **是的**, 但是原因还需要继续向下看。

问题的关键在于, **`global_epoch`到底是在什么条件下进行的更新?** 根据上面的代码可以看到, 如果所有的线程都不是active, 且都更新到了全局的epoch, 那么`global_epoch`就会进行加1操作。这个条件很关键, 因为这说明, 当前所有的线程都读到了最新的变量的值(否则它的epoch不会等于当前最新的global_epoch), 所以这时候**不会再有任何线程引用epoch值小于global_epoch的对象**, 这些对象的确是可释放的。但是马上global_epoch进行了加1操作, 所以这里需要释放的是epoch等于`(e - 2) % 3 = (e + 1) % 3`的所有元素。

这就是EBR的整个思想, 可以看到它对于读写都是无锁的, 但是它有一个缺点, 就是如果有一个线程没有及时更新epoch的值的话, 那么剩下的线程就无法及时删除元素, 这样内存消耗会立即增大。

### 2.2 General-Purpose RCU 

general的意思就是为了让普通的应用也能用, 而不仅仅是almost reader的情况, 但是代价就是reader的overhead会比较大, 不过作者说这个overhead仍然比一个硬件的`compare-and-swap`操作的消耗要小。

其读者的操作比较简单, 具体而言就是对每个线程都分配一个变量rcu_reader_gp, 每次进行`rcu_read_lock`的时候就把对应`per_thread(rcu_reader_gp)`进行加1操作, 然后每次进行`rcu_read_unlock`的时候就把变量进行减1操作。它之所以比quiescent state的reader要慢是因为lock和unlock的时候会有额外操作, 而且还有memory barrier。

至于writer的操作, 它有个比较特殊的地方, 首先看下general writer的代码

```c
static inline int rcu_old_gp_ongoing(int t)
{
    int v = ACCESS_ONCE(per_thread(rcu_reader_gp, t));

    return (v & RCU_GP_CTR_NEST_MASK) &&
        ((v ˆ rcu_gp_ctr) &  ̃RCU_GP_CTR_NEST_MASK);
}

static void flip_counter_and_wait(void)
{
    int t;

    rcu_gp_ctr ˆ= RCU_GP_CTR_BOTTOM_BIT;
    for_each_thread(t) {
        while (rcu_old_gp_ongoing(t)) {
            poll(NULL, 0, 10);
            barrier();
        }
    }
}

void synchronize_rcu(void)
{
    smp_mb();
    mutex_lock(&rcu_gp_lock);
    flip_counter_and_wait();
    flip_counter_and_wait();
    mutex_unlock(&rcu_gp_lock);
    smp_mb();
}
```

`flip_counter_and_wait`将`rcu_gp_ctr`的bottom bit设置为1, 代表现在有writer进入到了grace period, 然后等待所有的reader线程做完。

如何判断呢, 就是rcu_old_gp_ongoing这个函数, 主要有两个判断, 第一个判断`(v & RCU_GP_CTR_NEST_MASK)`容易理解, 看是否有多个reader；那么第二个判断`((v ˆ rcu_gp_ctr) &  ̃RCU_GP_CTR_NEST_MASK)`又是用来判断什么呢?

这个是判断`RCU_GP_CTR_BOTTOM_BIT`的, 如果线程变量和全局变量的bottom bit不相同, 那么就需要等待。为什么？

* 如果`rcu_thread_gp`的bottom bit为1而全局`gp_rcu_ctr`的bottom bit为0, 那么说明这个reader是在两个`flip_counter_and_wait`之间进入的，需要等待；
* 如果`rcu_thread_gp`的bottom bit为0而全局`gp_rcu_ctr`的bottom bit为1, 那么说明writer正在进入grace period, 而reader是在grace period之前做的, 所以要等待这个reader完成。

这里还有一个问题需要解决, 就是为什么`flip_counter_and_wait`要做两次? 这是因为防止race condition中reader没做完writer就进入了critical section的问题, 因为rcu的语义不像rwlock那样强, writer在执行的时候reader(如果强行执行的话)也是可以执行的。为了说明这个问题, 先把general reader的代码也贴上来

```c
#define RCU_GP_CTR_BOTTOM_BIT 0x80000000
#define RCU_GP_CTR_NEST_MASK (RCU_GP_CTR_BOTTOM_BIT - 1)
long rcu_gp_ctr = 1;
DEFINE_PER_THREAD(long, rcu_reader_gp);

static inline void rcu_read_lock(void)
{
    long tmp;
    long *rrgp;

    rrgp = &__get_thread_var(rcu_reader_gp);
    tmp = *rrgp;
    if ((tmp & RCU_GP_CTR_NEST_MASK) == 0) {
        *rrgp = ACCESS_ONCE(rcu_gp_ctr);
        smp_mb();
    } else {
        *rrgp = tmp + 1;
    }
}

static inline void rcu_read_unlock(void)
{
    long tmp;

    smp_mb();
    __get_thread_var(rcu_reader_gp)--;
}
```

现在考虑如下场景,

(1) 首先线程A是一个reader, 执行reader中的11-13行, 然后发现13行的判断满足, 于是获取了`rcu_gp_ctr`的值, 但是还没有存到线程的`rcu_reader_gp`变量中去, 此时`rcu_reader_gp`的值还为0.

(2) 线程B是一个writer, 执行synchronize_rcu, 进入到`flip_counter_and_wait`中, 首先B更新了`rcu_gp_ctr`的bottom bit位, 然后14-19行判断每一个线程。判断到线程A的时候, 因为线程A的`rcu_reader_gp`值还没有更新仍然是0, 所以B判断的时候会跳过线程A;

(3) 这时候A又继续执行, 但是A的`rcu_gp_ctr`使用的是旧值(记得(1)中说的, ACCESS_ONCE只会获取一次, 而B是在A获取了之后才去修改`rcu_gp_ctr`的值的), 所以线程A的`rcu_reader_gp`会被设置成为**旧的**`rcu_gp_ctr`, 而旧的`rcu_gp_ctr`的bottom bit为0

(3) 假定这时候只做一次`flip_counter_and_wait`的话, B的grace period就结束了, 此时`rcu_gp_ctr`的bottom bit仍然为1, 然后B又做一次`synchronize_rcu`, 这次又调用`flip_counter_and_wait`使得`rcu_gp_ctr`的bottom bit又变为了0, 然后发现线程A的`rcu_thread_gp`的bottom bit和`rcu_gp_ctr`的bottom bit是相同的, 所以又会忽略线程A, 就导致B返回后进入对象的删除阶段, 但这个时候A还在reader critical section里! 这就违反了RCU的语义。(**// TODO: **这里我不是很明白论文里面的那个分析, 为什么B要再做了一次`synchronize_rcu`才会产生违反语义的情况? 我个人觉得第一次`synchronize_rcu`调用返回后就会违反语义的情况啊!)

所以需要做两次`flip_counter_and_wait`, 这样第二次调用`synchronize_rcu`的时候就可以检查到上次仍然有reader在critical secion里, 就会等待. **// TODO.** 但在第一次调用`synchronize_rcu`的时候不需要等待这个reader完成吗? 存疑

**~~// TODO~~ 不过这里我有个问题不明白,如果说writer在做完两次`flip_counter_and_wait`后, 又来了一个reader怎么办, 这样岂不是reader和writer都进入了critical section?** ~~~这应该就是RCU的语义特性, 就是如果writer还在执行的时候不应该有reader执行(注意这里说的是不应该而不是不可能, 代码中只保证writer之前的reader已经做完了, 但并没有禁止writer做的时候不能有新的reader, 如果reader强行要进是会进的, 只是可能读到stale/inconsistent data, 而不像rwlock那样有很强的语义顺序性保证)~~~

RCU的语义确实比较弱化, 但是这里和语义没有太大关系, 之前说过了, writer采取的是COW的机制, 只是会延迟回收旧的对象, 其实在grace period开始之前, updater已经把指针更新成为了新的对象(通过原子操作就能实现), 所以在grace period开始后再开始的reader, 读到的是writer修改过的数据, 而在grace period之前开始的reader, 读到的则是旧的数据。具体的可以参考腾讯云社区的这篇[深入理解Linux的RCU机制][10]

### 2.3 Signal-based RCU

作者分析了现有的一些RCU的实现方案后发现, general RCU的overhead之所以太高, 是因为read-side有像memory barrier这样的primitive操作, 这些操作很expensive。作者于是就提出, 其实不是每次调用`rcu_read_lock`的时候, 都必须要memory barrier这类操作的。

那什么时候做呢? 论文给出的方法是在writer进入到`synchronize_rcu`的时候再做, 可是reader没法主动知道updater什么时候进入到grace period呀, 那么就需要updater去通知reader, 采用的通知的方法就是使用signal (个人觉得signal其实不会很快啊... 因为signal会下陷经过内核, 而且只有目标thread被调度到的时候才会进行)

这里因为大致原理都是一样, 所以就不展开描述, 要更多细节可以直接看他们的论文。

最后它还讲了一个wait-free update, 但是它的描述非常奇怪, 其实最后还是会阻塞的, 但是它说

> Of course, the use of synchronize rcu() causes call rcucleanup() to be blocking. However, as long as the callback function func that was passed to call rcu() does nothing other than free memory, as long as the synchronization mechanism used to coordinate RCU updates is wait-free, and as long as there is sufficient memory for allocations to succeed without blocking, RCU-based algorithms that use call rcu() will themselves be wait-free.

可是我如果有wait-free的coordination算法, 即使不用这一套也可以wait-free啊...不是很理解

## 3. Conclusion

读这篇paper花了我好长时间, 因为之前对于RCU不是很了解, 所以耗费在背景知识上的时间很多, 加上一些锁算法的设计都比较精巧, 所以有时候完全理解算法细节也要花一番功夫, 加上有些地方和自己的理解有出入又要重新看。

不过总体的收获还是很大的, 了解了RCU, 以及QSBR和EBR两种回收策略。感觉应该找时间revisit一下RCU, 顺便学习一些lock-free的知识。然后就可以开始了解C++中对应的库了, 为以后做准备。



[1]: https://lwn.net/Articles/573424/
[2]: http://liburcu.org/

[3]: http://preshing.com/20160726/using-quiescent-states-to-reclaim-memory/
[4]: https://blog.csdn.net/zhangyifei216/article/details/52767236
[5]: http://concurrencykit.org/presentations/ebr.pdf
[6]: http://www.cs.toronto.edu/~tomhart/papers/tomhart_thesis.pdf
[7]: http://www.jkeabc.com/182201.html
[8]: http://blog.jobbole.com/107955/
[9]: http://kukuruku.co/hub/cpp/lock-free-data-structures-the-inside-memory-management-schemes

[10]: https://cloud.tencent.com/developer/article/1006204
[11]: https://stackoverflow.com/a/42224909



