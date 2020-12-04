https://logz.io/blog/a-practical-guide-to-kubernetes-logging/

做统一日志采集吗？其实建议用Fluentd 
https://eksworkshop.com/intermediate/230_logging/

https://kubernetes.io/docs/concepts/cluster-administration/logging/


journald logging driver sends container logs to the systemd journal. 这个把日志直接打到host上面，其实不如直接采走好。而且journald 是docker世界的东西
一般聊在K8S Node logging的时候会提及 journald。By default, if a container restarts, the kubelet keeps one terminated container with its logs.


OK，我们考虑怎么把cluster-autoscaler日志收集到ES里，目前最佳方案应该是怎么让该autoscaler把日志发送到logstash日志收集器

"梅秀清 MaxQ: 「梁睿 Ray SWAT：journald logging driver sends container logs to the systemd journal. 这个把日志直接打到host上面，其实不如直接采走好。而且journald 是docker世界的东西
一般聊在K8S Node logging的时候会提及 journald。By default, if a container restarts, the kubelet keeps one terminated container with its logs.」
- - - - - - - - - - - - - - -
OK，我们考虑怎么把cluster-autoscaler日志收集到ES里，目前最佳方案应该是怎么让该autoscaler把日志发送到logstash日志收集器"
- - - - - - - - - - - - - - -
spec:
      containers:
      - command:
        - ./cluster-autoscaler
        - --cloud-provider=aws
        - --namespace=infra
        - --node-group-auto-discovery=asg:tag=k8s.io/cluster-autoscaler/enabled,k8s.io/cluster-autoscaler/production
        - --logtostderr=true
        - --stderrthreshold=ERROR
        - --v=4

使用 Fluent Bit 实现集中式容器日志记录
https://aws.amazon.com/cn/blogs/china/centralized-container-logging-fluent-bit/

filebeat
https://www.metricfire.com/blog/kubernetes-logging-with-filebeat-and-elasticsearch-part-2
