```shell
sda      500G
├─sda1   1G  /boot
└─sda2  49G
   ├─centos-root 48G /
   └─centos-swap 1G
```

```
原磁盘扩容
sda：50G → 500G
根目录：/dev/mapper/centos-root
文件系统：XFS 或 EXT4
```

------

# Linux 根目录扩容（LVM）操作笔记

先确认当前情况：

```bash
lsblk
df -Th
```

查看：

- 磁盘：`sda`
- 分区：`sda2`
- 根目录：`/`
- 文件系统：`xfs` 或 `ext4`

例如：

```bash
NAME            SIZE MOUNTPOINT
sda             500G
├─sda1            1G /boot
└─sda2           49G
  ├─centos-root  48G /
```

------

# 方案一：系统有 growpart（推荐）

安装：

```bash
yum install -y cloud-utils-growpart
```

检查：

```bash
which growpart
```

输出：

```bash
/usr/bin/growpart
```

------

## 第一步：扩容分区

扩大 sda2：

```bash
growpart /dev/sda 2
```

检查：

```bash
lsblk
```

结果：

```bash
sda
└─sda2 499G
```

------

## 第二步：扩容 LVM

刷新 PV：

```bash
pvresize /dev/sda2
```

查看：

```bash
pvs
```

------

扩容根卷：

```bash
lvextend -l +100%FREE /dev/centos/root
```

------

## 第三步：扩容文件系统

### XFS

查看：

```bash
df -Th
```

输出：

```bash
xfs
```

执行：

```bash
xfs_growfs /
```

------

### EXT4

查看：

```bash
df -Th
```

输出：

```bash
ext4
```

执行：

```bash
resize2fs /dev/mapper/centos-root
```

------

## 第四步：验证

```bash
df -h
```

完成。

------

# 方案二：没有 growpart（parted 扩容）

适用于：

```bash
yum安装失败
磁盘满
无网络
```

确认：

```bash
which parted
```

------

## 第一步：扩容分区

进入：

```bash
parted /dev/sda
```

执行：

```bash
print
resizepart 2 100%
Yes
quit
```

刷新：

```bash
partprobe
```

确认：

```bash
lsblk
```

结果：

```bash
sda
└─sda2 499G
```

------

## 第二步：扩容 LVM

刷新：

```bash
pvresize /dev/sda2
```

查看：

```bash
pvs
```

扩容：

```bash
lvextend -l +100%FREE /dev/centos/root
```

------

## 第三步：扩容文件系统

### XFS

```bash
xfs_growfs /
```

------

### EXT4

```bash
resize2fs /dev/mapper/centos-root
```

------

## 第四步：验证

```bash
df -h
```

------

# 常用检查命令

查看分区：

```bash
lsblk
```

查看文件系统：

```bash
df -Th
```

查看 LVM：

```bash
pvs
vgs
lvs
```

查看磁盘：

```bash
fdisk -l
```

------

# 一键思路（记忆版）

### 有 growpart

```
growpart
↓
pvresize
↓
lvextend
↓
xfs_growfs / resize2fs
```

------

### 无 growpart

```
parted
↓
partprobe
↓
pvresize
↓
lvextend
↓
xfs_growfs / resize2fs
```