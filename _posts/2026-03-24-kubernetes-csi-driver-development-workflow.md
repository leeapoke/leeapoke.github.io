---
layout: post
title: "Kubernetes CSI Driver 开发全流程：从 0 到 1 做一个可用的存储插件"
subtitle: "从需求建模、挂载设计到测试排障，讲清楚 CSI 的工程化开发路径"
date: 2026-03-24 17:35:00 +0800
author: "Poke"
header-img: "img/post-bg-unix-linux.jpg"
catalog: true
tags:
  - Kubernetes
  - CSI
  - Storage
categories:
  - Kubernetes
  - Storage
---

### 摘要

很多人第一次接触 CSI（Container Storage Interface）时，会觉得它不过就是一组 RPC 接口：把 `CreateVolume`、`DeleteVolume`、`NodePublishVolume`、`NodeUnpublishVolume` 这些方法实现出来，一个 CSI Driver 差不多就完成了。

但真正开始动手做一个可上线、可维护、可排障的 CSI Driver 后，很快就会发现：**CSI 不是“接口开发”，而是“把底层存储系统完整接入 Kubernetes 生命周期”的系统工程。**

从卷模型怎么设计，到 Controller 和 Node 的职责怎么划分；从挂载架构该选“每卷独立挂载”还是“shared mount + bind mount”，到是否要声明 `STAGE_UNSTAGE_VOLUME`；再到后面的 `csi-sanity`、回归测试、线上问题定位、残留挂载清理，这些都不是实现几个接口就能自动解决的问题。

这篇文章不想重复官方文档里的概念说明，而是希望从工程实践角度，把 **“从 0 到 1 做一个 Kubernetes CSI Driver”** 的完整开发流程梳理清楚：

- 开发前到底该先想清楚什么
- 哪些能力应该先做，哪些能力不该急着做
- Controller / Node 分别应该承担什么职责
- 一个 CSI 从 MVP 走向产品化，中间到底还差什么

如果你正准备开始做一个 CSI Driver，或者已经写到一半、但感觉越写越乱，希望这篇文章能帮你把整个开发路径重新整理出来。

---

### 1 为什么要开发一个 CSI Driver

在 Kubernetes 里，应用使用存储时，表面上接触到的是：

- `PersistentVolumeClaim`
- `PersistentVolume`
- `StorageClass`
- Pod 中的 `volumeMounts`

但这些只是 Kubernetes 暴露给用户的抽象。真正负责把底层存储系统接进 Kubernetes 的，是 CSI Driver。

通常只有在以下几类场景里，团队才会认真考虑自己开发一个 CSI：

- 需要接入自研存储系统
- 需要把已有 NAS / NFS / CephFS / SMB / FUSE 文件系统接入 Kubernetes
- 需要支持存储系统自己的特性，比如 quota、回收站、快照、分层、冷热迁移、元数据查询等
- 需要让 Kubernetes 能统一管理底层卷对象的生命周期

如果只是“让 Pod 能挂一个目录”，那件事本身并不复杂；但一旦进入生产环境，问题会立刻变成：

- 卷是怎么创建出来的？
- 卷的唯一标识是什么？
- 节点上到底怎么挂载？
- 多个 Pod / 多个节点共享时怎么处理？
- 删除顺序谁来保证？
- 节点异常、kubelet 重试、挂载残留怎么处理？
- 你宣称支持的 CSI 能力，是否真的和底层实现一致？

所以，开发 CSI 的本质，从来不是“把接口补齐”，而是：

> **把一个存储系统真正映射到 Kubernetes 的卷生命周期模型上。**

而这件事，本质上是一个系统工程。

---

### 2 在写代码前，必须先做的建模工作

很多 CSI 最后会变得又乱又难维护，并不是因为开发者不会写 Go，也不是因为不会调用 `mount`，而是在一开始就跳过了“建模”这一步。

一个很常见的误区是：先照着 CSI proto 文件把接口都建出来，边写边想。结果写到一半才发现：

- 卷 ID 不够用
- 静态卷和动态卷语义冲突
- Controller 该做的事跑到了 Node 里
- NodeStage / NodePublish 语义混乱
- 删除流程和 kubelet 的预期不一致

所以，真正动手写代码前，建议先把下面这些问题定清楚。

---

#### 2.1 先决定你的存储到底是什么类型

这一步看起来基础，实际上决定了后面 80% 的实现风格。

##### 块存储型
例如：

- 云盘
- RBD
- iSCSI
- 本地块设备

这类 CSI 的特点通常是：

- 卷更像一个设备，而不是一个目录
- 更依赖 attach / detach
- 常常需要 `ControllerPublishVolume`
- `STAGE_UNSTAGE_VOLUME` 往往是合理的
- 扩容也更接近“块设备 resize + 文件系统扩容”

##### 文件存储型
例如：

- NFS
- CephFS
- SMB
- FUSE 文件系统
- NAS 子目录

这类 CSI 的特点则很不一样：

- 卷更像“一个路径”或“一个子目录”
- `ControllerPublishVolume` 很可能只是逻辑空实现
- `NodePublishVolume` 才是主角
- `STAGE_UNSTAGE_VOLUME` 未必必要
- 更容易踩 `globalmount`、bind mount、`GetMountRefs`、共享引用等问题

这一点如果一开始没分清，后面就容易出现一种“接口都对，但实现味道完全不对”的状态。

---

#### 2.2 先决定卷的生命周期模型

CSI 最大的价值，不是“能挂载”，而是让卷对象进入 Kubernetes 的生命周期管理中。

所以在开发前，需要先回答这些问题：

##### 你支持哪些卷类型？

- 只支持动态卷？
- 只支持静态卷？
- 还是动态卷和静态卷都支持？

##### 动态卷在后端对应什么实体？

- 子目录
- 子卷
- LUN
- Bucket
- 一个 quota 空间
- 一个逻辑对象

##### 删除语义是什么？

- 物理删除
- 回收站
- 标记删除
- 延迟清理

##### 扩容语义是什么？

- 扩块设备
- 调整目录 quota
- 更新元数据容量上限

这些问题没有标准答案，但必须一开始就明确，否则后面接口虽然写完了，语义却收不住。

---

#### 2.3 先设计 `VolumeId`

CSI 的 `VolumeId` 设计，很多团队一开始不太重视，结果后面越用越痛苦。

一个好的 `VolumeId` 应该满足：

- 唯一
- 稳定
- 可解析
- 能让 Controller 和 Node 都拿到必要信息
- 能兼容静态卷与动态卷

例如目录型存储里，常见的设计是：

```text
<fs_address>|<api_port>|<volume_name>
```

或者：

```text
clusterA:pool1:pvc-xxxx
```

如果一开始卷 ID 设计太随意，后面会出现很多连锁问题：

- `DeleteVolume` 无法可靠反查后端对象
- `NodePublishVolume` 拿不到足够信息
- 静态卷语义难以兼容
- 调试日志里根本看不出卷到底是谁

所以，**卷 ID 不是一个字符串细节，而是整个驱动的数据骨架。**

---

#### 2.4 先决定挂载架构

这一步尤其重要。很多 CSI 的复杂度，不来自协议，而来自你选了哪种挂载模型。

##### 方案 A：每卷独立挂载

例如每个卷都单独建立一条底层远端挂载。

优点：

- 生命周期清楚
- 卸载顺序简单
- 不容易被 kubelet 误判引用
- 问题定位相对容易

缺点：

- 开销大
- 网络连接数多
- 对节点和后端都不够友好

##### 方案 B：shared mount + bind mount

例如：

- 先建立一个共享底层挂载
- 再把不同卷对应的子目录 bind 到 `globalmount` / Pod 路径

优点：

- 复用底层连接
- 适合共享目录型存储
- 能显著减少重复 mount 成本

缺点：

- 生命周期复杂度陡增
- 很容易踩：
  - `globalmount` 残留
  - bind mount 引用链
  - kubelet `GetMountRefs`
  - shared mount 长期引用问题
- `NodeStage` / `NodePublish` 的边界更难定义

这一步没有绝对对错，但一旦选了 shared mount 架构，后面就必须同步设计：

- refCounter
- sharedPath 生命周期
- bind mount 清理顺序
- kubelet 卸载检查兼容性

否则问题只是“早爆还是晚爆”。

---

### 3 搞清楚 CSI 的三个服务到底做什么

CSI 从接口上看分三块，但很多实现里最大的问题是：这三块职责没分清，导致 Controller 和 Node 的边界互相污染。

---

#### 3.1 Identity Service

这是最轻的一层，但不能忽略。

典型接口：

- `GetPluginInfo`
- `GetPluginCapabilities`
- `Probe`

它们主要解决：

- 这个插件叫什么
- 插件宣称支持哪些能力
- 插件是否存活

Identity 看起来简单，但它决定了 sidecar、kubelet、external-provisioner 是否能正确识别这个驱动。

特别是 `GetPluginCapabilities` 和 `Probe`，不要随便写成“都支持”或者“永远成功”，否则后面你会发现系统按你宣称的能力模型来调你，而你其实并没准备好。

---

#### 3.2 Controller Service

Controller 负责的是“卷对象本身”的生命周期，而不是节点上的挂载动作。

常见接口包括：

- `CreateVolume`
- `DeleteVolume`
- `ValidateVolumeCapabilities`
- `ControllerExpandVolume`
- `ListVolumes`
- `GetCapacity`
- `CreateSnapshot`
- `DeleteSnapshot`

Controller 更像是存储控制平面，解决的问题是：

- 卷怎么创建
- 卷怎么删除
- 卷是否存在
- 卷的容量怎么变更
- 卷当前对外暴露什么能力

如果你的实现里，Controller 开始关心节点路径、Pod targetPath、bind mount 细节，那通常说明职责已经混了。

---

#### 3.3 Node Service

Node 才是 CSI 最难也最容易出问题的部分。

典型接口包括：

- `NodePublishVolume`
- `NodeUnpublishVolume`
- `NodeStageVolume`
- `NodeUnstageVolume`
- `NodeGetInfo`
- `NodeGetCapabilities`
- `NodeExpandVolume`

Node 这层真正面对的是：

- kubelet 的挂载语义
- Linux VFS
- mount namespace
- bind mount
- 文件系统驱动行为差异
- 多引用卸载
- 并发重试
- 节点异常恢复

你最后踩的大坑，十有八九都在 Node 层。

---

### 4 正确的开发顺序：先 MVP，再补完整能力

开发 CSI 时，一个非常实用的经验是：**不要一开始就追求“完整支持所有标准能力”**。先把最小可用闭环打通，再逐步补全，是更稳的方式。

---

#### 4.1 第一阶段：先做最小可用闭环

建议最先实现这些：

##### Identity
- `GetPluginInfo`
- `GetPluginCapabilities`
- `Probe`

##### Controller
- `CreateVolume`
- `DeleteVolume`
- `ValidateVolumeCapabilities`

##### Node
- `NodePublishVolume`
- `NodeUnpublishVolume`
- `NodeGetInfo`
- `NodeGetCapabilities`

做到这里，通常已经可以打通：

```text
StorageClass -> PVC -> PV -> Pod
```

这就是一个真正有价值的 MVP。

这里的重点不是“功能少”，而是“主链路完整”：

- 可以创建卷
- 可以绑定 PVC
- 可以挂到 Pod
- 可以卸载和删除

一个主链路完整的 MVP，比一堆半成品高级能力更有意义。

---

#### 4.2 第二阶段：把“能跑”补成“能用”

MVP 之后，下一步要补的是工程细节，而不是马上冲 snapshot。

建议优先补：

- 静态卷支持
- 动态卷参数校验
- 目录权限 / uid / gid
- volume stats
- 删除语义
- 幂等性
- 基础扩容
- 更稳定的错误码

这一步的目标是让驱动从“能通 demo”变成“业务能试用”。

---

#### 4.3 第三阶段：再补标准能力

等主链路、语义和回归稳定之后，再决定是否补这些：

- `ListVolumes`
- `GetCapacity`
- `Snapshot`
- `NodeExpandVolume`
- `ControllerGetVolume`

这样做的好处是：

- 不会一开始就在边缘能力里迷路
- 主链路一旦有问题也更容易排查
- 你更容易判断哪些能力对当前业务是真需求，哪些只是“标准上有但现在没必要”

---

### 5 Controller 侧完整开发流程

Controller 层最重要的不是“能返回一个 response”，而是你如何把 Kubernetes 的卷生命周期映射到底层存储对象上。

---

#### 5.1 `CreateVolume`

这是第一个核心接口。

一个完整的 `CreateVolume` 至少要考虑四件事：

1. **参数校验**
   - 卷名
   - 请求容量
   - access mode
   - volume capability
   - `StorageClass.parameters`

2. **幂等性**
   - 同名卷重复创建怎么办
   - 同名不同容量怎么办
   - 已存在时返回成功还是报错

3. **后端对象创建**
   - 创建目录 / 子卷 / LUN / 逻辑对象
   - 初始化 quota
   - 初始化元数据

4. **返回稳定的 `VolumeId`**
   - 让后续 `DeleteVolume` 和 `NodePublishVolume` 都能可靠解析

很多 CSI 的第一批 bug，实际上都出在这里：卷是能创建出来，但重复调用、异常重试、部分成功场景全都没设计。

---

#### 5.2 `DeleteVolume`

`DeleteVolume` 真正的重点，不是“删不删”，而是**删除语义**。

常见几种实现：

- 直接物理删除
- 先移动到回收站
- 标记删除，由异步清理器回收
- 仅删除 Kubernetes 侧对象，不立即删除后端内容

对于生产环境来说，“回收站”往往是更稳的方案：

- 便于误删恢复
- 便于排查
- 便于延迟清理

所以 `DeleteVolume` 不是单纯一个操作，而是你对“卷生命周期结束”这件事的设计表达。

---

#### 5.3 `ValidateVolumeCapabilities`

这个接口很容易被轻视，但它有很大价值。

它可以用来提前发现：

- 卷根本不存在
- 卷支持的访问模式和请求不一致
- `volumeMode` 不兼容
- 这根本不是你的卷

比起把错误拖到 `NodePublishVolume` 再暴露，提前在 Controller 侧拦下来，排障成本会低很多。

---

#### 5.4 `ControllerExpandVolume`

很多人会下意识地把“扩容”理解成块设备扩容。但在目录型或共享文件系统型 CSI 里，扩容的含义未必一样。

可能是：

- 提高 quota
- 增加逻辑容量上限
- 更新元数据记录

所以同样叫 `ControllerExpandVolume`，背后的存储语义可能完全不同。这一点文档里最好明确说清楚，不要默认读者会自动理解。

---

### 6 Node 侧完整开发流程

Node 侧是 CSI 开发里真正的主战场。Controller 侧很多问题是“设计没想清楚”，而 Node 侧很多问题则是：**你设计清楚了，也不一定能在 Linux 和 kubelet 的现实里顺利落地。**

---

#### 6.1 `NodePublishVolume`

这是最核心的节点接口之一。

典型流程通常是：

1. 校验请求参数
2. 准备 `targetPath`
3. 确定源路径
4. 如有需要，建立或复用 shared mount
5. 将卷源路径 bind 到 `targetPath`

如果是共享目录型存储，还要继续区分：

- 动态卷目录是否由 CSI 自动创建
- 静态卷目录是否要求预先存在
- 卷路径到底是目录名还是完整路径

这一层的代码看起来大多是 mount / mkdir / stat / bind，但真正的复杂点在于：

- 这个源路径到底是什么语义
- 它和 `VolumeId`、`volumeContext`、StorageClass 参数之间的关系是否稳定

---

#### 6.2 `NodeUnpublishVolume`

很多人会把这个接口想成“对着 `targetPath` 做一次 `umount`”。

但实际要处理的问题远比这个复杂：

- `targetPath` 不存在怎么办
- 重复调用是否应该返回成功
- 卸载失败能否安全重试
- Pod mount 卸掉后目录是否删除
- shared mount 的引用计数是否减少
- 遇到残留、僵死 mount、transport endpoint 异常时怎么处理

Node 侧真正的质量，很多时候不在“能挂起来”，而在“能不能安全卸掉”。

---

#### 6.3 `NodeStageVolume / NodeUnstageVolume`

这是目录型 CSI 最容易被误用的一组接口。

很多实现会默认把它们做出来，但实际上，`STAGE_UNSTAGE_VOLUME` 是可选能力，而不是默认必需能力。

一旦启用了它，你就要真正面对：

- `globalmount` 生命周期
- kubelet 的 `GetMountRefs`
- bind mount 引用链
- shared mount 是否会被误判为残余引用
- Stage 和 Publish 的边界到底怎么切

对块设备型驱动来说，这套模型通常很自然；但对目录型共享存储来说，这套模型有时不但不帮忙，反而会放大复杂度。

所以在文件存储型 CSI 上，建议在设计阶段认真问一句：

> 我真的需要 `STAGE_UNSTAGE_VOLUME` 吗？

---

#### 6.4 `NodeGetInfo / NodeGetCapabilities`

这两个接口看起来像样板代码，但它们实际上决定了 kubelet 如何理解你的 Node 能力模型。

这里要特别注意：

- Node ID 怎么设计
- topology 是否真的有必要
- `NodeGetCapabilities` 里声明的能力是否和真实实现一致

尤其不要“先把 capability 都宣称支持，后面再补”。因为 kubelet 和 sidecar 会真的按照你宣称的能力来调你。

换句话说：

> **能力声明不是文案，是合约。**

---

### 7 静态卷与动态卷怎么做才不乱

很多 CSI 在静态卷这一块的问题，不是功能没做，而是语义定义从一开始就不清楚。

---

#### 7.1 动态卷

动态卷关注的核心是：

- `VolumeId` 怎么生成
- `StorageClass` 参数怎么设计
- 后端卷对象如何创建
- 卷的元数据和容量如何记录

动态卷通常更容易做成闭环，因为生命周期完全掌握在 CSI 手里。

---

#### 7.2 静态卷

静态卷最需要写清楚的是：

- `volumeHandle` 代表什么？
  - 目录名
  - 完整路径
  - 后端对象 ID
- 哪些目录由 CSI 自动创建？
- 哪些目录必须预先存在？
- 静态卷是否允许挂任意后端路径？

如果语义没写清楚，最后用户会把“实现限制”误解成“驱动 bug”。

所以静态卷真正难的，不是写个 if 分支，而是把**约束清晰地产品化**。

---

### 8 测试与回归流程

做 CSI，测试绝不能只是“随手起个 Pod 看看能不能挂上”。

CSI 的很多问题，只有在：

- 删除
- 重试
- 多引用
- 节点重启
- 插件重启
- 故障恢复

这些场景里才会暴露出来。

---

#### 8.1 功能测试

先做基础功能验证：

- PVC 创建绑定
- Pod 挂载
- Pod 内读写
- 删除 Pod 后重建验证数据仍在

这一步用来确认主链路真正通了。

---

#### 8.2 共享测试

如果你的驱动支持共享目录 / RWX，那这一块必须单独测：

- 同 PVC 多 Pod
- 同节点 / 多节点
- 同时读写
- 删除部分 Pod 后是否影响其他 Pod

共享场景往往比单 Pod 场景更容易暴露挂载和引用问题。

---

#### 8.3 静态卷测试

静态卷至少要覆盖：

- 已存在目录成功
- 不存在目录失败
- `volumeHandle` 语义校验
- 权限和路径约束是否符合文档

---

#### 8.4 生命周期测试

一定要把这些动作单独拉出来测：

- Create / Delete
- Publish / Unpublish
- Stage / Unstage（如果支持）
- 重复调用幂等性

因为线上最常见的问题不是“第一次调用失败”，而是“重试后的状态机错乱”。

---

#### 8.5 故障场景测试

真正像样的 CSI，一定要测一些“不正常”的情况：

- 节点重启
- 插件重启
- kubelet 重启
- 挂载残留
- 网络闪断
- runtime 异常
- 卷已不存在但 Kubelet 还在重试

这类测试通常最痛苦，但也是决定你驱动能不能真正上线的关键。

---

#### 8.6 标准兼容测试

建议至少把 `csi-sanity` 跑起来。

它并不能保证你的驱动完全适合业务场景，但它非常适合作为：

> 协议层回归的最小基线

很多“看起来业务能跑”的实现，在 `csi-sanity` 下会把幂等性、返回码、边界语义问题都暴露出来。

---

### 9 做 CSI 最容易踩的坑

这部分其实最有阅读价值。很多时候一篇真正有用的工程文章，不在于它讲了多少接口，而在于它提前告诉读者哪里最容易翻车。

常见坑包括：

- 把 CSI 当 RPC 开发，而不是生命周期设计
- `VolumeId` 设计太随意
- 幂等性没做好
- 误用 `STAGE_UNSTAGE_VOLUME`
- 静态卷语义不清
- 权限写死
- 卸载顺序设计错误
- shared mount 引用无法收敛
- 以为挂上了就等于做完了
- 不做回归测试，后面一改就炸

如果一定要只说一个坑，那我会说：

> **CSI 最容易错的地方，不在“挂载”，而在“卸载”和“重试”。**

---

### 10 从 MVP 到产品化还要补什么

一个能跑的 CSI，不等于一个成熟的 CSI。

从 MVP 走向产品化，通常还要继续补：

- `ListVolumes`
- `GetCapacity`
- `NodeExpandVolume`
- `Snapshot`
- 日志与指标
- 健康检查
- 回归测试体系
- 更清晰的错误码
- 配置和权限参数化

如果是目录型 CSI，还要额外关注：

- shared mount 生命周期
- refCounter 恢复
- 挂载残留自动清理
- kubelet 与内核文件系统之间的兼容性问题

很多团队做到 MVP 后就感觉“CSI 已经完成了”，但实际上，真正的产品化工作往往从这一步才开始。

---

### 11 我的建议：开发 CSI 的正确节奏

如果让我总结一个比较稳的开发节奏，我会建议这样走：

#### 第一步：先打通主链路

- `CreateVolume`
- `NodePublishVolume`
- `NodeUnpublishVolume`
- `PVC -> PV -> Pod`

先让卷从 Kubernetes 到 Pod 真正流转起来。

#### 第二步：补静态卷、删除、幂等和测试

- 静态卷
- 动态卷
- 删除语义
- 基础回归

这一步的目标是从“能跑”变成“能用”。

#### 第三步：再决定是否支持 `Stage/Unstage`、`Expand`、`Snapshot`

这些能力不要“先上了再说”，而应该根据底层存储模型判断是否真的适合。

#### 第四步：最后做产品化

- 观测
- 兼容性
- 健康检查
- 自动化测试
- 排障能力

这样做的好处是：你始终知道自己现在在解决哪个层面的问题，而不是一开始就被复杂性拖进泥潭。

---

### 12 总结

开发一个 CSI Driver，从来不只是“把几个接口补齐”。

真正难的是：

- 先选对存储模型
- 再选对挂载模型
- 再让 Kubernetes、kubelet、Linux 挂载语义和你的驱动实现真正对齐

尤其在目录型网络文件系统里，很多问题不是 CSI 接口本身的问题，而是：

> **你的架构设计，是否和 kubelet 的卸载模型、Linux 的挂载语义真正兼容。**

如果说“从 0 到 1 做一个 CSI”最重要的一条经验是什么，我会把它总结成一句话：

> **先把生命周期设计明白，再写代码。**
