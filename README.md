# 概述
ragflow k8s部署，所有组件统一使用一个pvc进行存储，方便管理。

支持ragflow以下版本
```bash
0.18.0
0.19.0
0.19.1
0.20.0
0.20.1
```

# 目录结构
```bash
env --> 全局环境变量
pvc --> 所有组件，统一使用一个pvc来进行持久化存储
databases --> 数据库相关：mysql，redis
middleware --> 中间件相关：elasticsearch，minio
services --> 服务相关：ragflow
```

# 准备工作
详细步骤，请参考文章：https://www.cnblogs.com/xiao987334176/p/18849826
## 1. 创建命名空间ragflow
```bash
kubectl create namespace ragflow
```
## 2. 创建全局环境变量
```bash
kubectl apply -f env/env.yaml
```
## 3. 创建pv和pvc
这里的pv是自建的NFS，请根据实际情况修改
```bash
kubectl apply -f pvc/storageClass.yaml
kubectl apply -f pvc/pv.yaml
kubectl apply -f pvc/pvc.yaml
```
进入到NFS根目录，创建ragflow相关持久化文件，并设置权限
```bash
mkdir -p ragflow/volumes/elasticsearch/data
mkdir -p ragflow/volumes/minio/data
mkdir -p ragflow/volumes/mysql/data
mkdir -p ragflow/volumes/redis/data
mkdir -p ragflow/volumes/ragflow/logs
chmod 777 -R ragflow
```
# 数据库相关
## MySQL
发布应用，注意执行顺序，先执行configMap，再执行下面的。
```bash
kubectl apply -f databases/mysql/mysql-cm1-configmap.yaml
kubectl apply -f databases/mysql/mysql-StatefulSet.yaml
kubectl apply -f databases/mysql/mysql-Service.yaml
```
## Valkey
```bash
kubectl apply -f databases/redis/redis-StatefulSet.yaml
kubectl apply -f databases/redis/redis-Service.yaml
```
# 中间件相关
## Elasticsearch
```bash
kubectl apply -f middleware/elasticsearch/elasticsearch-StatefulSet.yaml
kubectl apply -f middleware/elasticsearch/elasticsearch-Service.yaml
```
## MinIO
```bash
kubectl apply -f middleware/minio/minio-StatefulSet.yaml
kubectl apply -f middleware/minio/minio-Service.yaml
```
# 服务相关
## Ragflow
发布应用，注意执行顺序，先执行configMap，再执行下面的。
```bash
kubectl apply -f services/ragflow/ragflow-cm1-configmap.yaml
kubectl apply -f services/ragflow/ragflow-cm2-configmap.yaml
kubectl apply -f services/ragflow/ragflow-cm3-configmap.yaml
kubectl apply -f services/ragflow/ragflow-cm5-configmap.yaml
kubectl apply -f services/ragflow/ragflow-Deployment.yaml
kubectl apply -f services/ragflow/ragflow-Service.yaml
```
等待6分钟，这个镜像特别大，请耐心等待！
查看pod，确保是Running状态
```bash
# kubectl -n ragflow get pods|grep ragflow
ragflow-6bddc85f97-6xpkd   1/1     Running   0             6m48s
```
# 访问ragflow
直接使用ragflow的nodeport端口访问即可

如果需要域名访问，则添加一条ingress规则，指向到ragflow的svc，端口是80。
# 常见问题
## 1. 镜像拉取失败
由于国内无法访问docker hub，正常拉取会失败

首先准备一台docker服务器，修改文件`/etc/docker/daemon.json`
```bash
{
	"registry-mirrors": [
		"https://docker.1ms.run",
		"https://docker.xuanyuan.me"
	]
}
```
重启docker进程，就可以拉取docker hub镜像了

然后将本地的docker镜像推送到私有仓库，比如：harbor

最后将yaml文件中的镜像改为私有仓库地址即可，注意添加私有仓库密钥。

## 2. 某些组件响应慢
这些组件限制的cpu和内存，用的是最小资源。如果使用场景访问量很大，请根据实际情况调整。
## 3. ragflow注册失败
请检查mysql组件运行是否正常，登录到mysql的pod里面，运行以下命令
```bash
mysql -h localhost -u root -pinfini1#raGflow
mysql> show databases;
```
确保能正常显示数据库列表

注意：mysql8默认是启动了密码复杂性校验的，请确保root密码符合密码复杂性要求。
