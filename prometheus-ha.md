# Prometheus高可用部署

## 数据持久化

### 说明

#### 本地存储

Prometheus提供了本地存储，即tsdb时序数据库。它的特点决定了它不能用于long-term数据存储，只能用于短期窗口的timeseries数据的保存和查询，并且不具有高可用性（宕机会导致历史数据无法读取），一般Prometheus推荐只保留几周和几个月的数据。

在Kubernetes这样的动态集群环境下，如果Prometheus的实例被重新调度，所有历史监控数据都会丢失。而且本地存储也导致Prometheus无法进行弹性扩展。

#### 远程存储

Prometheus没有自己实现集群存储，而是提供了远程读写的接口，让用户自己选择合适的时序数据库来实现Prometheus的扩展性。

Prometheus通过下面两种方式来实现与其它远端存储系统的对接：

- Prometheus按照标准格式将metrics写到远端存储

- Prometheus按照标准格式从远端URL来读取metrics

![](images/ha-remote-storage.png)

##### 远程读

在远程读的流程中，当用户发起查询请求后，Prometheus将向remote_read中配置的URL发起查询请求（matchers,ranges），Adaptor根据请求条件从第三方存储服务中获取数据，同时将数据转换为Prometheus的原始样本数据返回给Prometheus Server。

当获取到样本数据后，Prometheus在本地使用PromQL对样本数据进行二次处理。

##### 远程写

用户可以在Prometheus配置文件中指定remote_write的URL地址，一旦设置了该配置项，Prometheus将样本数据通过HTTP的形式发送给适配器（Adaptor）。而用户则可以在适配器中对接外部任意的服务。外部服务可以是真正的存储系统，公有云存储服务，也可以是消息队列等任意形式。

Prometheus支持的远程存储系统

- AppOptics: write
- Azure Data Explorer: read and write
- Azure Event Hubs: write
- Chronix: write
- Cortex: read and write
- CrateDB: read and write
- Elasticsearch: write
- Gnocchi: write
- Google Cloud Spanner: read and write
- Graphite: write
- InfluxDB: read and write
- IRONdb: read and write
- Kafka: write
- M3DB: read and write
- MetricFire: read and write
- OpenTSDB: write
- PostgreSQL/TimescaleDB: read and write
- QuasarDB: read and write
- SignalFx: write
- Splunk: read and write
- TiKV: read and write
- Thanos: read and write
- VictoriaMetrics: write
- Wavefront: write

##### 远程存储系统选型

- 需要同时支持读写

Influxdb支持

- 满足数据的安全性，需要支持备份，最好支持集群

Influxdb OSS版本没有提供集群功能（Influxdb Enterprise提供集群功能），但提供了数据的备份和恢复功能。需要定时脚本进行数据的备份。

- 性能要好

Influxdb本身是时序数据库，时序数据的读写性能应该足够。

- Grafana读取支持，优先考虑

Grafana支持Influxdb。

- 技术方案不复杂

Influxdb部署不复杂，查询语法类似于SQL。




下载[https://dl.influxdata.com/influxdb/releases/influxdb-1.8.0.x86_64.rpm](https://dl.influxdata.com/influxdb/releases/influxdb-1.8.0.x86_64.rpm)，上传`influxdb-1.8.0.x86_64.rpm`，安装influxdb。

    yum install -y influxdb-1.8.0.x86_64.rpm

    systemctl start influxdb
    systemctl enable influxdb

创建Prometheus数据库。

    influx
    Connected to http://localhost:8086 version 1.8.0
    InfluxDB shell version: 1.8.0
    > show databases;
    name: databases
    name
    ----
    _internal
    > create database prometheus;
    > alter retention policy "autogen" on "prometheus" duration 365d default;
    > show databases;
    name: databases
    name
    ----
    _internal
    prometheus

查看数据库，可以看到还没有指标数据。

    > use prometheus;
    Using database prometheus
    > show measurements;
    > quit

在Prometheus配置文件中添加远程读写配置。

    vi /usr/local/prometheus/prometheus/prometheus.yml
    global:
      scrape_interval:     15s
      evaluation_interval: 15s

    scrape_configs:
      - job_name: 'prometheus'
        static_configs:
        - targets: ['192.168.1.71:9090']
          labels:
            instance: prom-node3

    remote_write:
    - url: "http://192.168.1.71:8086/api/v1/prom/write?db=prometheus"
    remote_read:
    - url: "http://192.168.1.71:8086/api/v1/prom/read?db=prometheus"

检查配置文件。

    ./promtool check config prometheus.yml
    Checking prometheus.yml
      SUCCESS: 0 rule files found

重启Prometheus服务。

    systemctl restart prometheus

查看prometheus数据库，发现已经写入指标数据。

    influx
    Connected to http://localhost:8086 version 1.8.0
    InfluxDB shell version: 1.8.0
    > use prometheus;
    Using database prometheus
    > show measurements;
    name: measurements
    name
    ----
    go_gc_duration_seconds
    go_gc_duration_seconds_count
    go_gc_duration_seconds_sum
    go_goroutines
    go_info
    go_memstats_alloc_bytes
    ......
    prometheus_tsdb_wal_truncations_total
    prometheus_tsdb_wal_writes_failed_total
    prometheus_wal_watcher_current_segment
    prometheus_wal_watcher_record_decode_failures_total
    prometheus_wal_watcher_records_read_total
    prometheus_wal_watcher_samples_sent_pre_tailing_total
    promhttp_metric_handler_requests_in_flight
    promhttp_metric_handler_requests_total
    scrape_duration_seconds
    scrape_samples_post_metric_relabeling
    scrape_samples_scraped
    scrape_series_added
    up

查询up指标，和SQL语句很相似。

    > select * from up;
    name: up
    time                __name__ instance   job        value
    ----                -------- --------   ---        -----
    1595082892044000000 up       prom-node3 prometheus 1
    1595082907043000000 up       prom-node3 prometheus 1
    1595082922044000000 up       prom-node3 prometheus 1
    1595082937043000000 up       prom-node3 prometheus 1
    1595082952043000000 up       prom-node3 prometheus 1
    1595082967044000000 up       prom-node3 prometheus 1
    1595082982043000000 up       prom-node3 prometheus 1
    1595082997043000000 up       prom-node3 prometheus 1
    > quit

> 可以和Prometheus查出来的指标`up{instance="prom-node1",job="prometheus",userLabel1="value1",userLabel2="value2"}  1`对比一下。

## 安装influxdb_exporter

    tar zxvf influxdb_exporter-0.4.2.linux-amd64.tar.gz
    mv influxdb_exporter-0.4.2.linux-amd64/influxdb_exporter /usr/local/bin/

配置自启动。

    vi /etc/systemd/system/influxdb_exporter.service
    [Unit]
    Description=Influxdb Exporter
    After=network.target
    
    [Service]
    Type=simple
    User=root
    ExecStart=/usr/local/bin/influxdb_exporter
    Restart=on-failure
    
    [Install]
    WantedBy=multi-user.target

    systemctl start influxdb_exporter
    systemctl enable influxdb_exporter

查看监控指标。

    curl 192.168.1.71:9122/metrics
    # HELP go_gc_duration_seconds A summary of the pause duration of garbage collection cycles.
    # TYPE go_gc_duration_seconds summary
    go_gc_duration_seconds{quantile="0"} 0
    go_gc_duration_seconds{quantile="0.25"} 0
    go_gc_duration_seconds{quantile="0.5"} 0
    go_gc_duration_seconds{quantile="0.75"} 0
    go_gc_duration_seconds{quantile="1"} 0
    go_gc_duration_seconds_sum 0
    go_gc_duration_seconds_count 0
    # HELP go_goroutines Number of goroutines that currently exist.
    # TYPE go_goroutines gauge
    go_goroutines 9
    ......
    # HELP influxdb_last_push_timestamp_seconds Unix timestamp of the last received influxdb metrics push in seconds.
    # TYPE influxdb_last_push_timestamp_seconds gauge
    influxdb_last_push_timestamp_seconds 0
    # HELP influxdb_udp_parse_errors_total Current total udp parse errors.
    # TYPE influxdb_udp_parse_errors_total counter
    influxdb_udp_parse_errors_total 0
    # HELP process_cpu_seconds_total Total user and system CPU time spent in seconds.
    # TYPE process_cpu_seconds_total counter
    process_cpu_seconds_total 0.03
    ......
    # HELP process_resident_memory_bytes Resident memory size in bytes.
    # TYPE process_resident_memory_bytes gauge
    process_resident_memory_bytes 8.077312e+06
    # HELP process_start_time_seconds Start time of the process since unix epoch in seconds.
    # TYPE process_start_time_seconds gauge
    process_start_time_seconds 1.59506698429e+09
    # HELP process_virtual_memory_bytes Virtual memory size in bytes.
    # TYPE process_virtual_memory_bytes gauge
    process_virtual_memory_bytes 7.2880128e+08
    # HELP process_virtual_memory_max_bytes Maximum amount of virtual memory available in bytes.
    # TYPE process_virtual_memory_max_bytes gauge
    process_virtual_memory_max_bytes -1
    # HELP promhttp_metric_handler_requests_in_flight Current number of scrapes being served.
    # TYPE promhttp_metric_handler_requests_in_flight gauge
    promhttp_metric_handler_requests_in_flight 1
    # HELP promhttp_metric_handler_requests_total Total number of scrapes by HTTP status code.
    # TYPE promhttp_metric_handler_requests_total counter
    promhttp_metric_handler_requests_total{code="200"} 0
    promhttp_metric_handler_requests_total{code="500"} 0
    promhttp_metric_handler_requests_total{code="503"} 0

配置Prometheus，监控Influxdb。

    vi /usr/local/prometheus/prometheus/prometheus.yml
    global:
      scrape_interval:     15s
      evaluation_interval: 15s

    scrape_configs:
      - job_name: 'prometheus'
        static_configs:
        - targets: ['192.168.1.71:9090']
          labels:
            instance: prom-node3

      - job_name: 'influxdb_exporter'
        static_configs:
        - targets: ['192.168.1.71:9122']
          labels:
            instance: prom-node3

    remote_write:
    - url: "http://192.168.1.71:8086/api/v1/prom/write?db=prometheus"
    remote_read:
    - url: "http://192.168.1.71:8086/api/v1/prom/read?db=prometheus"

    systemctl start prometheus

查看Prometheus数据库。

    influx
    > use prometheus;
    Using database prometheus
    > show measurements;
    show measurements;
    name: measurements
    name
    ----
    ......
    go_threads
    influxdb_exporter_build_info
    influxdb_last_push_timestamp_seconds
    influxdb_udp_parse_errors_total
    net_conntrack_dialer_conn_attempted_total
    net_conntrack_dialer_conn_closed_total
    net_conntrack_dialer_conn_established_total
    ......
    prometheus_tsdb_wal_truncate_duration_seconds_count
    prometheus_tsdb_wal_truncate_duration_seconds_sum
    prometheus_tsdb_wal_truncations_failed_total
    prometheus_tsdb_wal_truncations_total
    prometheus_tsdb_wal_writes_failed_total
    prometheus_wal_watcher_current_segment
    prometheus_wal_watcher_record_decode_failures_total
    prometheus_wal_watcher_records_read_total
    prometheus_wal_watcher_samples_sent_pre_tailing_total
    promhttp_metric_handler_requests_in_flight
    promhttp_metric_handler_requests_total
    scrape_duration_seconds
    scrape_samples_post_metric_relabeling
    scrape_samples_scraped
    scrape_series_added
    up
    select * from up order by time desc limit 10;
    name: up
    time                __name__ instance   job               value
    ----                -------- --------   ---               -----
    1595085157042000000 up       prom-node3 prometheus        1
    1595085152810000000 up       prom-node3 influxdb_exporter 1
    1595085142043000000 up       prom-node3 prometheus        1
    1595085137809000000 up       prom-node3 influxdb_exporter 1
    1595085127044000000 up       prom-node3 prometheus        1
    1595085122815000000 up       prom-node3 influxdb_exporter 1
    1595085112043000000 up       prom-node3 prometheus        1
    1595085107810000000 up       prom-node3 influxdb_exporter 1
    1595085097049000000 up       prom-node3 prometheus        1
    1595085092810000000 up       prom-node3 influxdb_exporter 1
    > quit

## 分层联邦+远程存储

### 说明

在大的数据中心或者Kubernetes集群中，比如说在有500个节点的K8S集群环境下，每个节点有50个Pod，每个Pod样本数为500+，就是1250w，采样间隔15s的话，需要的采样速度为83w/s，单个的Prometheus肯定是不足以满足性能需要的。

Prometheus提供了联邦功能支持分区采集，提高采集性能。

#### 分层联邦（Hieratchical Federation）

分层联邦拓扑类似于一棵树，更高级别的Prometheus服务器从大量从属服务器收集聚合的时间序列数据。



Prometheus是通过记录规则（Recording Rules）实现时序数据的聚合。下级Prometheus通过记录规则可以预计算上级Prometheus需要的聚合时序数据。其实Prometheus Operator已经预置了很多的记录规则，以下是一个例子。


。。。。。。。。。。。。。。。。。。。。图


上级Prometheus在拉取下级Prometheus时还可以通过label过滤掉无用的时序数据。

        params:
          'match[]':
            - '{job="prometheus"}'
            - '{job="node_exporter"}'
            - '{job="influxdb_exporter"}'

#### 跨服务联邦（Cross-service Federation）

在跨服务联邦中，将一项服务的Prometheus服务器配置为从另一项服务的Prometheus服务器抓取所选数据，以针对单个服务器内的两个数据集启用告警和查询。

例如，运行多个服务的集群调度程序可能会公开有关集群上运行的服务实例的资源使用情况信息（如内存和CPU使用情况）。另一方面，在该集群上运行的服务将仅公开特定于应用程序的服务指标。通常，这两组指标是由单独的Prometheus服务器抓取的。使用联邦，包含服务级别指标的Prometheus服务器可以从集群Prometheus中获取有关其特定服务的集群资源使用指标，以便可以在该服务器中使用者两组指标。

。。。。。。。。。。。。。。。。。。。。图






### 环境

| IP地址 | 主机名 | 安装软件 |
| :------| :------ | :------ |
| 192.168.1.69 | prom-node1 | prometheus,node_exporter |
| 192.168.1.70 | prom-node2 | prometheus,node_exporter |
| 192.168.1.71 | prom-node3 | prometheus,node_exporter,influxdb,influxdb_exporter |

> prom-node1的Prometheus监控prom-node1和prom-node2所有目标，prom-node2的Prometheus监控prom-node3所有目标。

> 将prom-node3的Prometheus作为全局节点。

> prom-node1和prom-node2的Prometheus使用本地存储，prom-node3的Prometheus采用Influxdb远程存储。

### 配置

配置prom-node1节点。

    vi /usr/local/prometheus/prometheus/prometheus.yml
    global:
      scrape_interval:     15s
      evaluation_interval: 15s

    scrape_configs:
      - job_name: 'prometheus'
        static_configs:
        - targets: ['192.168.1.69:9090']
          labels:
            instance: prom-node1
        - targets: ['192.168.1.70:9090']
          labels:
            instance: prom-node2

      - job_name: 'node_exporter'
        static_configs:
        - targets: ['192.168.1.69:9100']
          labels:
            instance: prom-node1
        - targets: ['192.168.1.70:9100']
          labels:
            instance: prom-node2

    systemctl restart prometheus

配置prom-node2节点。

    vi /usr/local/prometheus/prometheus/prometheus.yml
    global:
      scrape_interval:     15s
      evaluation_interval: 15s

    scrape_configs:
      - job_name: 'prometheus'
        static_configs:
        - targets: ['192.168.1.71:9090']
          labels:
            instance: prom-node3

      - job_name: 'node_exporter'
        static_configs:
        - targets: ['192.168.1.71:9100']
          labels:
            instance: prom-node3

      - job_name: 'influxdb_exporter'
        static_configs:
        - targets: ['192.168.1.71:9122']
          labels:
            instance: prom-node3

    systemctl restart prometheus

配置prom-node3节点。

    vi /usr/local/prometheus/prometheus/prometheus.yml
    global:
      scrape_interval:     15s
      evaluation_interval: 15s

    scrape_configs:
      - job_name: 'federate'
        scrape_interval: 15s

        honor_labels: true
        metrics_path: '/federate'

        params:
          'match[]':
            - '{job="prometheus"}'
            - '{job="node_exporter"}'
            - '{job="influxdb_exporter"}'

        static_configs:
          - targets:
            - '192.168.1.69:9090'
            - '192.168.1.70:9090'

    remote_write:
    - url: "http://192.168.1.71:8086/api/v1/prom/write?db=prometheus"
    remote_read:
    - url: "http://192.168.1.71:8086/api/v1/prom/read?db=prometheus"

    systemctl restart prometheus

> Global Prometheus会每15秒钟从下级Prometheus拉取一次指标数据，并写入远程数据库中。
> 指标数据的label需要匹配match中的条件，以上定义将所有的job指标数据都拉取。

可以查看Prometheus数据库，会发现各个job的指标数据都存在。

    influx
    Connected to http://localhost:8086 version 1.8.0
    InfluxDB shell version: 1.8.0
    > use prometheus
    Using database prometheus
    > show measurements;
    ......
    prometheus_wal_watcher_records_read_total
    prometheus_wal_watcher_samples_sent_pre_tailing_total
    promhttp_metric_handler_requests_in_flight
    promhttp_metric_handler_requests_total
    scrape_duration_seconds
    scrape_samples_post_metric_relabeling
    scrape_samples_scraped
    scrape_series_added
    up
    > select * from up order by time desc limit 15;
    name: up
    time                __name__ instance          job               value
    ----                -------- --------          ---               -----
    1595121465858000000 up       192.168.1.69:9090 federate          1
    1595121464840000000 up       prom-node2        prometheus        1
    1595121462399000000 up       prom-node1        node_exporter     1
    1595121458586000000 up       prom-node1        prometheus        1
    1595121456878000000 up       prom-node2        node_exporter     1
    1595121451215000000 up       192.168.1.70:9090 federate          1
    1595121450858000000 up       192.168.1.69:9090 federate          1
    1595121449838000000 up       prom-node2        prometheus        1
    1595121448537000000 up       prom-node3        influxdb_exporter 1
    1595121447398000000 up       prom-node1        node_exporter     1
    1595121443586000000 up       prom-node1        prometheus        1
    1595121441879000000 up       prom-node2        node_exporter     1
    1595121439962000000 up       prom-node3        prometheus        1
    1595121437728000000 up       prom-node3        node_exporter     1
    1595121436216000000 up       192.168.1.70:9090 federate          1
    > quit






















