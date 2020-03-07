# 步骤7 在EKS中使用IAM Role进行权限管理
我们将要为ServiceAccount配置一个S3的访问角色，并且部署一个job应用到EKS集群，完成S3的写入。

[官方文档](https://aws.amazon.com/blogs/opensource/introducing-fine-grained-iam-roles-service-accounts/)

7.1 配置IAM Role、ServiceAccount

>7.1.1 使用eksctl 创建service account 

```bash
# 在步骤3我们已经创建了OIDC身份提供商 
# 请检查IAM Open ID Connect provider已经创建
aws eks describe-cluster --name ${CLUSTER_NAME} --region ${AWS_REGION} --query cluster.identity.oidc.issuer --output text
# 如果上述命令无输出，run below command to create one
eksctl utils associate-iam-oidc-provider --cluster=${CLUSTER_NAME} --approve --region ${AWS_REGION}

#创建serviceaccount s3-echoer with IAM role
eksctl create iamserviceaccount --name s3-echoer --namespace default \
    --cluster ${CLUSTER_NAME} --attach-policy-arn arn:aws-cn:iam::aws:policy/AmazonS3FullAccess \
    --approve --override-existing-serviceaccounts --region ${AWS_REGION}

```

>7.1.2 修复eksctl 创建的Role
参考4.2.1.4 (可选）eksctl 0.15-rc.0 已知issue 处理
https://github.com/weaveworks/eksctl/issues/1871, 需要手动修复。eksctl 0.15-rc.1已经修复这个问题


7.2 部署测试应用
*请确保bucket名字唯一,s3 bucket才能创建成功

```bash
git clone https://github.com/mhausenblas/s3-echoer.git && cd s3-echoer

# 准备S3 bucket
TARGET_BUCKET=ray-gcr-eksworkshop-irsa-2019
if [ $(aws s3 ls | grep $TARGET_BUCKET | wc -l) -eq 0 ]; then
    aws s3api create-bucket  --bucket $TARGET_BUCKET  --create-bucket-configuration LocationConstraint=$AWS_REGION  --region $AWS_REGION
else
    echo "S3 bucket $TARGET_BUCKET existed, skip creation"
fi

# 部署
sed -e "s/TARGET_BUCKET/${TARGET_BUCKET}/g;s/us-west-2/${AWS_REGION}/g" s3-echoer-job.yaml.template > s3-echoer-job.yaml
kubectl apply -f s3-echoer-job.yaml

# 验证
kubectl get job/s3-echoer
kubectl logs job/s3-echoer
## 参考输出
Uploading user input to S3 using ray-gcr-eksworkshop-irsa-2019/s3echoer-1583415691

# 检查S3 bucket上面的文件
aws s3api list-objects --bucket $TARGET_BUCKET --query 'Contents[].{Key: Key, Size: Size}'  --region $AWS_REGION
[
    {
        "Key": "s3echoer-1583415691",
        "Size": 27
    }
]

#清理
kubectl delete job/s3-echoer
```
