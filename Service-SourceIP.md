# Service Overview
Applications running in a Kubernetes cluster find and communicate with each other, and the outside world, through the Service abstraction. 
This document explains what happens to the source IP of packets sent to different types of Services, and how you can toggle this behavior according to your needs

> Objectives
1. Expose a simple application through various types of Services
2. Understand how each Service type handles source IP NAT
3. Understand the tradeoffs involved in preserving source IP

## Prerequisites
Deploy a sample app use a small nginx webserver that echoes back the source IP of requests it receives through an HTTP header
```bash
kubectl get nodes
NAME                                                STATUS   ROLES    AGE    VERSION
ip-192-168-14-19.cn-northwest-1.compute.internal    Ready    <none>   6d2h   v1.14.9-eks-1f0ca9
ip-192-168-40-132.cn-northwest-1.compute.internal   Ready    <none>   6d2h   v1.14.9-eks-1f0ca9
ip-192-168-86-36.cn-northwest-1.compute.internal    Ready    <none>   5d4h   v1.14.9-eks-1f0ca9

# deployment
kubectl create deployment source-ip-app --image=k8s.gcr.io/echoserver:1.4
deployment.apps/source-ip-app created
```


## Source IP for Services with Type=ClusterIP

* Packets sent to ClusterIP from within the cluster are never SNAT. 
* The client pod IP is always the same as pod’s IP accepted by server pod, whether the client pod and server pod are in the same node or in different nodes.

1. Get the pod IP
```bash
SOURCE_IP_APP=$(kubectl get pods | egrep -o source-ip-app[a-zA-Z0-9-]+)
kubectl get pod ${SOURCE_IP_APP} -o wide
NAME                             READY   STATUS    RESTARTS   AGE     IP               NODE                                               NOMINATED NODE   READINESS GATES
source-ip-app-7b999b975f-cndmh   1/1     Running   0          8m30s   192.168.29.229   ip-192-168-14-19.cn-northwest-1.compute.internal   <none>           <none>
```

2. Creating a Service over the source IP app
```bash
kubectl expose deployment source-ip-app --name=clusterip --port=80 --target-port=8080
service/clusterip exposed

kubectl get svc clusterip
NAME        TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)   AGE
clusterip   ClusterIP   10.100.68.218   <none>        80/TCP    32s
```
From the output, we can find the service only have the cluster IP without External IP which means service can not be accessible from external, but can be accessed within cluster.

3. Hitting the ClusterIP of sample service from a pod in the same cluster
```bash
kubectl run busybox -it --image=busybox --restart=Never --rm

/ # ip addr
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
3: eth0@if26: <BROADCAST,MULTICAST,UP,LOWER_UP,M-DOWN> mtu 9001 qdisc noqueue
    link/ether 5a:4c:35:b1:6c:57 brd ff:ff:ff:ff:ff:ff
    inet 192.168.67.163/32 brd 192.168.67.163 scope global eth0
       valid_lft forever preferred_lft forever

/ # wget -qO - 10.100.68.218
CLIENT VALUES:
client_address=192.168.67.163
....
request_uri=http://10.100.68.218:8080/

....
HEADERS RECEIVED:
connection=close
host=10.100.68.218
user-agent=Wget

# From other terminal
kubectl get pod busybox -o wide
NAME      READY   STATUS    RESTARTS   AGE   IP               NODE                                               NOMINATED NODE   READINESS GATES
busybox   1/1     Running   0          17s   192.168.67.163   ip-192-168-86-36.cn-northwest-1.compute.internal   <none>           <none>
88e9fe7b8a30:aws-instance-scheduler ruiliang$
```

*The client_address is always the client pod’s IP address, whether the client pod and server pod are in the same node or in different nodes.*


## Source IP for Services with Type=NodePort
The traffic packets sent to Services with Type=NodePort are SNAT by default. (> Kubernetes 1.5)
```bash
kubectl expose deployment source-ip-app --name=nodeport --port=80 --target-port=8080 --type=NodePort
service/nodeport exposed

NODEPORT=$(kubectl get -o jsonpath="{.spec.ports[0].nodePort}" services nodeport)
NODES=$(kubectl get nodes -o jsonpath='{ $.items[*].status.addresses[?(@.type=="ExternalIP")].address }')

# open up a security group for the nodeport
for node in $NODES; do curl -s $node:$NODEPORT | grep -i client_address; done
client_address=192.168.14.19
client_address=192.168.40.132
client_address=192.168.86.36

kubectl get nodes -o jsonpath='{ $.items[*].status.addresses[?(@.type=="ExternalIP")].address }'
52.82.8.57 161.189.48.222 161.189.112.149

kubectl get nodes -o jsonpath='{ $.items[*].status.addresses[?(@.type=="InternalIP")].address }'
192.168.14.19 192.168.40.132 192.168.86.36
# We can see the client_address SNAT to Node InternalIP

#kubectl exec busybox -it -- /bin/sh
```

## Source IP for Services with Type=LoadBalancer

As of Kubernetes 1.5, traffic packets sent to Services with Type=LoadBalancer are SNAT by default. All schedulable nodes in the Ready state are eligible for loadbalanced traffic. If packets arrive at a node without an endpoint, the system proxies it to a node with an endpoint, replacing the source IP on the packet with the IP of the node.

```bash
kubectl expose deployment source-ip-app --name=loadbalancer --port=80 --target-port=8080 --type=LoadBalancer
service/loadbalancer exposed

kubectl get svc loadbalancer -o wide
NAME           TYPE           CLUSTER-IP      EXTERNAL-IP                                                                       PORT(S)        AGE   SELECTOR
loadbalancer   LoadBalancer   10.100.81.104   a26ce812c622211eaab7702aadf7bd76-1782920587.cn-northwest-1.elb.amazonaws.com.cn   80:32555/TCP   31s   app=source-ip-app

curl a26ce812c622211eaab7702aadf7bd76-1782920587.cn-northwest-1.elb.amazonaws.com.cn
CLIENT VALUES:
client_address=192.168.40.132
command=GET
real path=/
....
request_uri=http://a26ce812c622211eaab7702aadf7bd76-1782920587.cn-northwest-1.elb.amazonaws.com.cn:8080/

....
host=a26ce812c622211eaab7702aadf7bd76-1782920587.cn-northwest-1.elb.amazonaws.com.cn

# curl multiple times, you can find all client_address
curl a26ce812c622211eaab7702aadf7bd76-1782920587.cn-northwest-1.elb.amazonaws.com.cn | grep client_address
client_address=192.168.14.19
client_address=192.168.40.132
client_address=192.168.86.36

kubectl get pod -o wide $(kubectl get pods | egrep -o source-ip-app[a-zA-Z0-9-]+)
NAME                             READY   STATUS    RESTARTS   AGE     IP               NODE                                               NOMINATED NODE   READINESS GATES
source-ip-app-7b999b975f-cndmh   1/1     Running   0          6h12m   192.168.29.229   ip-192-168-14-19.cn-northwest-1.compute.internal   <none>           <none>
```


## Cleaning up
```bash
# Delete the Services:
kubectl delete svc -l app=source-ip-app
service "clusterip" deleted
service "loadbalancer" deleted
service "nodeport" deleted

# Delete the Deployment, ReplicaSet and Pod:
kubectl delete deployment source-ip-app
deployment.extensions "source-ip-app" deleted
```