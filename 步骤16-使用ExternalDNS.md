# ExternalDNS
ExternalDNS makes Kubernetes resources discoverable via public DNS servers. 

[ExternalDNS github](https://github.com/kubernetes-sigs/external-dns)

To configure ExternalDNS on AWS EKS, please follow up the guide

Follow the [Overall guide](https://github.com/kubernetes-sigs/external-dns/blob/master/docs/tutorials/aws.md) to setup ExternalDNS for use in Kubernetes clusters running in AWS. 

Specify the source=ingress argument so that ExternalDNS will look for hostnames in Ingress objects. Here is example [ExternalDNS  for alb-ingress](https://github.com/kubernetes-sigs/external-dns/blob/master/docs/tutorials/alb-ingress.md). Note that the ALB ingress controller uses the same tags for [subnet auto-discovery](https://kubernetes-sigs.github.io/aws-alb-ingress-controller/guide/controller/config/#subnet-auto-discovery) as Kubernetes does with the AWS cloud provider.

## IAM Policy
AllowExternalDNSUpdates allows ExternalDNS to update Route53 Resource Record Sets and Hosted Zones. 
```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": [
                "route53:ChangeResourceRecordSets"
            ],
            "Resource": [
                "arn:aws-cn:route53:::hostedzone/*"
            ]
        },
        {
            "Effect": "Allow",
            "Action": [
                "route53:ListHostedZones",
                "route53:ListResourceRecordSets"
            ],
            "Resource": [
                "*"
            ]
        },
        {
            "Effect": "Allow",
            "Action": "sts:*",
            "Resource": "*"
        }
    ]
}
```

## IAM Role
参考[步骤7-在EKS中使用IAMRole进行权限管理.md](步骤7-在EKS中使用IAMRole进行权限管理.md)

注意使用eksctl 0.16以上版本

```bash
# 在步骤3我们已经创建了OIDC身份提供商 
# 请检查IAM Open ID Connect provider已经创建
aws eks describe-cluster --name ${CLUSTER_NAME} --region ${AWS_REGION} --query cluster.identity.oidc.issuer --output text

eksctl utils associate-iam-oidc-provider --cluster=${CLUSTER_NAME} --approve --region ${AWS_REGION}

#创建serviceaccount external-dns with IAM role
AllowExternalDNSUpdates_ARN=
eksctl create iamserviceaccount --name external-dns --namespace default \
    --cluster ${CLUSTER_NAME} --attach-policy-arn ${AllowExternalDNSUpdates_ARN} \
    --approve --override-existing-serviceaccounts --region ${AWS_REGION}

```

## Set up a hosted zone
```bash
EXTERNAL_DNS_HOSTZONE=external-dns-test.ruiliang-zhy.com
aws route53 create-hosted-zone --name ${EXTERNAL_DNS_HOSTZONE} --caller-reference "external-dns-test-$(date +%s)" --endpoint-url https://route53.amazonaws.com.cn --region cn-northwest-1

HOSTZONE_ID=$(aws route53 list-hosted-zones-by-name --output json --dns-name ${EXTERNAL_DNS_HOSTZONE} --endpoint-url https://route53.amazonaws.com.cn --region cn-northwest-1 | jq -r '.HostedZones[0].Id')
echo ${HOSTZONE_ID}

aws route53 list-resource-record-sets --output json --hosted-zone-id ${HOSTZONE_ID} \
    --query "ResourceRecordSets[?Type == 'NS']" --endpoint-url https://route53.amazonaws.com.cn --region cn-northwest-1 | jq -r '.[0].ResourceRecords[].Value'
```

## Deploy ExternalDNS

**NOTE:"No OpenIDConnect provider found" error in AWS China regions with IAM Role for Service Account enabled**

可以采用类似 https://github.com/kubernetes-sigs/aws-alb-ingress-controller/issues/1180 添加 环境变量 AWS_REGION解决

**NOTE:level=error msg="RequestError: send request failed\ncaused by: Get https://route53.cn-northwest-1.amazonaws.com.cn/2013-04-01/hostedzone: dial tcp: lookup route53.cn-northwest-1.amazonaws.com.cn on 10.100.0.10:53: no such host"**
原因是 创建 route53 client的时候应该传递 正确的 中国区 endpoint

```go
sess, err := session.NewSession(&aws.Config{
Region: aws.String("cn-northwest-1"),
Endpoint: aws.String("https://route53.amazonaws.com.cn")},
)
client := route53.New(sess)
```

已经有 https://github.com/kubernetes-sigs/external-dns/issues/1544 进行跟踪这个问题

ExternalDNS supported arguments
https://github.com/kubernetes-sigs/external-dns/blob/83850ab21dada9c1200ac2be28e6b3021d46c3a5/pkg/apis/externaldns/types.go#L241

```bash
# Check the RBAC enabled
kubectl api-versions | grep rbac.authorization.k8s.io

# 添加环境变量
env:
    - name: AWS_REGION
    value: cn-northwest-1

# 部署ExternalDNS
kubectl apply -f resource/external-dns/external-dns-deploy.yaml

kubectl get pods

kubectl logs $(kubectl get pods| egrep -o "external-dns[a-zA-Z0-9-]+")
```

## Verify ExternalDNS works (Service example)
```bash
kubectl apply -f resource/external-dns/external-dns-service-demo.yaml
kubectl get pods

aws route53 list-resource-record-sets --output json --hosted-zone-id ${HOSTZONE_ID} \
    --query "ResourceRecordSets[?Name == 'nginx.external-dns-test.ruiliang-zhy.com']|[?Type == 'A']" \
    --endpoint-url https://route53.amazonaws.com.cn --region cn-northwest-1

curl nginx.external-dns-test.ruiliang-zhy.com

```

## Verify ExternalDNS works (Ingress example)

```bash
kubectl apply -f resource/external-dns/external-dns-alb-ingress-demo.yaml
kubectl get pods

aws route53 list-resource-record-sets --output json --hosted-zone-id ${HOSTZONE_ID} \
    --query "ResourceRecordSets[?Name == 'alb-nginx-ingress.external-dns-test.ruiliang-zhy.com']|[?Type == 'A']" \
    --endpoint-url https://route53.amazonaws.com.cn --region cn-northwest-1

curl alb-nginx-ingress.external-dns-test.ruiliang-zhy.com
```


## Cleanup
```bash
kubectl delete -f resource/external-dns/external-dns-service-demo.yaml
kubectl delete -f resource/external-dns/external-dns-alb-ingress-demo.yaml
kubectl delete -f resource/external-dns/external-dns-deploy.yaml
```