[TOC]

### CPU性能指标

- 
  平均负载

- CPU使用率
  - 用户CPU
  - 系统CPU
  - IOWAIT
  - 软中断
  - 硬中断
  - 窃取CPU
  - 客户CPU
- 上下文切换
  - 自愿上下文切换
  - 非自愿上下文切换
- CPU缓存命中率





#### 平均负载

- 命令
  - top
  - uptime
- 定义
  - 平均负载是指单位时间内，系统处于**可运行状态和不可中断状态**的平均进程数，也就是平均活跃进程数
    - 和 CPU 使用率并没有直接关系 （RD）
- [定义补充]( http://www.brendangregg.com/blog/2017-08-08/linux-load-averages.html) （ps aux 查看）
  - R 状态 ，可运行状态的进程，是指正在使用 CPU 或者正在等待 CPU 的进程
  - D状态， 不可中断状态的进程则是正处于内核态关键流程中的进程，并且这些流程是不可打断的
    - 如等待硬件设备的 I/O 响应，或进程向磁盘读写数据
- 期望值
  - 平均负载数值和CPU相当，当比CPU高时一半进程**竞争不到**CPU



```bash
uptime # 或top 显示的都是趋势  
#02:34:03 up 2 days, 20:14,  1 user,  load average: 0.63, 0.83, 0.88

grep 'model name' /proc/cpuinfo | wc -l # 2
```

#### CPU使用率

- 定义

  - 单位时间内 CPU 繁忙情况的统计

- 场景

  - CPU 密集型进程，使用大量 CPU 会导致平均负载升高，此时这两者是一致的；
  - **I/O 密集型进程**，等待 I/O 也会导致平均负载升高，但 **CPU 使用率不一定很高**；
  - **大量等待** CPU 的**进程调度**也会导致平均负载升高，此时的 CPU 使用率也会比较高。

- 工具

  - **sysstat**   （注意内核低会缺少部分数据）

##### 验证还原

###### CPU 密集型

  ~~~bash
  sudo apt install stress sysstat
  
  stress --cpu 1 --timeout 600
  
  watch -d uptime  #...,  load average: 1.00, 0.75, 0.39
  
  pidstat -u 5 1 #查出高cpu应用，没有 top也行 
  
  # -P ALL 表示监控所有CPU，后面数字5表示间隔5秒后输出一组数据
  mpstat -P ALL 5
  #03:21:14 AM  CPU    %usr   %nice    %sys %iowait    %irq   %soft  %steal  %guest  %gnice   %idle
  #03:21:19 AM    0  100.00#    0.00    0.00    0.00#    0.00    0.00    0.00    0.00    0.00    0.00
  #03:21:19 AM    1    0.20    0.00    0.20    0.00    0.00    0.00    0.00    0.00    0.00   99.60
  ~~~

  0号使用率为 100%，但它的 iowait 只有 0



###### I/O 密集型

~~~bash
stress -i 1 --timeout 600

watch -d uptime			#...,  load average: 1.06, 0.58, 0.37

pidstat -u 5 1 # 抓到stress，还是cpu略高，写数据库，写硬盘，网络大流量等  # pidstat -d 未验证

mpstat -P ALL 5 1
#13:41:28     CPU    %usr   %nice    %sys %iowait    %irq   %soft  %steal  %guest  %gnice 
#13:41:33       0    0.43    0.00   23.87#   67.53#    0.00    0.43    0.00    0.00    0.00    7.74
~~~

 **%sys** 升高到了 23.87，而 **iowait** 高达 67.53%



###### 大量进程

```bash
stress -c 8 --timeout 600

uptime   # ...,  load average: 7.97#, 5.93, 3.02

pidstat -u 5 1 # 这次直接查进程了，特征明细，多，平均，等待
14:23:25      UID       PID    %usr %system  %guest   %wait    %CPU   CPU  Command
14:23:30        0      3190   25.00    0.00    0.00   74.80#   25.00#     0  stress
14:23:30        0      3191   25.00    0.00    0.00   75.20   25.00     0  stress
14:23:30        0      3192   25.00    0.00    0.00   74.80   25.00     1  stress
14:23:30        0      3193   25.00    0.00    0.00   75.00   25.00     1  stress
14:23:30        0      3194   24.80    0.00    0.00   74.60   24.80     0  stress
14:23:30        0      3195   24.80    0.00    0.00   75.00   24.80     0  stress
14:23:30        0      3196   24.80    0.00    0.00   74.60   24.80     1  stress
14:23:30        0      3197   24.80    0.00    0.00   74.80   24.80     1  stress
14:23:30        0      3200    0.00    0.20    0.00    0.20    0.20     0  pidstat
```

load average 就很明显，pidstat 再验证一下



用户CPU

系统CPU

IOWAIT

软中断

硬中断

窃取CPU

客户CPU

#### CPU上下文切换

- 起因
  - 进程在竞争 CPU 的时候并没有真正运行，为什么还会导致系统的负载升高？
    - 分时复用，CPU 寄存器 xx 和程序计数器 PC 等称作上下文
    - CPU 上下文切换正常不需要特别关注，量大要注意
- 场景
  - 进程上下文切换
    - 例如用户程序访问内存，需要陷入内核态，通过系统调用，进程不变，CPU切换2次
    - 进程切换，内核状态和 CPU 寄存器，该进程的虚拟内存、栈等（资源基本单位）
    - 性能问题考虑方向，时间片，资源不足，sleep主动，高优先进程，硬件中断
  - 线程上下文切换
    - 一个线程时等于进程，多线程共享相同的虚拟内存和全局变量，另自有栈和寄存器
    - 线程切换
      - 不同进程的线程，同进程切换
      - 同进程线程，只切换线程的私有数据、寄存器
  - 中断上下文切换
    - 原因，中断处理会打断进程的正常调度和执行，强制切换
    - **中断不涉及用户态**
      - 只包括内核态中断服务程序执行所必需的状态，包括 CPU 寄存器、内核堆栈、硬件中断参数等
  - 排查 vmstat
    - cs（context switch）是每秒上下文切换的次数
    - in（interrupt）则是每秒中断的次数
    - r（Running or Runnable）是就绪队列的长度，也就是正在运行和等待 CPU 的进程数，最前
    - b（Blocked）则是处于不可中断睡眠状态的进程数。



~~~bash
vmstat 5
# r  b   swpd   free   buff  cache   si   so    bi    bo   in   cs us sy id wa st
 1  0      0 264836   2108 579080    0    0     2     2   40  221  1  0 99  0  0

pidstat -w 5 
~~~







自愿上下文切换

非自愿上下文切换

CPU缓存命中率

