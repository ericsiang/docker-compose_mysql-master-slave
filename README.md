---
title: Mysql 主從同步 (使用docker-compose)
date: 2022-03-07
categories:
- mysql  docker
tags:
- mysql docker
---


# Mysql 主從同步 (使用docker)

>製作者：Ericsiang
###### tags:  `mysql`  `docker`

#### 參考: 
[使用Docker配置MySQL主从数据库](https://www.modb.pro/db/81707)
[docker-compose搭建MySQL主从复制集群](https://juejin.cn/post/6844904136115388423)

### 第一步 : 設定master跟slave的docker-compose.yaml
master的docker-compose.yaml
```
version: '3.8'


services:
  master:
    image: mysql:5.7.31
    restart: always
    privileged: true
    ports:
      - 3307:3306
    volumes:
      - ./conf:/etc/mysql
      - ./data:/var/lib/mysql
      - ./log:/var/log/mysql/
    environment:
      MYSQL_ROOT_PASSWORD: "root"
    command: [
        '--character-set-server=utf8mb4',
        '--collation-server=utf8mb4_general_ci',
        '--max_connections=3000'
    ]
    networks:
      - master_slave  

networks:
  master_slave:
    driver: bridge 
    name: master_slave
```

slave的docker-compose.yaml
```
version: '3.8'


services:

  slave:
    image: mysql:5.7.31
    restart: always
    privileged: true
    ports:
      - 3308:3306
    volumes:
      - ./conf:/etc/mysql
      - ./data:/var/lib/mysql
      - ./log:/var/log/mysql/
    environment:
      MYSQL_ROOT_PASSWORD: "root"
    command: [
        '--character-set-server=utf8mb4',
        '--collation-server=utf8mb4_general_ci',
        '--max_connections=3000'
    ]
    networks:
      - master_slave  

networks:
  master_slave:
    driver: bridge 
    name: master_slave
```

**如會在同一台server上，也可以寫成一份docker-compose.yaml**
```
version: '3.8'

services:
  master:
    image: mysql:5.7.31
    restart: always
    privileged: true
    ports:
      - 3307:3306
    environment:
      MYSQL_ROOT_PASSWORD: "root"
    volumes:
      - ./master/conf:/etc/mysql
      - ./master/data:/var/lib/mysql
      - ./master/log:/var/log/mysql/
    networks:
      - master_slave_web  
  slave:
    image: mysql:5.7.31
    restart: always
    privileged: true
    ports:
      - 3308:3306
    environment:
      MYSQL_ROOT_PASSWORD: "root"
    volumes:
      - ./slave/conf:/etc/mysql
      - ./slave/data:/var/lib/mysql
      - ./slave/log:/var/log/mysql/
    networks:
      - master_slave_web    

networks:
  master_slave:
    driver: bridge
    name: master_slave_web
```

### 第二步 : 設定master跟slave的my.conf
master的my.conf
```
[client]
default-character-set = utf8mb4

[mysqld]

collation_server = utf8mb4_general_ci
character_set_server = utf8mb4

## 設定server_id 同一局域網中需要唯一
server_id=101

## 指定不需要同步的數據庫名稱
binlog-ignore-db=mysql

## 開啟二進制日誌功能
log-bin=mall-mysql-bin

## 設定二進制使用內存大小(事務)
binlog_cache_size=1M

## 設定使用的二進制日誌格式
binlog_format=mixed

## 二進制日誌過期清理時間 默認為0 表示不自動清理
expire_logs_days=7

##  跳過主從複製中遇到的所有錯誤或指定類型的錯誤 避免slave端複製中斷
##  EX: 1062錯誤是指一些主鍵重複 1032 是因為主從數據庫數據不一致
slave_skip_errors=1062
```

slave的my.conf

```
[client]
default-character-set = utf8mb4

[mysqld]

collation_server = utf8mb4_general_ci
character_set_server = utf8mb4

## 設定server_id 同一局域網中需要唯一
server_id=102

## 指定不需要同步的數據庫名稱
binlog-ignore-db=mysql

## 開啟二進制日誌功能
log-bin=mall-mysql-slave1-bin

## 設定二進制使用內存大小(事務)
binlog_cache_size=1M

## 設定使用的二進制日誌格式
binlog_format=mixed

## 二進制日誌過期清理時間 默認為0 表示不自動清理
expire_logs_days=7

##  跳過主從複製中遇到的所有錯誤或指定類型的錯誤 避免slave端複製中斷
##  EX: 1062錯誤是指一些主鍵重複 1032 是因為主從數據庫數據不一致
slave_skip_errors=1062 

## relay_log配置中繼日誌
relay_log=mall-mysql-relay-bin

## log_slave_updates 表示slave將複製事件寫進自己的二進制日誌
log_slave_updates=1

## slave 設定為只讀 (具有super權限用戶除外)
read_only=1
```

### 第三步 進到master的docker內操作mysql
1. **在master建立slave的使用者用戶，並將該用戶賦予權限**
```
CREATE USER 'slave'@'%' IDENTIFIED BY '123456';

GRANT REPLICATION SLAVE ,REPLICATION CLIENT ON *.*TO'slave'@'%';
```
刷新配置
```
FLUSH PRIVILEGES;
```

查詢使用者帳號
```
SELECT User,Host FROM mysql.user;
```

查看master數據庫的mysql status
```
show master status;
```

2. **在slave，設定master資料庫資訊**
查詢master的docker資訊，並找尋IPAddress
```
docker inspect [container_id]
```

![](https://i.imgur.com/0DJwydv.png)
```
docker inspect 容器ID | grep IPAddress
```
![](https://i.imgur.com/gpeLgO4.png)


```
master_host=主數據庫的IP
master_user=在主數據庫創建用來同步數據的用戶帳號
master_password=在主數據庫創建用來同步數據的用戶密碼
master_port=主數據庫的端口
master_log_file=指定從主數據庫要複製數據的日誌文件，透過查詢主數據庫狀態，取得File資訊
master_log_pos=指定從主數據庫從哪個位置開始複製數據，透過查詢主數據庫狀態，取得Position資訊
master_connect_retry=連接失敗重試的時間間隔，單位為秒
```
```
change master to master_host="master主機IP" ,master_user='slave',master_password='123456',master_port=3306,master_log_file='mall-mysql-bin.000001',master_log_pos=617,master_connect_retry=30;
```

查看從數據庫狀態
```
show slave status\G;
```
![](https://i.imgur.com/firiH2c.png)

**目前還是尚未同步**

### 第四步 在slave開啟主從同步
```
start slave;
```

查看從數據庫狀態
```
show slave status\G;
```
![](https://i.imgur.com/XepKvH6.png)

**成功同步連線**

如果無法實現主從同步，可以通過以下排查
![](https://i.imgur.com/V3Ysw0Z.png)


### 最後一步 主從同步測試

1. master資料庫建立資料表，並新增資料
2. slavey 資料庫查看是否有同步


### 備註
**如使用docker再同一台server或電腦，記得gateway要一致，不然連線會失敗**