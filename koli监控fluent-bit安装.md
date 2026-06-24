# 1、koli部署

```

```



# 2、fluent-bit部署

```yaml
# cat ns.yaml 
apiVersion: v1
kind: Namespace
metadata:
  name: logging

```



```yaml
# cat fluent-config.yaml 
apiVersion: v1
kind: ConfigMap
metadata:
  name: fluent-bit-config
  namespace: logging

data:
  fluent-bit.conf: |
    [SERVICE]
        Flush        1
        Log_Level    info

        Parsers_File parsers.conf

        HTTP_Server  On
        HTTP_Listen  0.0.0.0
        HTTP_Port    2020

        storage.path /var/log/flb-storage
        storage.sync normal
        storage.backlog.mem_limit 200M
        storage.metrics on


    [INPUT]
        Name              tail
        Path              /var/log/containers/*.log
        Parser            docker
        Tag               kube.*
        Mem_Buf_Limit     100MB
        Skip_Long_Lines   On


    [FILTER]
        Name              kubernetes
        Match             kube.*
        Merge_Log         On
        Keep_Log          Off
        K8S-Logging.Parser On
        K8S-Logging.Exclude On

    [FILTER]
        Name modify
        Match kube.*
        Add cluster liandanxia-token-test

    [OUTPUT]
        Name loki
        Match kube.*

        Host 172.22.5.245
        Port 3100

        Labels job=fluentbit,cluster=liandanxia-token-test

        label_keys $kubernetes['namespace_name'],$kubernetes['pod_name'],$kubernetes['container_name']


  parsers.conf: |
    [PARSER]
        Name        docker
        Format      json
        Time_Key    time
        Time_Format %Y-%m-%dT%H:%M:%S.%L
```



```yaml
# cat fluent-SA.yaml 
apiVersion: v1
kind: ServiceAccount
metadata:
  name: fluent-bit
  namespace: logging
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: fluent-bit
rules:
  - apiGroups: [""]
    resources:
      - pods
      - namespaces
    verbs: ["get", "list", "watch"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: fluent-bit
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: fluent-bit
subjects:
  - kind: ServiceAccount
    name: fluent-bit
    namespace: logging
```



```yaml
# cat fluent-DS.yaml 
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: fluent-bit
  namespace: logging
spec:
  selector:
    matchLabels:
      app: fluent-bit

  template:
    metadata:
      labels:
        app: fluent-bit

    spec:
      serviceAccountName: fluent-bit

      tolerations:
        - key: node-role.kubernetes.io/control-plane
          operator: Exists
          effect: NoSchedule
        - key: node-role.kubernetes.io/master
          operator: Exists
          effect: NoSchedule

      containers:
        - name: fluent-bit
          image: docker.m.daocloud.io/fluent/fluent-bit:3.2.10

          resources:
            requests:
              cpu: 100m
              memory: 100Mi
            limits:
              memory: 200Mi

          volumeMounts:
            - name: varlog
              mountPath: /var/log

            - name: varlibcontainers
              mountPath: /var/lib/docker/containers
              readOnly: true

            - name: config
              mountPath: /fluent-bit/etc/

            - name: storage
              mountPath: /var/log/flb-storage

      volumes:
        - name: varlog
          hostPath:
            path: /var/log

        - name: varlibcontainers
          hostPath:
            path: /var/lib/docker/containers

        - name: config
          configMap:
            name: fluent-bit-config

        - name: storage
          hostPath:
            path: /var/log/flb-storage

```



# 3、grafana部署

```

```



# 4、 打开日志查询

左边：

```
Explore
```

顶部选择：

```
Loki
```

输入：

查看所有日志：

```
{cluster="ldx-prod"}
```

------

查看某个 namespace：

```
{namespace="default"}
```

------

查看某个 Pod：

```
{pod=~".*nginx.*"}
```

------

查看错误：

```
{cluster="ldx-prod"} |= "error"
```

------

查看 ingress：

```
{namespace="ingress-nginx"}
```





```
swr.cn-north-4.myhuaweicloud.com/ddn-k8s/docker.io/grafana/alloy:v1.8.3
```

