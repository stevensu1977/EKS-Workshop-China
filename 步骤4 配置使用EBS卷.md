# 步骤4 配置使用EBS卷
4.1 创建所需要的IAM policy , EKS OIDC provider, service account

> 4.1.1 创建所需要的IAM policy


```bash

aws iam create-policy \
    --policy-name Amazon_EBS_CSI_Driver \
    --policy-document file://./resource/aws-ebs-csi-driver/example-iam-policy.json \
    --region cn-northwest-1
        
#返回示例,请记录返回的Plociy ARN
{
    "Policy": {
        "PolicyName": "Amazon_EBS_CSI_Driver",
        "PolicyId": "ANPASX274AZPHUBLZRDPJ",
        "Arn": "arn:aws-cn:iam::999999999999:policy/Amazon_EBS_CSI_Driver",
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
  rolearn: arn:aws-cn:iam::999999999999:role/eksctl-eksworkshop2-nodegroup-ng-NodeInstanceRole-GJGCGMEYPL5P
  username: system:node:{{EC2PrivateDNSName}}

Events:  <none>

aws iam attach-role-policy \
--policy-arn arn:aws-cn:iam::999999999999:policy/Amazon_EBS_CSI_Driver \
--role-name eksctl-eksworkshop2-nodegroup-ng-NodeInstanceRole-GJGCGMEYPL5P
```

* 使用updaterole.sh更新你的nodegroup role

```bash
EBS_CSI_POLICY=$(aws iam  list-policies  --query 'Policies[?contains(PolicyName,`Amazon_EBS_CSI_Driver`)]' --output text |awk {'print $1'})

sh ./resource/aws-ebs-csi-driver/updaterole.sh  $EBS_CSI_POLICY
```

4.2 配置使用EBS

>4.2.1 部署EBS CSI 驱动到EKS集群

```bash

kubectl apply -k ./resource/aws-ebs-csi-driver/deploy/kubernetes/overlays/stable

```
>4.2.2 部署动态卷实例应用

```bash
cd 
kubectl apply -f resource/aws-ebs-csi-driver/examples/kubernetes/dynamic-provisioning/specs

#查看storageclass
kubectl describe storageclass ebs-sc

#查看示例app状态
kubectl get pods --watch

kubectl get pv

kubectl exec -it app cat /data/out.txt

#删除示例程序
kubectl delete -f resource/aws-ebs-csi-driver/examples/kubernetes/dynamic-provisioning/specs
```