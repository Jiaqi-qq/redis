# Redis集群部署

---

## 规划

| 服务器 |      角色      |     IP     | 服务端口（默认6379） |
| :----: | :------------: | :--------: | :------------------: |
|   u1   | master1/slave2 | 172.17.0.3 |      7000/7001       |
|   u2   | master2/slave3 | 172.17.0.4 |      7000/7001       |
|   u3   | master3/slave1 | 172.17.0.5 |      7000/7001       |

---

## 1	docker创建三台ubuntu容器

```sh
# 准备三个容器
docker run -itd --name u1 ubuntu:20.04
docker run -itd --name u2 ubuntu:20.04
docker run -itd --name u3 ubuntu:20.04

# 分别进入3个容器
docker start u1 && docker attach u1

# 安装必要软件
apt update && apt install net-tools inetutils-ping vim  wget gcc make ntpdate -y

# 添加 hosts
cat >> /etc/hosts << EOF
>
> 172.17.0.3 u1
> 172.17.0.4 u2
> 172.17.0.5 u3
> EOF
```

---

## 2	下载安装

```sh
# 官网：https://redis.io/
# 1.下载编译安装
wget https://download.redis.io/releases/redis-6.2.6.tar.gz # 下载
tar -zxvf redis-6.2.6.tar.gz # 解压
cd redis-6.2.6 # 进入目录
make -j2 # 编译
make PREFIX=/opt/redis install # 安装
> redis-benchmark: 性能测试工具，可以运行看看性能
  redis-check-aof: 修复有问题的AOF文件
  redis-check-dump: 修复有问题的dump.rdb文件
  redis-sentinel: redis集群使用
  redis-server: redis服务器启动命令
  redis-cli: 客户端，操作入口

# 2.软连接
ln -s /opt/redis/bin/* /usr/local/bin/

# 3.拷贝conf文件到指定目录
mkdir -p /opt/redis/{conf,log,run,data/{7000,7001}}
cp /root/redis_dir/redis-6.2.6/redis.conf /opt/redis/conf/

# 4.修改conf参数（master1）
cd /opt/redis/conf
cp redis.conf redis_7000.conf
vi redis_7000.conf

bind 172.17.0.3 # 添加本机的ip
port 7000 # 端口　　
pidfile /opt/redis/run/redis_7000.pid # pid存储目录
logfile /opt/redis/log/redis_7000.log # 日志存储目录
dir /opt/redis/data/7000 # 数据存储目录，目录要提前创建好
cluster-enabled yes # 开启集群
cluster-config-file nodes-7000.conf # 集群节点配置文件，这个文件是不能手动编辑的。确保每一个集群节点的配置文件不通
cluster-node-timeout 15000 # 集群节点的超时时间，单位：ms，超时后集群会认为该节点失败
appendonly yes # 持久化
daemonize yes # 守护进程

# 继续修改7001（slave2）
cp redis_7000.conf redis_7001.conf
:%s/7000/7001 # 将每一行所有7000改为7001

# 其余机器重复上述操作

# 5.制作启动配置文件
# 启动脚本
cat > /opt/redis/bin/cluster_start.sh << EOF
./redis-server ../conf/redis_7000.conf
./redis-server ../conf/redis_7001.conf
EOF
chmod +x /opt/redis/bin/cluster_start.sh
# 关闭脚本
cat > /opt/redis/bin/cluster_shutdown.sh << EOF
pgrep redis-server | xargs -exec kill -9
EOF
chmod +x /opt/redis/bin/cluster_shutdown.sh

# 启动
cd /opt/redis/bin && ./cluster_start.sh
ps -ef | grep redis # 查看是否成功启动，不成功查看log日志
```

## 3	创建集群

```sh
# 创建集群
redis-cli --cluster create 172.17.0.3:7000 172.17.0.4:7000 172.17.0.5:7000 172.17.0.3:7001 172.17.0.4:7001 172.17.0.5:7001 --cluster-replicas 1

# 输出：
>>> Performing hash slots allocation on 6 nodes...
Master[0] -> Slots 0 - 5460
Master[1] -> Slots 5461 - 10922
Master[2] -> Slots 10923 - 16383
Adding replica 172.17.0.4:7001 to 172.17.0.3:7000
Adding replica 172.17.0.5:7001 to 172.17.0.4:7000
Adding replica 172.17.0.3:7001 to 172.17.0.5:7000
M: 11a3238c466a5c55e2345d8dfca7162b1a2f55ff 172.17.0.3:7000
   slots:[0-5460] (5461 slots) master
M: 70f6a5d0ff83b794f2a982edbfb9594164a72b08 172.17.0.4:7000
   slots:[5461-10922] (5462 slots) master
M: 49dc972b7e5f4822282d1f3ced3f6bb377377b0b 172.17.0.5:7000
   slots:[10923-16383] (5461 slots) master
S: 8e8d5980742f91c8f41884ddf5473da4424bab7c 172.17.0.3:7001
   replicates 49dc972b7e5f4822282d1f3ced3f6bb377377b0b
S: 2bccf1773748c305601eb2bd75f85fb1cba6cc96 172.17.0.4:7001
   replicates 11a3238c466a5c55e2345d8dfca7162b1a2f55ff
S: cd70d2d4e10c085d1978ed45fbc6a21037a5cec2 172.17.0.5:7001
   replicates 70f6a5d0ff83b794f2a982edbfb9594164a72b08
Can I set the above configuration? (type 'yes' to accept): yes
>>> Nodes configuration updated
>>> Assign a different config epoch to each node
>>> Sending CLUSTER MEET messages to join the cluster
Waiting for the cluster to join
.
>>> Performing Cluster Check (using node 172.17.0.3:7000)
M: 11a3238c466a5c55e2345d8dfca7162b1a2f55ff 172.17.0.3:7000
   slots:[0-5460] (5461 slots) master
   1 additional replica(s)
M: 70f6a5d0ff83b794f2a982edbfb9594164a72b08 172.17.0.4:7000
   slots:[5461-10922] (5462 slots) master
   1 additional replica(s)
S: 8e8d5980742f91c8f41884ddf5473da4424bab7c 172.17.0.3:7001
   slots: (0 slots) slave
   replicates 49dc972b7e5f4822282d1f3ced3f6bb377377b0b
S: 2bccf1773748c305601eb2bd75f85fb1cba6cc96 172.17.0.4:7001
   slots: (0 slots) slave
   replicates 11a3238c466a5c55e2345d8dfca7162b1a2f55ff
S: cd70d2d4e10c085d1978ed45fbc6a21037a5cec2 172.17.0.5:7001
   slots: (0 slots) slave
   replicates 70f6a5d0ff83b794f2a982edbfb9594164a72b08
M: 49dc972b7e5f4822282d1f3ced3f6bb377377b0b 172.17.0.5:7000
   slots:[10923-16383] (5461 slots) master
   1 additional replica(s)
[OK] All nodes agree about slots configuration.
>>> Check for open slots...
>>> Check slots coverage...
[OK] All 16384 slots covered.
# 注意看M和S，对照下集群角色表
```

## 4	查看集群状态

```sh
redis-cli -c -h 172.17.0.3 -p 7000 cluster info
# 输出：
cluster_state:ok
cluster_slots_assigned:16384
cluster_slots_ok:16384
cluster_slots_pfail:0
cluster_slots_fail:0
cluster_known_nodes:6
cluster_size:3
cluster_current_epoch:6
cluster_my_epoch:1
cluster_stats_messages_ping_sent:161
cluster_stats_messages_pong_sent:161
cluster_stats_messages_sent:322
cluster_stats_messages_ping_received:156
cluster_stats_messages_pong_received:161
cluster_stats_messages_meet_received:5
cluster_stats_messages_received:322
```

## 5	查看集群节点

```sh
redis-cli -c -h 172.17.0.3 -p 7000 cluster nodes
# 输出：
70f6a5d0ff83b794f2a982edbfb9594164a72b08 172.17.0.4:7000@17000 master - 0 1635774517000 2 connected 5461-10922
8e8d5980742f91c8f41884ddf5473da4424bab7c 172.17.0.3:7001@17001 slave 49dc972b7e5f4822282d1f3ced3f6bb377377b0b 0 1635774514000 3 connected
2bccf1773748c305601eb2bd75f85fb1cba6cc96 172.17.0.4:7001@17001 slave 11a3238c466a5c55e2345d8dfca7162b1a2f55ff 0 1635774517580 1 connected
11a3238c466a5c55e2345d8dfca7162b1a2f55ff 172.17.0.3:7000@17000 myself,master - 0 1635774515000 1 connected 0-5460
cd70d2d4e10c085d1978ed45fbc6a21037a5cec2 172.17.0.5:7001@17001 slave 70f6a5d0ff83b794f2a982edbfb9594164a72b08 0 1635774516576 2 connected
49dc972b7e5f4822282d1f3ced3f6bb377377b0b 172.17.0.5:7000@17000 master - 0 1635774515573 3 connected 10923-16383
```

## 6	测试

```sh
redis-cli -c -h 172.17.0.3 -p 7000
set name abc

redis-cli -c -h 172.17.0.4 -p 7000
get name
```

