#Linux 中断底半部机制
[TOC]

##linux中断底半部

因为硬中断ISR是在关闭中断的情况下执行的，故而在硬中断ISR中停留时间过长会影响系统性能。为了能尽可能地缩短硬中断的时间，linux设置了中断底半部。也就是说，Linux的中断分为顶半部（硬中断）和底半部（软中断），软中断是在打开中断响应的情况下执行的。硬中断中处理那些即时的不可中断关键部分，而底半部处理那些可中断的非关键部分。

#### 过去的中断底半部实现机制 bh_base[]
bh_base[]为一个32个元素的函数指针数组，并且内核设置了两个32位无符号整数：bh_active
和bh_mask用于管理这个数组，这两个无符号整数的每一位对应bh_base中的每一个函数指针。bh_base[]相当于硬中断当中的irq_desc[]。不过irq_desc[]中的每个元素都代表着一个中断通道，也就是说<u>*每个元素都管理一个中断服务程序队列*</u>。<u>*而bh_base中的每个元素却最多代表一个bh函数*</u>。

#####关于bh_active和 bh_mask
+ bh_active相当于硬件中断的“中断请求寄存器”，而bh_mask相当于“中断屏蔽寄存器”
+ 需要执行一个bh函数时候就通过一个mark_b将bh_active中的对应位置位。如果bh_mask中的相应位也是1,那么就允许这个bh函数被执行。每次执行完do_IRQ或者系统调用之后，就会在一个do_bottom_half()函数中执行这个bh函数。

## 新机制的出现
因为bh_base[]机制实现的底半部具有与硬中断一样的"严格串行化"的特点，造成了系统效率低下。于是我们需要一个新的底半部实现机制，用于实现非严格串行化的底半部。并且为了兼容bh_base[]机制（软件设计开闭原则：对增加开放，对修改关闭），linux的实现策略是这样的：*<u>增加一种底半部框架，然后将bh_base和新的“非串行化机制”纳入该框架当中</u>*，这个框架就是linux_kernel_2.4当中的“软中断(softirq)”.可以这么描述软中断：

+ 软中断是一种框架，也是一种机制
+ 这个框架内，有众多实现底半部的机制，bh机制就是其中一种。
+ bh机制是一种特殊的软中断，也可以说是设计嘴保守的，但是最简单，最安全的软中断。
+ 软中断一共有四种，定义于include/linux/interrupt.c
```cpp
 enum
 {
     HI_SOFTIRQ=0,
     NET_TX_SOFTIRQ,
     NET_RX_SOFTIRQ,
     TASKLET_SOFTIRQ
 };
```
其中NET_TX_SOFTIRQ和NET_RX_SOFTIRQ专门为网络设计。所以主要的软中断机制只有HI_SOFTIRQ和TASKLET_SOFTIRQ两种。

###软中断的分析
系统在初始化的时候调用softirq_init()函数对软中断进行初始化。
```cpp
281 void __init softirq_init()
282 {
283     int i;
284 
285     for (i=0; i<32; i++)
286         tasklet_init(bh_task_vec+i, bh_action, i);
287 
288     open_softirq(TASKLET_SOFTIRQ, tasklet_action, NULL);
289     open_softirq(HI_SOFTIRQ, tasklet_hi_action, NULL);
290 }
```

#### 机制的初始化
内核为bh机制设置了一个bh_task_vec[32]数组，这是一个tasklet_struct结构数组。该结构定义如下：
```cpp
 struct tasklet_struct
 {
     struct tasklet_struct *next;
     unsigned long state;
     atomic_t count;
     void (*func)(unsigned long);
     unsigned long data;
 };
```
一个tasklet_struct就代表这一个tasklet，因为bh的实现本身是一个tasklet，只是在此基础上增加了更加严格的"串行化"限制就成了原来的bh机制。在softirq_init中（285行），对bh的32个tasklet_init调用了tasklet_init之后，它们的函数指针都指向了bh_action。其它软中断的初始化都是通过open_softirq()来完成的。

```cpp
105 void open_softirq(int nr, void (*action)(struct softirq_action*), void *data)
106 {
107     unsigned long flags;
108     int i;
109 
110     spin_lock_irqsave(&softirq_mask_lock, flags);
111     softirq_vec[nr].data = data;
112     softirq_vec[nr].action = action;
113 
114     for (i=0; i<NR_CPUS; i++)
115         softirq_mask(i) |= (1<<nr);
116     spin_unlock_irqrestore(&softirq_mask_lock, flags);
117 }
```
内核为软中断设置了一个以“软中断号”为下标的数组softirq_vec[ ],类似与中断机制中的irq_desc[ ]。这是一个softirq_action结构数组，定义如下：
```cpp
 struct softirq_action
 {
     void (*action)(struct softirq_action *);
     void *data;
 };
```
softirq_vec是一个全局变量，所以SMP中的每个CPU看到的都是同一个数组。但是每个CPU各有自己的“软中断控制状态结构”，这些结构形成一个以CPU编号为下标的irq_stat[]。这个数组也是全局变量，每个CPU通过自己的编号来访问。irq_stat定义如下
```cpp
 /* entry.S is sensitive to the offsets of these fields */
 typedef struct 
 {
     unsigned int __softirq_active;
     unsigned int __softirq_mask;
     unsigned int __local_irq_count;
     unsigned int __local_bh_count;
     unsigned int __syscall_count;
     unsigned int __nmi_count;
 /* arch dependent */
 } ____cacheline_aligned irq_cpustat_t;
```
其中softirq_active相当于“软中断请求寄存器”，而softirq_mask则相当于“软中断屏蔽寄存器”。
函数open_softirq把函数指针填入相应的softirq_vec之外，把每个CPU上的softirq_mask的相应位设置为1，即允许该中断在每个CPU上运行。
内核还有一个以CPU编号为下标的数组tasklet_hi_vec[]，这是一个tasklet_head结构数组，每个tasklet_head结构就是一个tasklet_head结构的队列头。
```cpp
struct tasklet_head tasklet_hi_vec[NR_CPUS] __cacheline_aligned;
```


#总结
softirq框架：
在每次do_IRQ结束之后，或者系统调用结束之后，会检测是否有软中断需要处理。如果有的话就调用do_softirq()，而do_softirq()会检测是哪个softirq(HI_SOFTIRQ?TASKLET_SOFTIRQ?)有中断请求,然后查询softirq_vec数组，如果该cpu上的softirq_active对应位置位，就调用softirq_vec中对应位的action。比如说：
cpu上有bh请求需要处理，也就是说softirq_active中的HI_SOFTIRQ位置位了，这时候就会调用softirq_vec[HI_SOFTIRQ].action，也就是softirq_init()函数中安装的tasklet_hi_action函数，tasklet_hi_action会将该处理器上的tasklet_hi_vec[cpu]队列中的每一个tasklet。而对于bh来说，每个tasklet都会调用bhaction函数,bhaction函数则根据传入的nr调用bh_base[]中的函数。

从上面的叙述中可以看到：
+ softirq框架只有irq_stat[]数组，和softirq_vec数组。irq_stat用于查询中断状态，而softirq_vec则作为软中断向量表，保存这每个软中断服务程序入口。
+ 在softirq框架下的tasklet机制。实际上是在softirq_vec这个向量表内安装了相关的tasklet处理函数（tasklet软中断ISR），对每个CPU上的tasklet_hi_vec[cpu]队列中的每一个tasklet_struct进行处理。也就是说，tasklet机制只有softirq_vec中的“tasklet软中断ISR”、tasklet_hi_vec[cpu]队列tasklet_struct.（每个tasklet_struct就是一个tasklet实例，即一个小任务。）
+ 在softirq框架下的bh机制。bh机制实际上是借助tasklet实现的。它只是向CPU的tasklet_hi_vec[cpu]提交tasklet_struct,而每个tasklet_struct都负责调用一个bh_base中的函数。
