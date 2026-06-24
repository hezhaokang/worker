# 服务端

## 创建LVS磁盘

1. ### 创建物理卷 (PV)

将整块 `sdb` 磁盘初始化为 LVM 物理卷：

```Plain
pvcreate /dev/sdb
```

*验证命令：**`pvs`* *看到 sdb 即可。*

1. ### 创建卷组 (VG)

创建一个名为 `data_vg` 的卷组（名字可以自定义）：

```Plain
vgcreate data_vg /dev/sdb
```

*验证命令：**`vgs`* *看到 data_vg 即可。*

1. ### 创建逻辑卷 (LV)

在 `data_vg` 卷组中分配全部空间，创建一个名为 `data_lv` 的逻辑卷：

```Plain
lvcreate -l 100%FREE -n data_lv data_vg
```

*验证命令：**`lvs`* *就能看到你刚创建的 data_lv 了。*

1. ### 格式化逻辑卷

将该逻辑卷格式化为主流且稳定的 **`ext4`** 文件系统（由于你的系统 `/` 也是 ext4，保持一致即可）：

```Plain
mkfs.ext4 /dev/data_vg/data_lv
```

1. ### 挂载到 `/data` 并配置开机自动挂载

#### 临时挂载（立即生效）：

```Plain
mount /dev/data_vg/data_lv /data
```

#### 开机自动挂载（永久生效）：

为了防止服务器重启后挂载丢失，需要将挂载信息写入 `/etc/fstab`：

```Plain
echo '/dev/data_vg/data_lv /data ext4 defaults 0 0' >> /etc/fstab
```

### 验证最终结果

全部执行完后，你可以通过以下命令检查是否成功：

```Plain
# 1. 检查挂载和空间大小
df -Th /data

# 2. 检查 LVM 树状结构
lsblk
```

服务端操作

```PowerShell
apt update
apt install nfs-kernel-server -y

mkdir -p /data/nfs
chmod 777 /data/nfs

vim /etc/exports
/data/nfs 172.22.0.0/16(rw,sync,no_root_squash,no_subtree_check)

systemctl restart nfs-kernel-server 
systemctl enable nfs-kernel-server
systemctl status nfs-kernel-server.service
```

# 客户端

```PowerShell
apt-get install nfs-common -y
```

# K8s 配置 NFS 动态存储

我们需要创建 4个文件：`rbac.yaml`（权限）、`deployment.yaml`（启动驱动）、`storageclass.yaml`（存储类）和 `test-pvc.yaml`（测试验证）。

### 配置 RBAC 权限 (`rbac.yaml`)

```YAML
apiVersion: v1
kind: ServiceAccount
metadata:
  name: nfs-client-provisioner
  namespace: kube-system
---
kind: ClusterRole
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: nfs-client-provisioner-runner
rules:
  - apiGroups: [""]
    resources: ["persistentvolumes"]
    verbs: ["get", "list", "watch", "create", "delete"]
  - apiGroups: [""]
    resources: ["persistentvolumeclaims"]
    verbs: ["get", "list", "watch", "update"]
  - apiGroups: ["storage.k8s.io"]
    resources: ["storageclasses"]
    verbs: ["get", "list", "watch"]
  - apiGroups: [""]
    resources: ["events"]
    verbs: ["create", "update", "patch"]
---
kind: ClusterRoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: run-nfs-client-provisioner
subjects:
  - kind: ServiceAccount
    name: nfs-client-provisioner
    namespace: kube-system
roleRef:
  kind: ClusterRole
  name: nfs-client-provisioner-runner
  apiGroup: rbac.authorization.k8s.io
---
kind: Role
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: leader-locking-nfs-client-provisioner
  namespace: kube-system
rules:
  - apiGroups: [""]
    resources: ["endpoints"]
    verbs: ["get", "list", "watch", "create", "update", "patch"]
---
kind: RoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: leader-locking-nfs-client-provisioner
  namespace: kube-system
subjects:
  - kind: ServiceAccount
    name: nfs-client-provisioner
    namespace: kube-system
roleRef:
  kind: Role
  name: leader-locking-nfs-client-provisioner
  apiGroup: rbac.authorization.k8s.io
```

### 部署 Provisioner 驱动 (`deployment.yaml`)

**请注意修改文件最下方的`NFS_SERVER`、`NFS_PATH` 以及 `volumes` 中的服务器 IP 和路径。**

```YAML
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nfs-client-provisioner
  namespace: kube-system
  labels:
    app: nfs-client-provisioner
spec:
  replicas: 1
  strategy:
    type: Recreate
  selector:
    matchLabels:
      app: nfs-client-provisioner
  template:
    metadata:
      labels:
        app: nfs-client-provisioner
    spec:
      serviceAccountName: nfs-client-provisioner
      containers:
        - name: nfs-client-provisioner
          image: registry.k8s.io/sig-storage/nfs-subdir-external-provisioner:v4.0.2 # 如果国内无法下载，可换为 registry.cn-hangzhou.aliyuncs.com/rc_k8s/nfs-subdir-external-provisioner:v4.0.2
          volumeMounts:
            - name: nfs-client-root
              mountPath: /persistentvolumes
          env:
            - name: PROVISIONER_NAME
              value: k8s-sigs.io/nfs-subdir-external-provisioner
            - name: NFS_SERVER
              value: 172.22.5.246          # 填入你的 NFS 服务器 IP 
            - name: NFS_PATH
              value: /data/nfs             # 填入你的 NFS 共享路径
      volumes:
        - name: nfs-client-root
          nfs:
            server: 172.22.5.246           # 填入你的 NFS 服务器 IP
            path: /data/nfs                # 填入你的 NFS 共享路径
```

创建 StorageClass 存储类 (`storageclass.yaml`)

```YAML
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: nfs-storage
  annotations:
    storageclass.kubernetes.io/is-default-class: "true" # 设为集群默认存储类
provisioner: k8s-sigs.io/nfs-subdir-external-provisioner 
reclaimPolicy: Retain                                # 回收策略：Retain(保留数据) 或 Delete(自动删除数据)
volumeBindingMode: Immediate
```

应用上述配置：

```Plain
kubectl apply -f rbac.yaml
kubectl apply -f deployment.yaml
kubectl apply -f storageclass.yaml
```

检查 Pod 是否正常启动：

```Plain
kubectl get pod -n kube-system -l app=nfs-client-provisioner
```