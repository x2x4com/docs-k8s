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


## 镜像自动构建

GCP的云端触发器只支持3类的来源

- 云端源代码库(Cloud Source Repositories)
- Bitbucket
- Github

实际上无论是Bitbucket与Github都是通过镜像到云端源代码库来实现触发的

Bitbucket与Github的构建集成比较容易，根据页面提示一步一步就可以了

**Bitbucket集成私有项目时候需要注意，授权的用户需要拥有admin权限，否则授权后私有库在gcp会看不到**

接下来把主要介绍自定义Git库与GCP集成


### Gitee(码云)与GCP触发器集成

GCP只支持Bitbucket与Github，当用户使用其他Git仓库(如码云Gitee，或者自建的Gitlab)时就比较麻烦了

(Gitlab官方支持直接镜像到Bitbucket，可以直接集成)

我这里主要介绍使用Gitee的仓库来与GCP触发器集成

核心的思路就是曲线救国，使用Webhook将Gitee的库镜像到Bitbucket私有库，然后再授权给GCP触发器


使用到的工具

- Gitee账号
- Bitbucket账号
- GCP账号
- 具有公网地址的主机一台(ddns也OK)
- Webhook接收脚本，[这里可以获取到](https://github.com/x2x4com/gitee_trigger)

项目的名称
app_blockscanner

脚本使用Python3.4+，请自行安装。

请在app_blockscanner库里添加Webhook接收机的deploy key。

1. webhook接收机上创建镜像
   ```
   [ ! -d "/data/git" ] && sudo mkdir -p /data/git
   cd /data/git
   git clone --mirror git@gitee.com:crop1/app_blockscanner.git
   ```

2. 配置Webhook接收脚本环境
   1. 创建venv环境
      ```
      cd $HOME
      python3 -m venv webhook
      ```
   2. 下载webhook接收脚本
      ```
      cd webhook
      git clone "https://github.com/x2x4com/gitee_trigger.git" src
      ```
   3. 激活venv环境
      ```
      . bin/activate
      ```
   4. 安装依赖文件
      ```
      cd src
      pip install -r requirements.txt
      ```

3. 配置Webhook脚本


   <div>
   <script src="https://gist.github.com/x2x4com/b64b6288b4708dfc24f1fea2524c04ea.js"></script>
   </div>

4. 启动

   ```
   cd $HOME/webhook
   screen -AdmS mirror bin/python3 src/UpdateGitMirror.py
   ```
