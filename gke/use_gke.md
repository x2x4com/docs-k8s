# 使用GKE

## 文档版本
序号 | 内容 | 更新人 | 更新日期 | 版本
---| --- | --- | --- | ---
1 | 文档补全 | 许向 | 2019-2-9 | 0.1


## 安装gcloud-sdk

**注意：自 Cloud SDK 版本 206.0.0 起，gcloud CLI 对使用 Python 3.4+ 解释器运行提供实验性支持**

我使用Python3.6.2，具体的安装部署请[参考这里](https://cloud.google.com/sdk/docs/)


## 安装需要的组件

1. 更新gcloud工具到最新版本

   ```
   gcloud components update
   ```

2. 需要安装的组件

   container-builder-local, docker-credential-gcr, kubectl

   ```
   gcloud components install container-builder-local docker-credential-gcr kubectl
   ```

3. 检查安装的组件

   ```
   gcloud components list
   ```

4. 获得登陆授权

   ```
   gcloud auth application-default login
   ```

   输入之后会跳出浏览器，输入你的GCP用户名密码，点击授权即可


5. 设置项目


   ```
   gcloud config set project demo-1
   ```

   命令如下返回，成功

   ```
   Updated property [core/project].
   ```

6. 检查配置

   ```
   gcloud config list
   ```

   命令如下返回，成功

   ```
   [core]
   account = your-gmail@gmail.com
   disable_usage_reporting = True
   project = demo-1

   Your active configuration is: [default]
   ```


## 创建集群

创建一个K8S集群

- 集群名称: cluster-1
- 地区: asia-northeast1-a (日本节点)
- 机器类型:  1vcpu 3.75GB RAM
- 节点数量: 2

命令行

```
gcloud container --project "demo-1" clusters create "cluster-1" --zone "asia-northeast1-a" --username "admin" --cluster-version "1.9.3-gke.0" --machine-type "n1-standard-1" --image-type "COS" --disk-size "100" --scopes "https://www.googleapis.com/auth/compute","https://www.googleapis.com/auth/devstorage.read_only","https://www.googleapis.com/auth/logging.write","https://www.googleapis.com/auth/monitoring","https://www.googleapis.com/auth/servicecontrol","https://www.googleapis.com/auth/service.management.readonly","https://www.googleapis.com/auth/trace.append" --num-nodes "2" --network "default" --enable-cloud-logging --enable-cloud-monitoring --subnetwork "default" --enable-autoupgrade
```

检查集群创建情况

```
$ gcloud container clusters list
NAME        LOCATION           MASTER_VERSION  MASTER_IP       MACHINE_TYPE   NODE_VERSION  NUM_NODES  STATUS
cluster-1   asia-northeast1-a  1.9.3-gke.0     **.***.***.***  n1-standard-1  1.9.3-gke.0   3          RUNNING
```

绑定kubectl控制台

```
$ gcloud container clusters get-credentials cluster-1 --zone asia-northeast1-a
Fetching cluster endpoint and auth data.
kubeconfig entry generated for cluster-1.
```

检查节点情况

```
$ kubectl get nodes
NAME                                        STATUS    ROLES     AGE       VERSION
gke-cluster-1-default-pool-ddf96b65-3f5f    Ready     <none>    11m       v1.9.3-gke.0
gke-cluster-1-default-pool-ddf96b65-ggg3    Ready     <none>    11m       v1.9.3-gke.0
```
