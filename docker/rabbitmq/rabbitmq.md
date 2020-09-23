# 新建局域网段
> docker network create  --subnet=10.72.0.0/16  rabbitmq-net

# 新建haproxy.cfg
```
global
    log 127.0.0.1 local0
    maxconn 4096

defaults
    log     global
    mode    tcp
    option  tcplog
    retries 3
    option  redispatch
    maxconn 2000
    timeout connect 5000
    timeout client 50000
    timeout server 50000

# ssl for rabbitmq
# frontend ssl_rabbitmq
    # bind *:5673 ssl crt /root/rmqha_proxy/rmqha.pem
    # mode tcp
    # default_backend rabbitmq

listen stats
    bind *:1080 # haproxy容器1080端口显示代理统计页面，映射到宿主51080端口
    mode http
    stats enable
    stats hide-version
    stats realm Haproxy\ Statistics
    stats uri /
    stats auth root:ht

listen rabbitmq
    bind *:5672 # haproxy容器5672端口代理多个rabbitmq服务，映射到宿主56729端口
    mode tcp
    balance roundrobin
    timeout client 1h
    timeout server 1h
    option  clitcpka
    # server  rmqha_node0 rmqha_node0:5672  check inter 5s rise 2 fall 3
    server  rabbit-71 rabbit-71:5672  check inter 5s rise 2 fall 3
    server  rabbit-72 rabbit-72:5672  check inter 5s rise 2 fall 3
    server  rabbit-73 rabbit-73:5672  check inter 5s rise 2 fall 3
```
# 新建Dockerfile

```
from haproxy:1.8

copy haproxy.cfg /usr/local/etc/haproxy/haproxy.cfg

```

# 新建docker-compose.yml文件，内容：
```yaml
version: "3"
services:
  rabbit-71:
    image: rabbitmq:3.6-management
    container_name: rabbit-71
    restart: always
    networks:
      rabbitmq-net:
        ipv4_address: 10.72.0.71
    hostname: rabbit-71
    ports:
      - 7172:15672
      - 7156:5672
    environment:
      - RABBITMQ_DEFAULT_USER=root
      - RABBITMQ_DEFAULT_PASS=ht
      - RABBITMQ_ERLANG_COOKIE=rabbit-cookie
#      - RABBITMQ_HOSTNAME=rabbit-71
#      - RABBITMQ_NODENAME=rabbit-71
    volumes:
#      - /tools/docker/rabbitmq/etc/rabbitmq:/etc/rabbitmq
      - /tools/docker/rabbitmq/lib/rabbitmq:/var/lib/rabbitmq
      - /tools/docker/rabbitmq/log/rabbitmq/:/var/log/rabbitmq
      
  rabbit-72:
    image: rabbitmq:3.6-management
    container_name: rabbit-72
    restart: always
    links:
     - rabbit-71
    depends_on:
     - rabbit-71
    networks:
      rabbitmq-net:
        ipv4_address: 10.72.0.72
    hostname: rabbit-72
    ports:
      - 7272:15672
      - 7256:5672
    environment:
      - RABBITMQ_DEFAULT_USER=root
      - RABBITMQ_DEFAULT_PASS=ht
      - RABBITMQ_ERLANG_COOKIE=rabbit-cookie
#      - RABBITMQ_HOSTNAME=rabbit-72
#      - RABBITMQ_NODENAME=rabbit-72
      - RMQHA_RAM_NODE=true
    volumes:
#      - /tools/docker/rabbitmq/etc/rabbitmq:/etc/rabbitmq
      - /tools/docker/rabbitmq/lib/rabbitmq:/var/lib/rabbitmq
      - /tools/docker/rabbitmq/log/rabbitmq/:/var/log/rabbitmq

  rabbit-73:
    image: rabbitmq:3.6-management
    container_name: rabbit-73
    restart: always
    links:
     - rabbit-72
     - rabbit-71
    networks:
      rabbitmq-net:
        ipv4_address: 10.72.0.73
    depends_on:
     - rabbit-71
    hostname: rabbit-73
    ports:
      - 7372:15672
      - 7356:5672
    environment:
      - RABBITMQ_DEFAULT_USER=root
      - RABBITMQ_DEFAULT_PASS=ht
      - RABBITMQ_ERLANG_COOKIE=rabbit-cookie
#      - RABBITMQ_HOSTNAME=rabbit-73
#      - RABBITMQ_NODENAME=rabbit-73
      - RMQHA_RAM_NODE=true
    volumes:
#config      - /tools/docker/rabbitmq/etc/rabbitmq:/etc/rabbitmq
      - /tools/docker/rabbitmq/lib/rabbitmq:/var/lib/rabbitmq
      - /tools/docker/rabbitmq/log/rabbitmq/:/var/log/rabbitmq
  
  rabbit-haproxy:
    build:
     context: .
     dockerfile: Dockerfile
    image: ht/haproxy
    container_name: rabbit-haproxy
    restart: always
    networks:
      rabbitmq-net:
        ipv4_address: 10.72.0.77
    depends_on:
     - rabbit-71
     - rabbit-72
     - rabbit-73
    hostname: rabbit-haproxy
    ports:
      - 7772:5672
      - 7710:1080
    volumes:
      - /tools/docker/rabbitmq/haproxy.cfg:/usr/local/etc/haproxy/haproxy.cfg
    environment:
      - CONTAINER_NAME=rabbit-haproxy

networks:
 rabbitmq-net:
  external: true
```

# 构建集群配置
> 分别进入rabbit-71，rabbit-72，rabbit-73执行以下命令
```
 docker exec -it rabbit-71 bash  
 rabbitmqctl stop_app  
 rabbitmqctl reset  
 rabbitmqctl join_cluster rabbit@rabbit-71  # rabbit-71不执行  


```