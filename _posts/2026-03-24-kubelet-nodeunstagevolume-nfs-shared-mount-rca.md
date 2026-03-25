---
layout: post
title: "根因分析(RCA)：NFS Shared Mount 架构下触发 Kubelet NodeUnstageVolume 拦截的底层机制"
subtitle: "问题看起来在 CSI，结果锅甩到了 NFS 内核实现"
date: 2026-03-11 15:45:00 +0800
author: "Poke"
header-img: "img/post-bg-unix-linux.jpg"
catalog: true
tags:
  - Debug趣事
  - RCA
  - Kubernetes
  - CSI
  - NFS
categories:
  - Debug趣事
  - Kubernetes
  - Storage
---

### 摘要

在采用 **Shared Mount + Bind Mount** 架构的NFS CSI 中，我们观察到 NFS 与 FUSE 在删除最后一个 Pod 时表现出完全不同的行为：

>- **NFS 场景**：`NodeUnpublishVolume` 成功，但 `NodeUnstageVolume` 被 Kubelet 持续拦截，报错 `is still mounted by other references`，最终导致 `globalmount` 残留。
>- **FUSE 场景**：`NodeUnpublishVolume` 和 `NodeUnstageVolume` 均可正常执行，无挂载残留。

本问题的根因并不是 CSI 驱动本身的直接逻辑错误，也不能简单归因于“内核态文件系统 vs 用户态文件系统”的普遍差异，而是：

> **Kubelet 在执行 `NodeUnstageVolume` 前，会通过 `GetMountRefs/SearchMountPoints` 按 `mountinfo` 中的 `Root + Major + Minor` 搜索其他挂载引用；而 NFS 又通过 `show_path/show_devname` 特化了 `/proc/self/mountinfo` 的输出语义，导致 shared mount 与 globalmount 在 Kubelet 的判定维度上被识别为同源引用，最终持续拦截 `NodeUnstageVolume`。**

总结：

> **NFS 不是 bind 后 root 没变，而是 `mountinfo` 把它“显示成没变”；FUSE 则把真实 bind root 暴露出来了。**

---

### 1 问题背景

#### 1.1 架构模式

> **CSI是底层共享挂载 (Shared Mount) + 上层绑定挂载 (Bind Mount)的架构模式。在开发gluster存储的FUSE CSI时,考虑到客户的业务量和fuse客户端资源开销，决定使用shared mount的架构解决此些问题。在需要给客户提供ceph存储的NFS CSI时为了解决这些问题同样使用这种结构。**

这种架构的目标是：

- 复用底层网络连接
- 降低重复 mount 成本
- 用子目录实现多卷隔离

---

#### 1.2 故障现象
##### 故障描述
Kubernetes CSI 驱动中，在删除工作负载时，会在 `NodeUnstageVolume` 阶段发生完全相左的现象：
- 架构模式： 针对同一个网络文件系统，驱动只在 Node 上建立一个底层共享连接（如：`/var/lib/anna/csi/shared/...`）。随后通过 `mount --bind`，将该共享路径的不同子目录分别挂载给 Kubelet 的 `globalmount`（Stage 阶段）和 Pod 的目录（Publish 阶段）。
- 故障现象 (NFS 介质)： 当该节点上只剩 1 个运行的 Pod 时并将其删除，`NodeUnpublishVolume` 能够成功执行。但紧随其后的 `NodeUnstageVolume` 会一直重试并失败，Kubelet 报错：`is still mounted by other references`。这直接导致 `globalmount` 挂载路径残留（泄漏）。
- 对比现象 (FUSE 介质)： 在完全一样的 CSI 代码架构下，删除唯一的 Pod 时，`NodeUnpublishVolume` 和 `NodeUnstageVolume` 均能 100% 成功执行，无残留。

---

### 2 挂载信息检查
#### 2.1 Kubelet 安全检查机制

在调用 CSI 驱动的 `NodeUnstageVolume` 之前，Kubelet 会调用 `k8s.io/utils/mount.GetMountRefs(targetPath)` 进行安全性预检，以防误删正在被其他容器使用的块设备。
提取要卸载的目标（globalmount）在系统内核记录中的挂载源字符串（Source），在全系统的 `/proc/self/mountinfo`中搜寻所有具备相同 Source 字符串的其他挂载点。
```text
- fuse
	root@worker3:~# cat /proc/self/mountinfo | grep fuse
	957 31 0:70 / /var/lib/alamo/csi/shared/dnstest.test.cn_vol3 rw,relatime shared:326 - fuse_alamo.alamofs dnstest.test.cn:vol3 rw,user_id=0,group_id=0,default_permissions,allow_other,max_read=1048576
	992 31 0:70 /k8s-volumes/pvc-2d7f6a05-3ca4-4d6d-bfb3-8b235abe9556 /var/lib/kubelet/plugins/kubernetes.io/csi/alamo.csi.com/572944b0e09f41232bf084e030e83399f3944a38396af73f9d8cec561592716a/globalmount rw,relatime shared:326 - fuse_alamo.alamofs dnstest.test.cn:vol3 rw,user_id=0,group_id=0,default_permissions,allow_other,max_read=1048576
	1075 31 0:70 /k8s-volumes/pvc-2d7f6a05-3ca4-4d6d-bfb3-8b235abe9556 /var/lib/kubelet/pods/58c07a46-d314-4b54-8edc-2411c6f7c1f9/volumes/kubernetes.io~csi/pvc-2d7f6a05-3ca4-4d6d-bfb3-8b235abe9556/mount rw,relatime shared:326 - fuse_alamo.alamofs dnstest.test.cn:vol3 rw,user_id=0,group_id=0,default_permissions,allow_other,max_read=1048576
- nfs
	root@k8s-2:~# cat /proc/self/mountinfo | grep nfs
	1008 30 0:190 / /var/lib/anna/csi/shared/10.46.220.116 rw,relatime shared:192 - nfs4 10.46.220.116:/ rw,vers=4.0,rsize=1048576,wsize=1048576,namlen=255,hard,proto=tcp,timeo=600,retrans=2,sec=sys,clientaddr=10.61.52.187,local_lock=none,addr=10.46.220.116
	1090 30 0:190 / /var/lib/kubelet/plugins/kubernetes.io/csi/anna.csi.com/bd28b9558d1dfa24db6ab373a91aea83fd2d19b39c4a5635316e008a1906a151/globalmount rw,relatime shared:192 - nfs4 10.46.220.116:/k8s-volumes/pvc-923336bd-3a35-442c-b29a-5dfd26aac964 rw,vers=4.0,rsize=1048576,wsize=1048576,namlen=255,hard,proto=tcp,timeo=600,retrans=2,sec=sys,clientaddr=10.61.52.187,local_lock=none,addr=10.46.220.116
	1127 30 0:190 / /var/lib/kubelet/pods/cba3d5d4-ad70-47a9-aa7f-0555e444e8b9/volumes/kubernetes.io~csi/pvc-923336bd-3a35-442c-b29a-5dfd26aac964/mount rw,relatime shared:192 - nfs4 10.46.220.116:/k8s-volumes/pvc-923336bd-3a35-442c-b29a-5dfd26aac964 rw,vers=4.0,rsize=1048576,wsize=1048576,namlen=255,hard,proto=tcp,timeo=600,retrans=2,sec=sys,clientaddr=10.61.52.187,local_lock=none,addr=10.46.220.116
```

搜索出相同 Source 的挂载引用数 > 1（存在除了自己以外的其他引用），Kubelet 检查到某个 `globalmount` 仍有其他引用，就会拒绝继续执行卸载，从而拦截 `NodeUnstageVolume`,通过`/proc/self/mountinfo`发现nfs类型的`root`均为`/`。

#### 2.2 内核数据采样 (Linux findmnt)

我们对仅存在 1 个活跃 Pod 时的节点环境提取内核级的挂载树结构，重点观察由 Bind Mount 产生的 SOURCE 字符串字段。
```text 
[NFS 的内核上报特征 (Kernel FS)]
root@k8s-2:~# findmnt -o TARGET,SOURCE,FSTYPE,MAJ:MIN | grep 116
├─/var/lib/anna/csi/shared/10.46.220.116    10.46.220.116:/                                                            nfs4           0:190 
├─/var/lib/kubelet/xxxx/globalmount         10.46.220.116:/k8s-volumes/pvc-923336bd-3a35-442c-b29a-5dfd26aac964        nfs4           0:190
└─/var/lib/kubelet/xxxx/mount               10.46.220.116:/k8s-volumes/pvc-923336bd-3a35-442c-b29a-5dfd26aac964        nfs4           0:190

[FUSE 的内核上报特征 (Userspace FS)]
root@k8s-2:~# findmnt -o TARGET,SOURCE,FSTYPE,MAJ:MIN | grep vol3                
├─/var/lib/alamo/csi/shared/10.46.220.114_vol3              10.46.220.114:vol3                                                                fuse.alamofs   0:247
├─/var/lib/kubelet/xxxx/globalmount                         10.46.220.114:vol3[/k8s-volumes/pvc-59ee5d24-fb55-4735-af0a-1c8cb60f4caa]         fuse.alamofs   0:247
├─/var/lib/kubelet/xxxx/mount                               10.46.220.114:vol3[/k8s-volumes/pvc-59ee5d24-fb55-4735-af0a-1c8cb60f4caa]         fuse.alamofs   0:247
```

从该文本特征上看，NFS 和 FUSE 的 globalmount SOURCE 字符串，都与它们对应的底层 shared SOURCE 字符串不一样！ 如果仅仅依靠文本匹配，那它们都不应该找到底层 shared 目录。为何 NFS 却“精准踩雷”？这需要深入 Kubelet 内核级的设备检索机制。

#### 2.3 Kubelet 引擎模拟验证 (Go 代码注入)

我们在故障节点上直接调用运行了 Kubelet 底层的 GetMountRefs 算法，对其分析视角进行还原：

- 针对 NFS globalmount 运行： Kubelet 精确地检索到了 2 个冲突引用：一个是被 Pod 占用的 Mount，另一个是被我们作为底层架构缓存的 Shared Mount。因为计数为 2，此时即使 Pod 移除，底层 Shared 依然存在，导致引用数死锁为 1，永远无法卸载。
```text
root@k8s-2:~# ./check_mount /var/lib/kubelet/plugins/kubernetes.io/csi/anna.csi.com/bd28b9558d1dfa24db6ab373a91aea83fd2d19b39c4a5635316e008a1906a151/globalmount

=== 开始对挂载点进行 Kubelet 级深度体检 ===
检查目标: /var/lib/kubelet/plugins/kubernetes.io/csi/anna.csi.com/bd28b9558d1dfa24db6ab373a91aea83fd2d19b39c4a5635316e008a1906a151/globalmount
[第一项] IsLikelyNotMountPoint 返回: false (false 表示它确认这是一个合法的独立挂载根)

--- 原始底层 Stat 系统调用信息 ---
当前目录的 Device ID (Sys.Dev) : &{190 1099511627801 2 16895 0 0 0 0 0 1048576 1 {1773200806 192878352} {1773200806 192878352} {1773200807 297119250} [0 0 0]}
父级目录的 Device ID (Sys.Dev) : &{64768 2099682 3 16872 0 0 0 0 4096 4096 8 {1773216951 107098960} {773213055 732558564} {1773213055 732558564} [0 0 0]}
[分析]: 如果当前 Device ID 和 父级 Device ID 一样，Linux 就不认为这是一个跨文件系统的独立挂载根！

[输出] Kubelet GetMountRefs 认为在全系统中有 2 个其他挂载点与该 Device 冲突
    -> 冲突挂载 1: /var/lib/anna/csi/shared/10.46.220.116
    -> 冲突挂载 2: /var/lib/kubelet/pods/cba3d5d4-ad70-47a9-aa7f-0555e444e8b9/volumes/kubernetes.io~csi/pvc-923336bd-3a35-442c-b29a-5dfd26aac964/mount
```

- 针对 FUSE globalmount 运行： Kubelet 仅检索到 1 个冲突引用（即 Pod 本身）。这是因为 Kubelet 拿着 FUSE 那条变长的特殊 SOURCE 字符串去搜索全系统时，由于字符串长度不匹配，底层无后缀的 Shared 挂载点被 Kubelet 判定为“无关设备”，直接过滤出局。当 Pod 剥离后，Kubelet 检测冲突数为 0，从而“错误地”允许执行卸载。 
```text
root@k8s-2:~# mount | grep fuse
fusectl on /sys/fs/fuse/connections type fusectl (rw,nosuid,nodev,noexec,relatime)
10.46.220.114:vol3 on /var/lib/alamo/csi/shared/10.46.220.114_vol3 type fuse.alamofs (rw,relatime,user_id=0,group_id=0,default_permissions,allow_other,max_read=1048576)
10.46.220.114:vol3 on /var/lib/kubelet/plugins/kubernetes.io/csi/alamo.csi.com/b34a1dd34d9b450021d3daa0962747ed2a37aec77ae5cae0d557833ab0c99863/globalmount type fuse.alamofs (rw,relatime,user_id=0,group_id=0,default_permissions,allow_other,max_read=1048576)
10.46.220.114:vol3 on /var/lib/kubelet/pods/26a6633e-ce0d-4720-8886-3c10ad6f2997/volumes/kubernetes.io~csi/pvc-59ee5d24-fb55-4735-af0a-1c8cb60f4caa/mount type fuse.alamofs (rw,relatime,user_id=0,group_id=0,default_permissions,allow_other,max_read=1048576)
root@k8s-2:~# ./check_mount  /var/lib/kubelet/plugins/kubernetes.io/csi/alamo.csi.com/b34a1dd34d9b450021d3daa0962747ed2a37aec77ae5cae0d557833ab0c99863/globalmount 

=== 开始对挂载点进行 Kubelet 级深度体检 ===
检查目标: /var/lib/kubelet/plugins/kubernetes.io/csi/alamo.csi.com/b34a1dd34d9b450021d3daa0962747ed2a37aec77ae5cae0d557833ab0c99863/globalmount
[第一项] IsLikelyNotMountPoint 返回: false (false 表示它确认这是一个合法的独立挂载根)

--- 原始底层 Stat 系统调用信息 ---
当前目录的 Device ID (Sys.Dev) : &{247 12479527057063223492 2 16895 0 0 0 0 4096 131072 8 {1773215464 968548413} {1773215464 968548413} {1773215464 968548413} [0 0 0]}
父级目录的 Device ID (Sys.Dev) : &{64768 2364120 3 16872 0 0 0 0 4096 4096 8 {1773215547 782603043} {1773215547 782603043} {1773215547 782603043} [0 0 0]}
[分析]: 如果当前 Device ID 和 父级 Device ID 一样，Linux 就不认为这是一个跨文件系统的独立挂载根！

[输出] Kubelet GetMountRefs 认为在全系统中有 1 个其他挂载点与该 Device 冲突
    -> 冲突挂载 1: /var/lib/kubelet/pods/26a6633e-ce0d-4720-8886-3c10ad6f2997/volumes/kubernetes.io~csi/pvc-59ee5d24-fb55-4735-af0a-1c8cb60f4caa/mount
```

通过`GetMountRefs`检查挂载点发现能够判断出两个类型的引用数量有差别。

---

### 3 Kubelet refs机制

Kubernetes `mount-utils` 在实现 `SearchMountPoints()` 时，明确指出：对 bind mount 来说，单纯依赖 source 名称并不可靠，因此要用更稳定的维度标识挂载源。

其实际匹配条件如下：

```go
var refs []string
for i := range mis {
    if mis[i].ID == mountID {
        // Ignore mount entry for mount source itself.
        continue
    }
    if mis[i].Root == rootPath && mis[i].Major == major && mis[i].Minor == minor {
        refs = append(refs, mis[i].MountPoint)
    }
}
```

也就是说，**Kubelet 判定“是不是同源引用”依赖的是：**`Root`、`Major`、`Minor`。

##### 分析

FUSE 场景下：

> **shared path 的 `Root` 是：`/`, `globalmount` 和 `pod mount` 对应的`Root` 是`k8s-volumes/pvc-..`
虽然三者的 `Major:Minor` 一样：`0:70`,但由于 `Root` 不同，所以`shared path` 不会被 Kubelet 归并进 `globalmount` 的同源引用集合,`GetMountRefs(globalmount)` 只会看到 `Pod mount`。当 `Pod mount` 消失后，冲突引用数归零`NodeUnstageVolume` 能够继续执行。**

---

NFS 场景下：

> **`shared path`、`globalmount`、`pod mount` 三者的 `Root` 都是`/`,`Major:Minor` 也一致`0:190`,因此在 Kubelet 看来，这三者满足相同的`Root`、`Major`、`Minor`，于是它们会被归并到同一个同源引用集合里。这意味着`GetMountRefs(globalmount)` 不仅能找到 Pod mount，还会把底层 shared path 一并算进去。**

---

### 4 NFS mountinfo.root 独特风格

nfs挂载点在`mountinfo.root`中的输出并不是 VFS 固定写死的,`root`和`mount source`并不是同一套逻辑生成的。根据 `mountinfo(5)` 手册：
- `root` 表示：构成该挂载根的目录路径。
- `mount source` 表示：filesystem-specific information

也就是说`root` 与 `source` 本来就是两套语义，不要求文本一致。
Linux VFS 允许文件系统重写 `mountinfo` 的展示语义，Linux VFS 通过 `super_operations` 提供了两个关键钩子：
- `show_path` ：如何展示 `mountinfo` 的第 4 列 `root`
- `show_devname`:如何展示第 10 列 `mount source`

内核在 `fs/proc_namespace.c` 中的调用链就是：

```c
err = show_path(m, mnt->mnt_root);
...
err = sb->s_op->show_devname(m, mnt->mnt_root);
```
文件系统可以自己决定 `root` 怎么显示,也可以自己决定 `source` 怎么显示,二者允许不同步。

NFS 在 `super_operations` 中显式注册：

```c
.show_devname = nfs_show_devname,
.show_path = nfs_show_path,
```

其行为可以概括为：

- `nfs_show_path()`：固定输出 `/`
- `nfs_show_devname()`：根据传入的 `root dentry` 计算 `server:/path`

于是就产生了 NFS 场景中那个非常“反直觉”的组合：

```text
mountinfo.root   = /
mountinfo.source = 10.46.220.116:/k8s-volumes/pvc-...
```

这不是偶然，也不是日志错乱，而是 **NFS 代码的明确设计结果**。

---

### 5 根因定性

本问题的根因可以正式表述为：

> 本故障并非简单的 CSI 驱动逻辑缺陷，而是 **NFS 对 `/proc/self/mountinfo` 的 `show_path/show_devname` 展示语义** 与 **Kubelet `GetMountRefs/SearchMountPoints` 的同源挂载判定算法** 相互叠加后的结果。  
> Kubelet 实际按 `Root + Major + Minor` 搜索“其他引用”；而 NFS 通过 `show_path()` 将多个 bind 子目录挂载统一显示为 `root=/`，同时又通过 `show_devname()` 展示不同的 `server:/subdir`。  
> 在 shared mount + bind mount 架构下，这会导致 shared path 与 globalmount 在 Kubelet 视角中被识别为同源挂载，从而使 shared path 成为 `globalmount` 的长期残余引用，最终阻塞 `NodeUnstageVolume`。

因此，本问题的本质是：

> **NFS 对 mountinfo.root 的展示语义与 Kubelet 引用匹配算法之间的组合效应。而不是 CSI 驱动单独实现错误。**

---

### 6 设计与规避建议

#### 6.1 建议一：对 NFS/SMB 这类目录型网络文件系统，尽量不要声明 `STAGE_UNSTAGE_VOLUME`

这是最稳妥的规避方式。

Kubernetes CSI 文档本身就明确说明，Node 侧真正必需的方法只有：

- `NodePublishVolume`
- `NodeUnpublishVolume`
- `NodeGetCapabilities`

而：

- `STAGE_UNSTAGE_VOLUME`

属于**可选能力**，其典型适用场景是：

> 每节点的全局块设备挂载

对于 NFS / SMB / 目录型共享存储，这个能力并不是天然必要的。

###### 如果移除该能力：
- Kubelet 将跳过 `NodeStageVolume / NodeUnstageVolume`
- 也就不会触发 `GetMountRefs()` 这套拦截逻辑
- 卷生命周期可完全收敛到：
  - `NodePublishVolume`
  - `NodeUnpublishVolume`

这通常也是共享目录型 CSI 更稳的做法。

---

#### 6.2 建议二：如果必须保留 Stage/Unstage，就不要让 shared path 成为 kubelet 视角下的长期额外引用

如果出于架构原因必须保留 `NodeStageVolume / NodeUnstageVolume`，那就必须满足：

> shared mount 不能作为 kubelet 所在 mount namespace 中，与 `globalmount` 共同落入同一组 `Root + Major + Minor` 视图的长期额外引用。

否则：

- Kubelet 会一直把 shared path 视作残余引用
- `NodeUnstageVolume` 就无法完成

这一点在 NFS 上尤其敏感。

---

### 7 最终结论

本问题不是“CSI 协议层的纯实现 bug”，而是以下三者共同作用的结果：

1. **共享挂载 + bind mount** 的 CSI 架构
2. **NFS 对 `/proc/self/mountinfo` 的特化展示语义**
3. **Kubelet `SearchMountPoints()` 基于 `Root + Major + Minor` 的同源判定**

最终导致：

- shared mount 被 Kubelet 识别为 `globalmount` 的长期残余引用
- `NodeUnstageVolume` 被持续拦截
- `globalmount` 无法卸载

总结：

> **NFS 不是 bind 后 root 没变，而是 `mountinfo` 把它显示成没变；FUSE 则把真实 bind root 暴露出来了。**  
> **Kubelet 又恰好按 `Root + Major + Minor` 判断同源引用，于是 NFS 会把 shared mount 识别成 globalmount 的残余引用，而 FUSE 不会。**
