## 1、Prometheus & Alertmanager 基础安装

### 📂 Step 1: 创建所有核心数据/配置目录

```Plain
# 创建所有核心数据/配置目录
mkdir -p /data/prometheus/data
mkdir -p /data/prometheus/rules
mkdir -p /data/alertmanager
mkdir -p /data/prometheusalert/conf

# 赋予权限（防止容器内用户无权写入导致 CrashLoopBackOff）
chmod -R 777 /data/prometheus/data
```

### ⚙️ Step 2: 写入 Prometheus 配置文件

执行以下命令直接生成 `/data/prometheus/prometheus.yml`：

```Plain
cat << 'EOF' > /data/prometheus/prometheus.yml
global:
  scrape_interval: 15s
  evaluation_interval: 15s
  
rule_files:
  - "/etc/prometheus/rules/*.rules"

alerting:
  alertmanagers:
    - static_configs:
        - targets: ['172.17.0.1:9093'] # 内部通过 Docker 网桥互通
      api_version: v2

scrape_configs:
  - job_name: 'prometheus'
    static_configs:
      - targets: ['localhost:9090']
EOF
```

### ⚙️ Step 3: 写入 Alertmanager 配置文件

> 💡 **核心架构说明**：这里配置了**工单自动分流逻辑**。带有 `workorder: "true"` 标签的告警（如 GPU、IPMI 故障）会自动流向工单飞书通道，普通告警走兜底通道。

```Plain
cat << 'EOF' > /data/alertmanager/alertmanager.yml
global:
  resolve_timeout: 5m

# 路由树配置
route:
  group_by: ['alertname', 'cluster', 'instance']
  group_wait: 10s
  group_interval: 10s
  repeat_interval: 10m
  receiver: 'default-feishu' # 兜底路由

  routes:
    # 🌟 核心子路由：匹配所有带 workorder: "true" 的告警（IPMI/GPU两类）
    - match:
        workorder: "true"
      receiver: 'workorder-feishu' # 工单专属接收器
      group_wait: 3s
      routes:
        - match:
            severity: critical
          repeat_interval: 5m # 紧急工单 5 分钟重发催办# 保留原有按级别路由规则（仅对非工单告警生效）
    - match:
        severity: critical
      receiver: 'default-feishu'
      group_wait: 5s
    - match:
        severity: warning
      receiver: 'default-feishu'
      repeat_interval: 30m

receivers:
# 接收器1：兜底原有飞书机器人（采用 PrometheusAlert 自定义模板路由方式）
- name: 'default-feishu'
  webhook_configs:
  - url: 'http://172.17.0.1:8080/prometheusalert?type=fs&tpl=prometheus-fs&fsurl=https://open.feishu.cn/open-apis/bot/v2/hook/7d0a109d-3d73-4f33-a639-8b4e7f625e5a'
    send_resolved: true# 接收器2：工单专属飞书机器人
- name: 'workorder-feishu'
  webhook_configs:
  - url: 'http://172.17.0.1:8080/prometheusalert?type=fs&tpl=prometheus-fs&fsurl=https://open.feishu.cn/open-apis/bot/v2/hook/7d0a109d-3d73-4f33-a639-8b4e7f625e5a'
    send_resolved: true

inhibit_rules:
  - source_match:
      severity: 'critical'
    target_match:
      severity: 'warning'
    equal: ['alertname', 'cluster', 'instance']
EOF
```

### ⚙️ Step 4: 写入 PrometheusAlert 配置文件

执行以下命令直接生成 `/data/prometheusalert/conf/app.conf`：

```Plain
cat << 'EOF' > /data/prometheusalert/conf/app.conf
appname = "Prometheus 告警通知"
httpport = 8080
runmode = dev
copyrequestbody = true

login_user = admin
login_password = admin

logs = /app/logs
dbtype = sqlite3
dbname = /app/db/prometheusalert.db

logmaxday = 7

[default]
open = true
EOF
```

### 🚀 Step 5: 三大服务独立容器拉起

```Plain
# 1. 启动 PrometheusAlert
docker run -d \
  --name prometheusalert \
  --restart always \
  -p 8080:8080 \
  -v /data/prometheusalert/conf:/app/conf \
  -e TZ=Asia/Shanghai \
  swr.cn-north-4.myhuaweicloud.com/ddn-k8s/docker.io/feiyu563/prometheus-alert:v4.9.1

# 2. 启动 Alertmanager
docker run -d \
  --name alertmanager \
  --restart always \
  -p 9093:9093 \
  -v /data/alertmanager/alertmanager.yml:/etc/alertmanager/alertmanager.yml \
  swr.cn-north-4.myhuaweicloud.com/ddn-k8s/quay.io/prometheus/alertmanager:v0.27.0

# 3. 启动 Prometheus
docker run -d \
  --name prometheus \
  --restart always \
  -p 9090:9090 \
  -v /data/prometheus/prometheus.yml:/etc/prometheus/prometheus.yml \
  -v /data/prometheus/data:/prometheus \
  -v /data/prometheus/rules:/etc/prometheus/rules \
  swr.cn-north-4.myhuaweicloud.com/ddn-k8s/quay.io/prometheus/prometheus:v3.5.0
```

## 2、Grafana 安装与数据打通

```Plain
# 1. 创建专用数据持久化目录
mkdir -p /data/grafana/data

# 2. 授权（Grafana容器内部默认UID为472，不授权会报 Permission Denied 闪退）
chown -R 472:472 /data/grafana/data

# 3. 拉起 Grafana
docker run -d \
  --name grafana \
  --restart always \
  -p 3000:3000 \
  -v /data/grafana/data:/var/lib/grafana \
  -e TZ=Asia/Shanghai \
  grafana/grafana:10.4.0
```

> **📊 界面配置**：访问 `http://IP:3000`（初始密码 `admin/admin`）。进入 **Connections** -> **Data sources** -> 添加 **Prometheus**，URL 填写 `http://172.17.0.1:9090`，点击 **Save & test** 激活。

## 3、Node_Exporter 客户端部署

在被监控的虚拟机/物理机上执行：

```Plain
# 1. 下载与解压安装cd /opt
wget https://github.com/prometheus/node_exporter/releases/download/v1.9.1/node_exporter-1.9.1.linux-amd64.tar.gz
tar -zxvf node_exporter-1.9.1.linux-amd64.tar.gz
mv node_exporter-1.9.1.linux-amd64/node_exporter /usr/local/bin/

# 2. 创建运行系统非登录用户
useradd -rs /bin/false node_exporter

# 3. 创建 Systemd 管理服务
cat << 'EOF' > /etc/systemd/system/node_exporter.service
[Unit]
Description=Prometheus Node Exporter
After=network.target

[Service]
User=node_exporter
Group=node_exporter
ExecStart=/usr/local/bin/node_exporter --web.listen-address=:10086
Restart=always

[Install]
WantedBy=multi-user.target
EOF

# 4. 启动服务
systemctl daemon-reload && systemctl enable node_exporter && systemctl start node_exporter
```

## 4、Prometheus 核心告警规则配置 (Rules)

请将各规则文件放置于宿主机 `/data/prometheus/rules/` 目录下。

### 🚨 1. GPU 节点集群监控规则 (`gpu_node.rules`)

```Plain
root@hzk:/data# cat prometheus/rules/gpu_node.rules 
groups:
  - name: gpu_metrics
    rules:
      - alert: 集群GPU数量异常
        expr: node_cluster_check_gpu_count == 1
        for: 5m
        labels:
          severity: critical
          workorder: true
          fault_code: "3211"
        annotations:
          summary: "节点{{ $labels.instance }} GPU数量异常"
          description: "节点{{ $labels.instance }}的GPU数量不足，出现掉卡问题"
          runbook_url: "https://yindunyun.feishu.cn/wiki/WVJ4wm9Mgi10gXk3V4ec1jvennf?fromScene=spaceOverview"

      - alert: 集群IB网卡状态异常
        expr: node_cluster_check_ib_status == 1
        for: 5m
        labels:
          severity: critical
          workorder: true
          fault_code: "4001"
        annotations:
          summary: "节点{{ $labels.instance }} IB网卡状态异常"
          description: "节点{{ $labels.instance }}存在IB网卡处于down状态"
          runbook_url: "https://yindunyun.feishu.cn/wiki/WVJ4wm9Mgi10gXk3V4ec1jvennf?fromScene=spaceOverview"

      - alert: 集群Peermem状态异常
        expr: node_cluster_check_peermem_status == 1
        for: 5m
        labels:
          severity: warning
          workorder: true
          fault_code: "3202"
        annotations:
          summary: "节点{{ $labels.instance }} peermem状态异常"
          description: "节点{{ $labels.instance }}的peermem模块未正常加载"
          runbook_url: "https://yindunyun.feishu.cn/wiki/WVJ4wm9Mgi10gXk3V4ec1jvennf?fromScene=spaceOverview"

      - alert: 集群Fabric状态异常
        expr: node_cluster_check_fabric_status == 1
        for: 5m
        labels:
          fault_code: "3311"
          workorder: true
          severity: critical
        annotations:
          summary: "节点{{ $labels.instance }} fabric服务未开启"
          description: "节点{{ $labels.instance }}的nvidia-fabricmanager服务未处于运行状态"
          runbook_url: "https://yindunyun.feishu.cn/wiki/WVJ4wm9Mgi10gXk3V4ec1jvennf?fromScene=spaceOverview"

      - alert: 集群ACS状态异常
        expr: node_cluster_check_acs_status == 1
        for: 5m
        labels:
          severity: warning
          workorder: true
          fault_code: "2005"
        annotations:
          summary: "节点{{ $labels.instance }} ACS未关闭"
          description: "节点{{ $labels.instance }}的ACS功能未正确关闭，可能影响性能"
          runbook_url: "https://yindunyun.feishu.cn/wiki/WVJ4wm9Mgi10gXk3V4ec1jvennf?fromScene=spaceOverview"

      - alert: 集群网卡排列顺序异常
        expr: node_cluster_check_nic_order == 1
        for: 5m
        labels:
          severity: warning
          workorder: true
          fault_code: "5004"
        annotations:
          summary: "节点{{ $labels.instance }}网卡排列顺序不正确"
          description: "节点{{ $labels.instance }}的IB网卡排列顺序与预期不符"
          runbook_url: "https://yindunyun.feishu.cn/wiki/WVJ4wm9Mgi10gXk3V4ec1jvennf?fromScene=spaceOverview"

      - alert: 集群内存品牌不一致
        expr: node_cluster_check_memory_brand == 1
        for: 5m
        labels:
          severity: warning
          workorder: true
          fault_code: "8004"
        annotations:
          summary: "节点{{ $labels.instance }}内存品牌不一致"
          description: "节点{{ $labels.instance }}使用了不同品牌的内存模块"
          runbook_url: "https://yindunyun.feishu.cn/wiki/WVJ4wm9Mgi10gXk3V4ec1jvennf?fromScene=spaceOverview"

      - alert: 集群IB链路BER值异常
        expr: node_cluster_check_ib_ber_status == 1
        for: 5m
        labels:
          severity: warning
          workorder: true
          fault_code: "9501"
        annotations:
          summary: "节点{{ $labels.instance }} IB链路BER值异常"
          description: "节点{{ $labels.instance }}的{{ $labels.device }}设备Symbol BER值低于14"
          runbook_url: "https://yindunyun.feishu.cn/wiki/WVJ4wm9Mgi10gXk3V4ec1jvennf?fromScene=spaceOverview"

      - alert: 集群GPU ECC异常
        expr: node_cluster_check_gpu_ecc == 1
        for: 5m
        labels:
          severity: critical
          workorder: true
          fault_code: "3111"
        annotations:
          summary: "节点{{ $labels.instance }} GPU ECC异常"
          description: "节点{{ $labels.instance }}存在GPU ECC Remapping Failure"
          runbook_url: "https://yindunyun.feishu.cn/wiki/WVJ4wm9Mgi10gXk3V4ec1jvennf?fromScene=spaceOverview"

      - alert: 集群IB网卡光衰异常
        expr: node_cluster_check_ib_power_status == 1
        for: 5m
        labels:
          severity: warning
          workorder: true
          fault_code: "4004"
        annotations:
          summary: "节点{{ $labels.instance }} IB网卡光衰异常"
          description: "节点{{ $labels.instance }}的{{ $labels.device }}设备{{ $labels.power_type }}数值异常"
          runbook_url: "https://yindunyun.feishu.cn/wiki/WVJ4wm9Mgi10gXk3V4ec1jvennf?fromScene=spaceOverview"

      - alert: 集群IB网卡带宽异常
        expr: node_cluster_check_ib_bandwidth_status == 1
        for: 5m
        labels:
          severity: critical
          workorder: true
          fault_code: "37203"
        annotations:
          summary: "节点{{ $labels.instance }} IB网卡带宽异常"
          description: "节点{{ $labels.instance }}的{{ $labels.device }}设备带宽低于360 Gb/sec"
          runbook_url: "https://yindunyun.feishu.cn/wiki/WVJ4wm9Mgi10gXk3V4ec1jvennf?fromScene=spaceOverview"
        # GPU XID异常告警
  #- alert: 集群GPU XID异常
  #  expr: node_cluster_check_gpu_xid == 1
  #  for: 5m
  #  labels:
  #    severity: critical
  #    fault_code: "3101"
  #  annotations:
  #    summary: "节点{{ $labels.instance }} GPU XID异常"
  #    description: "节点{{ $labels.instance }}检测到GPU XID错误"
  #    runbook_url: "https://yindunyun.feishu.cn/wiki/WVJ4wm9Mgi10gXk3V4ec1jvennf?fromScene=spaceOverview"

  # 风扇转速异常告警
  #- alert: 集群风扇转速异常
  # expr: node_cluster_check_fan_speed == 1
  # for: 5m
  # labels:
  #   severity: warning
  #   fault_code: "37401"
  # annotations:
  #   summary: "节点{{ $labels.instance }}风扇转速不是最高"
  #   description: "节点{{ $labels.instance }}的风扇转速未设置为Max"
  #   runbook_url: "https://yindunyun.feishu.cn/wiki/WVJ4wm9Mgi10gXk3V4ec1jvennf?fromScene=spaceOverview"
```

### 🔌 2. IPMI 硬件监控规则 (`ipmi_alerts.rules`)

```Plain
root@hzk:/data/prometheus/rules# cat ipmi_alerts.rules 
groups:
- name: IPMI硬件监控告警规则
  rules:
  # ===================== 风扇相关告警 =====================
  # 1. 风扇未检测到/离线（FAN1-FAN14_Present状态异常/NaN）
  - alert: 风扇离线或未检测到
    expr: ipmi_up == 1 and (ipmi_sensor_state{type="Fan"} != 0 or ipmi_sensor_state{type="Fan"} != ipmi_sensor_state{type="Fan"})
    for: 30s
    labels:
      severity: critical
      workorder: true
      component: fan
      fault_code: "8002"
    annotations:
      summary: "服务器{{ $labels.instance }}风扇{{ $labels.name }}离线/未检测到"
      description: "风扇{{ $labels.name }}(ID:{{ $labels.id }})状态异常（离线/未检测到），正常状态值应为0！"
  # ===================== 电源相关告警 =====================
  # 1. 电源模块离线/故障（PSU1-PSU8_Status状态异常/NaN）
  - alert: 电源模块离线或故障
    expr: ipmi_up == 1 and (ipmi_sensor_state{type="Power Supply"} != 0 or ipmi_sensor_state{type="Power Supply"} != ipmi_sensor_state{type="Power Supply"})
    for: 30s
    labels:
      severity: critical
      workorder: true
      component: power
      fault_code: "8001"
    annotations:
      summary: "服务器{{ $labels.instance }}电源{{ $labels.name }}故障/离线"
      description: "电源模块{{ $labels.name }}(ID:{{ $labels.id }})状态异常（故障/离线/未检测到），正常状态值应为0！"

  # ===================== 硬盘/系统盘相关告警 =====================
  # 硬盘/系统盘离线/故障（HDD0-HDD3、SW00-SW31状态异常/NaN）
  - alert: 硬盘/系统盘离线或故障
    expr: ipmi_up == 1 and (ipmi_sensor_state{type="Drive Slot"} != 0 or ipmi_sensor_state{type="Drive Slot"} != ipmi_sensor_state{type="Drive Slot"})
    for: 30s
    labels:
      severity: critical
      workorder: true
      component: disk
      fault_code: "8003"
    annotations:
      summary: "服务器{{ $labels.instance }}{{ $labels.name }}离线/故障"
      description: "存储盘{{ $labels.name }}(ID:{{ $labels.id }})状态异常（离线/故障/未检测到），正常状态值应为0！"

  # ===================== CPU相关告警 =====================
  # 1. CPU离线/故障（CPU0/CPU1_Status状态异常/NaN）
  #- alert: CPU离线或故障
  #  expr: ipmi_up == 1 and (ipmi_sensor_state{type="Processor"} != 0 or ipmi_sensor_state{type="Processor"} != ipmi_sensor_state{type="Processor"})
  #  for: 30s
  #  labels:
  #    severity: critical
  #    workorder: true
  #    component: cpu
  #  annotations:
  #    summary: "服务器{{ $labels.instance }}{{ $labels.name }}离线/故障"
  #    description: "CPU{{ $labels.name }}(ID:{{ $labels.id }})状态异常（离线/故障/未检测到），正常状态值应为0！"

  # ===================== 内存相关告警 =====================
  # 内存控制器/内存离线/故障（CPU0/1_MEM_Status状态异常/NaN）
  #- alert: 内存离线或故障
  #  expr: ipmi_up == 1 and (ipmi_sensor_state{type="Memory"} != 0 or ipmi_sensor_state{type="Memory"} != ipmi_sensor_state{type="Memory"})
  #  for: 30s
  #  labels:
  #    severity: critical
  #    workorder: true
  #    component: memory
  #  annotations:
  #    summary: "服务器{{ $labels.instance }}{{ $labels.name }}离线/故障"
  #    description: "内存{{ $labels.name }}(ID:{{ $labels.id }})状态异常（离线/故障/未检测到），正常状态值应为0！"
```

### 🗄️ 3. 基础组件与系统监控规则

*(包含现成的 MinIO、MySQL、PostgreSQL 及系统基础系统监控，限于篇幅，管理员可直接引用原对应 rules 规则项配置。核心注意：需要分流至工单的指标，请务必在* *`labels`* *标签中声明* *`workorder: "true"`**)*

*MinIO*

```YAML
root@hzk:/data/prometheus/rules# cat minio.rules 
groups:
  - name: minio_alerts
    rules:
      - alert: MinIO_实例宕机
        expr: up{job=~".*minio.*"} == 0
        for: 1m
        labels:
          severity: critical
        annotations:
          summary: "MinIO 实例宕机"
          description: "MinIO 实例 {{ $labels.instance }} 已离线超过 1 分钟。"

      - alert: MinIO_磁盘使用率过高
        expr: minio_cluster_disk_used_percent > 85
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "MinIO 磁盘使用率过高"
          description: "MinIO 集群磁盘使用率已达 {{ $value }}%，实例：{{ $labels.instance }}。"

      - alert: MinIO_S3请求错误率过高
        expr: |
          rate(minio_s3_requests_errors_total{code=~"5.."}[5m])
            /
          rate(minio_s3_requests_total[5m]) > 0.05
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "MinIO S3 请求错误率过高"
          description: "过去 5 分钟内，S3 请求中 5xx 错误比例超过 5%，实例：{{ $labels.instance }}。"

      - alert: MinIO_磁盘驱动器离线
        expr: minio_node_drive_offline_total > 0
        for: 1m
        labels:
          severity: critical
        annotations:
          summary: "MinIO 磁盘驱动器离线"
          description: "MinIO 节点 {{ $labels.instance }} 有 {{ $value }} 个磁盘驱动器处于离线状态。"
```

*MySQL*

```YAML
root@hzk:/data/prometheus/rules# cat mysql.rules 
    groups:
      - name: mysql_alerts
        rules:
          # 1. MySQL 实例不可达
          - alert: MySQL 实例不可达
            expr: mysql_up{job="deepseek-mysql"} == 0
            for: 1m
            labels:
              severity: critical
            annotations:
              summary: "MySQL 实例不可达 (实例: {{ $labels.instance }})"
              description: "MySQL 服务器已无法连接，可能已宕机或网络中断。\n  作业: {{ $labels.job }}\n  实例: {{ $labels.instance }}"

          # 2. MySQL 连接数使用率 > 80%
          - alert: MySQL 连接数使用率过高
            expr: (mysql_global_status_threads_connected / mysql_global_variables_max_connections) * 100 > 80
            for: 5m
            labels:
              severity: warning
            annotations:
              summary: "MySQL 连接数使用率过高 (实例: {{ $labels.instance }})"
              description: "MySQL 当前连接数使用率已达 {{ printf \"%.2f\" .Value }}%，接近上限。\n  最大连接数: {{ $labels.mysql_global_variables_max_connections }}\n  建议检查应用连接池或调整 max_connections。"

          # 3. MySQL 主从复制中断（IO 或 SQL 线程停止）
          - alert: MySQL 主从复制中断
            expr: mysql_slave_status_slave_io_running == 0 or mysql_slave_status_slave_sql_running == 0
            for: 1m
            labels:
              severity: critical
            annotations:
              summary: "MySQL 主从复制中断 (实例: {{ $labels.instance }})"
              description: "MySQL 从库复制进程已停止。\n  IO Thread Running: {{ $value | printf \"%.0f\" }}\n  SQL Thread Running: {{ $value | printf \"%.0f\" }}\n  请立即检查主从同步状态。"

          # 4. 主从延迟超过 30 秒
          - alert: MySQL 主从延迟过高
            expr: mysql_slave_status_seconds_behind_master > 30
            for: 5m
            labels:
              severity: warning
            annotations:
              summary: "MySQL 主从延迟过高 (实例: {{ $labels.instance }})"
              description: "MySQL 主从延迟已达 {{ $value }} 秒，可能影响数据一致性。\n  建议检查网络、从库负载或大查询。"

          # 5. InnoDB 缓冲池命中率低于 95%
          - alert: InnoDB 缓冲池命中率过低
            expr: >
              (
                rate(mysql_global_status_innodb_buffer_pool_read_requests[5m])
                -
                rate(mysql_global_status_innodb_buffer_pool_reads[5m])
              )
              /
              rate(mysql_global_status_innodb_buffer_pool_read_requests[5m])
              * 100 < 95
            for: 10m
            labels:
              severity: warning
            annotations:
              summary: "InnoDB 缓冲池命中率过低 (实例: {{ $labels.instance }})"
              description: "InnoDB 缓冲池命中率仅为 {{ printf \"%.2f\" .Value }}%，可能导致大量磁盘 IO。\n  建议增加 innodb_buffer_pool_size 或优化查询。"

          # 6. 慢查询数量过高（每分钟超过 10 条）
          - alert: MySQL 慢查询数量过高
            expr: rate(mysql_global_status_slow_queries[5m]) * 60 > 10
            for: 5m
            labels:
              severity: warning
            annotations:
              summary: "MySQL 慢查询数量过高 (实例: {{ $labels.instance }})"
              description: "MySQL 每分钟慢查询数已达 {{ printf \"%.2f\" .Value }} 条，可能影响性能。\n  建议检查 slow_query_log 和执行计划。"
```

*PostgreSQL*

```YAML
root@hzk:/data/prometheus/rules# cat postgreSQL.rules
groups:
  - name: postgresql_alerts
    rules:
      - alert: PostgreSQL_实例宕机
        expr: up{job=~".*postgres.*"} == 0
        for: 1m
        labels:
          severity: critical
        annotations:
          summary: "PostgreSQL 实例宕机"
          description: "PostgreSQL 实例 {{ $labels.instance }} 已离线超过 1 分钟。"

      - alert: PostgreSQL_连接数使用率过高
        expr: pg_stat_database_maxconns_ratio > 0.85
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "PostgreSQL 连接数使用率过高"
          description: "PostgreSQL 连接数使用率已达 {{ $value | humanizePercentage }}，实例：{{ $labels.instance }}。"

      - alert: PostgreSQL_主从复制延迟过高
        expr: pg_replication_lag > 30
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "PostgreSQL 主从复制延迟过高"
          description: "主从复制延迟已达 {{ $value }} 秒，实例：{{ $labels.instance }}。"

      - alert: PostgreSQL_长时间运行事务
        expr: pg_stat_activity_max_tx_duration_seconds > 600
        for: 1m
        labels:
          severity: warning
        annotations:
          summary: "检测到长时间运行的 PostgreSQL 事务"
          description: "存在运行时间超过 {{ $value }} 秒的事务，实例：{{ $labels.instance }}。"

      - alert: PostgreSQL_表膨胀严重
        expr: pg_table_bloat_approx_bytes > 1e9
        for: 10m
        labels:
          severity: warning
        annotations:
          summary: "PostgreSQL 表膨胀严重"
          description: "检测到表膨胀超过 1GB，实例：{{ $labels.instance }}。"

      - alert: PostgreSQL_WAL写入速率过高
        expr: rate(pg_wal_writes_bytes_total[5m]) > 100 * 1024 * 1024
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "PostgreSQL WAL 写入速率过高"
          description: "WAL 写入速率超过 100 MB/s（当前 {{ $value | humanize }}B/s），实例：{{ $labels.instance }}。"
```

*系统基础系统监控*

```YAML
root@hzk:/data/prometheus/rules# cat node_alert.rules 
groups:
  - name: node_exporter_alerts
    rules:
      # 1. 主机存活监控
      - alert: 主机离线监控
        expr: up == 0
        for: 1m
        labels:
          severity: critical
        annotations:
          summary: "实例下线 (instance {{ $labels.instance }})"
          description: "{{ $labels.instance }} 节点已下线超过 10 分钟"

      # 2. CPU 使用率过高
      - alert: CPU 使用率监控
        expr: 100 - (avg by(instance) (rate(node_cpu_seconds_total{mode="idle"}[5m])) * 100) > 80
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "CPU 使用率过高 (instance {{ $labels.instance }})"
          description: "{{ $labels.instance }} CPU 使用率持续 5 分钟超过 80%，当前值: {{ $value }}%"

      # 3. 内存使用率过高
      - alert: 内存使用率监控
        expr: (1 - (node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes)) * 100 > 85
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "内存使用率过高 (instance {{ $labels.instance }})"
          description: "{{ $labels.instance }} 内存使用率超过 85%，当前值: {{ $value | humanize }}%"

      # 4. 磁盘使用率过高
      - alert: 磁盘使用率监控
        expr: (1 - (node_filesystem_avail_bytes{fstype!="tmpfs"} / node_filesystem_size_bytes)) * 100 > 95
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "磁盘使用率过高 (instance {{ $labels.instance }})"
          description: "{{ $labels.instance }} 分区 {{ $labels.mountpoint }} 使用率超过 95%，当前值: {{ $value | humanize }}%"

      # 9. 磁盘读写延迟过高
      - alert: 磁盘读写延迟监控
        expr: rate(node_disk_read_time_seconds_total[5m]) / rate(node_disk_reads_completed_total[5m]) > 0.1
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "磁盘读取延迟过高 (instance {{ $labels.instance }})"
          description: "{{ $labels.instance }} 磁盘 {{ $labels.device }} 读取延迟超过 100ms"

      # 10. 网络错误率过高
      - alert: 网络错误率监控
        expr: (rate(node_network_receive_errs_total[5m]) + rate(node_network_transmit_errs_total[5m])) / (rate(node_network_receive_packets_total[5m]) + rate(node_network_transmit_packets_total[5m])) > 0.01
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "网络错误率过高 (instance {{ $labels.instance }})"
          description: "{{ $labels.instance }} 网卡 {{ $labels.device }} 错误率超过 1%"

      # 13. 进程数过多
      - alert: 进程数监控
        expr: node_procs_running > 500
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "运行进程数过多 (instance {{ $labels.instance }})"
          description: "{{ $labels.instance }} 运行进程数超过 500，当前值: {{ $value }}"

      # 18. 硬件温度过高
      #- alert: 硬件温度过高
      #  expr: node_hwmon_temp_celsius > 80
      #  for: 5m
      #  labels:
      #    severity: critical
      #  annotations:
      #    summary: "硬件温度过高 (instance {{ $labels.instance }})"
      #    description: "{{ $labels.instance }} {{ $labels.sensor }} 温度超过 80°C，当前值: {{ $value }}°C"

      # 20. 僵尸进程检测
      - alert: 僵尸进程检测
        expr: node_procs_zombie > 0
        for: 10m
        labels:
          severity: warning
        annotations:
          summary: "发现僵尸进程 (instance {{ $labels.instance }})"
          description: "{{ $labels.instance }} 发现 {{ $value }} 个僵尸进程"
```

## 5、PrometheusAlert 飞书精美告警模板

> ⚠️ **避坑升级**：此版本已将原代码中的 `{{GetCSTtime}}` 升级替换为更稳健的内置时间算子 `{{TimeFormat}}`，完美解决时区乱码及高版本报错参数异常的问题。

- **配置路径**：登录 `http://IP:8080` ➔ **模板管理** ➔ **自定义模板** ➔ 修改或新增名为 **`prometheus-fs`** 的自定义模板。

```Plain
{{$var := .externalURL}}{{range $k,$v:=.alerts}}{{- if eq $v.status "resolved"}}
✅ **【恢复通知】**
【告警集群】 {{$v.labels.job}}
【告警名称】{{$v.labels.alertname}}
【开始时间】{{TimeFormat $v.startsAt "2006-01-02 15:04:05"}}
【结束时间】{{TimeFormat $v.endsAt "2006-01-02 15:04:05"}}
【节点地址】{{$v.labels.instance}}
【告警详情】{{$v.annotations.description}}
{{- else}}
🚨 **【告警通知】**
【告警集群】 {{$v.labels.job}}
【告警名称】{{$v.labels.alertname}}
【开始时间】{{TimeFormat $v.startsAt "2006-01-02 15:04:05"}}
【节点地址】{{$v.labels.instance}} 
【告警详情】{{$v.annotations.description}}
{{- end}}
{{end -}}
```
