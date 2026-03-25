---
layout: post
title: "根因分析(RCA):异常掉电后,XFS启动故障复盘"
subtitle: "`fstab`、dirty log 和 `fsck.xfs` 的段错误"
date: 2026-03-25 10:30:00 +0800
author: "Poke"
header-img: "img/post-bg-debug.png"
catalog: true
tags:
  - Debug趣事
  - RCA
  - XFS
categories:
  - Debug趣事
---

### 摘要

在一次 MTBF 异常掉电测试中，Kylin V2 国防版出现了无法正常启动的问题。最开始暴露出来的是磁盘异常：加盘加入集群时报错，后续磁盘无法识别，多次插拔仍无效，最终只能更换磁盘。更换后，新盘可以正常识别并加入集群，看起来硬件层面已经恢复；但系统重启时依然卡在 `A start job is running for /dev/disk/by-partuuid/...`，表现为长时间无法进入正常启动流程。

继续使用 U 盘进入系统排查后，首先发现 `/etc/fstab` 中残留了旧磁盘的挂载配置。删除残留项后，系统启动问题部分缓解。然而进一步分析 journal 后又发现，异常掉电同时导致 XFS 文件系统存在 dirty log，启动阶段的 `systemd-fsck` 进入了 `xfs_repair -e` / `xfs_repair -L` 的恢复路径。在部分节点上，`xfs_repair -L` 最终可以修复成功；但在另一些节点上，修复过程中出现 metadata CRC error，`xfs_repair` 甚至直接发生段错误，最终导致后续挂载失败并进入 emergency mode。

这次故障表面上像是“换盘后机器起不来”，实际上是 **失效挂载残留、XFS dirty log、元数据损坏以及 fsck 启动链路处理不一致** 叠加后的结果。

---

### 1 背景

这次问题出现在一轮 MTBF 异常掉电测试过程中，测试环境为 **Kylin V2 国防版**。

在这类测试里，大家最自然的关注点通常是：

- 异常掉电后，磁盘是否还能正常识别
- 文件系统是否会损坏
- 系统能否自行恢复启动
- 存储挂载和业务数据是否还能恢复

也正因为场景本身带有“硬件故障 / 文件系统损坏”的天然预期，排查一开始非常容易被带偏。

---

### 2 第一现场：加盘失败与磁盘无法识别

故障最初并不是直接表现为“系统开不了机”，而是从磁盘层面开始暴露异常。

#### 2.1 加盘时报错

在将新磁盘加入集群时，系统报错。报错后继续观察发现：

- 磁盘无法被正常识别
- 多次插拔之后仍然无效
- 现场看起来更像是设备本身异常，或者异常掉电影响了磁盘/控制器状态

#### 2.2 更换磁盘

之后更换磁盘，再次测试时：

- 新磁盘能够正常识别
- 也能够正常加入集群

到这一步，很容易产生一个判断：

> 前面的问题只是磁盘本身有问题，现在换盘后应该就恢复了。

但真正麻烦的地方也正是在这里——**硬件层面恢复了，不代表启动链路已经恢复。**

---

### 3 第二现场：系统重启后仍然卡住

更换磁盘后，机器重启时并没有顺利进入系统，而是在启动阶段出现了非常典型的 systemd 等待现象：

```text
A start job is running for /dev/disk/by-partuuid-...
```

#### 3.1 现场现象

- systemd 持续等待某个 `PARTUUID`
- 大约等待 1 分 30 秒
- 超时后继续进入后续异常界面或启动失败路径

这种现象的迷惑性很强，因为它表面上看像：

- 设备没起来
- 文件系统没准备好
- 磁盘还在异常
- 或者系统整体已经坏掉

但实际上，它首先说明的是一件事：

> **启动流程并不是完全卡死，而是在等一个它认为“应该出现”的块设备。**

这意味着，排查方向应该优先落到：

- `/etc/fstab`
- `UUID/PARTUUID`
- systemd mount unit
- 设备引用残留

---

### 4 用 U 盘进入系统后的第一结论：`fstab` 残留

为了避免继续被系统启动流程卡住，后续通过 U 盘进入系统，对原系统进行离线检查。

#### 4.1 核查 `/etc/fstab`

排查发现：

- `/etc/fstab` 中仍然存在旧 disk 的挂载项残留
- 该挂载项引用的是已经不存在的设备 UUID / PARTUUID

这就解释了为什么 systemd 会在启动阶段持续等待对应设备：

> **它不是“盲等”，而是在严格执行 `fstab` 中定义的挂载依赖。**

#### 4.2 删除残留项后的结果

删除无效的磁盘挂载项后：

- 系统可以继续启动
- 原先卡在 `A start job is running for /dev/disk/by-partuuid/...` 的问题得到解除

到这里，第一个根因已经比较明确：

> **系统启动异常的第一层原因，是 `/etc/fstab` 中残留了已经失效的磁盘挂载项。**

但这并不是全部。

---

### 5 继续往下看：问题不只是 `fstab`

如果把问题停留在“删除 `fstab` 残留后系统恢复”，这篇复盘就会显得有点普通。但从后续 journal 来看，事情远比这复杂。

异常掉电除了留下失效挂载残留，还触发了 **XFS 文件系统恢复链路**。

---

### 6 XFS 修复链路：从 dirty log 到 `xfs_repair -L`

日志显示，systemd 在启动阶段执行了如下流程：

```text
/usr/lib/systemd/systemd-fsck /dev/disk/by-uuid/294fde0b-b957-40fa-a27d-b480a0149bbe
```

这意味着启动时，systemd 不仅在处理挂载依赖，同时也在做文件系统检查。

#### 6.1 第一轮：`xfs_repair -e`

从日志中可以看到，`systemd-fsck` 先尝试了类似于：

```text
xfs_repair -e
```

但很快就碰到典型的 XFS dirty log 提示：

```text
ERROR: The filesystem has valuable metadata changes in a log which needs to
be replayed. Mount the filesystem to replay the log, and unmount it before
re-running xfs_repair. If you are unable to mount the filesystem, then use
the -L option to destroy the log and attempt a repair.
```

这里说明的不是“文件系统完全损坏”，而是：

> **文件系统日志还没有完成 replay。**

正常情况下，优先路径应该是：

1. 挂载文件系统
2. 回放日志（replay log）
3. 卸载
4. 再进行修复

但如果系统已经没法正常挂载，那就只能进入风险更高的路径：

```text
xfs_repair -L
```

也就是：**直接丢弃日志并尝试修复。**

---

#### 6.2 第二轮：`xfs_repair -L`

在日志中可以看到系统确实进入了 `-L` 路径：

```text
yyyyyyyyy repair -L
```

后续流程继续推进：

- Phase 1：查找并验证 superblock
- Phase 2：使用内部日志
- zero log
- 扫描 inode / freespace
- 遍历 AG
- 检查重复块
- 重建 AG headers
- 检查 inode 连通性

这说明当时并不是“刚开始修就失败”，而是已经进入了较深的修复阶段。

#### 6.3 一个容易被忽略的关键点：我们实际修改过 `fsck.xfs`

这次问题里还有一条非常关键、但如果不单独写出来就很容易被忽略的背景：**现场并不是完全使用系统原生的 `fsck.xfs` 行为，而是对 `initramfs` 中的 `/sbin/fsck.xfs` 做过修改。**

大致处理过程是：

```text
a. cp /boot/initramfs-3.10.0-514.26.2.el7.x86_64.img /var/ws/
b. mv initramfs-3.10.0-514.26.2.el7.x86_64.img initramfs-3.10.0-514.26.2.el7.x86_64.img.gz
c. gzip -d initramfs-3.10.0-514.26.2.el7.x86_64.img.gz
d. cpio -i < initramfs-3.10.0-514.26.2.el7.x86_64.img
e. 进入 initramfs 目录，修改 /sbin/fsck.xfs
f. 重新打包：find . | cpio -ov -H newc | gzip > ../initramfs-3.10.0-514.26.2.el7.x86_64.img
```

修改后的 `fsck.xfs` 逻辑大致如下：

```bash
echo "11111111 $FORCE"
case $- in
 *i*) FORCE=false ;;
esac
echo "22222222 $FORCE"
if [ -n "$PS1" -o -t 0 ]; then
 FORCE=false
fi
echo "33333333 $FORCE"
 
if $FORCE; then
 echo "xxxxxxxxxxx repair -e"
 xfs_repair -e $DEV
 repair_result=$?
 echo "xfs_repair -e result: $repair_result"
 repair2fsck_code $repair_result
 if [ $repair_result -eq 0 ]; then
  repair2fsck_code 0
  exit 0
 elif [ $repair_result -eq 4 ]; then
  repair2fsck_code 4
  exit 0
 else
  echo "yyyyyyyyy repair -L"
  xfs_repair -L $DEV
  result=$?
  echo "xfs_repair -L result: $result"
  if [ $result -eq 139 ]; then
   echo "zzzzzzzz repair -L"
   xfs_repair -L $DEV
   echo "second xfs_repair -L result: $?"
   chmod -x /usr/sbin/boot_test
  fi
  #repair2fsck_code $result
  exit 0
 fi
fi
```

这段脚本至少说明了三件事：

1. 现场确实人为修改了 `fsck.xfs` 的恢复逻辑，而不是完全依赖系统默认实现；
2. 一旦 `xfs_repair -e` 失败，就会自动进入 `xfs_repair -L`；
3. 更关键的是，在 `xfs_repair -L` 执行后，无论结果如何，脚本最后都直接 `exit 0`，而没有把真实结果继续映射给 systemd。

这也解释了前面日志中的那个反常现象：

```text
xfs_repair -L result: 139
fsck succeeded.
systemd-fsck@...service: Main process exited, code=exited, status=0/SUCCESS
```

换句话说，这并不完全是 systemd 自己“误判成功”，而是：

> **修改后的 `fsck.xfs` 脚本主动屏蔽了底层修复失败的真实返回码，导致 `systemd-fsck` 只能看到一个人为构造出来的成功退出。**

这会直接带来一个后果：

- 文件系统修复过程即使已经失败，甚至已经 segfault；
- 启动链路仍然会被继续往后推进；
- 最终在真正执行后续挂载（例如 `/var/lib/docker`）时，问题才再次暴露出来。

---

### 7 分叉点：相同恢复路径，在不同机器上走出了不同结果

这也是这次故障最有意思的地方。你给出的日志，其实对应了两种不同结局。

#### 7.1 test03：修复过程中出现元数据损坏，`xfs_repair` 崩溃

在 `test03` 上，日志里出现了非常关键的错误：

```text
Metadata CRC error detected at 0xaaac385cdc28, xfs_dir3_block block 0x7b0f640/0x1000
corrupt block 0 in directory inode 137426478: junking block
```

这说明：

- 已经不是单纯的 dirty log 未 replay
- 文件系统元数据本身出现了损坏
- 出错对象是目录块（`xfs_dir3_block`）
- 某个目录 inode 已经存在坏块

接着又出现了更危险的现象：

```text
audit[3191]: ... comm="xfs_repair" ... sig=11
/usr/sbin/fsck.xfs：行 89: 3191 段错误 xfs_repair -L $DEV
xfs_repair -L result: 139
```

也就是说：

- `xfs_repair -L` 在修复过程中直接触发了 **SIGSEGV**
- 退出码 `139` 本质上就是程序因为段错误崩溃

但更诡异的是，后面日志却显示：

```text
fsck succeeded.
systemd-fsck@...service: Main process exited, code=exited, status=0/SUCCESS
```

这意味着：

> **虽然底层 `xfs_repair -L` 实际崩溃了，但外层 `fsck.xfs/systemd-fsck` 最终仍把它当成“成功”处理。**

启动流程于是继续往后推进，尝试挂载：

```text
/var/lib/docker
```

但内核紧接着又报出：

```text
XFS (sda7): Corruption warning: Metadata has LSN (146:3535) ahead of current LSN (1:2)
XFS (sda7): log mount/recovery failed: error -22
XFS (sda7): log mount failed
```

最终导致：

- `/var/lib/docker` 挂载失败
- `local-fs.target` 依赖失败
- systemd 触发 `emergency.target`

也就是说，`test03` 的问题最后升级成了：

> **修复工具已经崩了，但系统还在继续启动，结果在真正挂载时再次失败。**

---

#### 7.2 test02：同样走 `xfs_repair -L`，但最终恢复成功

而在 `test02` 上，同样的恢复路径却给出了不同结果：

```text
yyyyyyyyy repair -L
...
Maximum metadata LSN (1383:1815) is ahead of log (1:2).
Format log to cycle 1386.
done
xfs_repair -L result: 0
fsck succeeded.
```

这里说明：

- 同样遇到了 LSN 不一致
- 同样走了 `-L` 路径
- 但修复最终成功完成
- `xfs_repair -L` 返回 `0`
- `systemd-fsck-root.service` 正常退出

这也说明一个重要事实：

> **同样是异常掉电 + XFS dirty log，恢复结果并不是完全一致的。**

有的现场只是日志未回放，`-L` 后还能恢复；有的现场则已经叠加了更深层的 metadata corruption，甚至让 `xfs_repair` 自己崩溃。

---

### 8 根因归纳：这不是单一故障，而是三层问题叠加

综合整个排查链路，这次问题不能简单定义成“盘坏了”或者“`fstab` 写错了”。

更准确的定性应该是：

#### 第一层：磁盘更换后的挂载残留
- `/etc/fstab` 中保留了旧磁盘挂载项
- systemd 启动时持续等待不存在的设备
- 直接表现为启动阶段长时间阻塞

#### 第二层：异常掉电留下 XFS dirty log
- 启动阶段进入 `systemd-fsck`
- `xfs_repair -e` 无法直接处理
- 被迫走 `xfs_repair -L`

#### 第三层：部分节点存在更深层的元数据损坏
- 出现 metadata CRC error
- `xfs_repair -L` 本身 segfault
- 但上层启动链路没有准确把修复失败传播出去
- 最终在后续真正 mount 时再次失败，触发 emergency mode

因此，这次故障本质上是：

> **失效挂载残留、XFS dirty log、元数据损坏以及 fsck 启动链路处理不一致** 共同叠加后的结果。

---

### 9 这次排查里最值得记住的几个点

#### 9.1 “无法启动”不等于“系统彻底坏了”

如果启动界面出现：

```text
A start job is running for /dev/disk/by-uuid/...
```

第一反应不应该是“系统盘坏了”，而应该先检查：

- `/etc/fstab`
- UUID / PARTUUID 是否还存在
- 是否有历史设备引用残留

很多“像死机”的场景，本质只是 systemd 在等一个永远不会出现的设备。

---

#### 9.2 异常掉电后的 XFS 问题，不一定只是 dirty log

`dirty log` 只是第一层表象。继续往下看 journal 时，必须区分：

- 只是日志需要 replay
- 还是已经存在 metadata corruption
- `xfs_repair -L` 是成功修复，还是只是在修复途中崩溃

这一点会直接决定后续风险。

---

#### 9.3 `xfs_repair -L` 不是“万能修复开关”

`-L` 的本质是：

> **丢弃日志，强制尝试恢复。**

这在部分场景下能把系统拉回来，但它本身就是有风险的。而这次更进一步说明：

- 不是每次 `-L` 都能安全完成
- 在更严重的元数据损坏下，连 `xfs_repair` 本身都可能失败甚至崩溃

---

#### 9.4 启动链路的“成功”未必真的代表恢复成功

这次最值得警惕的一点是：

- `xfs_repair -L` 返回了 `139`
- 明明是 segfault
- 但 `systemd-fsck` 最终仍然给出了 `SUCCESS`

这说明在某些恢复链路里：

> **“启动脚本认为成功” 并不等于 “文件系统真的已经恢复到可挂载状态”。**

---

### 10 最终结论

这次看起来像“换盘后系统起不来”的问题，实际上经历了三次认知反转：

1. **先像硬件问题**  
   因为磁盘无法识别、需要更换，容易先怀疑盘或控制器。

2. **再像启动配置问题**  
   因为系统卡在 `A start job is running for ...`，最终在 `/etc/fstab` 里发现了失效设备残留。

3. **最后升级成文件系统恢复问题**  
   因为异常掉电同时带来了 XFS dirty log 和更深层 metadata corruption，甚至让 `xfs_repair -L` 在部分节点上直接崩溃。

所以，对这次问题最准确的一句话总结应该是：

> **不是硬盘坏了，也不是单纯 `fstab` 配错了，而是旧盘残留、XFS 恢复链路和启动流程处理不一致一起把系统拖进了启动失败。**

---

### 11 给类似场景的建议

如果以后再碰到类似问题，我会建议优先按下面顺序排查：

#### 启动卡住先看
- `A start job is running for ...`
- `/etc/fstab`
- UUID / PARTUUID 是否仍然存在

#### 异常掉电后再看
- `journalctl -xb`
- `systemd-fsck`
- `xfs_repair -e / -L`
- 是否有 dirty log / LSN mismatch / metadata CRC error

#### 真正决定是否能恢复的点
- 挂载残留清没清干净
- fsck 是真的成功，还是脚本误判成功
- 后续目标挂载（例如 `/var/lib/docker`）是否真正成功
