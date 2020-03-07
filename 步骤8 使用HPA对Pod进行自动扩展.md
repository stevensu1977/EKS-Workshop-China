# 步骤8 使用HPA对Pod进行自动扩展
我们将要为集群配置一个HPA，并且部署一个应用进行压力测试，验证Pod 横向扩展能力。

[官方文档](https://kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale-walkthrough/#increase-load)

8.1 Install Metrics Server

```bash
# 下载Metrics Server
mkdir -p hpa && cd hpa
curl -Ls https://api.github.com/repos/kubernetes-sigs/metrics-server/tarball/v0.3.6  -o metrics-server-v0.3.6.tar.gz
mkdir -p metrics-server-v0.3.6
tar -xzf metrics-server-v0.3.6.tar.gz --directory metrics-server-v0.3.6 --strip-components 1

# 假设您已经使用了image webhook, 否则请修改image地址为中国国内可访问的
kubectl apply -f metrics-server-v0.3.6/deploy/1.8+/
kubectl logs -n kube-system $(kubectl get po -n kube-system | egrep -o metrics-server[a-zA-Z0-9-]+)

# 验证 Metrics Server installation
kubectl get deployment metrics-server -n kube-system
kubectl get apiservice v1beta1.metrics.k8s.io -o yaml
```

8.2 安装 HPA sample application php-apache
```bash
# Set threshold to CPU30% auto-scaling, and up to 5 pod replicas
kubectl autoscale deployment php-apache --cpu-percent=30 --min=1 --max=5
kubectl get hpa
```

8.3 开启 load-generator
```bash
kubectl run --generator=run-pod/v1 -it --rm load-generator --image=busybox /bin/sh

# 提示框输入
while true; do wget -q -O- http://php-apache.default.svc.cluster.local; done
```

8.4 Check HPA
```bash

watch kubectl get hpa
NAME         REFERENCE               TARGETS    MINPODS   MAXPODS   REPLICAS   AGE
php-apache   Deployment/php-apache   250%/30%   1         5         4          3m22s

kubectl get deployment php-apache
NAME         READY   UP-TO-DATE   AVAILABLE   AGE
php-apache   5/5     5            5           6m2s

```
