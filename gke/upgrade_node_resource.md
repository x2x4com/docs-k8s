# 资源池升级

## 文档版本
序号 | 内容 | 更新人 | 更新日期 | 版本
---| --- | --- | --- | ---
1 | 文档初始化 | 许向 | 2018-5-13 | 0.1
2 | 文档补全 | 许向 | 2019-2-11 | 0.2


## 资源池扩容[^1]
在某些情况下资源节点的配置并不能满足实际应用需求，比如某个应用需要2颗CPU，但是物理节点只有1颗CPU。GCP的扩容只能在同级别的主机资源上，也就是说1Core的机器再扩容也是1Core * n，这个时候只能通过升级底层资源池的配置来达到扩容的需求

1. 显示当前pool

   ```
   $ gcloud container node-pools list --cluster cluster-1 --zone asia-northeast1-a
   NAME          MACHINE_TYPE   DISK_SIZE_GB  NODE_VERSION
   default-pool  n1-standard-1  100           1.9.3-gke.0
   ```

2. 创建一个大号的pool

   ```
   $ gcloud container node-pools create n1-standard-2 --cluster cluster-1 --zone asia-northeast1-a --machine-type=n1-standard-2 --num-nodes=2
   Creating node pool n1-standard-2...done.
   Created [https://container.googleapis.com/v1/projects/demo-1/zones/asia-northeast1-a/clusters/cluster-1/nodePools/n1-standard-2].
   NAME           MACHINE_TYPE   DISK_SIZE_GB  NODE_VERSION
   n1-standard-2  n1-standard-2  100           1.9.3-gke.0
   ```

3. 检查

   ```
   $ gcloud container node-pools list --cluster cluster-1 --zone asia-northeast1-a
   NAME           MACHINE_TYPE   DISK_SIZE_GB  NODE_VERSION
   default-pool   n1-standard-1  100           1.9.3-gke.0
   n1-standard-2  n1-standard-2  100           1.9.3-gke.0
   ```

   ```
   $ kubectl get nodes
   NAME                                         STATUS    ROLES     AGE       VERSION
   gke-cluster-1-default-pool-ddf96b65-3f5f    Ready     <none>    12d       v1.9.3-gke.0
   gke-cluster-1-default-pool-ddf96b65-drpv    Ready     <none>    10d       v1.9.3-gke.0
   gke-cluster-1-default-pool-ddf96b65-ggg3    Ready     <none>    12d       v1.9.3-gke.0
   gke-cluster-1-n1-standard-2-92d50459-tbv0   Ready     <none>    6m        v1.9.3-gke.0
   gke-cluster-1-n1-standard-2-92d50459-tlkz   Ready     <none>    6m        v1.9.3-gke.0
   ```

   ```
   $ kubectl get pods -o=wide
   NAME                                             READY     STATUS    RESTARTS   AGE       IP           NODE
   cluster-devel-service-os-1-675c6d95d7-nqtnt     1/1       Running   0          22m       10.32.4.89   gke-cluster-1-default-pool-ddf96b65-drpv
   cluster-prod-app-api-1-c7d7d9c79-9b8v6          2/2       Running   0          22m       10.32.2.44   gke-cluster-1-default-pool-ddf96b65-3f5f
   cluster-prod-app-chat-1-7dcdd4575c-8gs9q        1/1       Running   0          5d        10.32.0.60   gke-cluster-1-default-pool-ddf96b65-ggg3
   cluster-prod-app-daemons-1-566c6f5f58-rhrrc     2/2       Running   0          1h        10.32.4.87   gke-cluster-1-default-pool-ddf96b65-drpv
   cluster-prod-app-static-1-667bfb756f-kns9x      1/1       Running   2          1d        10.32.4.68   gke-cluster-1-default-pool-ddf96b65-drpv
   cluster-prod-service-geth-1-5b54bcf9bc-rmsr9    1/1       Running   0          11m       10.32.0.73   gke-cluster-1-default-pool-ddf96b65-ggg3
   cluster-prod-service-lb-1-9f68489dc-j5h2c       1/1       Running   0          2d        10.32.0.65   gke-cluster-1-default-pool-ddf96b65-ggg3
   cluster-prod-service-redis-1-66c9cdb5fd-jlctd   1/1       Running   0          11d       10.32.0.33   gke-cluster-1-default-pool-ddf96b65-ggg3
   cluster-prod-service-rmq-1-6b8848dccb-xn64n     1/1       Running   0          5d        10.32.0.61   gke-cluster-1-default-pool-ddf96b65-ggg3
   ```

4. 迁移工作负载

   此步骤又分成2个小步骤

   1. Cordon the existing node pool
   2. Drain the existing node pool

   - Cordon

      首先检查当前default-pool

      ```
      $ kubectl get nodes -l cloud.google.com/gke-nodepool=default-pool
      NAME                                        STATUS    ROLES     AGE       VERSION
      gke-cluster-1-default-pool-ddf96b65-3f5f   Ready     <none>    12d       v1.9.3-gke.0
      gke-cluster-1-default-pool-ddf96b65-drpv   Ready     <none>    10d       v1.9.3-gke.0
      gke-cluster-1-default-pool-ddf96b65-ggg3   Ready     <none>    12d       v1.9.3-gke.0
      ```

      cordon

      ```
      $ for node in $(kubectl get nodes -l cloud.google.com/gke-nodepool=default-pool -o=name); do kubectl cordon "$node"; done
      node "gke-cluster-1-default-pool-ddf96b65-3f5f" cordoned
      node "gke-cluster-1-default-pool-ddf96b65-drpv" cordoned
      node "gke-cluster-1-default-pool-ddf96b65-ggg3" cordoned

      ```

      查看

      ```
      $ kubectl get nodes
      NAME                                         STATUS                     ROLES     AGE       VERSION
      gke-cluster-1-default-pool-ddf96b65-3f5f    Ready,SchedulingDisabled   <none>    13d       v1.9.3-gke.0
      gke-cluster-1-default-pool-ddf96b65-drpv    Ready,SchedulingDisabled   <none>    10d       v1.9.3-gke.0
      gke-cluster-1-default-pool-ddf96b65-ggg3    Ready,SchedulingDisabled   <none>    13d       v1.9.3-gke.0
      gke-cluster-1-n1-standard-2-92d50459-tbv0   Ready                      <none>    55m       v1.9.3-gke.0
      gke-cluster-1-n1-standard-2-92d50459-tlkz   Ready                      <none>    55m       v1.9.3-gke.0      
      ```

   - Drain

      ```
      $ for node in $(kubectl get nodes -l cloud.google.com/gke-nodepool=default-pool -o=name); do kubectl drain --force --ignore-daemonsets --delete-local-data --grace-period=10 "$node"; done
      WARNING: Deleting pods not managed by ReplicationController, ReplicaSet, Job, DaemonSet or StatefulSet: kube-proxy-gke-cluster-1-default-pool-ddf96b65-3f5f; Deleting pods with local storage: cluster-prod-app-api-1-c7d7d9c79-9b8v6; Ignoring DaemonSet-managed pods: fluentd-gcp-v2.0.10-vkzvd
      pod "kube-dns-autoscaler-69c5cbdcdd-tmzq8" evicted
      pod "kube-dns-5c5884448b-lpkrf" evicted
      pod "kube-dns-5c5884448b-f6gsz" evicted
      pod "event-exporter-v0.1.7-7c4f8bb746-47rvr" evicted
      pod "cluster-prod-app-api-1-c7d7d9c79-9b8v6" evicted
      node "gke-cluster-1-default-pool-ddf96b65-3f5f" drained
      node "gke-cluster-1-default-pool-ddf96b65-drpv" already cordoned
      WARNING: Ignoring DaemonSet-managed pods: fluentd-gcp-v2.0.10-vc862; Deleting pods not managed by ReplicationController, ReplicaSet, Job, DaemonSet or StatefulSet: kube-proxy-gke-cluster-1-default-pool-ddf96b65-drpv; Deleting pods with local storage: cluster-prod-app-daemons-1-566c6f5f58-rhrrc
      pod "cluster-prod-service-geth-1-5b54bcf9bc-zqs5p" evicted
      pod "cluster-prod-service-geth-1-5b54bcf9bc-4rcq6" evicted
      pod "cluster-prod-service-geth-1-5b54bcf9bc-zhjz4" evicted
      pod "cluster-prod-service-geth-1-5b54bcf9bc-tj4wc" evicted
      pod "cluster-prod-service-geth-1-5b54bcf9bc-f2rcw" evicted
      pod "cluster-prod-service-geth-1-5b54bcf9bc-xjkcm" evicted
      pod "cluster-prod-service-geth-1-5b54bcf9bc-rl4w9" evicted
      pod "cluster-prod-service-geth-1-5b54bcf9bc-vv56l" evicted
      pod "cluster-prod-service-geth-1-5b54bcf9bc-xvzrz" evicted
      pod "cluster-prod-service-geth-1-5b54bcf9bc-8949p" evicted
      pod "cluster-prod-service-geth-1-5b54bcf9bc-4pvk2" evicted
      pod "cluster-prod-service-geth-1-5b54bcf9bc-5rq27" evicted
      pod "cluster-prod-service-geth-1-5b54bcf9bc-bdtfd" evicted
      pod "cluster-prod-service-geth-1-5b54bcf9bc-v5gwq" evicted
      pod "cluster-prod-service-geth-1-5b54bcf9bc-jlvcg" evicted
      pod "cluster-prod-service-geth-1-5b54bcf9bc-rvqm6" evicted
      pod "cluster-prod-service-geth-1-5b54bcf9bc-g9zvr" evicted
      pod "cluster-prod-service-geth-1-5b54bcf9bc-mt7b7" evicted
      pod "cluster-devel-service-os-1-675c6d95d7-nqtnt" evicted
      pod "heapster-v1.5.0-6d8547cdc9-5bn5w" evicted
      pod "cluster-prod-app-static-1-667bfb756f-kns9x" evicted
      pod "cluster-prod-app-daemons-1-566c6f5f58-rhrrc" evicted
      node "gke-cluster-1-default-pool-ddf96b65-drpv" drained
      node "gke-cluster-1-default-pool-ddf96b65-ggg3" already cordoned
      WARNING: Ignoring DaemonSet-managed pods: fluentd-gcp-v2.0.10-mw8n5; Deleting pods not managed by ReplicationController, ReplicaSet, Job, DaemonSet or StatefulSet: kube-proxy-gke-cluster-1-default-pool-ddf96b65-ggg3; Deleting pods with local storage: kubernetes-dashboard-775fc9968-9ntzq
      pod "cluster-prod-service-geth-1-5b54bcf9bc-rmsr9" evicted
      pod "cluster-prod-service-geth-1-5b54bcf9bc-wj25v" evicted
      pod "l7-default-backend-57856c5f55-46psz" evicted
      pod "cluster-prod-service-redis-1-66c9cdb5fd-jlctd" evicted
      pod "cluster-prod-app-chat-1-7dcdd4575c-8gs9q" evicted
      pod "kubernetes-dashboard-775fc9968-9ntzq" evicted
      pod "metrics-server-v0.2.1-7f8dd98c8f-vvcm8" evicted
      pod "cluster-prod-service-lb-1-9f68489dc-j5h2c" evicted
      pod "cluster-prod-service-rmq-1-6b8848dccb-xn64n" evicted
      node "gke-cluster-1-default-pool-ddf96b65-ggg3" drained
      ```

      查看

      ```
      $ kubectl get pods -o wide
      NAME                                             READY     STATUS    RESTARTS   AGE       IP            NODE
      cluster-devel-service-os-1-675c6d95d7-4b6bc     1/1       Running   0          3m        10.32.10.7    gke-cluster-1-n1-standard-2-92d50459-tlkz
      cluster-prod-app-api-1-c7d7d9c79-bzgkx          2/2       Running   0          4m        10.32.9.4     gke-cluster-1-n1-standard-2-92d50459-tbv0
      cluster-prod-app-chat-1-7dcdd4575c-zpqdl        1/1       Running   0          2m        10.32.10.10   gke-cluster-1-n1-standard-2-92d50459-tlkz
      cluster-prod-app-daemons-1-566c6f5f58-hhgxb     2/2       Running   0          3m        10.32.10.8    gke-cluster-1-n1-standard-2-92d50459-tlkz
      cluster-prod-app-static-1-667bfb756f-zjf68      1/1       Running   0          3m        10.32.10.6    gke-cluster-1-n1-standard-2-92d50459-tlkz
      cluster-prod-service-geth-1-5b54bcf9bc-xdd5p    1/1       Running   0          48m       10.32.10.3    gke-cluster-1-n1-standard-2-92d50459-tlkz
      cluster-prod-service-lb-1-9f68489dc-n8fsd       1/1       Running   0          2m        10.32.10.9    gke-cluster-1-n1-standard-2-92d50459-tlkz
      cluster-prod-service-redis-1-66c9cdb5fd-k54zb   1/1       Running   0          2m        10.32.10.11   gke-cluster-1-n1-standard-2-92d50459-tlkz
      cluster-prod-service-rmq-1-6b8848dccb-kbzf8     1/1       Running   0          2m        10.32.9.8     gke-cluster-1-n1-standard-2-92d50459-tbv0
      ```

5. 移除老pool

   ```
   $ gcloud container node-pools delete default-pool --cluster cluster-1 --zone asia-northeast1-a
   The following node pool will be deleted.
   [default-pool] in cluster [cluster-1] in [asia-northeast1-a]

   Do you want to continue (Y/n)?  y

   Deleting node pool default-pool...done.
   Deleted [https://container.googleapis.com/v1/projects/demo-1/zones/asia-northeast1-a/clusters/cluster-1/nodePools/default-pool].   
    ```

   查看

   ```
   $ gcloud container node-pools list --cluster cluster-1 --zone asia-northeast1-a
   NAME          MACHINE_TYPE   DISK_SIZE_GB  NODE_VERSION
   n1-standard-2  n1-standard-2  100           1.9.3-gke.0
   ```


## 参考文档

[^1]: [migrating-node-pool](https://cloud.google.com/kubernetes-engine/docs/tutorials/migrating-node-pool)
