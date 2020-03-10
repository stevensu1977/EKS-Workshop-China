# Configure Containers Using a ConfigMap

> Objectives
1. Create a kustomization.yaml file containing:
    1.1 a ConfigMap generator
    1.2 a Pod resource config using the ConfigMap
2. Apply the directory by running kubectl apply -k ./
3. Verify that the configuration was correctly applied.

## Configuring Redis using a ConfigMap
我们将学习如何利用ConfigMap配置一个 Redis 服务

1. Prepare
```bash
mkdir -p configuremap-demo && cd configuremap-demo
curl -OL https://k8s.io/examples/pods/config/redis-config
> cat redis-config
maxmemory 2mb
maxmemory-policy allkeys-lru
```

2. Create a kustomization.yaml containing a ConfigMap from the redis-config file
```bash
cat <<EOF >./kustomization.yaml
configMapGenerator:
- name: example-redis-config
  files:
  - redis-config
EOF
```

3. Create redis-pod.yam

```bash
curl -OL https://raw.githubusercontent.com/kubernetes/website/master/content/en/examples/pods/config/redis-pod.yaml

cat <<EOF >>./kustomization.yaml
resources:
- redis-pod.yaml
EOF

> cat redis-pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: redis
spec:
  containers:
  - name: redis
    image: redis:5.0.4
    command:
      - redis-server
      - "/redis-master/redis.conf"
    env:
    - name: MASTER
      value: "true"
    ports:
    - containerPort: 6379
    resources:
      limits:
        cpu: "0.1"
    volumeMounts:
    - mountPath: /redis-master-data
      name: data
    - mountPath: /redis-master
      name: config
  volumes:
    - name: data
      emptyDir: {}
    - name: config
      configMap:
        name: example-redis-config
        items:
        - key: redis-config
          path: redis.conf
```

4. Apply the kustomization directory to create both the ConfigMap and Pod objects
```bash
kubectl apply -k .
configmap/example-redis-config-dgh9dg555m created
pod/redis created

kubectl get -k .
NAME                                        DATA   AGE
configmap/example-redis-config-dgh9dg555m   1      20m

NAME        READY   STATUS              RESTARTS   AGE
redis                         1/1     Running   0          20m

# verify redis work properly
kubectl exec -it redis redis-cli
127.0.0.1:6379> CONFIG GET maxmemory
1) "maxmemory"
2) "2097152"
127.0.0.1:6379> CONFIG GET maxmemory-policy
1) "maxmemory-policy"
2) "allkeys-lru"

# clean up
kubectl delete -k .
```