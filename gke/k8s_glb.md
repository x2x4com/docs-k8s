# GKE使用全局负载均衡器

## 文档版本
序号 | 内容 | 更新人 | 更新日期 | 版本
---| --- | --- | --- | ---
1 | 文档补全 | 许向 | 2019-2-11 | 0.1

## 需求

项目原本是通过区域(日本)的Loadbalance服务接入(Nginx Service with Node type Loadbalance)，随着业务增长，其他区域的用户大量反应页面打开慢，于是需要谷歌的全局负载均衡器，GKE使用GLB并不能在Web页面上配置。

## 配置
GLB并不支持http => https的重定向，所以我在k8s内部配置了一组Nginx7层代理，用于http => https的重定向和其他自定义7层规则

1. 创建一个全局静态地址

   在VPC网络，外部IP地址，添加一个保留静态地址

   - 名称: ninechain-prod-gip-1
   - 网络服务层级: 优质
   - IP版本: IPv4
   - 类型: **全局**

2. 创建一个K8S的backend

   {% gist id="x2x4com/99eb04347070ea344d1f52517187f17b",file="k8s-glb-backend.yaml" %}{% endgist %}

3. 创建一个内部的LB

   {% gist id="x2x4com/99eb04347070ea344d1f52517187f17b",file="k8s-glb-internal-lb.yaml" %}{% endgist %}

...TODO

## 参考

[^1]: [k8s-http-balancer](https://cloud.google.com/kubernetes-engine/docs/tutorials/http-balancer)
