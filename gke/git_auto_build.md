# GCP自动触发构建

## 文档版本
序号 | 内容 | 更新人 | 更新日期 | 版本
---| --- | --- | --- | ---
1 | 初始版本 | 许向 | 2019-1-15 | 0.1
2 | 文档补全 | 许向 | 2019-2-11 | 0.2


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

2. 配置Bitbucket

   1. 创建一个新的repo

      - Owner: 你自己
      - Project: x2x4-demo-crop1
      - Repository name: app_blockscanner
      - Access level: private
      - Inculde a README? No
      - Version control: Git

   2. 将Webhook接收机上的公钥添加到Bitbucket账号的SSH Keys中

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

   创建cfg.py文件，如下内容

   {% gist id="https://gist.github.com/x2x4com/b64b6288b4708dfc24f1fea2524c04ea" %}{% endgist %}

   UpdateGitMirror脚本只用到了配置文件中的global_password

4. 启动Webhook接收器

   ```
   cd $HOME/webhook
   screen -AdmS mirror bin/python3 src/UpdateGitMirror.py
   ```

5. 添加Gitee的Webhook

   在项目或者Gitee企业后台添加Webhook

   URL:  http://your-ip:10080/oschina/update.json
   密码:  gps_1
   选择事件: Push, Pull Request, Tag Push

6. 检查配置

   提交代码到git@gitee.com:crop1/app_blockscanner，查看bitbucket是不是同步了

6. 配置GCP触发器

   确认同步之后就可以配置GCP的触发器

   点击Cloud Build，创建触发器，选择Bitbucket

   ![cb-1](https://github.com/x2x4com/docs-k8s/raw/master/gke/img/cloud-build-1.jpg)

   在授权页面填入Bitbucket账号

   ![cb-2](https://github.com/x2x4com/docs-k8s/raw/master/gke/img/cloud-build-2.jpg)

   选择要同步的代码，选择app_blockscanner

   ![cb-3](https://github.com/x2x4com/docs-k8s/raw/master/gke/img/cloud-build-3.jpg)

   接下来的页面内填写信息

   - 名称: 触发器的名称
   - 触发器类型: 分支
   - 分支: master
   - 构建配置: Dockerfile
   - Dockerfile目录: /(Dockerfile存放在项目源码的/目录处)
   - 映像名称: asia.gcr.io/demo-1/crop1/master/app_blockscanner:$COMMIT_SHA(可以自行定义)

   保存即可

   ![cb-4](https://github.com/x2x4com/docs-k8s/raw/master/gke/img/cloud-build-4.jpg)

   一个代码库可以针对不同分支建不同的触发器策略，完成如下

   ![cb-5](https://github.com/x2x4com/docs-k8s/raw/master/gke/img/cloud-build-5.jpg)

   最后就可以点击运行触发器来测试了
