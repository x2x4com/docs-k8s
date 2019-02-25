# GKE

GKE是GCP提供的托管式K8S环境

这片文档主要记录

1. GKE底层node更换运算资源(跨配置升级)
2. GKE入口Ingress类型，包括自定义LB(区域IP)与GLB整合
3. GKE with istio(TODO)


关于收费

GKE的收费可以通过[这里](https://cloud.google.com/products/calculator/)来计算

主要的成本来自于GCE实例的收费与网络流量，K8S引擎本身GCP是不收费的
