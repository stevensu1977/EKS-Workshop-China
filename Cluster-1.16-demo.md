1. Install the 1.16 cluster
```bash
# upgrade the eksctl and kubectl
brew upgrade weaveworks/tap/eksctl
brew upgrade kubectl

# create the 1.16 cluster
eksctl create cluster --name=eks-xray-demo --version 1.16 --nodegroup-name standard-workers \
 --nodes=2 --node-type t3.medium --managed --alb-ingress-access --region=${AWS_REGION}
```

2. 使用 Kubernetes webhook 自动更换 Kubernetes Pod 的容器镜像
```bash
kubectl apply -f https://raw.githubusercontent.com/nwcdlabs/container-mirror/master/webhook/mutating-webhook.yaml

kubectl run --generator=run-pod/v1 test --image=k8s.gcr.io/coredns:1.3.1
kubectl get pod test -o=jsonpath='{.spec.containers[0].image}'
# 结果应显示为048912060910.dkr.ecr.cn-northwest-1.amazonaws.com.cn/gcr/google_containers/coredns:1.3.1

# 清理
kubectl delete pod test
```

3. Microservice
```bash
git clone https://github.com/brentley/ecsdemo-frontend.git
git clone https://github.com/brentley/ecsdemo-nodejs.git

cd ecsdemo-nodejs
kubectl apply -f kubernetes/deployment.yaml
kubectl apply -f kubernetes/service.yaml
kubectl get deployment ecsdemo-nodejs

cd ../ecsdemo-frontend
kubectl apply -f kubernetes/deployment.yaml
kubectl apply -f kubernetes/service.yaml
kubectl get deployment ecsdemo-frontend

ELB=$(kubectl get service ecsdemo-frontend -o json | jq -r '.status.loadBalancer.ingress[].hostname')
echo ${ELB}
# open browser to access ${ELB}

# cleanup
kubectl delete -f kubernetes/service.yaml
kubectl delete -f kubernetes/deployment.yaml
cd ../ecsdemo-nodejs
kubectl delete -f kubernetes/service.yaml
kubectl delete -f kubernetes/deployment.yaml
kubectl get pods
```

4. Deploy ALB ingress controller
```bash
eksctl utils associate-iam-oidc-provider --cluster=eks-xray-demo --approve --region ${AWS_REGION}
POLICY_NAME=$(aws iam list-policies --query 'Policies[?PolicyName==`ALBIngressControllerIAMPolicy`].Arn' --output text --region ${AWS_REGION})
echo $POLICY_NAME

eksctl create iamserviceaccount --cluster=eks-xray-demo \
       --namespace=kube-system --name=alb-ingress-controller \
       --attach-policy-arn=$POLICY_NAME --override-existing-serviceaccounts \
       --region cn-northwest-1 --approve

wget -O rbac-role1.1.7.yaml  https://raw.githubusercontent.com/kubernetes-sigs/aws-alb-ingress-controller/v1.1.7/docs/examples/rbac-role.yaml
kubectl apply -f rbac-role1.1.7.yaml

wget -O alb-ingress-controller1.1.7.yaml https://raw.githubusercontent.com/kubernetes-sigs/aws-alb-ingress-controller/v1.1.7/docs/examples/alb-ingress-controller.yaml
#修改以下内容
  spec:
    containers:
      - name: alb-ingress-controller
        args:
          - --cluster-name=<步骤2 创建的集群名字>
          - --aws-vpc-id=<eksctl 创建的vpc-id>   
          - --aws-region=cn-northwest-1
          - --feature-gates=waf=false,wafv2=false
  env:
    - name: AWS_REGION
      value: cn-northwest-1
    
kubectl apply -f alb-ingress-controller1.1.7.yaml

kubectl logs -n kube-system $(kubectl get po -n kube-system | egrep -o "alb-ingress[a-zA-Z0-9-]+")
-------------------------------------------------------------------------------
AWS ALB Ingress controller
  Release:    v1.1.7
  Build:      git-8694851d
  Repository: https://github.com/kubernetes-sigs/aws-alb-ingress-controller.git
-------------------------------------------------------------------------------
....
I0520 14:25:21.887900       1 controller.go:154] kubebuilder/controller "level"=0 "msg"="Starting workers"  "controller"="alb-ingress-controller" "worker count"=1

kubectl apply -f nginx-alb-ingress.yaml

ALB=$(kubectl get ingress -o json | jq -r '.items[0].status.loadBalancer.ingress[].hostname')
curl -m3 -v $ALB

# 如果遇到问题，请查看日志
kubectl logs -n kube-system $(kubectl get po -n kube-system | egrep -o "alb-ingress[a-zA-Z0-9-]+")

# cleanup
kubectl delete -f nginx-alb-ingress.yaml

kubectl apply -f https://raw.githubusercontent.com/kubernetes-sigs/aws-alb-ingress-controller/v1.0.0/docs/examples/2048/2048-namespace.yaml
kubectl apply -f https://raw.githubusercontent.com/kubernetes-sigs/aws-alb-ingress-controller/v1.0.0/docs/examples/2048/2048-deployment.yaml
kubectl get pods -n 2048-game
kubectl apply -f https://raw.githubusercontent.com/kubernetes-sigs/aws-alb-ingress-controller/v1.0.0/docs/examples/2048/2048-service.yaml
kubectl get service service-2048 -o wide -n 2048-game
kubectl apply -f https://raw.githubusercontent.com/kubernetes-sigs/aws-alb-ingress-controller/v1.0.0/docs/examples/2048/2048-ingress.yaml

# 获取访问地址，在浏览器中访问2048游戏
kubectl get ingress/2048-ingress -n 2048-game

kubectl delete -f https://raw.githubusercontent.com/kubernetes-sigs/aws-alb-ingress-controller/v1.0.0/docs/examples/2048/2048-deployment.yaml
kubectl delete -f https://raw.githubusercontent.com/kubernetes-sigs/aws-alb-ingress-controller/v1.0.0/docs/examples/2048/2048-service.yaml
kubectl delete -f https://raw.githubusercontent.com/kubernetes-sigs/aws-alb-ingress-controller/v1.0.0/docs/examples/2048/2048-ingress.yaml
kubectl delete -f https://raw.githubusercontent.com/kubernetes-sigs/aws-alb-ingress-controller/v1.0.0/docs/examples/2048/2048-namespace.yaml
```

5. Kubernetes Dashboard