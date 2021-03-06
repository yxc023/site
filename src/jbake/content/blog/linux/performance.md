+++
date = "2016-04-01T23:29:32+08:00"
draft = false
title = "性能分析vmstat"
categories = [ "vmstat" ]
type="linux"

+++

1. 性能分析的目的

1）找出系统性能瓶颈（包括硬件瓶颈和软件瓶颈）；
2）提供性能优化的方案（升级硬件？改进系统系统结构？）；
3）达到合理的硬件和软件配置；
4）使系统资源使用达到最大的平衡。（一般情况下系统良好运行的时候恰恰各项资源达到了一个平衡体，任何一项资源的过渡使用都会造成平衡体系破坏，从而造成系统负载极高或者响应迟缓。比如CPU过渡使用会造成大量进程等待CPU资源，系统响应变慢，等待会造成进程数增加，进程增加又会造成内存使用增加，内存耗尽又会造成虚拟内存使用，使用虚拟内存又会造成磁盘IO增加和CPU开销增加）

<!-- more -->

2. 影响性能的因素

1）CPU（cpu的速度与性能很大一部分决定了系统整体的性能，是否使用SMP）

2）内存（物理内存不够时会使用交换内存，使用swap会带来磁盘I0和cpu的开销）

3）硬盘（存储系统）
a.Raid技术使用（RAID0, RAID1, RAID5, RAID0+1）
b.小文件读写瓶颈是磁盘的寻址（tps），大文件读写的性能瓶颈是带宽
c.Linux可以利用空闲内存作文件系统访问的cache，因此系统内存越大存储系统的性能也越好

4）网络带宽。


3. 性能分析的步骤

1)对资源的使用状况进行长期的监控和数据采集（nagios、cacti）

2）使用常见的性能分析工具（vmstat、top、free、iostat等)

3）经验积累
a.应用程序设计的缺陷和数据库查询的滥用最有可能导致性能问题
b.性能瓶颈可能是因为程序差/内存不足/磁盘瓶颈，但最终表现出的结果就是CPU耗尽，系统负载极高，响应迟缓，甚至暂时失去响应
c.物理内存不够时会使用交换内存，使用swap会带来磁盘I0和cpu的开销
d.可能造成cpu瓶颈的问题：频繁执Perl，php，java程序生成动态web；数据库查询大量的where子句、order by/group by排序……
e.可能造成内存瓶颈问题：高并发用户访问、系统进程多，java内存泄露……
f.可能造成磁盘IO瓶颈问题：生成cache文件，数据库频繁更新，或者查询大表……


4. vmstat详细介绍

vmstat是一个很全面的性能分析工具，可以观察到系统的进程状态、内存使用、虚拟内存使用、磁盘的IO、中断、上下文切换、CPU使用等。对于 Linux 的性能分析，100%理解 vmstat 输出内容的含义，并能灵活应用，那对系统性能分析的能力就算是基本掌握了。
下面是vmstat命令的输出结果：

    [root@monitor-www ~]# vmstat 1 5
    procs         —————memory—————      ——swap—— ——io——  ——system——   ——cpu——
    r   b    swpd     free       buff      cache   si    so    bi     bo      in    cs    us sy  id wa st
    1   0    84780    909744   267428    1912076   0     0     20     94       0     0     2  1  95  1  0
    1   2    84780    894968   267428    1912216   0     0      0   1396    2301 11337     8  3  89  0  0
    1   0    84780    900680   267428    1912340   0     0     76   1428    1854  8082     7  2  90  0  0
    1   0    84780    902544   267432    1912548   0     0    116    928    1655  7502     7  2  92  0  0
    2   0    84780    900076   267432    1912948   0     0    180    904    1963  8703    10  3  87  0  0

对输出解释如下：

1）procs

a.r列表示运行和等待CPU时间片的进程数，这个值如果长期大于系统CPU个数，就说明CPU资源不足，可以考虑增加CPU；
b.b列表示在等待资源的进程数，比如正在等待I/O或者内存交换等。

2）memory

a.swpd列表示切换到内存交换区的内存数量（以KB为单位）。如果swpd的值不为0或者比较大，而且si、so的值长期为0，那么这种情况一般不用担心，不会影响系统性能；
b.free列表示当前空闲的物理内存数量（以KB为单位）；
c.buff列表示buffers cache的内存数量，一般对块设备的读写才需要缓冲；
d.cache列表示page cached的内存数量，一般作文件系统的cached，频繁访问的文件都会被cached。如果cached值较大，就说明cached文件数较多。如果此时IO中的bi比较小，就说明文件系统效率比较好。

3）swap

a.si列表示由磁盘调入内存，也就是内存进入内存交换区的数量；
b.so列表示由内存调入磁盘，也就是内存交换区进入内存的数量
c.一般情况下，si、so的值都为0，如果si、so的值长期不为0，则表示系统内存不足，需要考虑是否增加系统内存。

4）IO

a.bi列表示从块设备读入的数据总量（即读磁盘，单位KB/秒）
b.bo列表示写入到块设备的数据总量（即写磁盘，单位KB/秒）
这里设置的bi+bo参考值为1000，如果超过1000，而且wa值比较大，则表示系统磁盘IO性能瓶颈。

5）system

a.in列表示在某一时间间隔中观察到的每秒设备中断数；
b.cs列表示每秒产生的上下文切换次数。
上面这两个值越大，会看到内核消耗的CPU时间就越多。

6）CPU

a.us列显示了用户进程消耗CPU的时间百分比。us的值比较高时，说明用户进程消耗的CPU时间多，如果长期大于50%，需要考虑优化程序啥的。
b.sy列显示了内核进程消耗CPU的时间百分比。sy的值比较高时，就说明内核消耗的CPU时间多；如果us+sy超过80%，就说明CPU的资源存在不足。
c.id列显示了CPU处在空闲状态的时间百分比；
d.wa列表示IO等待所占的CPU时间百分比。wa值越高，说明IO等待越严重。如果wa值超过20%，说明IO等待严重。
e.st列一般不关注，虚拟机占用的时间百分比。 （Linux 2.6.11）