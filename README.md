![img](./920601da775da08844d231bc2b4c301d.png)



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

###### 内存泄漏

```shell
#内存紧张时，系统通过回写脏页回收缓存页、利用swap，最终OOM （Out of Memory）机制杀死占用内存最多的进程
vmstat 1 #每秒1次打印内存信息
# pmap -x 查看进程内存分布

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



# I/O



# 网络



# 综合



