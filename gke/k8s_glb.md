# GKE使用全局负载均衡器

## 文档版本
序号 | 内容 | 更新人 | 更新日期 | 版本
---| --- | --- | --- | ---
1 | 文档补全 | 许向 | 2019-2-11 | 0.1

## 需求

项目原本是通过区域(日本)的Loadbalance服务接入(Nginx Service with Node type Loadbalance)，随着业务增长，其他区域的用户大量反应页面打开慢，于是需要谷歌的全局负载均衡器，GKE使用GLB并不能在Web页面上配置，具体参考[^1]

## 配置
GLB并不支持http => https的重定向，而且有一些自定义的7层规则，所以我在k8s内部配置了一组Nginx7层代理，用于http => https的重定向和其他自定义7层规则

1. 创建一个全局静态地址

   在VPC网络，外部IP地址，添加一个保留静态地址

   - 名称: ninechain-prod-gip-1
   - 网络服务层级: 优质
   - IP版本: IPv4
   - 类型: **全局**

2. 创建backend服务

   {% gist id="x2x4com/99eb04347070ea344d1f52517187f17b",file="k8s-glb-backend.yaml" %}{% endgist %}

   检查

   ```
   $ kubectl get backendconfig
   NAME                                 AGE
   artwook-prod-service-ing-1-backend   24d
   ```

3. 创建内部的LB

   关键点在于配置文件中增加backend-config，不要使用443，我用443做后端时死活不通，heathcheck failed

   ```
   annotations:
    beta.cloud.google.com/backend-config: '{"ports": {"80":"artwook-prod-service-ing-1-backend"}}'
   ```

   {% gist id="x2x4com/99eb04347070ea344d1f52517187f17b",file="k8s-glb-internal-lb.yaml" %}{% endgist %}

   Nginx配置文件是通过Configmap挂载到Pod中，Example如下

   {% gist id="x2x4com/99eb04347070ea344d1f52517187f17b",file="k8s-glb-l7proxy.conf" %}{% endgist %}

   检查

   ```
   $ kubectl get deployment/ninechain-prod-service-l7proxyinternal-1
   NAME                                       DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
   ninechain-prod-service-l7proxyinternal-1   1         1         1            1           24d
   ```

   ```
   $ kubectl get services/ninechain-prod-service-l7proxyinternal-1
   NAME                                       TYPE       CLUSTER-IP     EXTERNAL-IP   PORT(S)        AGE
   ninechain-prod-service-l7proxyinternal-1   NodePort   10.31.245.16   <none>        80:32756/TCP   24d
   ```

4. 创建Ingress

   站点的SSL证书需要上传到secret

   如果没有上传请执行下面的命令

   ```
   kubectl create secret tls prod-resource-ssl --key=prod-resource-ssl.key --cert=prod-resource-ssl.crt
   ```

   {% gist id="x2x4com/99eb04347070ea344d1f52517187f17b",file="k8s-glb-ing.yaml" %}{% endgist %}

   检查

   ```
   $ kubectl get ing
   NAME                           HOSTS     ADDRESS          PORTS     AGE
   ninechain-prod-service-ing-1   *         35.***.***.***   80, 443   23d
   ```

   ADDRESS已经绑定上了分配给我们的全局IP，这时在页面上就能看到负载均衡器的配置了

   检查

   ![图片1](https://raw.githubusercontent.com/x2x4com/docs-k8s/master/gke/img/glb-0.jpg)

   当有流量时候可以看到

   ![图片2](https://raw.githubusercontent.com/x2x4com/docs-k8s/master/gke/img/glb-1.jpg)


## http => https跳转实现

按照上面的配置，ingress到后端是走的80，后端的内部l7proxy怎么才能知道用户是通过前端80访问过来还是443访问过来的呢？

查了不少资料，发现GLB在代理的时会增加一个header, x-forwarded-proto 通过这个值是http还是https可以知道用户是通过哪个前端进来的，于是我就在每个proxy_pass之前进行了判断，加入

```
if ($http_x_forwarded_proto = 'http') {
   return 301 https://$host$request_uri;
}
```

## 参考

[^1]: [k8s-http-balancer](https://cloud.google.com/kubernetes-engine/docs/tutorials/http-balancer)
