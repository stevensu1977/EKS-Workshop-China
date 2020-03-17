# 步骤12 VPCPeering和DX支持

[External SNAT 官方文档](https://docs.aws.amazon.com/eks/latest/userguide/external-snat.html/)
[AWS-VPC-CNI 插件官方文档](https://github.com/aws/amazon-vpc-cni-k8s)

> 本节目的
1. 学习EKS集群如何进行VPC Peering (类似方法同样适用于通过Direct Connect连接数据中心或者其他AWS Region）


# 12.1 创建 Worker Node 在Private Subnet的集群
复用已经创建的VPC，创建Worker Node 位于Private Subnet的集群
[参考eksctl文档](https://eksctl.io/usage/vpc-networking/)
## 创建集群
```bash
# 参考 resource/private-cluster.yaml, 修改subnets
eksctl create cluster -f resource/private-cluster.yaml

# 多个EKS集群进行上下文切换, 确保位于gcr-zhy-eksworkshop-private
kubectl config get-contexts
kubectl config use-context xxxx

# 集群创建完毕后,查看EKS集群工作节点，由于位于 Private Subent 未分配EXTERNAL-IP
kubectl get node -o wide
NAME                                                 STATUS     INTERNAL-IP       EXTERNAL-IP   OS-IMAGE         KERNEL-VERSION                  CONTAINER-RUNTIME
ip-192-168-153-107.cn-northwest-1.compute.internal    Ready      192.168.153.107    <none>   
ip-192-168-177-73.cn-northwest-1.compute.internal     Ready      192.168.177.73   <none>
ip-192-168-97-177.cn-northwest-1.compute.internal     Ready      192.168.97.177   <none>

# 部署测试 nginx
kubectl apply -f resource/nginx-app/nginx-nlb.yaml
## Check deployment status
kubectl get pods
NAME                                READY   STATUS    RESTARTS   AGE
nginx-deployment-56db997f77-cbgrw   1/1     Running   0          11m
## Get the external access 确保 EXTERNAL-IP是一个有效的AWS Network Load Balancer的地址
ELB=$(kubectl get service service-nginx -o json | jq -r '.status.loadBalancer.ingress[].hostname')
curl -m3 -v $ELB
# Note: if you want to create the public facing ELB, you must have the private subnets and public subnets in your cluster. If you only have the private subnets, you must claim your load balance as intenal one via annotation: service.beta.kubernetes.io/aws-load-balancer-internal: "true". Otherwise,you will encounter error:
# CreatingLoadBalancerFailed   service/service-nginx                    Error creating load balancer (will retry): failed to ensure load balancer for service default/service-nginx: could not find any suitable subnets for creating the ELB
## clean up
kubectl delete -f resource/nginx-app/nginx-nlb.yaml

```

## Testing VPC peering
```bash

curl http://161.189.48.110
curl http://172.16.40.249 
```

# 12.2 如何利用 amazon-vpc-cni-k8s 支持 VPC Peering
## SNAT 原理
VPC中的通信（例如Pod到Pod）直接在Private IP地址之间进行，不需要SNAT。 当流量发往VPC之外的地址时，Amazon VPC CNI Kubernetes 网络插件会将每个Pod的Private IP地址转换为分配给EC2实例主要弹性网络接口（ENI）的primary private IP。默认情况下，运行Pod的节点。 

SNAT的主要作用：
1. 使Pod可以与Internet双向通信。 Worker Node 必须位于公共子网中，并且具有分配给其主网络接口的主Private IP的公共或弹性IP地址。 SNAT 是必需的，因为 Internet 网关只知道如何在 primary private IP与公有或弹性 IP 地址之间进行转换（这些地址已分配给 Pod 所在的 EC2 Worker Node的 primary ENI

2. 防止其他私有IP地址空间，例如VPC Peering，Transit VPC或Direct Connect直接与如下Pod通信: 未向此 Pod 分配 EC2 Worker Node的 primary ENI 的 primary private IP。

地址转换如图所示：
![SNAT-enabled](media/SNAT-enabled.jpg)

那么如果Pod需要与其他私有IP地址空间（例如VPC Peering，Transit VPC或Direct Connect）中的设备通信，那么：
1. Worker Node必须部署在私有子网中，该私有子网具有到公用子网中NAT设备或者NAT网关的路由。
2. 需要在VPC CNI插件 aws-node DaemonSet中启用外部SNAT

启用外部SNAT后，当流量发送的目标为 VPC 外部地址时，CNI插件不会将Pod的 private IP地址转换为 primary private IP 此地址分配给 Pod 正在其上运行的 EC2 Worker Node的 primary ENI）。在VPC之外。从Pod到Internet的流量在外部与NAT设备的公共IP地址进行双向转换，并通过Internet网关路由到Internet和从Internet路由，如下图所示。
![SNAT-disabled](media/SNAT-disabled.jpg)

## 利用amazon-vpc-cni-k8s的环境变量，实现VPC Peering
1. 方法1：AWS_VPC_K8S_CNI_EXTERNALSNAT
[AWS-VPC-CNI 插件官方文档](https://github.com/aws/amazon-vpc-cni-k8s)
指定是否需要外部 NAT gateway 提供 SNAT
```bash
kubectl set env daemonset -n kube-system aws-node AWS_VPC_K8S_CNI_EXTERNALSNAT=true
```

2. 方法2： AWS_VPC_K8S_CNI_EXCLUDE_SNAT_CIDRS (Since v1.6.0)
[AWS_VPC_K8S_CNI_EXCLUDE_SNAT_CIDRS发布说明](https://aws.amazon.com/cn/about-aws/whats-new/2020/02/amazon-eks-announces-release-of-vpc-cni-version-1-6/)
[AWS-VPC-CNI 插件官方文档](https://github.com/aws/amazon-vpc-cni-k8s)
指定一个逗号分隔的IPv4 CIDR列表，以将其排除在 CNI 插件的 SNAT 外。
使用该环境变量时，需要设置 AWS_VPC_K8S_CNI_EXTERNALSNAT=false.


# 12.3. cleanup
```bash
清除VPC Peering

eksctl delete cluster -f resource/private-cluster.yaml
```
