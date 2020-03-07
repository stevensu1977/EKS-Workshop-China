Docker / Kubernetes 镜像源
https://cloud.tencent.com/developer/article/1499649
http://mirror.azure.cn/help/gcr-proxy-cache.html

下面我们以拉取 mysql:5.7 和 360cloud/wayne 为例：
# 使用中科大镜像源 
$ docker pull docker.mirrors.ustc.edu.cn/library/mysql:5.7
$ docker pull docker.mirrors.ustc.edu.cn/360cloud/wayne

# 使用 Azure 中国镜像源
$ docker pull dockerhub.azk8s.cn/library/mysql:5.7
$ docker pull dockerhub.azk8s.cn/360cloud/wayne

下面我们以拉取 gcr.io/kubernetes-helm/tiller:v2.9.1 为例：
# helm使用中科大镜像源 
$ docker pull gcr.mirrors.ustc.edu.cn/kubernetes-helm/tiller:v2.9.1

# 使用 Azure 中国镜像源
$ docker pull gcr.azk8s.cn/kubernetes-helm/tiller:v2.9.1



下面我们以拉取 k8s.gcr.io/addon-resizer:1.8.3 为例：
# 使用中科大镜像源 
$ docker pull gcr.mirrors.ustc.edu.cn/google-containers/addon-resizer:1.8.3

# 使用 Azure 中国镜像源
$ docker pull gcr.azk8s.cn/google-containers/addon-resizer:1.8.3


下面我们以拉取 quay.io/coreos/kube-state-metrics:v1.5.0 为例：
# 使用中科大镜像源 
$ docker pull quay.mirrors.ustc.edu.cn/coreos/kube-state-metrics:v1.5.0

# 使用 Azure 中国镜像源
$ docker pull quay.azk8s.cn/coreos/kube-state-metrics:v1.5.0


docker-wrapper
一个 Python 编写的工具脚本，可以替代系统的 Docker 命令，自动从 Azure 中国拉取镜像并自动 Tag 为目标镜像和删除 Azure 镜像，一气呵成。

项目地址：https://github.com/silenceshell/docker_wrapper

docker-wrapper 安装

$ git clone https://github.com/silenceshell/docker-wrapper.git
$ sudo cp docker-wrapper/docker-wrapper.py /usr/local/bin/
docker-wrapper 使用

$ docker-wrapper pull k8s.gcr.io/kube-apiserver:v1.14.1
$ docker-wrapper pull gcr.io/google_containers/kube-apiserver:v1.14.1
$ docker-wrapper pull nginx
$ docker-wrapper pull silenceshell/godaddy:0.0.2
azk8spull
一个 Shell 编写的脚本，这个脚本功能和 docker-wrapper 类似。同样可以自动从 Azure 中国拉取镜像并自动 Tag 为目标镜像和删除 Azure 镜像。

项目地址：https://github.com/xuxinkun/littleTools#azk8spull


azk8spull 安装

$ git clone https://github.com/xuxinkun/littleTools
$ cd littleTools
$ chmod +x install.sh
$ ./install.sh
azk8spull 使用

$ azk8spull quay.io/kubernetes-ingress-controller/nginx-ingress-controller:0.24.1
$ azk8spull k8s.gcr.io/pause-amd64:3.1