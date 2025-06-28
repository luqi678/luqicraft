# ✅RocketMQ部署以及基础

# 部署
## 1.下载镜像
### 1.1 RocketMQ镜像
```linux
docker pull apache/rocketmq:5.1.4
```

###  1.2 Dashboard 管理界面镜像
```linux
docker pull apacherocketmq/rocketmq-dashboard:1.0.0
```

## 2. 部署镜像
### 设置log文件
```linux
mkdir -p /mydata/rocketmq/logs/rocketmqlogs
touch /mydata/rocketmq/logs/rocketmqlogs/stats.log
touch /mydata/rocketmq/logs/rocketmqlogs/rocketmq-broker.log
touch /mydata/rocketmq/logs/rocketmqlogs/other.log
chmod 644 /mydata/rocketmq/logs/rocketmqlogs/*.log

```



```dockerfile
# 创建网络
docker network create rocketmq-net

# 启动 NameServer（同步宿主机时区）
docker run -d --name rmq-namesrv --network rocketmq-net \
-p 9876:9876 \
-e TZ=Asia/Shanghai \
-v /etc/localtime:/etc/localtime:ro \
-v /etc/timezone:/etc/timezone:ro \
apache/rocketmq:5.1.4 sh mqnamesrv

# 启动 Broker（同步宿主机时区）
docker rm rmq-broker
docker run -d --name rmq-broker --network rocketmq-net \
-p 10911:10911 -p 10909:10909 \
-e "NAMESRV_ADDR=rmq-namesrv:9876" \
-e "MAX_POSSIBLE_HEAP=100000000" \
-e TZ=Asia/Shanghai \
-v /mydata/rocketmq/logs:/home/rocketmq/logs \
-v /mydata/rocketmq/store:/home/rocketmq/store \
-v /etc/localtime:/etc/localtime:ro \
-v /etc/timezone:/etc/timezone:ro \
apache/rocketmq:5.1.4 \
sh mqbroker -n rmq-namesrv:9876 --enable-proxy

# 启动 Dashboard（同步宿主机时区）
docker run -d --name rmq-dashboard --network rocketmq-net \
-e "JAVA_OPTS=-Drocketmq.namesrv.addr=rmq-namesrv:9876" \
-e TZ=Asia/Shanghai \
-v /mydata/rocketmq/dashboard/config:/root/config \
-v /etc/localtime:/etc/localtime:ro \
-v /etc/timezone:/etc/timezone:ro \
-p 10901:8080 apacherocketmq/rocketmq-dashboard:1.0.0

```



## 引用

**版权声明：本文为博主原创文章，遵循 CC 4.0 BY-SA 版权协议，转载请附上原文出处链接和本声明。**


---

> 作者: [luqiCraft](https://luqi678.github.io/luqicraft)  
> URL: https://luqi678.github.io/posts/rocketmq%E9%83%A8%E7%BD%B2%E4%BB%A5%E5%8F%8A%E5%9F%BA%E7%A1%80/  

