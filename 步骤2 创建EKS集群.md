# 步骤2 创建EKS集群

2.1 使用eksctl 创建EKS集群(操作需要10-15分钟),该命令同时会创建一个使用t3.small的受管节点组。

 ```bash
 #设置默认region cn-northwest-1 
 $export AWS_DEFAULT_REGION=cn-northwest-1
 
 #可以修改
 #--node-type 工作节点类型
 #--nodes 工作节点数量
 
 eksctl create cluster \
       --name eksworkshop \
       --node-type t3.small \
              
 ```
 
 ![](media/15764759782724/15764761011094.jpg)

  集群创建完毕后,查看EKS集群工作节点
  ```bash
   kubectl get node
  ```
  
  ![](media/15764759782724/15764762619982.jpg)


2.2 部署一个nginx测试eks集群基本功能

> resource/nginx.yaml

```yaml
---
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: nginx-deployment
  labels:
    app: nginx
spec:
  replicas: 1
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx
        ports:
        - containerPort: 80
---
apiVersion: v1
kind: Service
metadata:
  name: "service-nginx"
  annotations:
        service.beta.kubernetes.io/aws-load-balancer-type: nlb
spec:
  selector:
    app: nginx
  type: LoadBalancer
  ports:
  - protocol: TCP
    port: 80
    targetPort: 80

```
 
 > 部署nginx
 
 ```bash
 kubectl apply -f resource/nginx-app
 
 #查看deployment ,service 
 kubectl get deploy,svc
 
 #可以通过 service/service-nginx 显示的EXTERNAL-IP访问nginx应用
 curl -v $(kubectl get svc | grep service-nginx | awk '{print $4}')
 ```

![-w1306](media/15832910610879/15837427165613.jpg)
 


