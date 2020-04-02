# Helm部署Prometheus Operator和自定义监控

- ### 安装

建议将Prometheus Operator部署在专门的命名空间中，一般为monitoring。

    kubectl create namespace monitoring

Helm v3安装时，在crds/目录中的清单文件会自动提交给Kubernetes。

    helm install prometheus stable/prometheus-operator \
      --namespace monitoring \
      --set prometheusOperator.createCustomResource=false \
      --set prometheusOperator.cleanupCustomResource=true

查看Kubernetes资源。

    kubectl --namespace monitoring get all
    NAME                                                         READY   STATUS    RESTARTS   AGE
    pod/alertmanager-prometheus-prometheus-oper-alertmanager-0   2/2     Running   0          4m20s
    pod/prometheus-grafana-dc56bc899-vprqs                       2/2     Running   0          4m56s
    pod/prometheus-kube-state-metrics-67b765f8b8-wblcd           1/1     Running   0          4m56s
    pod/prometheus-prometheus-node-exporter-fxl6j                1/1     Running   0          4m56s
    pod/prometheus-prometheus-node-exporter-r8vhc                1/1     Running   0          4m56s
    pod/prometheus-prometheus-node-exporter-xcgkj                1/1     Running   0          4m56s
    pod/prometheus-prometheus-oper-operator-58566dd678-5c2zm     2/2     Running   0          4m56s
    pod/prometheus-prometheus-prometheus-oper-prometheus-0       3/3     Running   1          4m9s
    
    NAME                                              TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)                      AGE
    service/alertmanager-operated                     ClusterIP   None           <none>        9093/TCP,9094/TCP,9094/UDP   4m20s
    service/prometheus-grafana                        ClusterIP   10.1.45.41     <none>        80/TCP                       4m56s
    service/prometheus-kube-state-metrics             ClusterIP   10.1.35.41     <none>        8080/TCP                     4m56s
    service/prometheus-operated                       ClusterIP   None           <none>        9090/TCP                     4m9s
    service/prometheus-prometheus-node-exporter       ClusterIP   10.1.206.118   <none>        9100/TCP                     4m56s
    service/prometheus-prometheus-oper-alertmanager   ClusterIP   10.1.248.72    <none>        9093/TCP                     4m56s
    service/prometheus-prometheus-oper-operator       ClusterIP   10.1.170.8     <none>        8080/TCP,443/TCP             4m56s
    service/prometheus-prometheus-oper-prometheus     ClusterIP   10.1.132.191   <none>        9090/TCP                     4m56s
    
    NAME                                                 DESIRED   CURRENT   READY   UP-TO-DATE   AVAILABLE   NODE SELECTOR   AGE
    daemonset.apps/prometheus-prometheus-node-exporter   3         3         3       3            3           <none>          4m56s
    
    NAME                                                  READY   UP-TO-DATE   AVAILABLE   AGE
    deployment.apps/prometheus-grafana                    1/1     1            1           4m56s
    deployment.apps/prometheus-kube-state-metrics         1/1     1            1           4m56s
    deployment.apps/prometheus-prometheus-oper-operator   1/1     1            1           4m56s
    
    NAME                                                             DESIRED   CURRENT   READY   AGE
    replicaset.apps/prometheus-grafana-dc56bc899                     1         1         1       4m56s
    replicaset.apps/prometheus-kube-state-metrics-67b765f8b8         1         1         1       4m56s
    replicaset.apps/prometheus-prometheus-oper-operator-58566dd678   1         1         1       4m56s
    
    NAME                                                                    READY   AGE
    statefulset.apps/alertmanager-prometheus-prometheus-oper-alertmanager   1/1     4m20s
    statefulset.apps/prometheus-prometheus-prometheus-oper-prometheus       1/1     4m9s

查看创建的CRDs。

    kubectl get crd | grep coreos
    alertmanagers.monitoring.coreos.com     2020-04-01T01:42:58Z
    podmonitors.monitoring.coreos.com       2020-04-01T01:42:58Z
    prometheuses.monitoring.coreos.com      2020-04-01T01:42:58Z
    prometheusrules.monitoring.coreos.com   2020-04-01T01:42:58Z
    servicemonitors.monitoring.coreos.com   2020-04-01T01:42:59Z

为了在集群外访问Prometheus、Grafana和Alertmanager，我们安装开源网关Ambassador，通过该网关访问。后面我们配置Prometheus监控该网关。

添加Helm仓库，选择`datawire/ambassador` Chart。

    helm repo add datawire https://www.getambassador.io
    helm search repo ambassador
    NAME                            CHART VERSION   APP VERSION     DESCRIPTION
    aliyuncs/ambassador             4.4.7           0.85.0          A Helm chart for Datawire Ambassador
    datawire/ambassador             6.2.2           1.3.1           A Helm chart for Datawire Ambassador
    datawire/ambassador-operator    0.1.0           1.0.0           A Helm chart for Kubernetes
    stable/ambassador               5.3.0           0.86.1          A Helm chart for Datawire Ambassador

安装Ambassador Edge Stack。

    kubectl create namespace ambassador
    
    helm install ambassador datawire/ambassador \
      --namespace ambassador \
      --set authService.create=false \
      --set crds.create=false \
      --set licenseKey.licenseKey=false \
      --set rateLimit.create=false \
      --set service.type=NodePort

> 如果如[Ambassador系列-11-Helm安装Ambassador Edge Stack 1.1.0](../ambassador/11-ambassador-edge-stack-helm-installation.md)一文中提到的方式申请了License，可以在Helm命令行注册该License，licenseKey.value就是邮件中收到的License Key。

    helm install ambassador datawire/ambassador \
      --namespace ambassador \
      --set authService.create=false \
      --set crds.create=false \
      --set licenseKey.secretName=ambassador-edge-stack \
      --set licenseKey.value=eyJhbGciOiJQUzUxXXXXXXXXXXXXXXXXXXXX.eyJsaWNlbnNlX2tleV92ZXJzaW9uIjoidjIiLCJjdXN0b21lcl9pZCI6InR3aW5nYW9Ac2luYS5jbiIsImN1c3RvbWVyX2VtYWlsIjoidHdpbmdhb0BzaW5hLmNuIiwiZW5hYmxlZF9mZWF0dXJlcyI6WyIiLCJmaWx0ZXIiLCJyYXRlbGltaXQiLCJ0cmFmZmljIiwiZGV2cG9ydGFsIl0sImVuZm9yY2VkX2xpbWl0cyI6W3sibCI6ImRldnBvcnRhbC1zZXJ2aWNlcyIsInYiOjV9LHsibCI6InJhdGVsaW1pdC1zZXJ2aWNlIiwidiI6NX0seyJsIjoiYXV0aGZpbHRlci1zZXJ2aWNlIiwidiI6NX1dLCJtZXRhZGF0YSI6e30sImV4cCI6MTYxMjUzMDk4NCwiaWF0IjoxNTgwOTXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX.TKBPqYGitumz5iSbFQq8EN9KN_BAqCJs9x03K6W3WBJxUx4fp3Qc6whjc9lNZgNG6KfUh61DJ9dru8G-90SyjCxvz05QDvAUtL__7PYfTS-17Jq0ZJygOAC8hGtrOz8iCw--oFkAhpZ14mvc0-CpZEn0DgKAHel0WQY7nYGQ6aEh2GYQG80rf3KBSxZwbp-sawBANArwvCvWw1W_5tSpBy3FBG33J0IIb2rS9lAuFr0ZvVdocJr5vIKb1KQAH3Ww9sxLKfFdFOLN_5fUIsFiAOYiPuo0hpQp1BbIllxCYrKAMig3xKRIlJI7Z6C-YySSxBXXXXXXXXXXXXXXXXXXXX \
      --set rateLimit.create=false \
      --set service.type=NodePort

查看Kubernetes资源。

    kubectl get all -nambassador
    NAME                                    READY   STATUS    RESTARTS   AGE
    pod/ambassador-75b5688649-7pnl2         0/1     Running   1          83s
    pod/ambassador-75b5688649-jpvpv         0/1     Running   1          83s
    pod/ambassador-75b5688649-llkn2         0/1     Running   1          83s
    pod/ambassador-redis-8556cbb4c6-ssbt6   1/1     Running   0          83s
    
    NAME                       TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)                      AGE
    service/ambassador         NodePort    10.1.110.49    <none>        80:38024/TCP,443:32271/TCP   83s
    service/ambassador-admin   ClusterIP   10.1.166.205   <none>        8877/TCP                     83s
    service/ambassador-redis   ClusterIP   10.1.193.90    <none>        6379/TCP                     83s
    
    NAME                               READY   UP-TO-DATE   AVAILABLE   AGE
    deployment.apps/ambassador         0/3     3            0           83s
    deployment.apps/ambassador-redis   1/1     1            1           83s
    
    NAME                                          DESIRED   CURRENT   READY   AGE
    replicaset.apps/ambassador-75b5688649         3         3         0       83s
    replicaset.apps/ambassador-redis-8556cbb4c6   1         1         1       83s

通过管理Service端口访问metrics，其实就是envoy的metrics。

    curl http://10.1.166.205:8877/metrics
    # TYPE envoy_cluster_upstream_cx_connect_timeout counter
    envoy_cluster_upstream_cx_connect_timeout{envoy_cluster_name="cluster_127_0_0_1_8877_ambassador"} 0
    # TYPE envoy_cluster_upstream_flow_control_paused_reading_total counter
    envoy_cluster_upstream_flow_control_paused_reading_total{envoy_cluster_name="cluster_127_0_0_1_8877_ambassador"} 0
    # TYPE envoy_cluster_upstream_cx_close_notify counter
    envoy_cluster_upstream_cx_close_notify{envoy_cluster_name="cluster_127_0_0_1_8877_ambassador"} 0
    # TYPE envoy_cluster_lb_recalculate_zone_structures counter
    envoy_cluster_lb_recalculate_zone_structures{envoy_cluster_name="cluster_127_0_0_1_8877_ambassador"} 0
    # TYPE envoy_cluster_upstream_flow_control_resumed_reading_total counter
    envoy_cluster_upstream_flow_control_resumed_reading_total{envoy_cluster_name="cluster_127_0_0_1_8877_ambassador"} 0
    # TYPE envoy_cluster_upstream_rq_timeout counter
    envoy_cluster_upstream_rq_timeout{envoy_cluster_name="cluster_127_0_0_1_8877_ambassador"} 0
    # TYPE envoy_cluster_upstream_cx_connect_fail counter
    envoy_cluster_upstream_cx_connect_fail{envoy_cluster_name="cluster_127_0_0_1_8877_ambassador"} 0
    # TYPE envoy_cluster_upstream_rq_cancelled counter
    envoy_cluster_upstream_rq_cancelled{envoy_cluster_name="cluster_127_0_0_1_8877_ambassador"} 0
    # TYPE envoy_cluster_upstream_cx_rx_bytes_total counter
    envoy_cluster_upstream_cx_rx_bytes_total{envoy_cluster_name="cluster_127_0_0_1_8877_ambassador"} 0
    # TYPE envoy_cluster_upstream_cx_overflow counter
    envoy_cluster_upstream_cx_overflow{envoy_cluster_name="cluster_127_0_0_1_8877_ambassador"} 0
    # TYPE envoy_cluster_upstream_cx_destroy_remote counter
    envoy_cluster_upstream_cx_destroy_remote{envoy_cluster_name="cluster_127_0_0_1_8877_ambassador"} 0
    # TYPE envoy_cluster_upstream_cx_http2_total counter
    envoy_cluster_upstream_cx_http2_total{envoy_cluster_name="cluster_127_0_0_1_8877_ambassador"} 0
    ......

为Prometheus、Grafana和Alertmanager配置Mapping。没有找到Ambassador如何配置重定向后的URL重写，只能按照域名配置路由，但又发现浏览器在host请求头中竟然带了端口号，只能在mapping的host配置中加上端口号，如`prom.twingao.com:32271`。只能将就，留待后续优化，或者有没有人告诉我如何处理？

    vi prometheus-mapping.yaml
    ---
    apiVersion: getambassador.io/v2
    kind: Mapping
    metadata:
      name: prometheus-mapping
      namespace: ambassador
    spec:
      host: prom.twingao.com:32271
      prefix: /
      service: prometheus-prometheus-oper-prometheus.monitoring:9090
    ---
    apiVersion: getambassador.io/v2
    kind: Mapping
    metadata:
      name: grafana-mapping
      namespace: ambassador
    spec:
      host: grafana.twingao.com:32271
      prefix: /
      service: prometheus-grafana.monitoring:80
    ---
    apiVersion: getambassador.io/v2
    kind: Mapping
    metadata:
      name: alert-mapping
      namespace: ambassador
    spec:
      host: alert.twingao.com:32271
      prefix: /
      service: prometheus-prometheus-oper-alertmanager.monitoring:9093
    
    kubectl apply -f prometheus-mapping.yaml

在浏览器主机增加hosts，Windows系统在`C:\Windows\System32\drivers\etc\hosts`。

    # Prometheus Start
    192.168.1.55 prom.twingao.com
    192.168.1.55 grafana.twingao.com
    192.168.1.55 alert.twingao.com
    # Prometheus End

访问Prometheus，并切换到Targets页面，地址[https://prom.twingao.com:32271/targets](https://prom.twingao.com:32271/targets)。没有看出来monitoring/prometheus-prometheus-oper-kube-etcd和monitoring/prometheus-prometheus-oper-kube-proxy出问题的原因，但可以看出使用node地址+Service端口访问，这是无法访问的。

monitoring/prometheus-prometheus-oper-kubelet是通过https-metrics（10250）端口访问的。monitoring/prometheus-prometheus-oper-kube-controller-manager和monitoring/prometheus-prometheus-oper-kube-scheduler已经配置正确。

![](images/prometheus-targets-helm.png)

访问Grafana，缺省密码为`prom-operator`，获取方式：

    helm show values stable/prometheus-operator | grep adminPassword
      adminPassword: prom-operator

Grafana缺省内置了多个dashboard。

![](images/grafana-dashboard-list-helm.png)

访问Alertmanager，地址：[https://alert.twingao.com:32271/#/alerts](https://alert.twingao.com:32271/#/alerts)。

![](images/alert-alert-helm.png)

- ### 监控Ambassador

我们从service/ambassador-admin抓取metrics，查看一下ports。

    kubectl get service/ambassador-admin -oyaml -nambassador
    apiVersion: v1
    kind: Service
    metadata:
      creationTimestamp: "2020-04-01T07:15:16Z"
      labels:
        app.kubernetes.io/instance: ambassador
        app.kubernetes.io/managed-by: Helm
        app.kubernetes.io/name: ambassador
        app.kubernetes.io/part-of: ambassador
        helm.sh/chart: ambassador-6.2.2
        product: aes
        service: ambassador-admin
      name: ambassador-admin
      namespace: ambassador
      resourceVersion: "43258"
      selfLink: /api/v1/namespaces/ambassador/services/ambassador-admin
      uid: 955b23af-c023-4196-a8f7-224194bac419
    spec:
      clusterIP: 10.1.166.205
      ports:
      - name: ambassador-admin
        port: 8877
        protocol: TCP
        targetPort: admin
      selector:
        app.kubernetes.io/instance: ambassador
        app.kubernetes.io/name: ambassador
      sessionAffinity: None
      type: ClusterIP
    status:
      loadBalancer: {}

查看Prometheus自定义资源，其中定义了`serviceMonitorSelector.matchLabels=release: prometheus`，Prometheus据此关联ServiceMonitor。

    kubectl get prometheuses.monitoring.coreos.com/prometheus-prometheus-oper-prometheus -nmonitoring -oyaml
    apiVersion: monitoring.coreos.com/v1
    kind: Prometheus
    metadata:
      creationTimestamp: "2020-04-01T01:43:35Z"
      generation: 1
      labels:
        app: prometheus-operator-prometheus
        chart: prometheus-operator-8.5.0
        heritage: Helm
        release: prometheus
      name: prometheus-prometheus-oper-prometheus
      namespace: monitoring
      resourceVersion: "14492"
      selfLink: /apis/monitoring.coreos.com/v1/namespaces/monitoring/prometheuses/prometheus-prometheus-oper-prometheus
      uid: 2b97c71a-5ec6-41bf-a42a-565136821ae5
    spec:
      alerting:
        alertmanagers:
        - name: prometheus-prometheus-oper-alertmanager
          namespace: monitoring
          pathPrefix: /
          port: web
      baseImage: quay.io/prometheus/prometheus
      enableAdminAPI: false
      externalUrl: http://prometheus-prometheus-oper-prometheus.monitoring:9090
      listenLocal: false
      logFormat: logfmt
      logLevel: info
      paused: false
      podMonitorNamespaceSelector: {}
      podMonitorSelector:
        matchLabels:
          release: prometheus
      portName: web
      replicas: 1
      retention: 10d
      routePrefix: /
      ruleNamespaceSelector: {}
      ruleSelector:
        matchLabels:
          app: prometheus-operator
          release: prometheus
      securityContext:
        fsGroup: 2000
        runAsNonRoot: true
        runAsUser: 1000
      serviceAccountName: prometheus-prometheus-oper-prometheus
      serviceMonitorNamespaceSelector: {}
      serviceMonitorSelector:
        matchLabels:
          release: prometheus
      version: v2.13.1

创建ServiceMonitor。其中几个需要注意的关键点。
- ServiceMonitor的name最终会反应到Prometheus的配置中，作为job_name。
- 由于Prometheus自定义资源中定义了`serviceMonitorSelector.matchLabels=release: prometheus`，表示ServiceMonitor需要定义一个标签release: prometheus，Prometheus据此可以关联ServiceMonitor。
- ServiceMonitor的命名空间必须和Prometheus所在的命名空间相同，此处为monitoring。
- endpoints.port需要和Service中的拉取metrics的ports.name对应，此处和上面对应为ambassador-admin。
- namespaceSelector.matchNames需要和被监控的Service所在的命名空间相同，此处为ambassador。
- selector.matchLabels的标签必须和被监控的Service中能唯一标明身份的标签对应。

创建ambassador-admin服务对应的ServiceMonitor。

    vi prometheus-serviceMonitorAmbassador.yaml
    ---
    apiVersion: monitoring.coreos.com/v1
    kind: ServiceMonitor
    metadata:
      name: ambassador-monitor
      labels:
        release: prometheus
      namespace: monitoring
    spec:
      endpoints:
      - port: ambassador-admin
      namespaceSelector:
        matchNames:
        - ambassador
      selector:
        matchLabels:
          service: ambassador-admin

    kubectl apply -f prometheus-serviceMonitorAmbassador.yaml

Prometheus的Targets。

![](images/prometheus-targets-helm-custom.png)

Prometheus监控指标`rate(envoy_http_rq_total[1m])`。

![](images/prometheus-graph-helm-custom.png)

