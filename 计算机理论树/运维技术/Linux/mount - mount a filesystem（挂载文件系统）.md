## 一、核心定位

`mount` 是 Linux 上**最核心、最基础的文件系统挂载工具**，属于「存储管理类」，普通用户通常不能挂载（需 root），用于**将硬盘分区、U 盘、ISO 镜像、网络文件系统（NFS/CIFS）等 “挂载” 到指定目录（挂载点）**，利用 “Linux 一切皆文件” 的特性，让设备可通过文件系统访问。

> 补充说明：`mount` 名称来源于 “mount”（挂载）；与 `umount`（卸载）配合使用；开机自动挂载需配置 `/etc/fstab` 文件；查看已挂载的文件系统可直接运行 `mount` 或 `df -h`。

---

## 二、语法格式

### 1. 查看已挂载的文件系统

bash

运行

```
mount [选项]
```

### 2. 挂载文件系统

bash

运行

```
mount [选项] 设备路径 挂载点目录
```

> 关键说明：
> 
> 1. **设备路径**：必选，如 `/dev/sda1`（硬盘分区）、`/dev/cdrom`（光盘）、`/path/to/file.iso`（ISO 镜像）。
> 2. **挂载点目录**：必选，需**先创建**（`mkdir`），且最好为空（非空也可，但原文件会被 “遮挡”）。
> 3. **默认行为**：无参数时显示所有已挂载的文件系统。

---

## 三、高频核心选项

表格

|选项|核心作用|高频使用场景|
|:--|:--|:--|
|`-t, --types TYPE`|指定**文件系统类型**（如 `ext4`、`xfs`、`vfat`、`ntfs-3g`、`iso9660`、`nfs`、`cifs`）|挂载特定类型的文件系统|
|`-o, --options OPTIONS`|指定**挂载选项**（逗号分隔多个选项）|自定义挂载行为，**最常用的选项之一**|
|`-a, --all`|挂载 `/etc/fstab` 中**所有未挂载的文件系统**|测试 `/etc/fstab` 配置、开机后手动挂载所有|
|`-l, --show-labels`|显示文件系统的**标签（Label）**|查看分区标签|
|`-h, --help`|输出帮助信息|快速查看参数|

### 常用 `-o` 挂载选项

表格

|选项|含义|说明|
|:--|:--|:--|
|`defaults`|默认选项|等价于 `rw,suid,dev,exec,auto,nouser,async`，日常最常用|
|`ro`|只读挂载|挂载为只读，不可写入|
|`rw`|读写挂载|挂载为可读写（默认）|
|`loop`|使用 loop 设备|挂载 ISO 镜像、文件作为块设备时**必须加**|
|`noatime`|不更新访问时间|减少磁盘 I/O，提升性能，推荐用于服务器|
|`nodiratime`|不更新目录访问时间|类似 `noatime`，仅针对目录|
|`uid/gid`|指定所有者 / 组 ID|挂载 FAT32/NTFS 等无权限的文件系统时使用|

---

## 四、核心示例

### 1. 查看已挂载的文件系统

bash

运行

```
# 查看所有已挂载的文件系统
mount

# 查看已挂载的文件系统，显示标签
mount -l

# 更直观的查看（推荐用 df -h）
df -h
```

### 2. 挂载本地硬盘分区

bash

运行

```
# 先创建挂载点
sudo mkdir -p /mnt/mydisk

# 挂载 /dev/sda1（ext4/xfs）到 /mnt/mydisk，用默认选项
sudo mount /dev/sda1 /mnt/mydisk

# 显式指定文件系统类型为 ext4
sudo mount -t ext4 /dev/sda1 /mnt/mydisk

# 只读挂载
sudo mount -o ro /dev/sda1 /mnt/mydisk

# 不更新访问时间挂载（提升性能）
sudo mount -o noatime /dev/sda1 /mnt/mydisk
```

### 3. 挂载 U 盘 / 移动硬盘

bash

运行

```
# 先查看 U 盘设备路径（用 lsblk 或 fdisk -l）
lsblk

# 假设 U 盘是 /dev/sdb1，FAT32 文件系统
sudo mkdir -p /mnt/usb
sudo mount /dev/sdb1 /mnt/usb

# 如果是 NTFS，需先安装 ntfs-3g（sudo apt install ntfs-3g）
sudo mount -t ntfs-3g /dev/sdb1 /mnt/usb
```

### 4. 挂载 ISO 镜像（必须加 loop！）

bash

运行

```
# 先创建挂载点
sudo mkdir -p /mnt/iso

# 挂载 ISO 镜像，必须加 -o loop！
sudo mount -o loop /path/to/file.iso /mnt/iso
```

### 5. 挂载网络文件系统（NFS）

bash

运行

```
# 先安装 NFS 客户端（sudo apt install nfs-common）
sudo mkdir -p /mnt/nfs

# 挂载 NFS 共享（服务器 IP:共享路径）
sudo mount -t nfs 192.168.1.10:/srv/nfs /mnt/nfs
```

### 6. 挂载 /etc/fstab 里的所有

bash

运行

```
# 挂载 /etc/fstab 中所有未挂载的文件系统（测试配置用）
sudo mount -a
```

---

## 五、避坑提示与注意事项

1. **挂载点必须先创建，且最好为空**
    
    - 挂载点目录必须存在，否则会报错；
    - 推荐：先 `sudo mkdir -p /mnt/xxx` 创建挂载点；
    - 典型错误：`sudo mount /dev/sda1 /mnt/mydisk`，但 /mnt/mydisk 不存在，报错；
    - 正确示例：先 `sudo mkdir -p /mnt/mydisk`，再 mount。
    
2. **普通用户不能挂载，要加 sudo**
    
    - 普通用户通常没有挂载权限（除非 /etc/fstab 配置了 `user` 选项）；
    - 解决方法：加 `sudo`；
    - 典型错误：普通用户直接 mount，报错；
    - 正确示例：`sudo mount ...`。
    
3. **挂载后原挂载点的文件会被 “遮挡”，卸载后才可见**
    
    - 如果挂载点目录非空，挂载后原文件会被 “遮挡”，看不到；
    - 解决方法：最好用空目录作为挂载点；
    - 典型错误：用非空目录挂载，原文件不见了；
    - 正确示例：用空目录挂载，或卸载后查看原文件。
    
4. **不要在挂载点目录下操作时卸载，会报错 “device is busy”**
    
    - 如果当前终端在挂载点目录下，或有进程在访问挂载点，卸载会报错；
    - 解决方法：先退出挂载点目录，或关闭访问的进程；
    - 典型错误：在 /mnt/usb 目录下执行 `sudo umount /mnt/usb`，报错；
    - 正确示例：先 `cd ~` 退出，再 umount。
    
5. **ISO 镜像必须加 `-o loop`**
    
    - ISO 是文件，不是块设备，必须用 loop 设备挂载；
    - 典型错误：`sudo mount file.iso /mnt/iso`，报错；
    - 正确示例：`sudo mount -o loop file.iso /mnt/iso`。
    
6. **开机自动挂载要改 /etc/fstab，且先测试 `mount -a`**
    
    - 开机自动挂载需在 `/etc/fstab` 中添加一行；
    - 推荐：添加后先 `sudo mount -a` 测试，没问题再重启，避免开不了机；
    - 典型错误：直接改 /etc/fstab 不测试，重启后挂载失败开不了机；
    - 正确示例：改完 /etc/fstab 先 `sudo mount -a` 测试。
    

---

## 六、同类对比

表格

|工具|核心作用|与 `mount` 的区别|
|:--|:--|:--|
|`umount`|卸载文件系统|mount 是挂载，umount 是卸载，两者配合使用。挂载用 mount，卸载用 umount。|
|`df`|查看已挂载文件系统的磁盘使用情况|mount 查看挂载信息；df 查看磁盘使用情况。挂载信息用 mount，磁盘使用用 df -h。|
|`lsblk`|查看块设备信息|mount 查看挂载；lsblk 查看块设备（硬盘 / 分区）结构。块设备用 lsblk，挂载用 mount。|

---

## 七、课后练习

1. **查看已挂载**：用 `mount` 和 `df -h` 查看已挂载的文件系统，对比两者的输出。
2. **创建挂载点**：`sudo mkdir -p /mnt/test` 创建一个测试挂载点。
3. **挂载 ISO（如果有）**：找一个 ISO 镜像文件，`sudo mount -o loop /path/to/file.iso /mnt/test` 挂载，然后 `ls /mnt/test` 查看内容，最后 `sudo umount /mnt/test` 卸载。
4. **挂载 U 盘（如果有）**：插入 U 盘，用 `lsblk` 查看设备路径，然后挂载到 /mnt/test，查看内容，最后卸载。
5. **测试 mount -a**：如果修改了 /etc/fstab（仅测试环境！），用 `sudo mount -a` 测试配置。
6. **查看 /etc/fstab**：用 `cat /etc/fstab` 查看开机自动挂载的配置文件，理解每一列的含义。