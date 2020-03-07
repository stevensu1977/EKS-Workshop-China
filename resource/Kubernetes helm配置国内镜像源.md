删除默认的源
helm repo remove stable

增加新的国内镜像源
helm repo add stable http://mirror.azure.cn/kubernetes/charts/
helm repo add stable https://burdenbear.github.io/kube-charts-mirror/

查看helm源添加情况
helm repo list

安装好 Helm 后，通过键入如下命令，在 Kubernetes 群集上安装 Tiller：
helm init --upgrade -i registry.cn-hangzhou.aliyuncs.com/google_containers/tiller:v2.5.1 --stable-repo-url https://kubernetes.oss-cn-hangzhou.aliyuncs.com/charts


通过 Helm 部署 WordPress
helm install --name wordpress-test --set "persistence.enabled=false,mariadb.persistence.enabled=false" stable/wordpress
echo http://$(kubectl get svc wordpress-test-wordpress -o jsonpath='{.status.loadBalancer.ingress[0].ip}')
echo Username: user
echo Password: $(kubectl get secret --namespace default wordpress-test-wordpress -o jsonpath="{.data.wordpress-password}" | base64 --decode)