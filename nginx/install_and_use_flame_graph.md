# 安装与使用 (for centos)

## 查看内核版本 

```
uname -r (ex: 3.10.0-123.el7.x86_64)
```





## 安装内核包

#### 根据包名到 `http://rpm.pbone.net/` 搜索

```
rpm -ivh kernel-debuginfo-$(uname -r).rpm
rpm -ivh kernel-debuginfo-common-x86_64-$(uname -r).rpm || kernel-debuginfo-common-$(uname -r).rpm 
rpm -ivh kernel-devel-$(uname -r).rpm
```

## 安装systemtap

```
yum install gcc gcc-c++ elfutils-devel
wget https://sourceware.org/systemtap/ftp/releases/systemtap-3.1.tar.gz
tar -xvf systemtap-3.1.tar.gz
cd systemtap-3.1/
./configure --prefix=/opt/stap --disable-docs --disable-publican --disable-refdocs CFLAGS="-g -O2"
make -j8  
sudo make install
```






## 下载systemtap套件以及画图工具

### systemtap套件

```
wget https://github.com/openresty/openresty-systemtap-toolkit/archive/master.zip
```

### 画图工具(火焰图)

> systemtap 的数据转换 `stackcollapse-stap.pl` 和 最后的画图 `flamegraph.pl`

```
wget https://github.com/brendangregg/FlameGraph/archive/master.zip
```


## 测试是否安装成功

```
# stap -v -e 'probe vfs.read {printf("read performed\n"); exit()}'

Pass 1: parsed user script and 465 library scripts using 175108virt/43196res/2208shr/41464data kb, in 120usr/10sys/138real ms.
Pass 2: analyzed script: 1 probe, 1 function, 7 embeds, 0 globals using 294564virt/163780res/3476shr/160920data kb, in 790usr/180sys/965real ms.
Pass 3: using cached /root/.systemtap/cache/3b/stap_3b9eab162d1643cde029ab9801833b91_2677.c
Pass 4: using cached /root/.systemtap/cache/3b/stap_3b9eab162d1643cde029ab9801833b91_2677.ko
Pass 5: starting run.
read performed
Pass 5: run completed in 0usr/10sys/316real ms.
```


## 使用

> 在进程有数据访问的情况下

```
./openresty-systemtap-toolkit/sample-bt  -p 12605 -t 30 -u > flame.bt 
./FlameGraph/stackcollapse-stap.pl flame.bt > flame.cbt
./FlameGraph/flamegraph.pl flame.cbt > flame.svg
```





> 注：有时候运行 stap -v -e 'probe vfs.read {printf("read performed\n"); exit()}' 命令的时候会出错，错误如下：

```
Pass 1: parsed user script and 465 library scripts using 212868virt/44160res/2824shr/41600data kb, in 140usr/0sys/152real ms. 
Pass 2: analyzed script: 1 probe, 1 function, 7 embeds, 0 globals using 334808virt/167324res/4092shr/163540data kb, in 850usr/180sys/1024real ms. 
Pass 3: using cached /root/.systemtap/cache/91/stap_910821b500037ab6493098235f73ae89_2677.c 
Pass 4: using cached /root/.systemtap/cache/91/stap_910821b500037ab6493098235f73ae89_2677.ko 
Pass 5: starting run. ERROR: module version mismatch (#1 SMP Mon Mar 9 16:14:50 CDT 2015 vs #1 SMP Fri Mar 6 11:36:42 UTC 2015), release 3.10.0-229.el7.x86_64 WARNING: /opt/stap/bin/staprun exited with status: 1 
Pass 5: run completed in 0usr/10sys/44real ms.
Pass 5: run failed. [man error::pass5]
```

> 解决方案

```
修改/usr/src/kernels/3.10.0-123.el7.x86_64/include/generated/compile.h文件中的UTS_VERSION，改为uname -v的值
并重新编译systemtap
```

