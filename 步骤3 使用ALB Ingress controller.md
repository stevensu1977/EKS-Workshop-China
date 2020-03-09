# 步骤3 使用ALB Ingress controller

3.1 创建ALB Ingress Controller所需要的IAM policy , EKS OIDC provider, service account

> 3.1.1 创建EKS OIDC Provider (在整个实验中只需要操作1次）

```bash
eksctl utils associate-iam-oidc-provider --cluster=eksworkshop --approve 
             
```

修改OIDC Provider的Audience, 选择IAM服务>Providers>
![-w738](media/15832934698484/15837578824712.jpg)
点击Add an Audience , 输入sts.amazonaws.com,点击保存
![-w1079](media/15832934698484/15837579358417.jpg)


> 3.1.2 创建所需要的IAM policy

```bash
aws iam create-policy \
    --policy-name ALBIngressControllerIAMPolicy \
    --policy-document file://./resource/alb-ingress-controller/iam-policy.json
    
#请记录返回的ALBIngressControllerIAMPolicy的ARN
{
    "Policy": {
        "PolicyName": "ALBIngressControllerIAMPolicy",
        "PolicyId": "ANPASX274AZPIN2MOYJLS",
        "Arn": "arn:aws-cn:iam::999999999999:policy/ALBIngressControllerIAMPolicy",
        "Path": "/",
        "DefaultVersionId": "v1",
        "AttachmentCount": 0,
        "PermissionsBoundaryUsageCount": 0,
        "IsAttachable": true,
        "CreateDate": "2020-03-04T06:47:15Z",
        "UpdateDate": "2020-03-04T06:47:15Z"
    }
}
```

>3.1.3 请使用上述返回的policy ARN创建service account

```bash
ALB_POLICY=$(aws iam  list-policies  --query 'Policies[?contains(PolicyName,`ALBIngressControllerIAMPolicy`)]' --output text |awk {'print $1'})

eksctl create iamserviceaccount \
       --cluster=eksworkshop \
       --namespace=kube-system \
       --name=alb-ingress-controller \
       --attach-policy-arn=$ALB_POLICY \
       --override-existing-serviceaccounts \
       --approve

```


打开IAM>Role 输入iamserviceaccount 找到你刚才创建的role 点击Trust  ![-w797](media/15832934698484/15837580420522.jpg)
relationships, StringEquals里面的"sts.amazonaws.com.cn" 修改为"sts.amazonaws.com"

修改后的截图
![-w964](media/15832934698484/15837581990630.jpg)


3.2 部署 ALB Ingress Controller
 
 ```bash
   
 kubectl apply -f ./resource/alb-ingress-controller/rbac-role.yaml
 
 #修改 ./resource/alb-ingress-controller/alb-ingress-controller.yaml中的aws-vpc-id , 其他参数，环境变量示例文件已经配置好了.
 #(eksctl 自动创建的 vpc 默认为 eksctl-<集群名字>-cluster/VPC)
  
 
  #修改集群名字,aws-vpc-id
  - --cluster-name=eksworkshop
  - --aws-vpc-id=<eksctl 创建的vpc-id>   
  - --aws-region=cn-northwest-1
  
  #需要添加环境变量
  env:
            - name: AWS_REGION
              value: cn-northwest-1
  
    
  #部署ALB Ingress Controller
 kubectl apply -f ./resource/alb-ingress-controller/alb-ingress-controller.yaml
 
 #确认ALB Ingress Controller是否工作
 kubectl logs -n kube-system $(kubectl get po -n kube-system | egrep -o alb-ingress[a-zA-Z0-9-]+)

 ```

![-w564](media/15832934698484/15833085106168.jpg)

    

3.3 使用ALB Ingress   

我们重新创建一个使用ClusterIP的service 指向实验2的 nginx-app

nginx-ingress.yaml
```bash
---
apiVersion: v1
kind: Service
metadata:
  name: "service-nginx-clusterip"
  annotations:
        service.beta.kubernetes.io/aws-load-balancer-type: nlb
spec:
  selector:
    app: nginx
  type: ClusterIP
  ports:
  - protocol: TCP
    port: 80
    targetPort: 80
---
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: "alb-ingress"
  namespace: "default"
  annotations:
    kubernetes.io/ingress.class: alb
    alb.ingress.kubernetes.io/scheme: internet-facing
    alb.ingress.kubernetes.io/target-type: ip
  labels:
    app: nginx
spec:
  rules:
    - http:
        paths:
          - path: /*
            backend:
              serviceName: "service-nginx-clusterip"
              servicePort: 80
```

>3.3.2 验证

```bash
kubectl get ingress

curl -v $(kubectl get ingress | grep alb-ingress | awk '{print $4}')
```

