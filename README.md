# 日志救不了程序员

服务端程序上线后，总是会出现一些奇奇怪怪的问题，查到最后，要么是程序bug，要么是环境问题，要么是错误操作问题。那么问题来了，问题出现的时候，一般是用户或者运维先发现，再层层转达，最后可能交到程序员来排查。

作为一名被线上问题困扰已久的程序员，我知道线上问题是永远规避不完的，明确的需求和稳定/良好的代码风格/全面覆盖的测试用例只能减少线上问题，但总是会漏掉那么一些。bug一词的来源也是服务器上飞了一只虫子导致的机器问题。当然线上问题不止功能问题，也包含性能问题。

当线上问题传达到我的耳朵里时，很可能带有一定的迷惑性，因为我听到的只是现象。而最终现象和问题根源可能距离较远，直接联想可能很难想到，可能真是因为很难联想到它们之间的相关关系，才会让它在前面的阶段漏掉，再在线上跳出来。

一般排查线上问题，我会从两个层面去排查：1.程序层面，2.系统层面.

程序层面一般看程序日志，但有时日志没有覆盖到问题。系统层面就看系统日志，但是系统日志就更少了。所以我在查问题时，单看日志是很难看出问题的，如果问题到了程序员这一层，一般都是靠日志是看不出来的问题。特别是之前在做网络块设备服务时，一般遇到的都是性能问题，例如要求64KB大小的块的读延迟在1ms以内，这种时候，就连linux内核里面一个不起眼的策略配置，或者是网络稍微抖一下，都会导致最终看到的结果是大于1ms。这些时候，更需要合理使用性能排查工具观察问题出在哪里。

幸亏，linux提供了像/proc虚拟文件系统，perf采样，动态追踪等技术，并且市面上有完备的便捷工具，覆盖了几乎每一个点，能够让我们从各个角度去观察进程和系统内核，去观察每一个细节，把问题根源定位出来。(什么？你的程序没跑在linux上？)

以下是我学习倪鹏飞的《linux性能优化》作的笔记，记录了计算机的一些重要资源的观察方法。其中有一些还未完善的点，日后有时间一一补充。另外可以参考书籍《性能之巅》，里面也提供了大量工具和方法。

当然，我不是运维工程师，通过观察线上系统的运行状态，不仅仅能收获一种排查定位问题的方法，观察自己写的程序的运行期行为，更能理解linux的运作原理。

以下是线上服务器的软件堆栈图

![img](./920601da775da08844d231bc2b4c301d.png)

以下是观察各个层面/角度提供的工具

![img](./9ee6c1c5d88b0468af1a3280865a6b7a.png)

# CPU

###### 系统平均负载高

```shell
uptime
02:34:03 up 2 days, 20:14, 1 user, load average: 0.63, 0.83, 0.88
# 当前时间 系统运行时间       正在登录的用户数    1/5/15分钟平均负载
# 平均负载：系统处于可运行状态[running and runnable]和不可中断状态[uninterruptible sleep]的平均进程数[及平均活跃进程数]
# [uninterruptible sleep]:ps时，stat显示D，如等待硬件设备I/O响应的进程，系统对进程和硬件的一种保护机制

grep 'model name' /proc/cpuinfo | wc -l # 得出CPU核心数量
2
# CPU 密集型进程，使用大量 CPU 会导致平均负载升高，此时这两者是一致的；
# I/O 密集型进程，等待 I/O 也会导致平均负载升高，但 CPU 使用率不一定很高；
# 大量等待 CPU 的进程调度也会导致平均负载升高，此时的 CPU 使用率也会比较高；

# sysstat：性能工具包。包含mpstat和pidstat。
# mpstat：多核CPU性能分析工具，实时查看每个CPU的性能指标，以及所有CPU平均指标。
# pidstat：进程性能分析工具，实时查看进程的CPU/内存/IO/上下文切换等性能指标。
apt install stress sysstat

# stress:系统压力测试工具
stress --cpu 1 --timeout 600 #模拟持续600秒一个CPU使用率100%
stress -i 1 --timeout 600 #模拟持续600秒一个CPU不停地执行sync 使用率100%，及模拟IO压力
stress -c 8 --timeout 600 #模拟8个进程

watch -d uptime # watch观察命令输出, -d 参数表示高亮显示变化的区域
mpstat -P ALL 5 # 监控所有CPU的使用率，每5秒输出一组数据
pidstat -u 5 1 # 输出各个进程的CPU使用情况；观察5秒后输出一组数据
```

###### 上下文切换严重[硬件中断/软件中断 [上下文切换到中断处理程序]；同进程的线程间上下文切换/进程间上下文切换 的工作量和开销不同]

```shell
apt install sysbench sysstat
# 以10个线程运行5分钟的基准测试，模拟多线程切换的问题
sysbench --threads=10 --max-time=300 threads run

vmstat 5 # 每间隔5秒输出一组数据；系统的一些信息
procs -----------memory---------- ---swap-- -----io---- -system-- ------cpu-----
 r  b   swpd   free   buff  cache   si   so    bi    bo   in   cs us sy id wa st
 0  0  96256 4265788 2308036 58498780    0    0     0     4    0    0  0  0 100  0  0
 0  0  96256 4265640 2308036 58498812    0    0     0     0  164  138  0  0 100  0  0
r[running or runnable]:可运行队列和运行中的进程数量
b[blocked]:处于不可中断睡眠状态的进程数
in[interrupt]:每秒中断的次数
cs[context switch]:每秒上下文切换的次数
us[user]:用户态使用CPU时间
sy[system]:内核态使用CPU时间

pidstat -w 5 #每隔5秒输出1组数据
Average:      UID       PID   cswch/s nvcswch/s  Command
Average:        0         1      0.80      0.00  systemd
Average:        0         9     10.32      0.00  rcu_sched
Average:        0        11      0.27      0.00  watchdog/0
  cswch:进程每秒自愿上下文切换次数，高的话表示系统很少强占
nvcswch:进程每秒非自愿上下文切换次数，高的话表示系统较忙

pidstat -w -u 1# -w参数表示输出进程切换指标，而-u参数则表示输出CPU使用指标
Average:      UID       PID    %usr %system  %guest    %CPU   CPU  Command
Average:     1009      4398    0.66    1.66    0.00    2.32     -  pidstat
Average:        0     28571    0.00    0.33    0.00    0.33     -  kworker/8:1

Average:      UID       PID   cswch/s nvcswch/s  Command
Average:        0         1      0.66      0.00  systemd
Average:        0         9     14.24      0.00  rcu_sched

pidstat -wt 1 # -t 参数表示输出线程的指标

watch -d cat /proc/interrupts #观察各个CPU的中断情况
```

###### CPU利用率

```shell
#时间中断：每一个节拍触发一次。[内核中使用全局变量Jiffies记录开机以来的节拍数]。查看当前系统的节拍率(HZ)
$ grep 'CONFIG_HZ=' /boot/config-$(uname -r)
CONFIG_HZ=1000
# 内核还提供了一个用户空间节拍率 USER_HZ，它总是固定为 100

#/proc/stat 提供系统的 CPU 和任务统计信息
cat /proc/stat | grep ^cpu # 查看cpu开头的行：不同场景下CPU的累加节拍数，单位是USER_HZ
cpu  16604077 4158 3919911 15868803405 955663 0 285262 0 0 0
cpu0 1061392 139 316848 1322763764 54651 0 8857 0 0 0
cpu1 926280 104 264630 1323075364 23700 0 111 0 0 0
#每一列表示一种场景[man proc; 查找/proc/stat可以找到说明]，依次是:
#user   (1) Time spent in user mode.
#nice   (2) Time spent in user mode with low[1-19] priority (nice).
#system (3) Time spent in system mode.
#idle   (4) Time spent in the idle task.  This value should be USER_HZ times the second entry in the /proc/uptime pseudo-file. no include iowait.
#iowait (since Linux 2.5.41) (5) Time waiting for I/O to complete.
#irq (since Linux 2.6.0-test4) (6) Time servicing interrupts.
#softirq (since Linux 2.6.0-test4) (7) Time servicing softirqs.
#steal (since Linux 2.6.11) (8) Stolen time, which is the time spent in other operating systems when running in a virtualized environment
#guest (since Linux 2.6.24) (9) Time spent running a virtual CPU for guest operating systems under the control of the Linux kernel.
#guest_nice (since Linux 2.6.33) (10) Time spent running a niced guest (virtual CPU for guest operating systems under the control of the Linux kernel).
```

![img](./image-20200605153013090.png)

各种性能工具:取开始和结束的值，计算一小段时间内的使用率，然后显示:

![img](./8408bb45922afb2db09629a9a7eb1d5a.png)

top:默认起始间隔3秒，3秒刷新一次；

ps:默认进程整个生命周期为间隔时间；

```shell
#进程的CPU使用情况,值域通过man手册了解
/proc/[pid]/stat

#现成的方便性能工具:top/ps/pidstat

#程序哪行代码费CPU：
gdb:attach调试进程，但是会中断进程正常执行
perf:以性能事件采样为基础，分析系统和应用程序的性能问题
perf list # 列出所有能够触发perf采样点的事件
perf top #类似于top命令的交互界面,显示CPU始终占用的进程函数或指令排行
perf top -g -p 21515# -g 开启调用关系采样;-p指定进程id;
```

![img](./image-20200605154829171.png)

Samples:perf采样事件数量，过少的话可能不准

event:事件类型

Event count:事件总数量

Overhead:该符号的性能事件在采样中的比例

Shared:该函数或指令所在的动态共享对象[内核/进程名/动态链接库名/内核模块名]

Object:动态共享对象的类型 [.]:用户空间的可执行程序/动态库；[k]:内核空间

Syabol:符号名，及函数名，函数名未知时用16进制的地址表示

```shell
perf record #采样保存到当前目录perf.data，ctrl-C停止
perf record -g # -g 开启调用关系采样
perf report #展示类似于perf top的报告
```

###### 案例中用到的工具

```shell
# -a 表示输出命令行选项； p表PID； s表示指定进程的父进程
pstree -aps 3084
pstree | grep stress #查看stress进程的父进程

execsnoop # 专为短时进程设计的工具。通过ftrace[动态追踪技术]实时监控进程的exec()行为，输出短时进程的基本信息
```

top命令中，进程的状态:

R 是 Running 或 Runnable 的缩写，表示进程在 CPU 的就绪队列中，正在运行或者正在等待运行。

D 是 Disk Sleep 的缩写，也就是不可中断状态睡眠（Uninterruptible Sleep），一般表示进程正在跟硬件交互，并且交互过程不允许被其他进程或中断打断。

Z 是 Zombie 的缩写，如果你玩过“植物大战僵尸”这款游戏，应该知道它的意思。它表示僵尸进程，也就是进程实际上已经结束了，但是父进程还没有回收它的资源（比如进程的描述符、PID 等）。

S 是 Interruptible Sleep 的缩写，也就是可中断状态睡眠，表示进程因为等待某个事件而被系统挂起。当进程等待的事件发生时，它会被唤醒并进入 R 状态。

I 是 Idle 的缩写，也就是空闲状态，用在不可中断睡眠的内核线程上。前面说了，硬件交互导致的不可中断进程用 D 表示，但对某些内核线程来说，它们有可能实际上并没有任何负载，用 Idle 正是为了区分这种情况。要注意，D 状态的进程会导致平均负载升高， I 状态的进程却不会。

T 或者 t，也就是 Stopped 或 Traced 的缩写，表示进程处于暂停或者被跟踪状态。

​	SIGSTOP信号:使进程进入暂停状态

​	SIGCONT信号:使进程从T状态恢复运行

​	gdb断点：使进程进入被跟踪状态

X 是Dead缩写，表示进程以及消亡，在top和ps中看不到，僵尸进程被回收后用此状态表示。

s 表示该进程是一个会话的领导进程，+表示前台进程组

```shell
#dstat 同时观察CPU/磁盘IO/网络和内存的使用情况，需要安装
dstat 1 10 #每1秒输出10组数据

#-d 展示 I/O 统计数据，-p 指定进程号，间隔 1 秒输出 3 组数据
pidstat -d 1 3

strace -p 6082 #跟踪进程系统调用
```

###### 中断

上半部：硬件请求，直接打断CPU正在执行的任务，在中断禁止模式[不响应任何中断]下运行。

下半部：内核触发，以内核线程的方式运行，每个CPU都对应一个软中断内核线程[ksoftirqd/CPU 编号]，延迟处理在上半部中未完成的工作。下半部包含于软中断

```shell
ps aux | grep softirq
#/proc/softirqs 提供了软中断的运行情况；会显示软中断类型和在CPU上的分布
#/proc/interrupts 提供了硬中断的运行情况；
watch -d cat /proc/softirqs
                     CPU0           CPU1
HI[]:                  0              0
TIMER[定时中断]:       1083906        2368646
NET_TX[网络发送]:      53             9
NET_RX[网络接收]:      1550643        1916776
BLOCK[]:              0              0
IRQ_POLL[]:          0              0
TASKLET[]:          333637         3930
SCHED[内核调度]:       963675         2293171
HRTIMER[]:             0              0
RCU[RCU锁]:         1542111        1590625
```

```shell
# sar:系统活动报告工具。
# hping3:可构造TCP/IP协议数据包的工具。
# tcpdump:网络抓包工具。

# -S参数表示设置TCP协议的SYN（同步序列号）[SYN FLOOD攻击]，-p表示目的端口为80
# -i u100表示每隔100微秒发送一个网络帧
# 注：如果你在实践过程中现象不明显，可以尝试把100调小，比如调成10甚至1
hping3 -S -p 80 -i u100 192.168.0.30 #Ctrl+C 停止

# -n DEV 表示显示网络收发的报告，间隔1秒输出一组数据
sar -n DEV 1

# -i eth0 只抓取eth0网卡，-n不解析协议名和主机名
# tcp port 80表示只抓取tcp协议并且端口号为80的网络帧
tcpdump -i eth0 -n tcp port 80
15:11:32.678966 IP 192.168.0.2.18238 > 192.168.0.30.80: Flags [S], seq 458303614, win 512, length 0...
```



```shell
# 工具总结
uptime
top
ps
mpstat
vmstat
pidstat
perf
pstree
execsnoop
dstat
strace
/proc/cpuinfo
/proc/interrupts
/proc/stat
/proc/[pid]/stat
/boot/config-$(uname -r)
```



# 内存

###### buffer & cache

```shell
free -h
# buffer:/proc/meminfo中的Buffers值。内核缓冲区用到的内存。读写磁盘的缓存
# cache:/proc/meminfo中的Cached与SReclaimable 之和。内核页缓存和slab用到的内存。读写文件的缓存

# 清理文件页、目录项、Inodes等各种缓存
echo 3 > /proc/sys/vm/drop_caches

vmstat 1 #每秒输出1组，观察变化
dd if=/dev/urandom of=/tmp/file bs=1M count=500 #写文件时，cache值增大
dd if=/dev/urandom of=/dev/sdb1 bs=1M count=2048 #写磁盘时，buffer值增大
dd if=/tmp/file of=/dev/null #读文件时，cache值增大
dd if=/dev/sda1 of=/dev/null bs=1M count=1024 #读磁盘时，buffer值增大

# cachestat 提供了整个操作系统缓存的读写命中情况，bcc 软件包的一部分
# cachetop 提供了每个进程的缓存命中情况，bcc 软件包的一部分
# ubuntu安装bcc-tools,需要内核4.1以上
sudo apt-key adv --keyserver keyserver.ubuntu.com --recv-keys 4052245BD4284CDD
echo "deb https://repo.iovisor.org/apt/xenial xenial main" | sudo tee /etc/apt/sources.list.d/iovisor.list
sudo apt-get update
sudo apt-get install -y bcc-tools libbcc-examples linux-headers-$(uname -r)
export PATH=$PATH:/usr/share/bcc/tools

cachestat 1 3
   TOTAL   MISSES     HITS  DIRTIES   BUFFERS_MB  CACHED_MB
       2        0        2        1           17        279
       2        0        2        1           17        279
       2        0        2        1           17        279 
  总IO次数  缓存未命中次 缓存命中 新增到缓存的脏页 
cachetop # 类top命令

# pcstat:指定文件的缓存大小。先安装go语言，再安装pcstat
export GOPATH=~/go
export PATH=~/go/bin:$PATH
go get golang.org/x/sys/unix
go get github.com/tobert/pcstat/pcstat

pcstat /bin/ls #查看ls文件的缓存情况
strace -p $(pgrep app) # 利用shell语法生成app进程的pid，直接strace
```

centos7升级内核

```shell
# 升级系统
yum update -y

# 安装ELRepo
rpm --import https://www.elrepo.org/RPM-GPG-KEY-elrepo.org
rpm -Uvh https://www.elrepo.org/elrepo-release-7.0-3.el7.elrepo.noarch.rpm

# 安装新内核
yum remove -y kernel-headers kernel-tools kernel-tools-libs
yum --enablerepo="elrepo-kernel" install -y kernel-ml kernel-ml-devel kernel-ml-headers kernel-ml-tools kernel-ml-tools-libs kernel-ml-tools-libs-devel

# 更新Grub后重启
grub2-mkconfig -o /boot/grub2/grub.cfg
grub2-set-default 0
reboot

# 重启后确认内核版本已升级为4.20.0-1.el7.elrepo.x86_64
uname -r
```

安装bcc-tools

```shell

# 安装bcc-tools
yum install -y bcc-tools

# 配置PATH路径
export PATH=$PATH:/usr/share/bcc/tools

# 验证安装成功
cachestat 
```

###### 内存泄漏

```shell
#内存紧张时，系统通过回写脏页回收缓存页、利用swap，最终OOM （Out of Memory）机制杀死占用内存最多的进程
vmstat 1 #每秒1次打印内存信息
# pmap -x 查看进程内存分布

# cgroups:限制进程资源，保证系统内存不会被异常进程耗尽
# proc/pid/oom_adj 调整核心应用的oom_score，保证内存紧张进程也不被OOM杀死，值越大越容易被OOM杀死
dmesg | grep -i "Out of memory" # OOM杀死进程后查看OOM日志

# memleak:bcc软件包中的一个工具,用于检测内存泄漏[valgrind也可以检测内存泄漏]
# -a 表示显示每个内存分配请求的大小以及地址
# -p 指定案例应用的PID号
/usr/share/bcc/tools/memleak -a -p $(pidof app) #pidof如果有多个结果不换行，pgrep有多个结果时换行
```

###### swap

回写脏页的两种方式：

1：系统调用fsync，将脏页同步写到磁盘中

2：内核线程pdflush负责刷新脏页数据到磁盘中

swap机制，不能回收的不活跃页，就使用swap机制换到磁盘分区，用到时再换回内存。也可以用来实现开机恢复上次运行状态

使用swap分区的情况：

1：直接内存回收：有内存分配请求但剩余内存不够时，会触发系统通过swap回收内存用来响应内存分配请求

2：kswapd0：专门的内核线程定期swap回收内存。一旦剩余内存小于pages_low，就会触发内存的swap回收

![img](./c1054f1e71037795c6f290e670b29120.png)

可以通过内核选项/proc/sys/vm/min_free_kbytes设置pages_min，然后内核自动得出其他值：

pages_low = pages_min * 5/4
pages_high = pages_min * 3/2

**numa 与 swap**

Non-Uniform Memory Access 处理器架构:多个处理器被划分到不同的node上，每个node拥有自己的本地内存空间。同一个node内部，进一步分为不同的内存域(zone)，直接内存访问区(DMA)/普通内存区(NORMAL)/伪内存区(MOVABLE) 等

![img](./be6cabdecc2ec98893f67ebd5b9aead9.png)

```shell
# 查看处理器再node的分布情况，以及每个node的内存使用情况
numactl --hardware
available: 1 nodes (0)
node 0 cpus: 0 1
node 0 size: 7977 MB
node 0 free: 4416 MB
...
#示例输出为1核心2线程

#通过/proc/zoneinfo [内存域在proc中的接口]观察 页最小阈值[min]、页低阈值[low]和页高阈值[high]
$ cat /proc/zoneinfo
...
Node 0, zone   Normal
 pages free     227894 #剩余内存页数
       min      14896 #pages_min
       low      18620 #pages_low
       high     22344 #pages_high
...
     nr_free_pages 227894 #剩余内存页数
     nr_zone_inactive_anon 11082 #非活跃匿名页数
     nr_zone_active_anon 14024 #活跃匿名页数
     nr_zone_inactive_file 539024 #非活跃文件页数
     nr_zone_active_file 923986 #活跃文件页数
...

# 当一个node内存不足时：cat /proc/sys/vm/zone_reclaim_mode
# 0：可以去其他node寻找空闲内存,也可以从本地回收内存
# 1：从本地回收内存。
# 2：从本地回收内存。可以回写脏数据回收内存
# 4：从本地回收内存。可以用swap回收内存
```

**swappiness**

对文件页的回收，当然就是直接回收缓存，或者把脏页写回磁盘后再回收。

而对匿名页的回收，其实就是通过 Swap 机制，把它们写入磁盘后再释放内存。

/proc/sys/vm/swappiness  值域为[0,100]，越大系统越倾向于回收匿名页。

```shell
# 开启swap
# 创建Swap文件
$ fallocate -l 8G /mnt/swapfile
# 修改权限只有根用户可以访问
$ chmod 600 /mnt/swapfile
# 配置Swap文件
$ mkswap /mnt/swapfile
# 开启Swap
$ swapon /mnt/swapfile

$ free # 查看

$ swapoff -a # 关闭swap
$ swapoff -a && swapon -a # 关闭再打开，达到清空swap的效果

# -r表示显示内存使用情况，-S表示显示Swap使用情况
$ sar -r -S 1 # 间隔1秒输出一组数据
04:39:56    kbmemfree   kbavail kbmemused  %memused kbbuffers  kbcached  kbcommit   %commit  kbactive   kbinact   kbdirty
04:39:57      6249676   6839824   1919632     23.50    740512     67316   1691736     10.22    815156    841868         4

04:39:56    kbswpfree kbswpused  %swpused  kbswpcad   %swpcad
04:39:57      8388604         0      0.00         0      0.00
# kbcommit 当前系统负载需要的内存[估值]
# kbactive 活跃内存
# kbinact 非活跃，可能被回收

# cachetop观察缓存使用情况
cachetop 5

# 查看进程swap换出到swap的虚拟内存大小
/proc/pid/status 中的 VmSwap
# 按VmSwap使用量对进程排序，输出进程名称、进程ID以及SWAP用量
$ for file in /proc/*/status ; do awk '/VmSwap|Name|^Pid/{printf $2 " " $3}END{ print ""}' $file; done | sort -k 3 -n -r | head
dockerd 2226 10728 kB
docker-containe 2251 8516 kB
snapd 936 4020 kB
networkd-dispat 911 836 kB
polkitd 1004 44 kB

smem --sort swap # 将进程按照swap使用量排序显示

#mlock()/mlockall()库函数可以锁定内存，阻止它们被换出到swap分区
```

![img](./e28cf90f0b137574bca170984d1e6736.png)

```shell
# 工具总结
free
vmstat
sar
ps
top
pmap
cachestat
cachetop
memleak # 内存泄漏检测
valgrind # 内存泄漏检测
systemtap # 动态追踪
pcstat
/proc/meminfo
/proc/pid/status
/proc/pid/smaps
```

# I/O

###### 原理

一切皆文件:所有对象都通过统一的文件系统管理

inode:记录文件元数据

dentry:记录文件名/inode指针/dentry关系(用于构成目录结构)，从磁盘加载后作为特殊文件放到内存

![img](./328d942a38230a973f11bae67307be47.png)

抽象一层VFS用于统一接口，支持多种文件系统并存:

基于磁盘的文件系统:ext4,xfs

基于内存的文件系统:/proc, /sys

基于网络的文件系统:NFS，iSCSI

![img](./728b7b39252a1e23a7a223cdf4aa1612.png)

将文件系统挂载到VFS目录树的某个挂载点，才能访问。

缓冲IO 与 非缓冲IO:是否利用标准库访问

直接IO 与 非直接IO:是否跳过系统页缓存（裸IO：跳过文件系统直接读写磁盘设备）

堵塞IO 与 非堵塞IO:对于I/O 调用者来说，IO函数是否会卡住一段时间

同步IO 与 异步IO:对于I/O执行者来说，调用IO函数是立即得到结果还是通过事件通知

disk file:

O_SYNC：数据和元数据写入磁盘才返回

O_DSYNC：数据写入磁盘才返回

net socket:

O_ASYNC：异步IO

###### 工具

```shell
df -h /dev/sda1 # 查看文件系统的磁盘空间使用情况
df -i /dev/sda1 #存储inode的部分使用情况
```

```shell
# 与free命令输出的cache是 页缓存和可回收slab缓存的和
$ cat /proc/meminfo | grep -E "SReclaimable|Cached" 
Cached:           748316 kB 
SwapCached:            0 kB 
SReclaimable:     179508 kB 
```

```shell
# 查看slab缓存中，细分到各种文件的缓存
$ cat /proc/slabinfo | grep -E '^#|dentry|inode' 
# name            <active_objs> <num_objs> <objsize> <objperslab> <pagesperslab> : tunables <limit> <batchcount> <sharedfactor> : slabdata <active_slabs> <num_slabs> <sharedavail> 
xfs_inode              0      0    960   17    4 : tunables    0    0    0 : slabdata      0      0      0 
... 
ext4_inode_cache   32104  34590   1088   15    4 : tunables    0    0    0 : slabdata   2306   2306      0hugetlbfs_inode_cache     13     13    624   13    2 : tunables    0    0    0 : slabdata      1      1      0 
sock_inode_cache    1190   1242    704   23    4 : tunables    0    0    0 : slabdata     54     54      0 
shmem_inode_cache   1622   2139    712   23    4 : tunables    0    0    0 : slabdata     93     93      0 
proc_inode_cache    3560   4080    680   12    2 : tunables    0    0    0 : slabdata    340    340      0 
inode_cache        25172  25818    608   13    2 : tunables    0    0    0 : slabdata   1986   1986      0 
dentry             76050 121296    192   21    1 : tunables    0    0    0 : slabdata   5776   5776      0 
# dentry 目录项缓存
# inode_cache VFS索引节点缓存
# 其他行 各具体文件系统的inode缓存

#实用工具 slabtop
slabtop # 按下c按照缓存大小排序，按下a按照活跃对象数排序
```

SATA & SSD : 接口/传输通道/传输协议 : RAID/分区

**通用块层**

在文件系统和磁盘驱动中间的一个抽象层，用于上下层接口封装抽象，还会给文件系统发来的IO请求排队并优化(重排序/请求合并等)。

排序：4种可选的IO调度算法

NONE:无任何处理

NOOP:FIFO队列，做基本的请求合并，常用于SSD

CFQ:完全公平调度器，每个进程一个IO调度队列，按时间片把IO处理机会均匀分给每个进程，适合进程数多的系统
DeadLine:读写请求分别创建IO队列，并确保达到deadline的请求优先处理，多用于IO压力重的场景

*Linux I/O栈=文件系统+通用块层+设备层*

**磁盘性能指标**

使用率：磁盘处理IO的时间百分比，没考虑IO的大小

饱和度：磁盘处理IO的繁忙程度，100%则无法接受新IO请求

IOPS：每秒IO次数

吞吐量：每秒IO的数据量

响应时间：从发出请求到收到响应的时间间隔

```shell
# 观察每块磁盘的使用情况
iostat -d -x 1 #-d -x表示显示所有磁盘I/O的指标
Device            r/s     w/s     rkB/s     wkB/s   rrqm/s   wrqm/s  %rrqm  %wrqm r_await w_await aqu-sz rareq-sz wareq-sz  svctm  %util 
loop0            0.00    0.00      0.00      0.00     0.00     0.00   0.00   0.00    0.00    0.00   0.00     0.00     0.00   0.00   0.00 
sda              0.00    0.00      0.00      0.00     0.00     0.00   0.00   0.00    0.00    0.00   0.00     0.00     0.00   0.00   0.00 
# 输出列含义依次是：
# 每秒发给磁盘的读/写 请求数 数据量[KB]
# 每秒合并的读/写 请求数 百分比
# 读/写 请求处理完成等待的时间[包括通用块层排队时间+设备实际处理时间]
# 请求队列平均长度
# 每个读/写请求平均大小[KB]
# 处理IO请求所需的平均时间[ms，不包括等待时间，推断数据]
# 磁盘处理IO的所花时间占总时间的百分比

# 观察每个进程的IO情况
pidstat -d 1 
13:39:51      UID       PID   kB_rd/s   kB_wr/s kB_ccwr/s iodelay  Command 
13:39:52      102       916      0.00      4.00      0.00       0  rsyslogd
# 每秒读取的数据大小/写请求数据大小/取消写请求数据大小/块IO延迟[包括等待同步块IO和换入块IO的时间]/

# iotop 类top命令 观察IO较大的进程排名
iotop

lsof -p $pid #查看进程打开的文件，可以和strace打印的的系统调用对应上，包括各种类型的文件
strace -T -tt -pf $pid # -f 多线程进程时，-f能观察除主线程外的其他线程的系统调用，-T显示系统调用所用时长，-tt显示跟踪时间
pstree -t -a -p $pid # 查看进程树。-t表示线程，-a表示显示命令行参数
```

```shell
# filetop,跟踪内核中文件读写情况，并输出线程ID/读写大小/读写类型/文件名称，bcc软件包的一部分
./filetop -C #-C 选项表示输出新内容时不清空屏幕
ps -efT # -T显示线程ID,和filetop命令的输出对应上

# opensnoop 跟踪内核中的open系统调用，属于bcc软件包
opensnoop
```

```shell
# nsenter 能进入linux namespace
nsenter --target $PID --net -- lsof -i # --net进入net namespace;lsod -i显示tcp socket文件的tcp连接信息
```

###### fio

```shell
# 磁盘的性能如何，操作系统没有提供接口去查询，只有通过实际测试去测出磁盘的性能，fio是一个很好的测试工具
# 使用例子
# 随机读
fio -name=randread -direct=1 -iodepth=64 -rw=randread -ioengine=libaio -bs=4k -size=1G -numjobs=1 -runtime=1000 -group_reporting -filename=/dev/sdb

# 随机写
fio -name=randwrite -direct=1 -iodepth=64 -rw=randwrite -ioengine=libaio -bs=4k -size=1G -numjobs=1 -runtime=1000 -group_reporting -filename=/dev/sdb

# 顺序读
fio -name=read -direct=1 -iodepth=64 -rw=read -ioengine=libaio -bs=4k -size=1G -numjobs=1 -runtime=1000 -group_reporting -filename=/dev/sdb

# 顺序写
fio -name=write -direct=1 -iodepth=64 -rw=write -ioengine=libaio -bs=4k -size=1G -numjobs=1 -runtime=1000 -group_reporting -filename=/dev/sdb 

# direct，表示是否跳过系统缓存。上面示例中，我设置的 1 ，就表示跳过系统缓存。
# iodepth，表示使用异步 I/O（asynchronous I/O，简称 AIO）时，同时发出的 I/O 请求上限。在上面的示例中，我设置的是 64。
# rw，表示 I/O 模式。我的示例中， read/write 分别表示顺序读 / 写，而 randread/randwrite 则分别表示随机读 / 写。
# ioengine，表示 I/O 引擎，它支持同步（sync）、异步（libaio）、内存映射（mmap）、网络（net）等各种 I/O 引擎。上面示例中，我设置的 libaio 表示使用异步 I/O。
# bs，表示 I/O 的大小。示例中，我设置成了 4K（这也是默认值）。
# filename，表示文件路径，当然，它可以是磁盘路径（测试磁盘性能），也可以是文件路径（测试文件系统性能）。示例中，我把它设置成了磁盘 /dev/sdb。不过注意，用磁盘路径测试写，会破坏这个磁盘中的文件系统，所以在使用前，你一定要事先做好数据备份。

# fio输出测试结果
slat ，是指从 I/O 提交到实际执行 I/O 的时长（Submission latency）；
clat ，是指从 I/O 提交到 I/O 完成的时长（Completion latency）；
lat ，指的是从 fio 创建 I/O 到 I/O 完成的总时长。
对同步 I/O 来说，由于 I/O 提交和 I/O 完成是一个动作，所以 slat 实际上就是 I/O 完成的时间，而 clat 是 0。而从示例可以看到，使用异步 I/O（libaio）时，lat 近似等于 slat + clat 之和。

# fio支持IO重复
blktrace /dev/sdb # 跟踪磁盘IO行为，生成sdb.blktrace.0 sdb.blktrace.1文件
blkparse sdb -d sdb.bin # 将结果转化为二进制文件
fio --name=replay --filename=/dev/sdb --direct=1 --read_iolog=sdb.bin # 使用fio重放日志
```

```shell

# 工具总结
fio # 磁盘性能基准测试工具
iostat
pidstat
sar
dstat
iotop
slabtop
vmstat
blktrace
biosnoop
biotop
strace
perf
df
mount
du
tune2fs
hdparam
/proc/slabinfo
/proc/meminfo
/proc/diskstats
/proc/pid/io
```

# 网络

![img](./c7b5b16539f90caabb537362ee7c27ac.png)

```shell
# 网络配置，网络接口收发包统计信息
ethtool eth0 | grep Speed # 网卡最大带宽

ifconfig eth0
eth0: flags=4163<UP,BROADCAST,RUNNING,MULTICAST> mtu 1500 # 没有RUNNING则没插网线
...
ip -s addr show dev eth0
2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc mq state UP group default qlen 1000 # 没有LOWER_UP则没插网线
...
# TX和RX中的errors、dropped、overruns、carrier 以及 collisions 等指标不为0时通常表示出现了网络IO问题
# errors 表示发生错误的数据包数，比如校验错误、帧同步错误等；
# dropped 表示丢弃的数据包数，即数据包已经收到了 Ring Buffer，但因为内存不足等原因丢包；
# overruns 表示超限数据包数，即网络 I/O 速度过快，导致 Ring Buffer 中的数据包来不及处理（队列满）而导致的丢包；
# carrier 表示发生 carrirer 错误的数据包数，比如双工模式不匹配、物理电缆出现问题等；
# collisions 表示碰撞数据包数；


# 套接字信息
# 推荐使用ss；-l 表示只显示监听套接字；-n 表示显示数字地址和端口(而不是名字)；-p 表示显示进程信息；-t 表示只显示 TCP 套接字
$ ss -ltnp | head -n 3 # netstat -nlp | head -n 3
State    Recv-Q    Send-Q        Local Address:Port        Peer Address:Port
LISTEN   0         128           127.0.0.53%lo:53               0.0.0.0:*        users:(("systemd-resolve",pid=840,fd=13))
LISTEN   0         128                 0.0.0.0:22               0.0.0.0:*        users:(("sshd",pid=1459,fd=3))
# 连接状态（Established）时：
#	Read-Q:还没被应用程序取走的字节数[接收队列长度]
#	Send-Q:还没有被远端主机确认的字节数[发送队列长度]
# 监听状态（Listening）时：(全连接：完成了3次握手但还没accept()的套接字)(半连接：收到了第一个SYN包的套接字)
#	Read-Q:全连接队列的长度
#	Send-Q:全连接队列的最大长度

$ netstat -s # 查看协议栈信息 ss -s


# 网络吞吐和PPS
sar -n DEV 1


# 主机间连通性和延时
$ ping -c3 114.114.114.114 # -c3表示发送三次ICMP包后停止
```

**IO模型优化**

非堵塞IO+水平触发通知：select or pool。单线程等待所有socket，不需要使用多线程或者多进程来占用系统资源。有网络事件发生时需要轮询[select(O(n^2))   poll(O(n))]，用户侧和内核侧存在参数[大数据结构]拷贝。

非堵塞IO+边缘触发通知：epoll。不需要频繁拷贝大数据结构，由内核自己维护socket列表。只返回发生网络事件的socket，不需要轮询所有使用的socket。

AIO：Asynchronous I/O。

**工作模型优化**

主进程bind+listen后，fork多个子进程，每个子进程内各自accept和epoll_wait。v2.6前惊群问题(一个三次握手触发所有进程的epoll_wait唤醒[nginx通过进程间全局锁规避惊群])。多进程也可以改为多线程。

SO_REUSEPORT选项[Linux 3.9]，多进程可以同时监听相同的接口，内核负责将请求负载均衡到这些监听进程中。Nginx1.9.1开始支持这种模式。

**跨过内核瓶颈**

DPDK:用户态轮询网卡，大页，CPU绑定，内存对齐，流水线并发等

XDP:和bcc-tools一样，都基于eBPF机制实现。4.8版本以上。

#### 各协议层的性能测试

**转发性能**

```shell
# ping
# hping3
# 高性能网络测试工具pktgen,pktgen作为内核线程运行，需要内核编译时配置过CONFIG_NET_PKTGEN=m，加载pktgen内核模块，然后通过/proc文件系统交互。
$ modprobe pktgen # 载入内核模块
$ ps -ef | grep pktgen | grep -v grep
root     26384     2  0 06:17 ?        00:00:00 [kpktgend_0] #CPU0的内核线程
root     26385     2  0 06:17 ?        00:00:00 [kpktgend_1] #CPU1的内核线程
$ ls /proc/net/pktgen/
kpktgend_0  kpktgend_1  pgctrl # 每个文件与一个内核线程交互，pgctrl用来控制测试的启停
```

```shell
#使用pktgen
# 定义一个工具函数，方便后面配置各种测试选项
function pgset() {
    local result
    echo $1 > $PGDEV

    result=`cat $PGDEV | fgrep "Result: OK:"`
    if [ "$result" = "" ]; then
         cat $PGDEV | fgrep Result:
    fi
}

# 为0号线程绑定eth0网卡
PGDEV=/proc/net/pktgen/kpktgend_0
pgset "rem_device_all"   # 清空网卡绑定
pgset "add_device eth0"  # 添加eth0网卡

# 配置eth0网卡的测试选项
PGDEV=/proc/net/pktgen/eth0
pgset "count 1000000"    # 总发包数量
pgset "delay 5000"       # 不同包之间的发送延迟(单位纳秒)
pgset "clone_skb 0"      # SKB包复制
pgset "pkt_size 64"      # 网络包大小[Byte]
pgset "dst 192.168.0.30" # 目的IP
pgset "dst_mac 11:11:11:11:11:11"  # 目的MAC

# 启动测试
PGDEV=/proc/net/pktgen/pgctrl
pgset "start"

# 结束后查看测试报告
$ cat /proc/net/pktgen/eth0
Params: count 1000000  min_pkt_size: 64  max_pkt_size: 64
     frags: 0  delay: 0  clone_skb: 0  ifname: eth0
     flows: 0 flowlen: 0
...
Current: # 测试进度
     pkts-sofar: 1000000  errors: 0
     started: 1534853256071us  stopped: 1534861576098us idle: 70673us
...
Result: OK: 8320027(c8249354+d70673) usec, 1000000 (64byte,0frags) # 测试结果
  120191pps 61Mb/sec (61537792bps) errors: 0
```

**TCP/UDP性能**

```shell
# netperf
# iperf : yum install iperf3
# -s表示启动服务端，-i表示汇报间隔，-p表示监听端口
$ iperf3 -s -i 1 -p 10000 # A机器运行服务端
# -c表示启动客户端，192.168.0.30为目标服务器的IP
# -b表示目标带宽(单位是bits/s)
# -t表示测试时间
# -P表示并发数，-p表示目标服务器监听端口
$ iperf3 -c 192.168.0.30 -b 1G -t 15 -P 2 -p 10000 # B机器运行客户端
```

**HTTP性能**

```shell
# webbench
# ab:apache自带的http压测工具
# Ubuntu
$ apt-get install -y apache2-utils
# CentOS
$ yum install -y httpd-tools
# -c表示并发请求数为1000，-n表示总的请求数为10000
$ ab -c 1000 -n 10000 http://192.168.0.30/
```

**模拟应用负载[模拟用户行为真实负载]**

```shell
# TCPCopy
# jmeter
# LoadRunner
# wrk: http性能测试工具，内置LuaJIT，可以使用lua构造请求负载
$ https://github.com/wg/wrk
$ cd wrk
$ apt-get install build-essential -y
$ make
$ sudo cp wrk /usr/local/bin/
$ wrk -c 1000 -t 2 http://192.168.0.30/ # 简单使用，-c表示并发连接数1000，-t表示线程数为2
$ wrk -c 1000 -t 2 -s auth.lua http://192.168.0.30/ # lua脚本参考官方示例
```

#### DNS解析

DNS服务：应用层协议，基于UDP的较多，一般监听53端口。

```shell
cat /etc/resolv.conf # DNS服务地址配置
time nslookup github.com # 查询DNS,获取指定域名对应的IP。time获取nslookup命令的执行时间

# 跟踪DNS服务查询递归过程
$ dig +trace +nodnssec github.com # +trace表示开启跟踪查询; +nodnssec表示禁止DNS安全扩展
```

#### tcpdump & wireshark

```shell
# example 
$ tcpdump -nn udp port 53 or host 35.190.27.188 # -nn ，表示不解析抓包中的域名（即不反向解析）、协议以及端口号
#输出格式: 时间戳 协议 源地址.源端口 > 目的地址.目的端口 网络包详细信息
```

![img](./859d3b5c0071335429620a3fcdde4fff.png)

![img](./4870a28c032bdd2a26561604ae2f7cb3.png)

DoS攻击:Denail of Service，使用大量合理请求占用服务器资源

DDoS攻击:Distributed Denial of Service，在DoS基础上，采用分布式架构，堕胎机器同时攻击目标服务器

```shell
sysctl net.ipv4.tcp_max_syn_backlog # 查看半连接队列最大容量
sysctl -w net.ipv4.tcp_max_syn_backlog=1024 # 设置半连接队列最大容量
sysctl -w net.ipv4.tcp_synack_retries=1 # 设置每个SYN_RECV失败时自动重试次数为1次。默认为5次
sysctl -w net.ipv4.tcp_syncookies=1 # 开启TCP SYN Cookies，专门防御 SYN Flood 攻击的方法
cat /etc/sysctl.conf # 以上临时设置想要持久化，需要写到该文件中。
```

**网络延迟**

Nagle:合并小包，降低网络流量开销，增加网络延迟。

```shell
$ hping3 -c 3 -S -p 80 baidu.com # -c表示发送3次请求，-S表示设置TCP SYN，-p表示端口号为80
$ traceroute --tcp -p 80 -n baidu.com # --tcp表示使用TCP协议，-p表示端口号，-n表示不对结果中的IP地址执行反向域名解析
```

**NAT**

NAT：重写ip报文头中的ip地址。普遍用来解决公网ip地址短缺问题；也用来做网络安全隔离。又根据一些实现方式分为 软件实现/硬件实现；静态[永久]映射/动态映射；

NAPT[内网IP的端口映射为外网IP的一个端口]：SNAT[目标地址不变，只替换源IP和端口]；DNAT[源地址不变，只替换目标IP或端口]；



# 综合





