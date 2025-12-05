# 配置信息
## docker compose 
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
