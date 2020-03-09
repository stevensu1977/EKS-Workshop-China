# 步骤5 配置使用Amazon EFS卷

5.1 创建EFS文件系统

>5.1.1 查找EKS集群使用的VPC 使用的CIDR

```bash
EKS_VPC=$(aws eks describe-cluster --name eksworkshop --query "cluster.resourcesVpcConfig.vpcId" --output text)

aws ec2 describe-vpcs --vpc-ids $EKS_VPC --query "Vpcs[].CidrBlock" --output text


```
>5.1.2 创建EFS需要的安全组
添加一个名字为sg-nfs的安全组，并且添加一个类型为NFS,源为上面VPC的CIDR的入站规则，允许EKS的工作节点访问EFS
![-w676](media/15836602498237/15837596877215.jpg)

>5.1.3 创建EFS文件系统
安全组请选择sg-nfs
![-w881](media/15836602498237/15837598666553.jpg)
并获取EFS id (fs-****)
![-w445](media/15836602498237/15837599707448.jpg)

5.2 部署EFS驱动和示例程序
> 5.2.3 部署EFS CSI 驱动到eks 集群

```bash
#使用EFS CSI v0.3.0 镜像
kubectl apply -k ./resource/aws-efs-csi-driver/deploy/kubernetes/overlays/stable

```
>5.2.4 部署动态卷实例应用
请修改./resource/aws-efs-csi-driver/examples/kubernetes/multiple_pods/specs/pv.yaml 修改volumeHandle为你创建的EFS id (fs-**** )

```bash
#vi pv.yaml
#csi:
#    driver: efs.csi.aws.com
#    volumeHandle: fs-9c21a579
    
vi ./resource/aws-efs-csi-driver/examples/kubernetes/multiple_pods/specs/pv.yaml

kubectl apply -f ./resource/aws-efs-csi-driver/examples/kubernetes/multiple_pods/specs/

#查看storageclass
kubectl describe storageclass efs-sc

#查看示例app状态
kubectl get pods --watch

kubectl get pv

kubectl exec -it app cat /data/out.txt

#删除示例程序
kubectl delete -f specs/
```