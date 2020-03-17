# 步骤2 创建EKS集群

2.1 使用eksctl 创建EKS集群(操作需要10-15分钟),该命令同时会创建一个使用t3.small的受管节点组。

详细参考手册
* [creating-and-managing-clusters](https://eksctl.io/usage/creating-and-managing-clusters/)
* [managing-nodegroups](https://eksctl.io/usage/managing-nodegroups/)


 ```bash
 #可以修改
 --node-type 工作节点类型
 --nodes 工作节点数量
 CLUSTER_NAME 集群名称
 AWS_REGION cn-northwest-1：宁夏区； cn-north-1：北京区

AWS_REGION=cn-northwest-1
AWS_DEFAULT_REGION=cn-northwest-1
CLUSTER_NAME=gcr-zhy-eksworkshop

eksctl create cluster --name=${CLUSTER_NAME} --version 1.14 、
  --nodes=2 --node-type t3.medium --managed --alb-ingress-access --region=${AWS_REGION}

 ```
 
参考输出
```bash
[ℹ]  eksctl version 0.15.0-rc.1
[ℹ]  using region cn-northwest-1
[ℹ]  setting availability zones to [cn-northwest-1b cn-northwest-1a cn-northwest-1c]
[ℹ]  subnets for cn-northwest-1b - public:192.168.0.0/19 private:192.168.96.0/19
[ℹ]  subnets for cn-northwest-1a - public:192.168.32.0/19 private:192.168.128.0/19
[ℹ]  subnets for cn-northwest-1c - public:192.168.64.0/19 private:192.168.160.0/19
[ℹ]  using Kubernetes version 1.14
[ℹ]  creating EKS cluster "gcr-zhy-eksworkshop" in "cn-northwest-1" region with managed nodes
[ℹ]  will create 2 separate CloudFormation stacks for cluster itself and the initial managed nodegroup
[ℹ]  if you encounter any issues, check CloudFormation console or try 'eksctl utils describe-stacks --region=cn-northwest-1 --cluster=gcr-zhy-eksworkshop'
[ℹ]  CloudWatch logging will not be enabled for cluster "gcr-zhy-eksworkshop" in "cn-northwest-1"
[ℹ]  you can enable it with 'eksctl utils update-cluster-logging --region=cn-northwest-1 --cluster=gcr-zhy-eksworkshop'
[ℹ]  Kubernetes API endpoint access will use default of {publicAccess=true, privateAccess=false} for cluster "gcr-zhy-eksworkshop" in "cn-northwest-1"
[ℹ]  2 sequential tasks: { create cluster control plane "gcr-zhy-eksworkshop", create managed nodegroup "ng-f6ef3930" }
[ℹ]  building cluster stack "eksctl-gcr-zhy-eksworkshop-cluster"
[ℹ]  deploying stack "eksctl-gcr-zhy-eksworkshop-cluster"
[ℹ]  building managed nodegroup stack "eksctl-gcr-zhy-eksworkshop-nodegroup-ng-f6ef3930"
[ℹ]  deploying stack "eksctl-gcr-zhy-eksworkshop-nodegroup-ng-f6ef3930"
[✔]  all EKS cluster resources for "gcr-zhy-eksworkshop" have been created
[✔]  saved kubeconfig as "/Users/ruiliang/.kube/config"
[ℹ]  nodegroup "ng-f6ef3930" has 2 node(s)
[ℹ]  node "ip-192-168-14-19.cn-northwest-1.compute.internal" is ready
[ℹ]  node "ip-192-168-40-132.cn-northwest-1.compute.internal" is ready
[ℹ]  waiting for at least 2 node(s) to become ready in "ng-f6ef3930"
[ℹ]  nodegroup "ng-f6ef3930" has 2 node(s)
[ℹ]  node "ip-192-168-14-19.cn-northwest-1.compute.internal" is ready
[ℹ]  node "ip-192-168-40-132.cn-northwest-1.compute.internal" is ready
[ℹ]  kubectl command should work with "/Users/ruiliang/.kube/config", try 'kubectl get nodes'
[✔]  EKS cluster "gcr-zhy-eksworkshop" in "cn-northwest-1" region is ready

```

  集群创建完毕后,查看EKS集群工作节点
  ```bash
   kubectl get node
  ```
  
  参考输出
```bash
NAME                                                STATUS   ROLES    AGE    VERSION
ip-192-168-14-19.cn-northwest-1.compute.internal    Ready    <none>   4d1h   v1.14.9-eks-1f0ca9
ip-192-168-40-132.cn-northwest-1.compute.internal   Ready    <none>   4d1h   v1.14.9-eks-1f0ca9

```

（可选）配置环境变量用于后续使用
```bash
STACK_NAME=$(eksctl get nodegroup --cluster ${CLUSTER_NAME} --region=${AWS_REGION} -o json | jq -r '.[].StackName')
echo $STACK_NAME
ROLE_NAME=$(aws cloudformation describe-stack-resources --stack-name $STACK_NAME --region=${AWS_REGION} | jq -r '.StackResources[] | select(.ResourceType=="AWS::IAM::Role") | .PhysicalResourceId')
echo $ROLE_NAME
export ROLE_NAME=${ROLE_NAME}
export STACK_NAME=${STACK_NAME}
```

2.2 部署一个nginx测试eks集群基本功能

> 下载本git repository, 参考 resource/nginx-app目录的nginx-nlb.yaml, 创建一个nginx pod，并通过LoadBalancer类型对外暴露

```bash
git clone git@github.com:liangruibupt/EKS-Workshop-China.git
kubectl create namespace test
kubectl apply -f nginx-nlb.yaml --namespace test

## Check deployment status
kubectl get pods -n test
kubectl get deployment nginx-deployment -n test

## Get the external access 确保 EXTERNAL-IP是一个有效的AWS Network Load Balancer的地址
kubectl get service service-nginx -o wide -n test

## 下面可以试着访问这个地址
ELB=$(kubectl get service service-nginx -n test -o json | jq -r '.status.loadBalancer.ingress[].hostname')
curl -m3 -v $ELB
```

> 清理
```bash
kubectl delete -f nginx-nlb.yaml -n test
```

2.3 扩展集群节点
> 我们之前通过eksctl创建了一个2节点的集群，下面我们来扩展集群节点到3
```bash
NODE_GROUP=$(eksctl get nodegroup --cluster ${CLUSTER_NAME} --region=${AWS_REGION} -o json | jq -r '.[].Name')
eksctl scale nodegroup --cluster=${CLUSTER_NAME} --nodes=3 --name=${NODE_GROUP} --region=${AWS_REGION}
```
> 检查结果
```bash
eksctl get nodegroup --cluster ${CLUSTER_NAME} --region=${AWS_REGION}
kubectl get node
```

参考输出
```bash
CLUSTER			NODEGROUP	CREATED			MIN SIZE	MAX SIZE	DESIRED CAPACITY	INSTANCE TYPE	IMAGE ID
gcr-zhy-eksworkshop	ng-f6ef3930	2020-03-03T08:09:39Z	2		3		3			t3.medium

NAME                                                STATUS   ROLES    AGE    VERSION
ip-192-168-14-19.cn-northwest-1.compute.internal    Ready    <none>   4d1h   v1.14.9-eks-1f0ca9
ip-192-168-40-132.cn-northwest-1.compute.internal   Ready    <none>   4d1h   v1.14.9-eks-1f0ca9
ip-192-168-86-36.cn-northwest-1.compute.internal    Ready    <none>   3d4h   v1.14.9-eks-1f0ca9
```

2.4 （可选）中国区镜像处理

由于防火墙或安全限制，海外gcr.io, quay.io的镜像可能无法下载，为了不手动修改原始yaml文件的镜像路径，采用下面webhook的方式，自动修改国内配置的镜像路径。
详情参考 [amazon-api-gateway-mutating-webhook-for-k8](https://github.com/aws-samples/amazon-api-gateway-mutating-webhook-for-k8)
```bash
git clone git@github.com:aws-samples/amazon-api-gateway-mutating-webhook-for-k8.git
cd amazon-api-gateway-mutating-webhook-for-k8
Deploy cloudformation api-gateway.yaml，注意修改下面的镜像地址为你偏好的地址
image_mirrors = {
            'gcr.io/': '<镜像地址>/',
            'k8s.gcr.io/': '<镜像地址>/google-containers/',
            'quay.io/': '<镜像地址>/'
}
kubectl apply -f mutating-webhook.yaml
```

部署样例应用，验证webhook工作正常
```bash
kubectl apply -f ./nginx-gcr.yaml
kubectl get pods
kubectl get pod nginx-gcr-deployment-xxxx -o=jsonpath='{.spec.containers[0].image}'
# 结果应显示为配置的gcr.io的路径
# 清理
kubectl delete -f ./nginx-gcr.yaml
```

注意：China region EKS service have two official account id, cn-northwest-1 961992271922, cn-north-1 91830976355，因此如果你需要下载EKS 官方的镜像，需要正确使用上面的两个id
```bash
#例如宁夏区
#aws-efs-csi-driver
961992271922.dkr.ecr.cn-northwest-1.amazonaws.com.cn/eks/aws-efs-csi-driver

#aws-ebs-csi-driver
961992271922.dkr.ecr.cn-northwest-1.amazonaws.com.cn/eks/aws-ebs-csi-driver
```

2.5 (可选) Kubernetes service different expose type

尽管每个Pod都有一个唯一的IP地址，但是如果没有Service，这些IP不会暴露在群集外部。 Service 允许您的应用程序接收流量

Services 可以根据ServiceSpec中定义的Type被暴露为不同的方式:

1. ClusterIP (default) - Exposes the Service on an internal IP in the cluster. This type makes the Service only reachable from within the cluster.
2. NodePort - Exposes the Service on the same port of each selected Node in the cluster using NAT. Makes a Service accessible from outside the cluster using <NodeIP>:<NodePort>. Superset of ClusterIP.
3. LoadBalancer - Creates an external load balancer in the current cloud (if supported) and assigns a fixed, external IP to the Service. Superset of NodePort.
4. ExternalName - Exposes the Service using an arbitrary name (specified by externalName in the spec) by returning a CNAME record with the name. No proxy is used. This type requires v1.7 or higher of kube-dns.

通过一个简单实验解释不同的ServiceSpec类型，详情参考

[ServiceSpec type](Service-SourceIP.md)