# Redis简介
Redis 是完全开源免费的，遵守 BSD 协议，是一个灵活的高性能 key-value 数据结构存储，可以用来作为数据库、缓存和消息队列。

Redis 比其他 key-value 缓存产品有以下三个特点：

* Redis 支持数据的持久化，可以将内存中的数据保存在磁盘中，重启的时候可以再次加载到内存使用。
* Redis 不仅支持简单的 key-value 类型的数据，同时还提供 list，set，zset，hash 等数据结构的存储。
* Redis 支持主从复制，即 master-slave，哨兵，cluster模式的数据备份。

# Redis特点
* 高性能： Redis 将所有数据集存储在内存中，可以在入门级 Linux 机器中每秒写（SET）11 万次，读（GET）8.1 万次。Redis 支持 Pipelining 命令，可一次发送多条命令来提高吞吐率，减少通信延迟。
* 持久化：当所有数据都存在于内存中时，可以根据自上次保存以来经过的时间和/或更新次数，使用灵活的策略将更改异步保存在磁盘上。Redis 支持仅附加文件（AOF）持久化模式。
* 数据结构： Redis 支持各种类型的数据结构，例如字符串、散列、集合、列表、带有范围查询的有序集、位图、超级日志和带有半径查询的地理空间索引。
* 原子操作：处理不同数据类型的 Redis 操作是原子操作，因此可以安全地 SET 或 INCR 键，添加和删除集合中的元素等。
* 支持的语言： Redis 支持许多语言，如 C、C++、Erlang、Go、Haskell、Java、JavaScript（Node.js)、Lua、Objective-C、Perl、PHP、Python、R、Ruby、Rust、Scala、Smalltalk 等。
* 主/从复制： Redis 遵循非常简单快速的主/从复制。配置文件中只需要一行来设置它，而 Slave 在 Amazon EC2 实例上完成 10 MM key 集的初始同步只需要 21 秒。
* 分片： Redis 支持分片。与其他键值存储一样，跨多个 Redis 实例分发数据集非常容易。
* 可移植： Redis 是用 C 编写的，适用于大多数 POSIX 系统，如 Linux、BSD、Mac OS X、Solaris 等。

# Redis架构
Redis 主要由有两个程序组成：

* Redis 客户端 redis-cli
* Redis 服务器 redis-server

Redis集群(Redis Cluster) 是Redis提供的分布式数据库方案，通过 分片(sharding) 来进行数据共享，并提供复制和故障转移功能。相比于主从复制、哨兵模式，Redis集群实现了较为完善的高可用方案，解决了存储能力受到单机限制，写操作无法负载均衡的问题。

# Redis HA方案
1. master
2. master,slave主从复制
3. 哨兵模式
4. cluster集群模式

本文以1,2,4进行搭建和测试

# 测试方法
* redis-benchmark
```shell
Usage: redis-benchmark [-h <host>] [-p <port>] [-c <clients>] [-n <requests]> [-k <boolean>]
 -h <hostname>      # redis服务地址 (default 127.0.0.1)
 -p <port>          # redis服务的端口 (default 6379)
 -s <socket>        # 服务器的socket (overrides host and port)
 -a <password>      # 数据库密码
 -c <clients>       # 并发链接的客户端数量 (default 50)
 -n <requests>      # 请求的总数 (default 100000)
 -d <size>          # 单个数据包的大小 (单位是bytes，默认是2)
 --dbnum <db>       # 执行默认的数据库 (default 0)
 -k <boolean>       #客户端是否使用keepalive，1为使用，0为不使用，默认值为1
 -r <keyspacelen>   #  key的长度;SET/GET/INCR 使用随机 key, SADD 使用随机值。使用-r的话可以生成更多的key
 -P <numreq>        # 通过管道请求的数量 Default 1 (no pipeline).
 -q                 # 强制退出 redis。仅显示 query/sec 值
 --csv              # 以 CSV 格式输出
 -l                 # 循环执行，永久执行测试
 -t <tests>         # 仅运行以逗号分隔的测试命令列表。如果不指定会依次压测多个命令（get,set,incr,lpush）结果
 -I                 # Idle 模式。仅打开 N 个 idle 连接并等待。
```
# 部署
## master
1. 安装

```shell
echo -e "[base]\nname=CentOS- - Base\nbaseurl=http://192.168.66.196:8080/centos/\$releasever/os/\$basearch/\nenabled=1\ngpgcheck=0\n[updates]\nname=CentOS- - Updates\nbaseurl=http://192.168.66.196:8080/centos/\$releasever/updates/\$basearch/\nenabled=1\ngpgcheck=0\n[extras]\nname=CentOS- - Extras\nbaseurl=http://192.168.66.196:8080/centos/\$releasever/extras/\$basearch/\nenabled=1\ngpgcheck=0">/etc/yum.repos.d/CentOS-Base.repo

echo -e "[centos-openstack-stein]\nname=CentOS-7 - OpenStack stein\nbaseurl=http://192.168.66.196:8080/centos/\$releasever/cloud/\$basearch/openstack-stein/\ngpgcheck=0\nenabled=1\nexclude=sip,PyQt4">/etc/yum.repos.d/CentOS-OpenStack-stein.repo

yum clean all
yum makecache
yum -y install redis

mkdir -p /data/redis
echo -e "daemonize yes\nbind 0.0.0.0\nport 6379\ndbfilename dump.rdb\ndir /data/redis" > /etc/redis.conf
chown -R redis:redis /data/redis
sysctl vm.overcommit_memory=1
setenforce 0
sed -i  "s/SELINUX=enforcing/SELINUX=disabled/g" /etc/selinux/config
systemctl endable redis.service
systemctl start redis.service
```
或者 

```shell
wget https://packages.redis.io/redis-stack/redis-stack-server-6.2.4-v1.rhel7.x86_64.tar.gz
tar zxvf redis-stack-server-6.2.4-v1.rhel7.x86_64.tar.gz
mv redis-stack-server-6.2.4-v1 redis
mkdir -p /data/redis
echo -e "daemonize yes\nbind 0.0.0.0\nport 6379\ndbfilename dump.rdb\ndir /data/redis" > /etc/redis.conf
chown -R redis:redis /data/redis
sysctl vm.overcommit_memory=1
setenforce 0
sed -i  "s/SELINUX=enforcing/SELINUX=disabled/g" /etc/selinux/config
```
2. 配置
```shell
port 6379
logfile "redis-server-6379.log"
dbfilename "dump-6379.rdb"
daemonize yes
requirepass "123456"
masterauth "123456"
save 3600 1
save 300 100
save 60 10000
maxclients 10000
maxmemory 14G
maxmemory-policy allkeys-lru
maxmemory-samples 5
```

3. 启动
```shell
./bin/redis-server /etc/redis.conf
./bin/redis-cli -p 6379
```

## master,slave主从复制
1. 安装

```shell
echo -e "[base]\nname=CentOS- - Base\nbaseurl=http://192.168.66.196:8080/centos/\$releasever/os/\$basearch/\nenabled=1\ngpgcheck=0\n[updates]\nname=CentOS- - Updates\nbaseurl=http://192.168.66.196:8080/centos/\$releasever/updates/\$basearch/\nenabled=1\ngpgcheck=0\n[extras]\nname=CentOS- - Extras\nbaseurl=http://192.168.66.196:8080/centos/\$releasever/extras/\$basearch/\nenabled=1\ngpgcheck=0">/etc/yum.repos.d/CentOS-Base.repo

echo -e "[centos-openstack-stein]\nname=CentOS-7 - OpenStack stein\nbaseurl=http://192.168.66.196:8080/centos/\$releasever/cloud/\$basearch/openstack-stein/\ngpgcheck=0\nenabled=1\nexclude=sip,PyQt4">/etc/yum.repos.d/CentOS-OpenStack-stein.repo

yum clean all
yum makecache
yum -y install redis

mkdir -p /data/redis
chown -R redis:redis /data/redis
sysctl vm.overcommit_memory=1
setenforce 0
sed -i  "s/SELINUX=enforcing/SELINUX=disabled/g" /etc/selinux/config

```

或者 

```shell
wget https://packages.redis.io/redis-stack/redis-stack-server-6.2.4-v1.rhel7.x86_64.tar.gz
tar zxvf redis-stack-server-6.2.4-v1.rhel7.x86_64.tar.gz
mv redis-stack-server-6.2.4-v1 redis
mkdir -p /data/redis

sysctl vm.overcommit_memory=1
setenforce 0
sed -i  "s/SELINUX=enforcing/SELINUX=disabled/g" /etc/selinux/config
```

2. 配置

* master
  
```shell
vim /etc/redis.conf
daemonize yes
bind 0.0.0.0
port 6379
dbfilename dump.rdb
dir /data/redis
requirepass "123456"
masterauth "123456"
save 3600 1
save 300 100
save 60 10000
maxclients 10000
maxmemory 14G
maxmemory-policy allkeys-lru
maxmemory-samples 5
  ```
* slave
```shell
daemonize yes
bind 0.0.0.0
port 26379
slaveof {master_ip}  6379
dbfilename dump.rdb
dir /data/redis
requirepass "123456"
masterauth "123456"
save 3600 1
save 300 100
save 60 10000
maxclients 10000
maxmemory 14G
maxmemory-policy allkeys-lru
maxmemory-samples 5
```
3. 启动

* master
```shell
./bin/redis-server /etc/redis.conf
./bin/redis-cli -p 6379
```
* slave
 ```shell
./bin/redis-server /etc/redis.conf
./bin/redis-cli -p 26379
``` 


## cluster集群模式
1. 安装
```shell
echo -e "[base]\nname=CentOS- - Base\nbaseurl=http://192.168.66.196:8080/centos/\$releasever/os/\$basearch/\nenabled=1\ngpgcheck=0\n[updates]\nname=CentOS- - Updates\nbaseurl=http://192.168.66.196:8080/centos/\$releasever/updates/\$basearch/\nenabled=1\ngpgcheck=0\n[extras]\nname=CentOS- - Extras\nbaseurl=http://192.168.66.196:8080/centos/\$releasever/extras/\$basearch/\nenabled=1\ngpgcheck=0">/etc/yum.repos.d/CentOS-Base.repo

echo -e "[centos-openstack-stein]\nname=CentOS-7 - OpenStack stein\nbaseurl=http://192.168.66.196:8080/centos/\$releasever/cloud/\$basearch/openstack-stein/\ngpgcheck=0\nenabled=1\nexclude=sip,PyQt4">/etc/yum.repos.d/CentOS-OpenStack-stein.repo

yum clean all
yum makecache
yum -y install redis

mkdir -p /data/redis
chown -R redis:redis /data/redis
sysctl vm.overcommit_memory=1
setenforce 0
sed -i  "s/SELINUX=enforcing/SELINUX=disabled/g" /etc/selinux/config

```

或者

```shell
wget https://packages.redis.io/redis-stack/redis-stack-server-6.2.4-v1.rhel7.x86_64.tar.gz
tar zxvf redis-stack-server-6.2.4-v1.rhel7.x86_64.tar.gz
mv redis-stack-server-6.2.4-v1 redis
mkdir -p /data/redis
sysctl vm.overcommit_memory=1
setenforce 0
sed -i  "s/SELINUX=enforcing/SELINUX=disabled/g" /etc/selinux/config
```

2. 配置
例子以3个master，3个slave，分别部署不同机器上。
端口分别为 6379 6479 6380 6480 6381 6481

```shell
vim etc/redis_6379.conf

port 6379
cluster-enabled yes
cluster-config-file "node-6379.conf"
logfile "redis-server-6379.log"
dbfilename "dump-6379.rdb"
daemonize yes

vim etc/redis_6479.conf

port 6479
cluster-enabled yes
cluster-config-file "node-6479.conf"
logfile "redis-server-6479.log"
dbfilename "dump-6479.rdb"
daemonize yes

vim etc/redis_6380.conf

port 6380
cluster-enabled yes
cluster-config-file "node-6380.conf"
logfile "redis-server-6380.log"
dbfilename "dump-6380.rdb"
daemonize yes

vim etc/redis_6480.conf

port 6480
cluster-enabled yes
cluster-config-file "node-6480.conf"
logfile "redis-server-6480.log"
dbfilename "dump-6480.rdb"
daemonize yes

vim etc/redis_6381.conf

port 6381
cluster-enabled yes
cluster-config-file "node-6381.conf"
logfile "redis-server-6381.log"
dbfilename "dump-6381.rdb"
daemonize yes

vim etc/redis_6481.conf

port 6481
cluster-enabled yes
cluster-config-file "node-6481.conf"
logfile "redis-server-6481.log"
dbfilename "dump-6481.rdb"
daemonize yes
```

每个节点相同
```shell
bind {本机ip}
port 6379
cluster-enabled yes
cluster-config-file "node-6379.conf"
logfile "redis-server-6379.log"
loglevel notice
dbfilename "dump-6379.rdb"
activedefrag yes 
active-defrag-ignore-bytes 1000mb #内存碎片字节数到1000m开始清理
active-defrag-threshold-lower 30 #内存碎片空间站操作系统分配给redis的百分之30开始清理
active-defrag-cycle-min 25 #自动清理所用cpu时间比例不低于25% 保证清理正常开展
active-defrag-cycle-max 75 #自动清理过程所用cpu时间比例不高于75% 一单超过 停止清理 避免清理时大量内存拷贝阻塞redis 导致响应延迟提高00mb开始清理
active-defrag-threshold-lower 30 #内存碎片空间站操作系统分配给redis的百分之30开始清理
active-defrag-cycle-min 25 #自动清理所用cpu时间比例不低于25% 保证清理正常开展
active-defrag-cycle-max 75 #自动清理过程所用cpu时间比例不高于75% 一单超过 停止清理 避免清理时大量内存拷贝阻塞redis 导致响应延迟提高
repl-backlog-size 1G #副本backlog大小
maxclients 10000 #最大客户端
maxmemory 14G #最大内存
maxmemory-policy allkeys-lru #内存满策略
maxmemory-samples 5 #随机5个keys最少使用的key清理i
io-threads 2 #开启线程进行写操作
save 3600 1 #表示如果每3600秒内至少1个key发生变化（新增、修改和删除），则重写rdb文件
save 300 100
save 60 10000
stop-writes-on-bgsave-error yes #如果后台save出错，前台要停止操作，保证数据一致性
requirepass "123456" #添加访问认证
masterauth "123456" #如果主节点开启了访问认证，从节点访问主节点需要认证
appendonly yes #是否开启 AOF 持久化模式
appendfilename "appendonly6379.aof"
replica-read-only yes #副本只读
replica-ignore-maxmemory yes
cluster-replica-validity-factor 0 #保证副本始终可以进行故障转移
cluster-node-timeout 20000 #节点连接超时时间
cluster-announce-ip 192.168.231.50 #集群节点的ip，当前节点的ip
cluster-announce-port 6379 #集群节点映射端口
cluster-announce-bus-port 16379 #集群节点总线端口,节点之间互相通信，常规端口+1万
```
3. 启动服务
分别启动6个服务

```shell
./bin/redis-server ./redis_6379.conf
./bin/redis-server ./redis_6479.conf
./bin/redis-server ./redis_6380.conf
./bin/redis-server ./redis_6480.conf
./bin/redis-server ./redis_6381.conf
./bin/redis-server ./redis_6481.conf
```

4. 初始化集群
```shell
./bin/redis-cli --cluster create 127.0.0.1:6379 127.0.0.1:6479 127.0.0.1:6380 127.0.0.1:6480 127.0.0.1:6381 127.0.0.1:6481 --cluster-replicas 1

>>> Performing hash slots allocation on 6 nodes...
Master[0] -> Slots 0 - 5460
Master[1] -> Slots 5461 - 10922
Master[2] -> Slots 10923 - 16383
Adding replica 127.0.0.1:6381 to 127.0.0.1:6379
Adding replica 127.0.0.1:6481 to 127.0.0.1:6479
Adding replica 127.0.0.1:6480 to 127.0.0.1:6380
>>> Trying to optimize slaves allocation for anti-affinity
[WARNING] Some slaves are in the same host as their master
M: 0a4eec34984a5666d763b8f7787e8d24c2f15ebd 127.0.0.1:6379
   slots:[0-5460] (5461 slots) master
M: 76e5424e486d9fe4eb6d4ece74ed7b6d0e68cbe8 127.0.0.1:6479
   slots:[5461-10922] (5462 slots) master
M: 23a461b6fbc08a4c31fc6fc1d25d94a651234b40 127.0.0.1:6380
   slots:[10923-16383] (5461 slots) master
S: a1aaedcf4fb2c7a4976561fee20bf35702b1e991 127.0.0.1:6480
   replicates 76e5424e486d9fe4eb6d4ece74ed7b6d0e68cbe8
S: 662ca6e92f01b1b13956c3b425b4a52023d924b6 127.0.0.1:6381
   replicates 23a461b6fbc08a4c31fc6fc1d25d94a651234b40
S: 1e555bcf52c6f178f82a5d3b0a668bc9c5c1bbeb 127.0.0.1:6481
   replicates 0a4eec34984a5666d763b8f7787e8d24c2f15ebd
Can I set the above configuration? (type 'yes' to accept): yes
>>> Nodes configuration updated
>>> Assign a different config epoch to each node
>>> Sending CLUSTER MEET messages to join the cluster
Waiting for the cluster to join
.........
>>> Performing Cluster Check (using node 127.0.0.1:6379)
M: 0a4eec34984a5666d763b8f7787e8d24c2f15ebd 127.0.0.1:6379
   slots:[0-5460] (5461 slots) master
   1 additional replica(s)
S: a1aaedcf4fb2c7a4976561fee20bf35702b1e991 127.0.0.1:6480
   slots: (0 slots) slave
   replicates 76e5424e486d9fe4eb6d4ece74ed7b6d0e68cbe8
M: 23a461b6fbc08a4c31fc6fc1d25d94a651234b40 127.0.0.1:6380
   slots:[10923-16383] (5461 slots) master
   1 additional replica(s)
M: 76e5424e486d9fe4eb6d4ece74ed7b6d0e68cbe8 127.0.0.1:6479
   slots:[5461-10922] (5462 slots) master
   1 additional replica(s)
S: 662ca6e92f01b1b13956c3b425b4a52023d924b6 127.0.0.1:6381
   slots: (0 slots) slave
   replicates 23a461b6fbc08a4c31fc6fc1d25d94a651234b40
S: 1e555bcf52c6f178f82a5d3b0a668bc9c5c1bbeb 127.0.0.1:6481
   slots: (0 slots) slave
   replicates 0a4eec34984a5666d763b8f7787e8d24c2f15ebd
[OK] All nodes agree about slots configuration.
>>> Check for open slots...
>>> Check slots coverage...
[OK] All 16384 slots covered.
```
5. 加入集群
```shell
bin/redis-cli --cluster add-node 192.168.231.27:6481 192.168.231.26:6479 --cluster-slave --cluster-master-id 8dc338bd22112cb3489374f582545f259ab10670 -a "123456"
```

6. 例子
```shell
[root@redis ]# ./bin/redis-cli -c -p 6379
127.0.0.1:6379> set foo bar
-> Redirected to slot [12182] located at 127.0.0.1:6380
OK
127.0.0.1:6380> set hello world
-> Redirected to slot [866] located at 127.0.0.1:6379
OK
127.0.0.1:6379> get foo
-> Redirected to slot [12182] located at 127.0.0.1:6380
"bar"
```


# 性能压测

## 平台
1. 平台：拟态云平台虚拟机
2. 操作系统：CentOS-7
3. 规格：4C(cpu)-16GB(内存)-200GB(硬盘)
4. 
## 压测方式
* redis-benchmark

## 性能压测对比
### 所有命令（非pipeline）

1. cluster集群模式
1000000个请求，1000个客户端
```shell
bin/redis-benchmark -n 1000000 -c 1000  -q -h 192.168.231.26 -p 6479 -a "123456" --cluster
```

```shell
PING_INLINE: 63000.06 requests per second, p50=15.111 msec                      
PING_MBULK: 60125.06 requests per second, p50=15.487 msec                       
SET: 63031.83 requests per second, p50=14.791 msec                     
GET: 62007.81 requests per second, p50=15.143 msec                     
INCR: 63047.73 requests per second, p50=14.815 msec                     
LPUSH: 62936.62 requests per second, p50=14.719 msec                     
RPUSH: 63043.75 requests per second, p50=14.879 msec                     
LPOP: 62968.33 requests per second, p50=14.999 msec                     
RPOP: 62042.44 requests per second, p50=15.135 msec                     
SADD: 61139.64 requests per second, p50=15.135 msec                        
HSET: 63031.83 requests per second, p50=15.071 msec                      
SPOP: 61020.26 requests per second, p50=15.471 msec                      
ZADD: 61109.75 requests per second, p50=15.319 msec                     
ZPOPMIN: 62023.20 requests per second, p50=15.047 msec                     
LPUSH (needed to benchmark LRANGE): 62952.47 requests per second, p50=14.967 msec                     
LRANGE_100 (first 100 elements): 51743.77 requests per second, p50=15.135 msec                     
LRANGE_300 (first 300 elements): 26768.75 requests per second, p50=23.311 msec                       
LRANGE_500 (first 500 elements): 18904.31 requests per second, p50=25.295 msec                    
LRANGE_600 (first 600 elements): 16942.84 requests per second, p50=26.783 msec                      
MSET (10 keys): 57494.39 requests per second, p50=14.087 msec
```
2. master
1000000个请求，1000个客户端
```shell
bin/redis-benchmark -n 1000000 -c 1000  -q -h 192.168.231.44 -p 6379 -a "123456"
```

```shell
PING_INLINE: 45429.77 requests per second, p50=17.199 msec                    
PING_MBULK: 48447.26 requests per second, p50=16.607 msec                    
SET: 49637.64 requests per second, p50=16.023 msec                    
GET: 49195.65 requests per second, p50=15.927 msec                    
INCR: 46541.93 requests per second, p50=16.703 msec                    
LPUSH: 47281.32 requests per second, p50=16.399 msec                    
RPUSH: 47890.43 requests per second, p50=16.063 msec                    
LPOP: 46561.44 requests per second, p50=16.863 msec                    
RPOP: 46317.74 requests per second, p50=16.783 msec                    
SADD: 46935.14 requests per second, p50=16.527 msec                    
HSET: 46095.70 requests per second, p50=16.287 msec                    
SPOP: 47323.84 requests per second, p50=15.791 msec                    
ZADD: 47812.57 requests per second, p50=16.063 msec                    
ZPOPMIN: 47005.73 requests per second, p50=15.903 msec                    
LPUSH (needed to benchmark LRANGE): 47963.93 requests per second, p50=16.479 msec                    
LRANGE_100 (first 100 elements): 31234.38 requests per second, p50=19.455 msec                    
LRANGE_300 (first 300 elements): 14019.54 requests per second, p50=36.863 msec                    
LRANGE_500 (first 500 elements): 10744.37 requests per second, p50=47.839 msec                    
LRANGE_600 (first 600 elements): 9551.92 requests per second, p50=53.759 msec                     
MSET (10 keys): 46431.72 requests per second, p50=17.071 msec
```
3. master,slave主从复制
1000000个请求，1000个客户端
```shell
bin/redis-benchmark -n 1000000 -c 1000  -q -h {master_ip} -p 6379 -a "123456" 
```
```shell
PING_INLINE: 67217.85 requests per second, p50=7.039 msec                     
PING_MBULK: 65023.73 requests per second, p50=7.311 msec                    
SET: 66769.05 requests per second, p50=7.071 msec                    
GET: 69338.51 requests per second, p50=6.919 msec                   
INCR: 65223.06 requests per second, p50=7.207 msec                   
LPUSH: 67078.08 requests per second, p50=7.191 msec                    
RPUSH: 69847.03 requests per second, p50=6.911 msec                   
LPOP: 68436.90 requests per second, p50=6.951 msec                   
RPOP: 66115.70 requests per second, p50=7.183 msec                   
SADD: 66992.70 requests per second, p50=7.015 msec                   
HSET: 70932.05 requests per second, p50=6.799 msec                   
SPOP: 68535.40 requests per second, p50=6.855 msec                    
ZADD: 65854.46 requests per second, p50=7.119 msec                    
ZPOPMIN: 67976.34 requests per second, p50=6.967 msec                   
LPUSH (needed to benchmark LRANGE): 69691.27 requests per second, p50=6.911 msec                     
LRANGE_100 (first 100 elements): 36782.29 requests per second, p50=12.903 msec                    
LRANGE_300 (first 300 elements): 16885.62 requests per second, p50=28.767 msec                    
LRANGE_500 (first 500 elements): 12229.27 requests per second, p50=40.479 msec                    
LRANGE_600 (first 600 elements): 10702.28 requests per second, p50=46.079 msec                    
MSET (10 keys): 63431.65 requests per second, p50=12.207 msec
```

### 所有命令（pipeline）
1. cluster集群模式
1000000个请求，1000个客户端,16个管道
```shell
bin/redis-benchmark -n 1000000 -c 1000 -P 16 -q -h 192.168.231.26 -p 6479 -a "123456" --cluster
```

```shell
PING_INLINE: 787401.56 requests per second, p50=14.575 msec                      
PING_MBULK: 782472.62 requests per second, p50=13.983 msec                      
SET: 655308.00 requests per second, p50=17.839 msec                      
GET: 655308.00 requests per second, p50=12.159 msec                      
INCR: 656598.81 requests per second, p50=16.719 msec                      
LPUSH: 657462.19 requests per second, p50=17.327 msec                      
RPUSH: 655737.69 requests per second, p50=16.847 msec                      
LPOP: 656598.81 requests per second, p50=12.527 msec                      
RPOP: 657030.25 requests per second, p50=14.535 msec                      
SADD: 561482.31 requests per second, p50=15.127 msec                      
HSET: 654450.25 requests per second, p50=18.399 msec                      
SPOP: 781250.00 requests per second, p50=14.847 msec                      
ZADD: 786782.06 requests per second, p50=16.735 msec                      
ZPOPMIN: 651890.44 requests per second, p50=16.895 msec                      
LPUSH (needed to benchmark LRANGE): 653594.81 requests per second, p50=16.399 msec                      
LRANGE_100 (first 100 elements): 106655.29 requests per second, p50=59.359 msec                      
LRANGE_300 (first 300 elements): 34306.49 requests per second, p50=174.719 msec                     
LRANGE_500 (first 500 elements): 21600.61 requests per second, p50=213.375 msec                     
LRANGE_600 (first 600 elements): 18466.54 requests per second, p50=225.791 msec                      
MSET (10 keys): 198019.80 requests per second, p50=56.511 msec
```

2. master
```shell
bin/redis-benchmark -n 1000000 -c 1000 -P 16 -q -h 192.168.231.44 -p 6379 -a "123456"
```

```shell
PING_INLINE: 632911.38 requests per second, p50=17.247 msec                     
PING_MBULK: 566251.38 requests per second, p50=20.895 msec                     
SET: 539083.56 requests per second, p50=24.815 msec                     
GET: 589622.62 requests per second, p50=19.567 msec                     
INCR: 566893.38 requests per second, p50=19.471 msec                     
LPUSH: 493339.94 requests per second, p50=29.183 msec                     
RPUSH: 555555.56 requests per second, p50=21.615 msec                     
LPOP: 588581.50 requests per second, p50=24.239 msec                     
RPOP: 506329.09 requests per second, p50=23.503 msec                     
SADD: 589970.50 requests per second, p50=22.271 msec                     
HSET: 581057.56 requests per second, p50=22.287 msec                     
SPOP: 674763.81 requests per second, p50=18.591 msec                     
ZADD: 557103.06 requests per second, p50=24.255 msec                     
ZPOPMIN: 674763.81 requests per second, p50=17.871 msec                     
LPUSH (needed to benchmark LRANGE): 498504.47 requests per second, p50=25.455 msec                     
LRANGE_100 (first 100 elements): 80231.07 requests per second, p50=102.975 msec                     
LRANGE_300 (first 300 elements): 22079.93 requests per second, p50=311.807 msec                     
LRANGE_500 (first 500 elements): 12805.74 requests per second, p50=476.415 msec                      
LRANGE_600 (first 600 elements): 11554.42 requests per second, p50=489.983 msec                      
MSET (10 keys): 233808.75 requests per second, p50=61.855 msec
```

3. master,slave主从复制
1000000个请求，1000个客户端
```shell
bin/redis-benchmark -n 1000000 -c 1000 -P 16 -q -h {master_ip} -p 6379 -a "123456"
```
```shell
PING_INLINE: 839630.56 requests per second, p50=14.487 msec                     
PING_MBULK: 834724.56 requests per second, p50=9.943 msec                      
SET: 520291.34 requests per second, p50=27.727 msec                     
GET: 904159.19 requests per second, p50=14.423 msec                     
INCR: 678886.62 requests per second, p50=20.223 msec                     
LPUSH: 490918.03 requests per second, p50=29.695 msec                     
RPUSH: 530504.00 requests per second, p50=27.647 msec                     
LPOP: 531914.88 requests per second, p50=27.535 msec                     
RPOP: 579710.12 requests per second, p50=24.319 msec                     
SADD: 887311.44 requests per second, p50=14.903 msec                     
HSET: 510725.25 requests per second, p50=29.103 msec                     
SPOP: 926784.06 requests per second, p50=13.975 msec                     
ZADD: 668449.19 requests per second, p50=20.575 msec                     
ZPOPMIN: 990099.00 requests per second, p50=11.519 msec                      
LPUSH (needed to benchmark LRANGE): 475511.19 requests per second, p50=31.359 msec                     
LRANGE_100 (first 100 elements): 86971.65 requests per second, p50=92.415 msec                      
LRANGE_300 (first 300 elements): 22012.37 requests per second, p50=357.631 msec                     
LRANGE_500 (first 500 elements): 15302.69 requests per second, p50=364.543 msec                     
LRANGE_600 (first 600 elements): 11572.47 requests per second, p50=594.943 msec                     
MSET (10 keys): 141242.94 requests per second, p50=115.327 msec 
```

### 缓存命中（非pipeline）
1.cluster集群模式
运行一百万次SET操作，每次操作从10万个key里面随机选一个
```shell
bin/redis-benchmark -n 1000000 -r 100000 -t set -q -h 192.168.231.26 -p 6479 -a "123456" --cluster
```

```shell
SET: 51706.31 requests per second, p50=0.815 msec
```

2. master
运行一百万次SET操作，每次操作从10万个key里面随机选一个
```shell
bin/redis-benchmark -n 1000000 -r 100000 -t set -q -h 192.168.231.44 -p 6379 -a "123456"
```

```shell
SET: 34939.38 requests per second, p50=1.183 msec
```

3. master,slave主从复制
1000000个请求，1000个客户端
```shell
bin/redis-benchmark -n 1000000 -r 100000 -t set -q -h {master_ip} -p 6379 -a "123456"
```
```shell
SET: 87009.48 requests per second, p50=0.359 msec
```


### 缓存命中（pipeline）
1.cluster集群模式
运行一百万次SET操作，每次操作从10万个key里面随机选一个
```shell
bin/redis-benchmark -n 1000000 -r 100000 -t set -P 16 -q -h 192.168.231.26 -p 6479 -a "123456" --cluster
```

```shell
SET: 567859.19 requests per second, p50=0.991 msec 
```

2. master
运行一百万次SET操作，每次操作从10万个key里面随机选一个
```shell
bin/redis-benchmark -n 1000000 -r 100000 -t set -P 16 -q -h 192.168.231.44 -p 6379 -a "123456"
```

```shell
SET: 411861.62 requests per second, p50=1.543 msec
```
3. master,slave主从复制
```shell
bin/redis-benchmark -n 1000000 -r 100000 -t set -P 16 -q -h {master_ip} -p 6379 -a "123456"
```

```shell
SET: 496277.88 requests per second, p50=1.487 msec
```
