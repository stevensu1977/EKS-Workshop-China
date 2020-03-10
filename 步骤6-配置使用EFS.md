# 步骤6 配置使用EFS

6.1 创建EFS file system
```bash
# 创建EFS Security group
VPC_ID=$(aws eks describe-cluster --name ${CLUSTER_NAME} --region ${AWS_REGION} --query "cluster.resourcesVpcConfig.vpcId" --output text)
VPC_CIDR=$(aws ec2 describe-vpcs --vpc-ids ${VPC_ID} --query "Vpcs[].CidrBlock"  --region ${AWS_REGION} --output text)
aws ec2 create-security-group --description ${CLUSTER_NAME}-efs-eks-sg --group-name efs-sg --vpc-id ${VPC_ID}
SGGroupID=上一步的结果访问
aws ec2 authorize-security-group-ingress --group-id ${SGGroupID}  --protocol tcp --port 2049 --cidr ${VPC_CIDR}

# 创建EFS file system 和 mount-target
aws efs create-file-system --creation-token eks-efs --region ${AWS_REGION}
aws efs create-mount-target --file-system-id FileSystemId --subnet-id SubnetID --security-group SGGroupID

```

6.2 在EKS中使用EFS

6.2.1: 选项1 efs-provisioner 

[官方指导手册](https://aws.amazon.com/premiumsupport/knowledge-center/eks-pods-efs/)

6.2.1.1 配置StorageClass and ConfigMap
```bash
# 获取EFS file.system.id 
mkdir -p eks-efs/efs-provisioner && cd efs-provisioner
aws efs describe-file-systems --query "FileSystems[*].[FileSystemId,Name]" --region ${AWS_REGION} --output text

# 获取 StorageClass and ConfigMap
wget https://raw.githubusercontent.com/kubernetes-incubator/external-storage/master/aws/efs/deploy/class.yaml
wget https://raw.githubusercontent.com/kubernetes-incubator/external-storage/master/aws/efs/deploy/configmap.yaml

# 编辑 configmap.yaml 中的 file.system.id and aws.region

# apply the StorageClass and ConfigMap to your cluster
kubectl apply -f class.yaml
kubectl apply -f configmap.yaml
```

6.2.1.2 配置 Deployment and ClusterRole 
```bash
# 获取 Deployment and ClusterRole 
wget https://raw.githubusercontent.com/kubernetes-incubator/external-storage/master/aws/efs/deploy/deployment.yaml
wget https://raw.githubusercontent.com/kubernetes-incubator/external-storage/master/aws/efs/deploy/rbac.yaml


# 编辑 deployment.yaml file
# change the volumes configuration to a path of / and file-system dynamic-provisioning.
# This allows the efs-provisioner to create child directories to back each persistent volume that it provisions at the root of the EFS volume

      volumes:
        - name: pv-volume
          nfs:
            server: fs-72d63897.efs.cn-northwest-1.amazonaws.com.cn
            path: /

# apply the RBAC resources and the Deployment
kubectl apply -f rbac.yaml
kubectl apply -f deployment.yaml
```

6.2.1.3 测试验证
```bash
# 获取 测试样例 and persistent volume claim 
wget https://raw.githubusercontent.com/kubernetes-incubator/external-storage/master/aws/efs/deploy/claim.yaml
wget https://raw.githubusercontent.com/kubernetes-incubator/external-storage/master/aws/efs/deploy/test-pod.yaml
kubectl apply -f claim.yaml
# https://github.com/kubernetes-incubator/external-storage/issues/1293
ProvisioningFailed     persistentvolumeclaim/efs  Error creating provisioned PV object for claim default/efs: PersistentVolume "pvc-76b7ee16-5fc4-11ea-ab77-02aadf7bd768" is invalid: spec.nfs.path: Invalid value: "fs-72d63897.efs.cn-northwest-1.amazonaws.com.cn:/efs-pvc-76b7ee16-5fc4-11ea-ab77-02aadf7bd768": must be an absolute path. Deleting the volume.

# 修改 test-pod.yaml 
"touch /mnt/SUCCESS && exit 0 || exit 1"
#改为
"while true; do echo $(date -u) >> /mnt/out.txt; sleep 5; done"

# 部署并验证
kubectl apply -f test-pod.yaml
kubectl exec -it test-pod cat /mnt/out.txt

# 清理
kubectl delete -f test-pod.yaml
kubectl delete -f claim.yaml
kubectl delete -f deployment.yaml
kubectl delete -f rbac.yaml
kubectl delete -f configmap.yaml
kubectl delete -f class.yaml
```


6.2.2： 选项2 efs-csi

[官方文档]（https://docs.aws.amazon.com/zh_cn/eks/latest/userguide/efs-csi.html）

```bash
git clone https://github.com/kubernetes-sigs/aws-efs-csi-driver.git
cd aws-efs-csi-driver
```

6.2.2.1 Deploy EFS CSI driver to EKS cluster 

> 已知问题：

https://github.com/kubernetes-sigs/aws-efs-csi-driver/issues/138

v0.2.0 image contains old version of efs-utils, efs-utils added China region support from v1.19
The v.0.3.0 does work, you can also use the awsguru/aws-efs-csi-driver/v0.2.1

```bash
# 修改 aws-efs-csi-driver/deploy/kubernetes/overlays/stable/kustomization.yaml
# 1. latest tag
# 2. image location for China region
kubectl apply -k ./deploy/kubernetes/overlays/stable
kubectl get pods -n kube-system

NAME                                      READY   STATUS    RESTARTS   AGE
alb-ingress-controller-649b854d75-m8c75   1/1     Running   0          2d18h
aws-node-ct6rz                            1/1     Running   0          4d18h
aws-node-sfjtn                            1/1     Running   0          3d21h
aws-node-xzfx9                            1/1     Running   0          4d18h
coredns-6565755d58-pd5nm                  1/1     Running   0          4d18h
coredns-6565755d58-v9nl7                  1/1     Running   0          4d18h
ebs-csi-controller-6dcc4dc6f4-6k4s5       4/4     Running   0          2d17h
ebs-csi-controller-6dcc4dc6f4-vtklz       4/4     Running   0          2d17h
ebs-csi-node-2zmct                        3/3     Running   0          2d17h
ebs-csi-node-plljf                        3/3     Running   0          2d17h
ebs-csi-node-s9lbz                        3/3     Running   0          2d17h
efs-csi-node-5jtlc                        3/3     Running   0          10h
efs-csi-node-lqdz9                        3/3     Running   0          10h
efs-csi-node-snqmh                        3/3     Running   0          10h
kube-proxy-g4mcw                          1/1     Running   0          4d18h
kube-proxy-mb88w                          1/1     Running   0          4d18h
kube-proxy-tpx4x                          1/1     Running   0          3d21h
kubernetes-dashboard-5f7b999d65-dcc6h     1/1     Running   0          2d23h
metrics-server-7fcf9cc98b-rntrh           1/1     Running   0          44h

kubectl exec -ti efs-csi-node-5jtlc -n kube-system -- mount.efs --version
# Make sure the version is > 1.19
# If your version is < 1.19, update the efs-utils
kubectl exec -ti efs-csi-node-5jtlc -n kube-system -- /bin/bash
yum update -y amazon-efs-utils && mount.efs --version
```

6.2.2.2 部署样例测试
```bash
## Deploy app use the EFS
cd examples/kubernetes/multiple_pods/
aws efs describe-file-systems --query "FileSystems[*].[FileSystemId,Name]" --region ${AWS_REGION} --output text

# 修改 the specs/pv.yaml file and replace the volumeHandle with FILE_SYSTEM_ID

# 部署 the efs-sc storage class, efs-claim pv claim, efs-pv, and app1 and app2 sample applications.
kubectl apply -f specs/

kubectl describe storageclass efs-sc
kubectl get pv
kubectl describe pv efs-pv
kubectl get pods --watch
kubectl get events

# 验证
kubectl exec -ti app1 -- tail /data/out1.txt
kubectl exec -ti app2 -- tail /data/out1.txt

# 清理
kubectl delete -f specs/
```
