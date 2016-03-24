title: mysql的一次优化
---
### 背景
服务器在我自己的macpro上登录是没有压力的，1000人登录完成大概在5左右，登录创建大概是8秒，基本达标。但是转移到外网压力测试机器上创建登录却达到了好几分钟，看日志一直可以以正常的速度看到save日志（保存到数据库）。后面也可以看到，一但全部1000人正常创建后，下次1000人登录在4秒以内。

### 数据库版本
一开始我对比了数据库版本，mac下时5.6，linux是5.1.7 (centos 6.7 final)。网上查了下版本对比，性能是有提升，20%，但不至于差那么多。不管怎样先升级再说，至少可以对比下配置。升级完后，查看了几个比较关键的参数，都是默认的一样。

### 压力测试对比
mysqlslap：mysql自带的工具压力测试

``` bash
mysqlslap -a --engine=myisam,innodb --auto-generate-sql-load-type=read
--number-of-queries 1000 --iterations=5 -uroot -people
```

| read       | myisam  |  innodb |
| --------   | -----:  	| :----:  |
| macpro     | 0.08 	|   0.1   |
| linux      | 0.15   	|   0.2   |

``` bash
mysqlslap -a --engine=myisam,innodb --auto-generate-sql-load-type=write
--number-of-queries 1000 --iterations=5 -uroot -people
```

| write       | myisam  |  innodb |
| --------   | -----:  	| :----:  |
| macpro     | 0.04 	|   0.1   |
| linux      | 0.03   	|   50   |

``` bash
mysqlslap -a --engine=myisam,innodb --auto-generate-sql-load-type=update
--number-of-queries 1000 --iterations=5 -uroot -people
```

| update     | myisam  |  innodb |
| --------   | -----:  	| :----:  |
| macpro     | 0.5 		|   0.1   |
| linux      | 0.3   	|   50   |

innodb在写操作达到惊人的500倍数。。。又对比了下双方innodb的一些关键参数，一样。你tm在逗我，这不会是磁盘坏的吧。。。

### 磁盘io读写测试

``` bash
＃ 写文件到当前目录
time dd if=/dev/zero of=10Gb.file bs=1024 count=10000000
＃ 从当前目录10Gb.file读取
time dd if=10Gb.file bs=64k |dd of=/dev/null
＃ sar观察磁盘io
sar -d 1 -p

```
写的观察结果如下

``` bash
       		DEV       tps  rd_sec/s  wr_sec/s  avgrq-sz  avgqu-sz     await     svctm     %util
       		sda     405.56     0.00    415288.89     1024      158.71     395.91     2.74     111.11
 VolGroup-lv_root    54613.33   0.00    436906.73      8.00      20311.46   376.25     0.02     111.11
 VolGroup-lv_swap    0.00     0.00      0.00      0.00      0.00      	0.00      0.00      0.00
 VolGroup-lv_home    0.00      0.00      0.00      0.00      0.00      0.00       0.00      0.00
```
之后又到mysql的data目录测试也是一样很快，看来不是磁盘坏了，呵呵

``` bash
mysql> show variables like "%dir%";
datadir ｜ /var/lib/mysql/
```
再看看用mysqlslap写时候的数据

``` bash
       		DEV       tps  rd_sec/s  wr_sec/s  avgrq-sz  avgqu-sz     await     svctm     %util
       		sda     126.26      0.00    824.24     6.53      1.01     8.02     7.97     100.61
 VolGroup-lv_root     103.03      0.00    824.24      8.00      1.01      9.82      9.76     100.61
 VolGroup-lv_swap      0.00      0.00      0.00      0.00      0.00      0.00      0.00      0.00
 VolGroup-lv_home      0.00      0.00      0.00      0.00      0.00      0.00      0.00      0.00
```

sar结果具体参数网摘如下

``` bash
参数-p可以打印出sda,hdc等磁盘设备名称,如果不用参数-p,设备节点则有可能是dev8-0,dev22-0

tps:每秒从物理磁盘I/O的次数.多个逻辑请求会被合并为一个I/O磁盘请求,一次传输的大小是不确定的.
rd_sec/s:每秒读扇区的次数.

wr_sec/s:每秒写扇区的次数.

avgrq-sz:平均每次设备I/O操作的数据大小(扇区).

avgqu-sz:磁盘请求队列的平均长度.

await:从请求磁盘操作到系统完成处理,每次请求的平均消耗时间,包括请求队列等待时间,单位是毫秒(1秒=1000毫秒).

svctm:系统处理每次请求的平均时间,不包括在请求队列中消耗的时间.

%util:I/O请求占CPU的百分比,比率越大,说明越饱和.
```
对比一下就可以得到有意义的数据

``` bash
wr_sec/s：差距巨大
avgqu-sz：几乎为1，本身说明io请求就是一个接着一个，很快联想到innodb的一个重要参数
innodb_flush_logs_at_trx_commit 默认设置的是1 也就是同步刷新log
avgrq-sz：相比较小（同步的话，每次数据io很小）
await：相比较短（同步的话，就不用再队列中等待了）
svctm：相比较长（因为不包括队列中时间，同步的话就显长了）

```

``` bash
mysql > set global innodb_flush_logs_at_trx_commit=2;
```
一切就顺滑如飞了

参考：

[优化系列 | 实例解析MySQL性能瓶颈排查定位](http://imysql.com/2016/01/13/mysql-optimization-case-howto-find-performance-bottleneck.shtml)

[MySQL自带的性能压力测试工具mysqlslap](http://xstarcd.github.io/wiki/MySQL/mysqlslap.html)

