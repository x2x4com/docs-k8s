# EKS

## 文档版本
序号 | 内容 | 更新人 | 更新日期 | 版本
---| --- | --- | --- | ---
1 | 文档初始化 | 许向 | 2019-2-12 | 0.1


## 工具

- kubectl
   kubectl 不在本文档讨论范围内，请自行查找安装

- aws-iam-authenticator
   aws-iam-authenticator 是基本工具，无论是awscli、kubectl、eksctl都对它有依赖，请先行安装

   发行版
   - Linux：https://amazon-eks.s3-us-west-2.amazonaws.com/1.10.3/2018-07-26/bin/linux/amd64/aws-iam-authenticator

   - MacOS：https://amazon-eks.s3-us-west-2.amazonaws.com/1.10.3/2018-07-26/bin/darwin/amd64/aws-iam-authenticator

   - Windows：https://amazon-eks.s3-us-west-2.amazonaws.com/1.10.3/2018-07-26/bin/windows/amd64/aws-iam-authenticator.exe

   MacOS安装

   ```
   curl -o aws-iam-authenticator https://amazon-eks.s3-us-west-2.amazonaws.com/1.10.3/2018-07-26/bin/darwin/amd64/aws-iam-authenticator
   [ ! -d $HOME/bin ] && mkdir $HOME/bin && echo "export PATH=$HOME/bin:$PATH" >> $HOME/.bashrc
   chmod +x aws-iam-authenticator && mv aws-iam-authenticator $HOME/bin
   ```

- eksctl(可选)

   eksctl是一个用go实现的aws k8s命令行控制器，
   https://eksctl.io/

   ```
   brew tap weaveworks/tap
   brew install weaveworks/tap/eksctl
   ```


## Web Console配置



## 参考
[^1]: [eks-userguide](https://docs.aws.amazon.com/zh_cn/eks/latest/userguide/what-is-eks.html)
