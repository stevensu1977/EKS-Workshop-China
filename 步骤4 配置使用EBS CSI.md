# 步骤4 配置使用EBS CSI

官方指导

4.1 创建所需要的IAM policy , EKS OIDC provider, service account


> 4.1.1 创建所需要的IAM policy
[https://raw.githubusercontent.com/kubernetes-sigs/aws-ebs-csi-driver/v0.4.0/docs/example-iam-policy.json](https://raw.githubusercontent.com/kubernetes-sigs/aws-ebs-csi-driver/v0.4.0/docs/example-iam-policy.json)

```bash
#curl -O https://raw.githubusercontent.com/kubernetes-sigs/aws-ebs-csi-driver/v0.4.0/docs/example-iam-policy.json

#请使用EKS-Workshop-China/resource/aws-ebs-csi-driver/example-iam-policy.json
aws iam create-policy \
    --policy-name Amazon_EBS_CSI_Driver \
    --policy-document file://./example-iam-policy.json \
    --region cn-northwest-1
        
#返回示例,请记录返回的Plociy ARN
{
    "Policy": {
        "PolicyName": "Amazon_EBS_CSI_Driver",
        "PolicyId": "ANPASX274AZPHUBLZRDPJ",
        "Arn": "arn:aws-cn:iam::188642756190:policy/Amazon_EBS_CSI_Driver",
        "Path": "/",
        "DefaultVersionId": "v1",
        "AttachmentCount": 0,
        "PermissionsBoundaryUsageCount": 0,
        "IsAttachable": true,
        "CreateDate": "2020-03-04T08:11:11Z",
        "UpdateDate": "2020-03-04T08:11:11Z"
    }
```
    
> 4.1.2 获取eks 工作节点的IAM role
注意这一步如果是多个nodegroup就会有多个role

```bash
kubectl -n kube-system describe configmap aws-auth

Name:         aws-auth
Namespace:    kube-system
Labels:       <none>
Annotations:  <none>

Data
====
mapRoles:
----
- groups:
  - system:bootstrappers
  - system:nodes
  rolearn: arn:aws-cn:iam::188642756190:role/eksctl-eksworkshop2-nodegroup-ng-NodeInstanceRole-GJGCGMEYPL5P
  username: system:node:{{EC2PrivateDNSName}}

Events:  <none>

aws iam attach-role-policy \
--policy-arn arn:aws-cn:iam::188642756190:policy/Amazon_EBS_CSI_Driver \
--role-name eksctl-eksworkshop2-nodegroup-ng-NodeInstanceRole-GJGCGMEYPL5P
```

* 我这里准备了一个脚本updaterole.sh
```bash
sh ./updaterole.sh "arn:aws-cn:iam::188642756190:policy/Amazon_EBS_CSI_Driver"
```

> 4.1.3 部署EBS CSI 驱动到eks 集群

[官方文档 https://docs.aws.amazon.com/zh_cn/eks/latest/userguide/ebs-csi.html](https://docs.aws.amazon.com/zh_cn/eks/latest/userguide/ebs-csi.html)

```bash
#因为中国区有些地方需要修改,所以我fork了一个官方的，并修改了
git clone xxxxx
kubectl apply -k ./aws-ebs-csi-driver/deploy/kubernetes/overlays/stable

```
4.2 部署动态卷实例应用

```bash
cd aws-ebs-csi-driver/examples/kubernetes/dynamic-provisioning/
kubectl apply -f specs/

#查看storageclass
kubectl describe storageclass ebs-sc

#查看示例app状态
kubectl get pods --watch

kubectl get pv

kubectl exec -it app cat /data/out.txt

#删除示例程序
kubectl delete -f specs/
```