# 步骤13 开启Load balancer-access-logs

- [The load balancer 服务 AWS annotations 官方文档](https://kubernetes.io/docs/concepts/cluster-administration/cloud-providers/#load-balancers)
- [AWS CLB 访问日志官方文档](https://docs.amazonaws.cn/en_us/elasticloadbalancing/latest/classic/enable-access-logs.html)
- [AWS ALB 访问日志官方文档](https://docs.amazonaws.cn/en_us/elasticloadbalancing/latest/application/load-balancer-access-logs.html)
- [AWS NLB 访问日志官方文档](https://docs.amazonaws.cn/en_us/elasticloadbalancing/latest/network/load-balancer-access-logs.html)

> 本节目的
1. 学习如何开启 load balancer 服务访问日志
2. 学习如何开启 aws-alb-ingress-controller 访问日志

# 13.1 配置 Classic load balancer 访问日志
## 按照文档配置存储日志的 S3 bucket
- [AWS CLB 访问日志官方文档](https://docs.amazonaws.cn/en_us/elasticloadbalancing/latest/classic/enable-access-logs.html)

注意：
1. S3 Bucket 必须与 Classic load balancer 位于同一区域
2. 存储桶策略必须授予将访问日志写入存储桶的权限。
参考存储桶策略，请将占位符替换为您的存储桶的 {bucket_name} 和 {prefix}，以及AWS账户的ID {aws-account-id}
- aws-account-id Beijing: 638102146993
- aws-account-id Ningxia: 037604701340
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "AWS": "arn:aws-cn:iam::{aws-account-id}:root"
      },
      "Action": "s3:PutObject",
      "Resource": "arn:aws-cn:s3:::<bucket-name>/{prefix}/*"
    }
  ]
}
```

## 添加 The load balancer 服务 AWS annotations
- [kubernetes 关于 CLB annotations 官方文档](https://kubernetes.io/docs/concepts/cluster-administration/cloud-providers/#load-balancers)

请将占位符替换为您的存储桶的 {bucket_name} 和 {prefix}
```yaml
apiVersion: v1
kind: Service
metadata:
  name: "nginx-clb-access-log"
  annotations:
        service.beta.kubernetes.io/aws-load-balancer-type: clb
        # The annotation to enable access log.
        service.beta.kubernetes.io/aws-load-balancer-access-log-emit-interval: "5"
        service.beta.kubernetes.io/aws-load-balancer-access-log-enabled: "true"
        service.beta.kubernetes.io/aws-load-balancer-access-log-s3-bucket-name: <bucket-name>
        service.beta.kubernetes.io/aws-load-balancer-access-log-s3-bucket-prefix: {prefix}
        
spec:
  selector:
    app: nginx
  type: LoadBalancer
  ports:
  - protocol: TCP
    port: 80
    targetPort: 80
```

## 测试验证
```bash
# 部署测试应用
kubectl apply -f resource/nginx-app/nginx-clb-access-log.yaml
kubectl get deployment nginx-clb-deployment -o wide
kubectl get pods -l app=nginx -o wide
CLB=$(kubectl get service nginx-clb-access-log -o json | jq -r '.status.loadBalancer.ingress[].hostname')
CLB_NAME=$(echo ${CLB} | cut -d "-" -f 1)
echo ${CLB_NAME}
for i in {1..10}; do curl -m3 -v $CLB; done

# 检查 CLB 的 access log已经配置
aws elb describe-load-balancer-attributes --load-balancer-name ${CLB_NAME} --region ${AWS_REGION} --query "LoadBalancerAttributes.AccessLog"

# 检查 S3 bucket，log日志已经生成
aws s3api list-objects-v2 --bucket ray-clb-accesslogs-zhy --prefix gcr-zhy-eksworkshop --max-items 10 --region ${AWS_REGION}

# 清理
kubectl delete -f resource/nginx-app/nginx-clb-access-log.yaml
```

# 13.2 配置 aws-alb-ingress-controller 访问日志
## 按照文档配置存储日志的 S3 bucket
- [AWS ALB 访问日志官方文档](https://docs.amazonaws.cn/en_us/elasticloadbalancing/latest/application/load-balancer-access-logs.html)

注意：
1. S3 Bucket 必须与 Application load balancer 位于同一区域
2. 存储桶策略必须授予将访问日志写入存储桶的权限。
参考存储桶策略，请将占位符替换为您的存储桶的 {bucket_name} 和 {prefix}，AWS账户的ID {aws-account-id}，您自己的AWS账户的ID {owner_account_id}
- aws-account-id Beijing: 638102146993
- aws-account-id Ningxia: 037604701340
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "AWS": "arn:aws-cn:iam::{aws-account-id}:root"
      },
      "Action": "s3:PutObject",
      "Resource": "arn:aws-cn:s3:::<bucket-name>/{prefix}/AWSLogs/{owner_account_id}/*"
    },
    {
      "Effect": "Allow",
      "Principal": {
        "Service": "delivery.logs.amazonaws.com"
      },
      "Action": "s3:PutObject",
      "Resource": "arn:aws-cn:s3:::<bucket-name>/{prefix}/AWSLogs/{owner_account_id}/*",
      "Condition": {
        "StringEquals": {
          "s3:x-amz-acl": "bucket-owner-full-control"
        }
      }
    },
    {
      "Effect": "Allow",
      "Principal": {
        "Service": "delivery.logs.amazonaws.com"
      },
      "Action": "s3:GetBucketAcl",
      "Resource": "arn-cn:aws:s3:::<bucket-name>"
    }
  ]
}
```

## 添加 aws-alb-ingress-controller 访问日志 annotations
- [kubernetes 关于 aws-alb-ingress-controller annotations 官方文档](https://github.com/kubernetes-sigs/aws-alb-ingress-controller/blob/master/docs/guide/ingress/annotation.md)

```yaml
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: "alb-ingress"
  namespace: "default"
  annotations:
    kubernetes.io/ingress.class: alb
    alb.ingress.kubernetes.io/scheme: internet-facing
    alb.ingress.kubernetes.io/target-type: ip
    alb.ingress.kubernetes.io/load-balancer-attributes: access_logs.s3.enabled=true,access_logs.s3.bucket=ray-alb-accesslogs,access_logs.s3.prefix=gcr-zhy-eksworkshop
```

## 测试验证
```bash
# 部署测试应用
kubectl apply -f resource/nginx-app/nginx-alb-ingress-access-log.yaml
ALB=$(kubectl get ingress -o json | jq -r '.items[0].status.loadBalancer.ingress[].hostname')
ALB_NAME=$(echo ${ALB} | cut -d "." -f 1 | cut -d "-" -f 1,2,3,4)
echo ${ALB_NAME}
ALB_ARN=$(aws elbv2 describe-load-balancers --names ${ALB_NAME} --region ${AWS_REGION} --query "LoadBalancers[0].LoadBalancerArn" | sed 's/"//g') 
echo ${ALB_ARN}
for i in {1..10}; do curl -m3 -v $ALB; done
# 如果遇到问题，请查看日志
kubectl logs -n kube-system $(kubectl get po -n kube-system | egrep -o alb-ingress[a-zA-Z0-9-]+)

# 检查 ALB 的 access log已经配置
aws elbv2 describe-load-balancer-attributes --load-balancer-arn ${ALB_ARN} --region ${AWS_REGION}

# 检查 S3 bucket，log日志已经生成
aws s3api list-objects-v2 --bucket ray-alb-accesslogs --prefix gcr-zhy-eksworkshop --max-items 10 --region ${AWS_REGION}

# 清理
kubectl delete -f resource/nginx-app/nginx-alb-ingress-access-log.yaml
```


# 13.3 配置 Network load balancer 访问日志
- [kubernetes 对于 AWS NLB access log annotations 支持的 issue](https://github.com/kubernetes/kubernetes/issues/81584)
- [add accessLogs support for aws nlb PR](https://github.com/kubernetes/kubernetes/pull/78497)
**根据该 Issue，目前 kubernetes 需要v1.15.0-beta.2以上版本 支持 Network load balancer 访问日志 annotations**

## 使用 NLB access log annotations
### 按照文档配置存储日志的 S3 bucket
[AWS NLB 访问日志官方文档](https://docs.amazonaws.cn/en_us/elasticloadbalancing/latest/network/load-balancer-access-logs.html)

注意：
1. S3 Bucket 必须与 Network load balancer 位于同一区域
2. 存储桶策略必须授予将访问日志写入存储桶的权限。
参考存储桶策略，请将占位符替换为您的存储桶的 {bucket_name} 和{prefix}，您自己的AWS账户的ID {owner_account_id}
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AWSLogDeliveryWrite",
      "Effect": "Allow",
      "Principal": {
        "Service": "delivery.logs.amazonaws.com"
      },
      "Action": "s3:PutObject",
      "Resource": "arn:aws-cn:s3:::{bucket_name}/{prefix}/AWSLogs/{owner_account_id}/*",
      "Condition": {
        "StringEquals": {
          "s3:x-amz-acl": "bucket-owner-full-control"
        }
      }
    },
    {
      "Sid": "AWSLogDeliveryAclCheck",
      "Effect": "Allow",
      "Principal": {
        "Service": "delivery.logs.amazonaws.com"
      },
      "Action": "s3:GetBucketAcl",
      "Resource": "arn:aws-cn:s3:::{bucket_name}"
    }
  ]
}
```

### 添加 The load balancer 服务 AWS annotations
```yaml
apiVersion: v1
kind: Service
metadata:
  name: "nginx-nlb-access-log"
  annotations:
        service.beta.kubernetes.io/aws-load-balancer-type: nlb
        # The annotation to enable access log.
        service.beta.kubernetes.io/aws-load-balancer-access-log-enabled: "true"
        service.beta.kubernetes.io/aws-load-balancer-access-log-s3-bucket-name: "ray-nlb-accesslogs-zhy"
        service.beta.kubernetes.io/aws-load-balancer-access-log-s3-bucket-prefix: "gcr-zhy-eksworkshop"
        
spec:
  selector:
    app: nginx
  type: LoadBalancer
  ports:
  - protocol: TCP
    port: 80
    targetPort: 80
```

### 测试验证
```bash
# 部署测试应用
kubectl apply -f resource/nginx-app/nginx-nlb-access-log.yaml
kubectl get deployment nginx-nlb-deployment -o wide
kubectl get pods -l app=nginx -o wide
NLB=$(kubectl get service nginx-nlb-access-log -o json | jq -r '.status.loadBalancer.ingress[].hostname')
NLB_NAME=$(echo ${NLB} | cut -d "-" -f 1)
echo ${NLB_NAME}
```
**NOTE**
**Access logs are created only if the load balancer has a TLS listener and they contain information only about TLS requests**

https://docs.aws.amazon.com/elasticloadbalancing/latest/network/load-balancer-access-logs.html

```bash
# 检查 NLB 的 access log已经配置
aws elbv2 describe-load-balancer-attributes --load-balancer-arn ${NLB_ARN} --region ${AWS_REGION}

# 如果 annotations 没有生效，通过 NLB 命令行 设置 access log
NLB_ARN=$(aws elbv2 describe-load-balancers --names ${NLB_NAME} --region ${AWS_REGION} --query "LoadBalancers[0].LoadBalancerArn" | sed 's/"//g') 
echo ${NLB_ARN}
aws elbv2 modify-load-balancer-attributes --load-balancer-arn ${NLB_ARN} \
  --attributes Key=access_logs.s3.enabled,Value=true Key=access_logs.s3.bucket,Value=ray-nlb-accesslogs-zhy Key=access_logs.s3.prefix,Value=gcr-zhy-eksworkshop \
  --region ${AWS_REGION}
for i in {1..10}; do curl -m3 -v $NLB; done

# 检查 S3 bucket，log日志是否生成
aws s3api list-objects-v2 --bucket ray-nlb-accesslogs-zhy --prefix gcr-zhy-eksworkshop --max-items 10 --region ${AWS_REGION}

# 清理
kubectl delete -f resource/nginx-app/nginx-nlb-access-log.yaml
```



