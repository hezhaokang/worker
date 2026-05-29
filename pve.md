# 退出集群

```bash
systemctl stop pvc-cluster && systemctl stop corosync
pmxcfs -l
rm /etc/pve/corosync.conf && rm -rf /etc/corosync/*
killall pmxcfs && systemctl start pvc-cluster
```



# 时钟同步

```bash
# 强制让 chrony 立即步进调整时间（即使偏差很大）
chronyc -a makestep

# 查看时间源同步情况
chronyc sources -v
```

# ceph检查

```bash
ceph -s
# 或者
ceph status
```

