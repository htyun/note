# 新建局域网段
> docker network create  --subnet=10.8.0.0/16 mysql-net

# 新建my.cnf
```
[mysqld]
default_authentication_plugin=mysql_native_password
log-bin = mysql-bin-s # 标记修改
server_id = 31 # 每台单独容器id不能相同
datadir = /var/lib/mysql
sql_mode=STRICT_TRANS_TABLES,NO_ENGINE_SUBSTITUTION
```

# 新建docker-compose.yml文件，内容：
```yaml
version: "3"
services:
 mysql8-33:
#  network_mode: "mysql-net"
  networks:
   mysql-net:
    ipv4_address: 10.8.0.33
  image: mysql
  restart: always
  container_name: mysql8-33
  hostname: mysql8-33
  ports:
   - 3308:3306
  volumes:
   - /tools/docker/mysql8/data_33//:/var/lib/mysql
   - /tools/docker/mysql8/conf/my_33.cnf:/etc/my.cnf
  environment:
   MYSQL_ROOT_PASSWORD: password # root密码
   MYSQL_USER: admin   #创建admin用户
   MYSQL_PASSWORD: password  #设置admin用户的密码
  command: mysqld --character-set-server=utf8 --collation-server=utf8_unicode_ci #设置utf8字符集

 mysql8-31:
#  network_mode: "mysql-net"
  networks:
   mysql-net:
    ipv4_address: 10.8.0.31
  image: mysql
  restart: always
  hostname: mysql8-31
  container_name: mysql8-31
  ports:
   - 3108:3306
  volumes:
   - /tools/docker/mysql8/data_31//:/var/lib/mysql
   - /tools/docker/mysql8/conf/my_31.cnf:/etc/my.cnf
  environment:
   MYSQL_ROOT_PASSWORD: password # root密码
   MYSQL_USER: admin   #创建admin用户
   MYSQL_PASSWORD: password  #设置admin用户的密码
  command: mysqld --character-set-server=utf8 --collation-server=utf8_unicode_ci #设置utf8字符集

networks:
 mysql-net:
  ipam:
   config:
    - subnet: 10.8.0.0/16

#networks:
# mysql-net:
#  external: true 外部，不创建
```

# 构建集群配置
- 进入主节点docker exec -it mysql8-33 mysql -uroot -pht  
  - use mysql 
  - 修改密码模式  
  -  ALTER USER 'root'@'%' IDENTIFIED WITH mysql_native_password BY 'ht';  
  - flush privileges;  
  - show master status; 查看主节点状态
- 进入备节点docker exec -it mysql8-31 mysql -uroot -pht
  - use mysql 修改密码模式  
  -  ALTER USER 'root'@'%' IDENTIFIED WITH mysql_native_password BY 'ht';  
  - change master to master_host='10.8.0.33',master_user='root',master_password='ht',**master_log_file='mysql-bin-33m.000003'**,**master_log_pos=155**,master_port=3306;
  - **master_log_file和master_log_pos在主节点状态里面复制出来**
  - start slave;
 


## mysql -e 直接执行sql