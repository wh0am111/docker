local记录3节点的docker swarm中使用docker stack部署各种服务配置文件


## 环境
* 主机1：192.168.0.65 swarm01 manager
* 主机2：192.168.0.66 swarm02 manager
* 主机3：192.168.0.67 swarm03 manager


## 服务
* 反向代理：Treafik，有v1.7和v2两个版本，其他服务主要基于traefik:v2来反向代理
* docker UI：portainer
* 数据库：mysql,redis,mongo
* 其他：zookeeper,kafka,dubbo,rabbitmq,haproxy


## 创建代理工具Treafik V2的在swarm中的网络
```bash
docker network create -d overlay --attachable traefik-public
```
## 创建新项目在swarm中的网络
```bash
docker network create -d overlay newproject
```


## 部署Traefik V2
```bash
docker stack deploy -c ./traefik/docker-compose-traefik.yml reverse_proxy
```

## 部署docker swarm集群的web管理工具portainer

```bash
docker stack deploy -c ./portainer/docker-compose-portainer-agent.yml tool
```

## 部署新项目所需的所有服务，redis，mongo，mysql，rabbitmq,zookeeper,dubbo-admin,kafka
```bash
docker stack deploy -c ./redis/docker-compose-redis.yml newproject_db
docker stack deploy -c ./mongo/docker-compose-mongo.yml newproject_db
docker stack deploy -c ./mysql/docker-compose-mysql.yml newproject_db
docker stack deploy -c ./rabbitmq/docker-compose-rabbitmq.yml newproject_queue
docker stack deploy -c ./haproxy/docker-compose-haproxy.yml newproject_queue

docker stack deploy -c ./zookeeper/docker-compose-zookeeper.yml newproject_service
docker stack deploy -c ./dubbo/docker-compose-dubbo.yml newproject_service
docker stack deploy -c ./kafka/docker-compose-kafka.yml newproject_service
```

# 部署docker swarm集群的日志收集以及状态监控系统
```bash
docker stack deploy -c ./swarmprom/docker-compose-traefik-v2-http.yml monitor
docker stack deploy -c ./elk/docker-compose-elk.yml log
```