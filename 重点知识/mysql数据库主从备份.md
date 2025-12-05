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

