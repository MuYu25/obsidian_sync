# 配置信息
## docker compose.yaml
```yaml
version: '3.8'

services:
  mysql-master:
    image: docker.1ms.run/mysql:8.0
    container_name: mysql-server-master
    restart: unless-stopped
    environment:
      MYSQL_ROOT_PASSWORD: '123456'
      MYSQL_DATABASE: mydb
    ports:
      - "3306:3306"
    volumes:
      - ./master_data:/var/lib/mysql
      - ./mysql-master/conf.d:/etc/mysql/conf.d
    networks:
      - app-network

  mysql-slave:
   image: docker.1ms.run/mysql:8.0
   container_name: mysql-server-slave
   restart: unless-stopped
   environment:
     MYSQL_ROOT_PASSWORD: '123456'
     MYSQL_DATABASE: mydb
   ports:
     - "3307:3306"
   volumes:
     - ./slave_data:/var/lib/mysql
     - ./mysql-slave/conf.d:/etc/mysql/conf.d
   depends_on:
     - mysql-master
   networks:
     - app-network

  redis:
    image: docker.1ms.run/redis:8.0
    container_name: redis-server
    restart: unless-stopped
    ports:
      - "6379:6379"
    volumes:
      - redis_data:/data
      - ./redis/redis.conf:/usr/local/etc/redis/redis.conf
    command: redis-server /usr/local/etc/redis/redis.conf
    networks:
      - app-network

  mongo:
    image: docker.1ms.run/mongo:8
    container_name: mongo-server
    restart: unless-stopped
    environment:
      MONGO_INITDB_DATABASE: mydb
    ports:
      - "27017:27017"
    volumes:
      - mongo_data:/data/db
      - ./mongo/init.js:/docker-entrypoint-initdb.d/init.js
    networks:
      - app-network
volumes:
  mysql_data:
  redis_data:
  mongo_data:

networks:
  app-network:
    driver: bridge
```

## mysql-master
文件位置均是自己在docker compose文件中自行配置
- 宿主机配置文件位置: ./mysql-master/conf.d
- 宿主机日志文件位置: ./master_data
- 虚拟机配置文件位置: /etc/mysql/conf.d
- 虚拟机日志文件位置: /var/lib/mysql
### master.cnf
当时有出现一些问题是，日志文件夹下已经存在二进制日志，并且名称为binlog.000003。故log_bin修改为binlog
```bash
[mysqld]
server-id               = 1                 # 唯一 ID
log_bin                 = binlog # 开启二进制日志
binlog_do_db            = mydb     # 可选：只复制特定数据库
skip_slave_start        = 1                 # 防止启动时自动开启复制
bind-address            = 0.0.0.0           # 允许外部连接
```

## slave-master
文件与mysql-master差不多，详细可以看docker compose 文件
### slave.cnf
```bash
[mysqld]
server-id               = 2                 # 唯一 ID，不能与 Master 相同
replicate_do_db         = mydb     # 可选：只复制特定数据库
skip_slave_start        = 1                 # 防止启动时自动开启复制
bind-address            = 0.0.0.0
```

# 操作
## 配置主从同步（MySQL 命令）
- `mysql-server-master` 是docker运行时容器的名称，我在docker compose 中设置为`mysql-server-master`.
- `123456`是我`mysql-server-master`中`root`用户的密码
### master执行操作
```bash
sudo docker exec -it mysql-server-master mysql -u root -p123456
```
#### 创建复制用户
- `123456`是我`mysql-server-master`中`repl_user`用户的密码
```SQL
-- 1. 创建复制用户
CREATE USER 'repl_user'@'%' IDENTIFIED BY '123456';

-- 2. 授予复制权限
GRANT REPLICATION SLAVE ON *.* TO 'repl_user'@'%';

-- 3. 刷新权限
FLUSH PRIVILEGES;
```

#### 查看 Master 状态（获取同步点）
这一步至关重要，它提供了 Slave 连接 Master 所需的起始点。
```SQL
-- 4. 锁定表（防止数据变动影响同步点）
FLUSH TABLES WITH READ LOCK;

-- 5. 查看 Master 状态
SHOW MASTER STATUS;
```
**记下输出结果中的两个值：**
- `File` (例如：`mysql-bin.000001`)
- `Position` (例如：`688`)

我当时的值如下:

|      File      | Position | Binlog_Do_DB | Binlog_Ignore_DB | Executed_Gtid_Set |
| :------------: | :------: | :----------: | :--------------: | :---------------: |
| binglog.000004 |   836    |     mydb     |                  |                   |

执行完后，解锁表：

```SQL
-- 6. 解锁表
UNLOCK TABLES;
```

### slave执行操作
```bash
docker exec -it mysql-server-slave mysql -u root -p123456
```
#### #### 设置复制连接信息
使用在 Master 步骤中获取的信息（File 和 Position）来设置 Slave 连接 Master 的参数。
- `binglog.000004`是我在`master`上获取到的，实际填入情况是自己`master`返回的file.
- `836`也是我在`master`上获取到的，实际填入情况是自己`master`返回的`Position`.
```SQL
-- 1. 停止 Slave 进程 
STOP SLAVE;

-- 2. 配置 Master 连接信息 -- 替换 MasterLogFile 和 MasterLogPos 为你在 Master 上查到的值 
CHANGE MASTER TO MASTER_HOST='mysql-server-master', -- Master 容器的服务名（通过 Docker 网络互通） 
-- 若不是使用docker network进行链接，则需要使用 **Master 宿主机 A 的实际 IP 地址**。
MASTER_USER='repl_user', MASTER_PASSWORD='your_replication_password', MASTER_LOG_FILE='binglog.000004', -- 替换为 Master 的 File
MASTER_LOG_POS=836, -- 替换为 Master 的 Position
GET_MASTER_PUBLIC_KEY=1;
```
#### 启动 Slave 进程并验证

```SQL
-- 3. 启动 Slave 进程
START SLAVE;

-- 4. 查看 Slave 状态，确认是否成功
SHOW SLAVE STATUS\G
```

#### 检查验证

在 Slave 上执行 `SHOW SLAVE STATUS\G` 后，重点检查以下两项：
- `Slave_IO_Running`: **Yes**
- `Slave_SQL_Running`: **Yes**
- `Seconds_Behind_Master`: **0** 或一个非常小的数字
若结果如上所示，那么恭喜，已经设置完成
# 问题排查
## 同步失败
```bash
Slave_IO_Running: Connecting
Slave_SQL_Running: Yes
Seconds_Behind_Master: NULL
```
### 可能情况1 docker network
两个容器不在同一个`docker network`中，排查方法
```bash
sudo docker network ls
```
查看`docker compose`中自己创建相同名称的`network`。我的如下:
```bash
sudo docker network ls
NETWORK ID     NAME                   DRIVER    SCOPE
1e09fb29ff2e   bridge                 bridge    local
9027af43c605   hl202417_app-network   bridge    local
60e6709c7708   host                   host      local
f03b49ba2da7   none                   null      local
```
依旧经验而谈，我锁定了`hl202417_app-network`
```bash
sudo docker network inspect hl202417_app-network
```
```json
"Containers": {
            "80af39073a647a6f9ea9379c29d18692b97c4a24d9786898b306af55d281dbee": {
                "Name": "mysql-server-master",
                "EndpointID": "d2cedd2f0f31df68a934f236e2e70b985348893874b8e40c35162ba2975c38ae",
                "MacAddress": "12:b0:04:01:a9:2f",
                "IPv4Address": "172.18.0.5/16",
                "IPv6Address": ""
            },
            "919f81debfbaa6c6f21fbad892d809d4bb6a80e56a955732b8d8ff29908295a9": {
                "Name": "mysql-server-slave",
                "EndpointID": "6bfa629ef08f072ec0d3f085e3dff4cfb284ddf58815ad83477531b22b062992",
                "MacAddress": "1e:11:7c:97:0f:35",
                "IPv4Address": "172.18.0.3/16",
                "IPv6Address": ""
            }
		}
```
我在网络中发现 `master`与`slave` 处于统一网络下。故此问题不存在.

若发现`master`或`slave`不在`docker compose`所设置的网路中，请修改 `docker compose`文件.
### 可能情况2  master下用于同步的账户权限不足或用户不存在(我遇到的情况)
我是再执行了一下在`master`上创建账户的流程
#### master流程
```bash
sudo docker exec -it mysql-server-master mysql -u root -p123456
```
修改密码并且修改权限:
```SQL
-- 1. 确保用户使用兼容的认证插件并设置新密码（使用您想要的密码替换 'YOUR_NEW_PASSWORD'）
ALTER USER 'repl_user'@'%' IDENTIFIED WITH mysql_native_password BY 'YOUR_NEW_PASSWORD';

-- 2. 确保用户有正确的复制权限
GRANT REPLICATION SLAVE ON *.* TO 'repl_user'@'%';

-- 3. 刷新权限
FLUSH PRIVILEGES;

-- 4. 退出 Master 客户端
QUIT;
```
记录同步点:
```SQL
-- 4. 锁定表（防止数据变动影响同步点）
FLUSH TABLES WITH READ LOCK;

-- 5. 查看 Master 状态
SHOW MASTER STATUS;
```
#### slave流程
进入容器:
```bash
sudo docker exec -it mysql-server-slave mysql -u root -p123456
```
**重新配置 Master 连接：** 确保密码正确，并使用正确的日志点。
```SQL
CHANGE MASTER TO 
  MASTER_HOST='mysql-server-master',
-- 若不是使用docker network进行链接，则需要使用 **Master 宿主机 A 的实际 IP 地址**。
  MASTER_USER='repl_user',
  MASTER_PASSWORD='YOUR_NEW_PASSWORD',  -- <<< 使用新密码
  MASTER_LOG_FILE='binlog.000003',      -- 确保 File 和 Pos 正确
  MASTER_LOG_POS=324,
  GET_MASTER_PUBLIC_KEY=1;
```
启动复制并且验证：
```SQL
START REPLICA;

SHOW SLAVE STATUS\G
```
确认 `Slave_IO_Running: Yes` 和 `Slave_SQL_Running: Yes`。
# 添加多线程的数据复制
## slave设置
slave.cnf
```bash
[mysqld]
server-id               = 2                 # 唯一 ID，不能与 Master 相同
replicate_do_db         = mydb     # 可选：只复制特定数据库
skip_slave_start        = 1                 # 防止启动时自动开启复制
bind-address            = 0.0.0.0
slave_parallel_type     = LOGICAL_CLOCK     # 基于逻辑时钟并行应用事务
slave_parallel_workers  = 32                # 设置并行工作线程数，通常设置为CPU核心数的2倍
```
新增
```
slave_parallel_type     = LOGICAL_CLOCK     # 基于逻辑时钟并行应用事务
slave_parallel_workers  = 32                # 设置并行工作线程数，通常设置为CPU核心数的2倍
```
## 检查多线程工作状态
```SQL
SHOW GLOBAL VARIABLES LIKE 'slave_parallel_workers'; --显示当前 Slave 实例实际运行的并行线程数 
```
**预期结果：**
- `Variable_name`: `slave_parallel_workers`
- `Value`: **32** (或者您在配置文件中设置的任何值)
```SQL
SHOW GLOBAL VARIABLES LIKE 'slave_parallel_type';
```
**预期结果：**
- `Variable_name`: `slave_parallel_type`
- `Value`: **LOGICAL_CLOCK**
```SQL
SHOW PROCESSLIST;
```
**预期结果：**
您应该能看到多个进程，其 `Command` 字段显示为 **`Slave_worker`** 或 `Replica_worker`。进程的数量应该与您设置的 `slave_parallel_workers` (即 32 个) 接近。