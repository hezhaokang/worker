# 1、ubuntu重启自动生成yaml文件

## 示例

```bash
root@ubuntu101:~# ip a
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host 
       valid_lft forever preferred_lft forever
2: ens33: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel state UP group default qlen 1000
    link/ether 00:50:56:2d:5a:8c brd ff:ff:ff:ff:ff:ff
    altname enp2s1
    inet 10.0.0.101/24 brd 10.0.0.255 scope global ens33
       valid_lft forever preferred_lft forever
    inet 10.0.0.32/24 brd 10.0.0.255 scope global secondary ens33
       valid_lft forever preferred_lft forever
    inet6 fe80::250:56ff:fe2d:5a8c/64 scope link 
       valid_lft forever preferred_lft forever

root@ubuntu101:~# ls /etc/netplan/
00-installer-config.yaml	50-cloud-init.yaml
```

## 解决办法

创建：

```bash
mkdir -p /etc/cloud/cloud.cfg.d
```

写入：

```bash
vim /etc/cloud/cloud.cfg.d/99-disable-network-config.cfg
```

内容：

```bash
network: {config: disabled}
```

保存后执行：

```bash
rm -f /etc/netplan/50-cloud-init.yaml
netplan generate
netplan apply
```

然后自己建：

```bash
vim /etc/netplan/00-installer-config.yaml
```

示例：

```yaml
network:
  ethernets:
    eth0:
      addresses:
        - 10.0.0.101/24
      routes:
        - to: default
          via: 10.0.0.2
      nameservers:
        addresses:
          - 8.8.8.8
          - 114.114.114.114
  version: 2
```



# 2、修改网卡名为eth0

### 1 编辑 GRUB

打开：

```bash
vim /etc/default/grub
```

找到：

```
GRUB_CMDLINE_LINUX=""
```

改成：

```
GRUB_CMDLINE_LINUX="net.ifnames=0 biosdevname=0"
```

保存。

------

### 2 更新 grub

Ubuntu：

```
update-grub
```

------

### 3 更新 netplan 配置

先看：

```
ls /etc/netplan
```

编辑：

```
vim /etc/netplan/00-installer-config.yaml
```

把：

```
ens33:
```

改成：

```
eth0:
```

例如：

```
network:
  version: 2
  ethernets:
    eth0:
      dhcp4: true
```

------

### 4 应用并重启

```
netplan generate
reboot
```

------

启动后查看：

```
ip a
```



 