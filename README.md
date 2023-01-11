# 测试步骤
## 系统调优

### 压测机
在/etc/sysctl.conf加入以下内容
```shell
# 系统级别最大打开文件
fs.file-max = 100000

# 单用户进程最大文件打开数
fs.nr_open = 100000

# 是否重用, 快速回收time-wait状态的tcp连接
net.ipv4.tcp_tw_reuse = 1
net.ipv4.tcp_tw_recycle = 1

# 单个tcp连接最大缓存byte单位
net.core.optmem_max = 8192

# 可处理最多孤儿socket数量，超过则警告，每个孤儿socket占用64KB空间
net.ipv4.tcp_max_orphans = 10240

# 最多允许time-wait数量
net.ipv4.tcp_max_tw_buckets = 10240

# 从客户端发起的端口范围,默认是32768 61000，则只能发起2w多连接，改为一下值，可一个IP可发起差不多6.4w连接。
net.ipv4.ip_local_port_range = 1024 65535
```

在/etc/security/limits.conf加入以下内容
```shell
# 最大不能超过fs.nr_open值, 分别为单用户进程最大文件打开数，soft指软性限制,hard指硬性限制
* soft nofile 100000
* hard nofile 100000
root soft nofile 1000000
root hard nofile 1000000
```
执行sysctl -p，使上述配置生效

### LB和RS
在/etc/sysctl.conf加入以下内容
```shell
# 系统最大文件打开数
fs.file-max = 20000000

# 单个用户进程最大文件打开数
fs.nr_open = 20000000

# 全连接队列长度,默认128
net.core.somaxconn = 10240
# 半连接队列长度，当使用sysncookies无效，默认128
net.ipv4.tcp_max_syn_backlog = 16384
net.ipv4.tcp_syncookies = 0

# 网卡数据包队列长度  
net.core.netdev_max_backlog = 41960

# time-wait 最大队列长度
net.ipv4.tcp_max_tw_buckets = 300000

# time-wait 是否重新用于新链接以及快速回收
net.ipv4.tcp_tw_reuse = 1  
net.ipv4.tcp_tw_recycle = 1

# tcp报文探测时间间隔, 单位s
net.ipv4.tcp_keepalive_intvl = 30
# tcp连接多少秒后没有数据报文时启动探测报文
net.ipv4.tcp_keepalive_time = 900
# 探测次数
net.ipv4.tcp_keepalive_probes = 3

# 保持fin-wait-2 状态多少秒
net.ipv4.tcp_fin_timeout = 15  

# 最大孤儿socket数量,一个孤儿socket占用64KB,当socket主动close掉,处于fin-wait1, last-ack
net.ipv4.tcp_max_orphans = 131072  

# 每个套接字所允许得最大缓存区大小
net.core.optmem_max = 819200

# 默认tcp数据接受窗口大小
net.core.rmem_default = 262144  
net.core.wmem_default = 262144  
net.core.rmem_max = 16777216  
net.core.wmem_max = 16777216

# tcp栈内存使用第一个值内存下限, 第二个值缓存区应用压力上限, 第三个值内存上限, 单位为page,通常为4kb
net.ipv4.tcp_mem = 786432 4194304 8388608
# 读, 第一个值为socket缓存区分配最小字节, 第二个，第三个分别被rmem_default, rmem_max覆盖
net.ipv4.tcp_rmem = 4096 4096 4206592
# 写, 第一个值为socket缓存区分配最小字节, 第二个，第三个分别被wmem_default, wmem_max覆盖
net.ipv4.tcp_wmem = 4096 4096 4206592

在/etc/security/limits.conf加入一下内容
# End of file
root      soft    nofile          2000000
root      hard    nofile          2000000
*         soft    nofile          2000000
*         hard    nofile          2000000
```
执行sysctl -p，使上述配置生效

## 多IP配置

### 压测机
配置压测机多IP，用来突破压测机 --> LB 并发 6w 的限制
```shell
for i in `seq 61 70`; do         sudo ip addr add 192.168.78.$i/32 dev lo; done
```
### LB
配置LB多IP，用来突破LB --> RS 并发 6w 的限制
```shell
for i in `seq 61 70`; do         sudo ip addr add 192.168.78.$i/32 dev lo; done
```

## 压测软件

### 压测机
自研client客户端，使用src IP 192.168.78.61-70，各自逐渐发起30000并发
```shell
for i in {61..70}; do  ./client -src 192.168.78.$i -addr 192.168.78.51:80 -batch 1 -conn 30000 & done
```

### LB
pipy，运行4LB脚本（根据CPu配置启动n个进程）

### RS
fortio ，使用fortio模拟tcp服务端
```shell
fortio server -tcp-port 30003
```

## 观测方式

### nmon
```shell
nmon -f  -t  -s 10 -c 60 -m /root/logs
```
-s 10 每 10 秒采集一次数据
-t 采集top数据
-c 60 采集 60 次，即为采集十分钟的数据
-f 生成的数据文件名中包含文件创建的时间
-m 生成的数据文件的存放目录

### ss
在LB上观察入站连接数
```shell
while true ; do sleep 1; ss -nt | grep 192.168.78.51:80 | wc -l ; done
```
在LB上观察出站连接数
```shell
while true ; do sleep 1; ss -nt | grep 30003 | wc -l ; done
```
