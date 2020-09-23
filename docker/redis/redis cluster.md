# 下载文件
- wget -c http://download.redis.io/redis-stable/redis.conf

# 修改redis.conf文件
> 为每个配置集群redis配置一个端口  
> 修改bind为0.0.0.0  
> cluster-announce-ip 192.168.1.3 设置为内网IP，连接公网设置为公网IP  
> cluster-announce-port 1111 集群单个端口  
> cluster-announce-bus-port 11111 集群总线端口，一般为redis端口加10000  
> cluster-enabled yes 开放  
> cluster-node-timeout 15000 开放  
> cluster-config-file nodes.conf 开放  
> daemonize no  
> masterauth password 设置集群密码  
> requirepass password 设置登陆密码 ， 两个密码一致  
> appendonly yes 持久化可选  
> appendfsync 默认即可

# 创建redis专用内网
```
docker network create --subnet=10.79.0.0/16 redis-net
```

# 新建docker-compose.yml文件

```yaml
version: "3"
services:
 redis-81:
  networks:
   redis-net:
    ipv4_address: 10.79.0.81
  image: redis
  restart: always
  container_name: redis-81
  hostname: redis-81
  ports:
    - 8179:8179
    - 18179:18179
  volumes:
   - /tools/docker/redis/conf/8179.conf:/etc/redis/8179.conf
   - /tools/docker/redis/data_81/:/data
  command: redis-server /etc/redis/8179.conf
  
 redis-82:
  depends_on:
   - redis-81
  networks:
   redis-net:
    ipv4_address: 10.79.0.82
  image: redis
  restart: always
  container_name: redis-82
  hostname: redis-82
  ports:
    - 8279:8279
    - 18279:18279
  volumes:
   - /tools/docker/redis/conf/8279.conf:/etc/redis/8279.conf
   - /tools/docker/redis/data_82/:/data
  command: redis-server /etc/redis/8279.conf
  
 redis-83:
  depends_on:
   - redis-81
  networks:
   redis-net:
    ipv4_address: 10.79.0.83
  image: redis
  restart: always
  container_name: redis-83
  hostname: redis-83
  ports:
    - 8379:8379
    - 18379:18379
  volumes:
   - /tools/docker/redis/conf/8379.conf:/etc/redis/8379.conf
   - /tools/docker/redis/data_83/:/data
  command: redis-server /etc/redis/8379.conf

networks:
 redis-net:
  external: true
```
**注意：** 头对齐最好使用空格，不要使用table

# 构建集群
> 执行 docker-compose up -d  
> docker exec -it redis-81 bash进入容器  
> 执行redis-cli -a password --cluster  create  ip:port ip:port ip:port --cluster-replicas 1 构建集群     
> --cluster-replicas 1 为集群创建备节点，数目为1，也可以不使用

```
客户端命令：redis-cli -c -p port -h ip）
登录redis后，在里面可以进行下面命令操作

集群
cluster info ：打印集群的信息
cluster nodes ：列出集群当前已知的所有节点（ node），以及这些节点的相关信息。
节点
cluster meet <ip> <port> ：将 ip 和 port 所指定的节点添加到集群当中，让它成为集群的一份子。
cluster forget <node_id> ：从集群中移除 node_id 指定的节点。
cluster replicate <master_node_id> ：将当前从节点设置为 node_id 指定的master节点的slave节点。只能针对slave节点操作。
cluster saveconfig ：将节点的配置文件保存到硬盘里面。
槽(slot)
cluster addslots <slot> [slot ...] ：将一个或多个槽（ slot）指派（ assign）给当前节点。
cluster delslots <slot> [slot ...] ：移除一个或多个槽对当前节点的指派。
cluster flushslots ：移除指派给当前节点的所有槽，让当前节点变成一个没有指派任何槽的节点。
cluster setslot <slot> node <node_id> ：将槽 slot 指派给 node_id 指定的节点，如果槽已经指派给
另一个节点，那么先让另一个节点删除该槽>，然后再进行指派。
cluster setslot <slot> migrating <node_id> ：将本节点的槽 slot 迁移到 node_id 指定的节点中。
cluster setslot <slot> importing <node_id> ：从 node_id 指定的节点中导入槽 slot 到本节点。
cluster setslot <slot> stable ：取消对槽 slot 的导入（ import）或者迁移（ migrate）。
键
cluster keyslot <key> ：计算键 key 应该被放置在哪个槽上。
cluster countkeysinslot <slot> ：返回槽 slot 目前包含的键值对数量。
cluster getkeysinslot <slot> <count> ：返回 count 个 slot 槽中的键 。
```