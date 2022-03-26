# 问题排查

## 整机：top

```shell
[root@121 ~]# top
top - 19:13:27 up 480 days,  8:42,  1 user,  load average: 0.08, 0.08, 0.12
Tasks: 228 total,   1 running, 227 sleeping,   0 stopped,   0 zombie
%Cpu(s):  1.2 us,  0.8 sy,  0.0 ni, 97.8 id,  0.3 wa,  0.0 hi,  0.0 si,  0.0 st
KiB Mem : 16265256 total,  4638172 free,  9629568 used,  1997516 buff/cache
KiB Swap:        0 total,        0 free,        0 used.  5538452 avail Mem 

  PID USER      PR  NI    VIRT    RES    SHR S  %CPU %MEM     TIME+ COMMAND                                                               
28338 root      20   0 6845536 527476  18168 S   1.7  3.2 414:41.00 java                                                                  
28561 root      20   0 6841432 506400  18120 S   1.3  3.1 370:28.44 java                                                                  
28460 root      20   0 6841432 483348  18116 S   1.0  3.0 366:10.10 java     
```

能从大体上定位问题

## 虚拟内存、进程、CPU概况 vmstat

```shell
[root@121 ~]# vmstat -n 2 3
procs -----------memory---------- ---swap-- -----io---- -system-- ------cpu-----
 r  b   swpd   free   buff  cache   si   so    bi    bo   in   cs us sy id wa st
 1  0      0 4636576 147080 1853640    0    0    14    66    0    0  7  1 91  0  0
 0  0      0 4636180 147080 1853640    0    0     0   102 3825 7128  1  1 99  0  0
 0  0      0 4636180 147080 1853640    0    0     0    94 3843 7170  1  0 99  0  0

```

- procs
  - r：运行队列中进程的数量running
  - b：等待中的进程数量block
- memory
  - swpd：使用虚拟内存大小
  - free：可用内存大小
  - buff：用作缓冲的内存大小
  - cache：用作缓存的大小
- swap
  - si：每秒从交换区写道内存的大小
  - so：每秒写入交换区的内存大小
- IO
  - bi：每秒读取的块数
  - bo：每秒写入的块数
- 系统
  - in: 每秒中断数，包括时钟中断。【interrupt】
  - cs: 每秒上下文切换数。    【count/second】
- CPU
  - us：用户进程执行时间
  - sy：系统进程执行时间
  - id：空间时间
  - wa：等待IO时间

我主要是看CPU，如果us+sy>80%CPU性能不足 



## 磁盘IO

### sar



### iostat

```she
[root@121 ~]# iostat -dxk 1 2
Linux 3.10.0-1062.12.1.el7.x86_64 (121.37.211.94) 	03/26/2022 	_x86_64_	(8 CPU)

Device:         rrqm/s   wrqm/s     r/s     w/s    rkB/s    wkB/s avgrq-sz avgqu-sz   await r_await w_await  svctm  %util
vda               0.14     6.61    1.18   18.53   103.73   469.32    58.15     0.03    1.69    2.71    1.62   0.69   1.36
vdb               0.00     0.03    0.07    0.56     9.50    57.34   212.17     0.01   38.31   14.38   41.15   0.81   0.05
dm-0              0.00     0.00    0.04    0.24     2.18    10.82    90.87     0.00    6.85    3.86    7.36   1.00   0.03
dm-1              0.00     0.00    0.03    0.26     7.32    46.53   376.46     0.02   79.02   31.56   83.70   1.21   0.03
```

其中主要看三个参数：

- await：平均每次设备IO操作的等待时间(毫秒)
- svctm：评卷每次设备IO操作的服务时间(毫秒)
- %util：一秒钟有百分之几的事件用域IO操作

await 的大小取决于scvtm的大小和IO队列长度，如果svctm的值与await很接近，代表没有多少等待时间

%util如果接近百分百，代表IO满载运行

## 磁盘容量

### df  -h

查看分区或者目录大小

### du

查看文件大小 

```she
du -h  --max-depth=1

```



## 网络IO：ifstat

```shell
[root@121 ~]# ifstat
#kernel
Interface        RX Pkts/Rate    TX Pkts/Rate    RX Data/Rate    TX Data/Rate  
                 RX Errs/Drop    TX Errs/Drop    RX Over/Rate    TX Coll/Rate  
lo                 4029M 0         4029M 0       248385K 0       248385K 0      
                       0 0             0 0             0 0             0 0      
eth0             981377K 0       705434K 0         2397M 0       810887K 0      
                       0 0             0 0             0 0             0 0      
docker0                0 0             0 0             0 0             0 0      
                       0 0             0 0             0 0             0 0  
```



## netstat

看网络连接情况

```shell
netstat -an |grep 'ESTABLISHED' |grep 'tcp' |wc -l  # 统计TCP连接数
```





