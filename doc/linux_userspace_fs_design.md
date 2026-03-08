# 面向 LevelDB 的 Linux 用户态文件系统设计（Direct I/O）

> 目标：为 LevelDB 提供一个“够用且高效”的专用文件系统，不追求完整 POSIX 语义，不使用 FUSE，直接在用户态通过 `O_DIRECT` 访问块设备。

## 1. 设计边界与假设

基于题设，约束如下：

1. 仅实现 LevelDB 需要的文件与目录语义（写入语义仅 append-only）；
2. 单实例文件数不超过 100k，绝大多数文件大小在 2MB~3MB；
3. 可将全部元数据常驻内存（启动时加载，运行时维护）；
4. 用户态直接 I/O（`O_DIRECT`），减少页缓存干扰和双缓存；
5. 单机单进程挂载（不考虑多主并发挂载）。

这使我们可以将系统设计成：**日志 + extent 分配 + 内存 inode 表 + 简化目录模型**。

---

## 2. LevelDB 需要支持的最小 I/O 语义

参考 `include/leveldb/env.h`，核心接口可映射为：

- `NewSequentialFile`：顺序读；
- `NewRandomAccessFile`：随机读（主要 SST）；
- `NewWritableFile`：创建并顺序追加写（WAL/SST 输出）；
- `NewAppendableFile`：可选（LevelDB 某些实现中可能不强依赖）；
- `FileExists / GetChildren / DeleteFile / CreateDir / DeleteDir / GetFileSize / RenameFile`；
- `LockFile / UnlockFile`：可简化为进程内锁文件语义。

### 可简化的 POSIX 语义

- 不支持硬链接、软链接、权限位、owner/group；
- 不支持稀疏文件；
- 明确不支持任意 offset 覆盖写；系统层只实现 append-only（append + sync + close）；
- `rename` 仅保证同目录原子替换（已足够覆盖 MANIFEST/CURRENT 更新）。

---

## 3. 总体架构


```text
+-------------------------------+
| LevelDB Env 实现（LdbFsEnv） |
+---------------+---------------+
                |
                v
+-------------------------------+
|   用户态 FS Core              |
|  - 元数据管理（内存）          |
|  - extent 分配器              |
|  - 日志与 checkpoint          |
|  - 文件句柄与 I/O 调度        |
+---------------+---------------+
                |
                v
+-------------------------------+
| Block Device + O_DIRECT       |
+-------------------------------+
```

核心思路：

- **数据面**：文件内容写入 data area，以 extent 形式分配；
- **控制面**：所有元数据更新先写元数据日志（WAL），再更新内存结构；
- **恢复**：启动时先加载最近 checkpoint，再重放元数据日志。

---

## 4. 磁盘布局

建议固定分区布局（单设备文件）：

1. `SuperBlock`（双副本，含版本/UUID/布局偏移）；
2. `MetaLog` 区（循环日志，记录元数据事务）；
3. `Checkpoint` 区（周期性快照：inode 表 + 目录项 + 空闲 extent 树）；
4. `Data` 区（文件数据）；
5. 可选 `DiscardMap`（TRIM/回收提示）。

块大小建议 4KiB，对齐到 `O_DIRECT` 要求（读写 buffer、offset、length 均按设备逻辑块对齐）。

---

## 5. 内存元数据模型

由于上限 100k 文件，内存常驻可行。

## 5.1 inode（简化）

- `inode_id`（64-bit）
- `type`（file/dir）
- `size`
- `ctime/mtime`
- `state`（open_writable, sealed, deleting）
- `vector<Extent>`（按逻辑 offset 升序）
- `refcnt`（句柄引用计数）

## 5.2 目录结构

- 采用 hash map：`dir_inode -> {name -> inode_id}`
- LevelDB 目录层级浅，`GetChildren` 开销可接受。

## 5.3 空闲空间管理

- 按 extent 维护 `free tree`（例如红黑树/分桶）
- 优先分配 2MB~4MB 大 extent（贴合文件分布，降低碎片）
- 后台合并相邻空闲 extent

---

## 6. I/O 路径设计

## 6.1 写路径（WAL/SST 生成）

1. `Create`：分配 inode，写入元数据日志事务（inode create）；
2. `Append`：
   - 维护用户态对齐缓冲（如 128KiB）；
   - 满块后 `pwrite` 到 data area；
   - 为新分配 extent 写元数据日志（extent add）；
3. `Close/Sync`：
   - flush 未满对齐块（尾块可采用“逻辑长度 + 物理补齐”）；
   - 提交 `size update + seal` 元数据事务并 `fdatasync` 元数据日志。

> 约束强化：数据文件只允许顺序追加，不提供覆盖写接口，也不做 read-modify-write 的随机更新路径。
>
> 注意：`O_DIRECT` 下尾写常不是块对齐，需要内部 bounce buffer 做对齐写。

## 6.2 读路径（SST 随机读）

- 通过内存 inode 的 extent 索引将逻辑 offset 映射到物理 offset；
- 对齐读取到 bounce buffer，再拷贝请求区间给上层；
- 可做小型读缓存（如 8MB~64MB）专门覆盖 index/filter block 热点。

## 6.3 删除与回收

- `DeleteFile`：先在目录摘链，再标记 inode deleting；
- 若 `refcnt==0` 立即回收 extent，否则延迟回收；
- 回收记录写入元数据日志，确保崩溃后不会泄漏空间。

---

## 7. 一致性与崩溃恢复

## 7.1 元数据事务模型

每条事务包含：

- `txid`
- 操作类型（create/delete/rename/extent_add/size_update/...）
- payload
- crc
- commit 标志

恢复规则：仅回放“完整且 commit 的事务”。

## 7.2 checkpoint 策略

- 每 N 秒或元数据日志达到阈值（如 64MB）触发一次；
- 写新 checkpoint 后原子更新 superblock epoch；
- 启动恢复：加载最新 epoch checkpoint + 回放后续日志。

## 7.3 LevelDB 关键原子性

- `CURRENT` 文件更新采用 `write temp -> fsync -> rename`；
- 在本 FS 中需保证同目录 `rename` 原子，并记录到单条元数据事务。

---

## 8. 并发模型

建议分层锁：

- 全局 `meta_mutex`（保护 inode 表/目录树/空闲树）；
- 每 inode `rwlock`（读写数据 extent 访问）；
- I/O 提交线程池（异步刷盘，调用线程等待完成或 future）。

常见优化：

- 将元数据日志串行化（单写线程）以简化顺序一致性；
- 数据 I/O 可并行，减少 WAL 与 SST 写互相阻塞。

---

## 9. 与 LevelDB 文件类型的针对性优化

1. **`.log`（WAL）**
   - 强顺序 append；
   - 可使用较小 extent（如 1MB）避免尾部浪费；
2. **`.sst`**
   - close 后 immutable；
   - 优先连续分配 2~4MB extent，提升顺序读与 compaction 读；
3. **`MANIFEST` / `CURRENT`**
   - 元数据强一致优先，`Sync` 策略保守；
4. **临时文件**
   - 支持快速 rename/replace，删除走延迟回收。

---

## 10. API 草案（用户态 FS Core）

可向 `Env` 适配层暴露如下接口：

- `fs_open(path, flags)`
- `fs_read(fd, off, buf, len)`
- `fs_append(fd, buf, len)`
- `fs_fsync(fd)`
- `fs_close(fd)`
- `fs_stat(path)`
- `fs_readdir(path)`
- `fs_rename(old, new)`
- `fs_unlink(path)`
- `fs_mkdir(path)`

其中 `fs_append` 明确只支持顺序追加，偏离则返回 `NotSupported`。

---

## 11. 容量估算（验证元数据内存化可行）

按 100k 文件估算：

- inode 基本信息 ~ 128B：约 12.8MB；
- 每文件平均 1~2 个 extent，按 32B 计：约 3.2MB~6.4MB；
- 目录项（文件名 + 哈希节点）假设平均 80B：约 8MB；
- 其他索引与碎片开销预留 2~3 倍：总计约 50MB~100MB。

结论：在现代服务器上内存常驻完全可行。

---

## 12. 实现里程碑建议

1. **M1：最小可用**
   - create/append/read/close/delete/rename + 恢复；
   - 单线程元数据日志；
2. **M2：稳定性**
   - checkpoint、崩溃注入测试、空间回收；
3. **M3：性能优化**
   - extent 分配策略调优、读缓存、异步 I/O；
4. **M4：与 LevelDB 深度联调**
   - db_bench 场景下验证写放大/尾延迟/恢复时间。

---

## 13. 风险与对策

- **Direct I/O 对齐复杂**：统一封装 aligned allocator + bounce buffer；
- **元数据日志膨胀**：checkpoint + 日志截断；
- **碎片问题**：按文件类型分配策略 + 后台 merge；
- **崩溃窗口**：严格“先日志后内存”与事务 commit 标记。

---

## 14. 小结

这个方案利用了题设中“小规模、文件大小集中、语义可裁剪”的特点，避免通用文件系统复杂度，重点保障 LevelDB 真正需要的三件事：

1. 顺序 append + 随机读性能；
2. `rename`/`sync` 驱动的元数据一致性；
3. 快速恢复与可控实现复杂度。

如果需要进一步落地，可以下一步给出：

- `Env` 到 `LdbFsEnv` 的具体类图；
- 元数据日志二进制格式（字段级）；
- 崩溃测试矩阵（kill -9 注入点）。

---

## 15. 是否可以省略 Journal？

可以，但要先明确：**省略 Journal 不是“零成本”**，而是把一致性复杂度转移到别的机制上。对于本设计约束（文件数 <= 100k、元数据可常驻内存、单机单进程），有两类可行路径：

### 15.1 方案 A：保留 Journal（基线方案）

- 优点：实现与恢复路径直观；
- 风险低：通过“先写日志再改内存”保证元数据原子性；
- 代价：多一次元数据写放大。

### 15.2 方案 B：去 Journal，利用 sector 原子写组织关键元数据

在“只追加写、元数据规模小”的前提下，可以走更轻量的无 Journal 方案：

1. 设备假设：4KiB 物理 sector（或设备声明的原子写单元）内写入原子；
2. 为每个文件维护一个 `TailMeta` 槽位，并保证其完整落在**单个 sector**：
   - `file_id` / `version`
   - `logical_size`
   - `tail_extent`（或最后一个 block 映射）
   - `crc`
3. 每次 append 持久化时，先落数据，再原子覆盖该文件 `TailMeta` 所在 sector；
4. 启动恢复时只接受 crc 正确且 version 最大的 `TailMeta`。

这样崩溃后看到的状态要么是旧 `TailMeta`，要么是新 `TailMeta`，不会出现“size 与 block 映射撕裂”。

> 关键点：你提到的“把 size 和 block 信息放在同一个 sector”是可行且有效的，正是该方案成立的核心。

### 15.3 无 Journal 时如何保证数据持久性（append-only）

无 Journal 场景下仍必须遵守写序：

1. 先把新追加数据写入 data area；
2. `fdatasync(data_fd)`；
3. 再写入包含 `logical_size + tail block/extent` 的 `TailMeta` 原子 sector；
4. `fdatasync(meta_fd)` 后才向上层返回 append 成功。

若顺序反过来，会出现“size 已前移但数据未落盘”的可见性错误。

### 15.4 适用边界与限制

该无 Journal 方案可行，但有前提：

- 必须是 append-only；若需要覆盖写/随机改写，单 sector `TailMeta` 不足以描述复杂原子更新；
- 依赖设备真实提供的 sector 原子写语义（需启动时探测并校验）；
- `TailMeta` 仅覆盖“文件尾增量提交”，目录 rename、文件创建/删除等仍建议用小型事务区或双副本元数据页处理。

结论：

- **在本题约束下（只追加写），“无 Journal + sector 原子 TailMeta”可以保证崩溃一致性与持久性。**
- **若未来扩展到覆盖写或更复杂元数据操作，应回到 Journal 或 COW 元数据树。**

---

## 16. 不用 FUSE 时，如何支持多进程可见性

前提是不使用 FUSE，因此不能依赖内核 VFS 帮你做“跨进程命名空间 + 缓存一致性”。
可行做法是：**一个用户态 FS 守护进程（唯一 owner）+ 多客户端进程通过 IPC 访问**。

### 16.1 推荐架构：FS Daemon + LibEnv Client

```text
+-----------------+        Unix Domain Socket / shm / eventfd
| process A       |  <----------------------------------------+
| LevelDB + Env   |                                           |
+-----------------+                                           |
                                                              v
+-----------------+                                 +---------------------+
| process B       |                                 |   ldbfsd daemon     |
| LevelDB + Env   |  <----------------------------> | - 唯一元数据主副本   |
+-----------------+                                 | - 唯一块设备写入口   |
                                                    | - 提交序列号(LSN)    |
+-----------------+                                 +----------+----------+
| process C       |                                            |
| LevelDB + Env   |  <-----------------------------------------+
+-----------------+                                         O_DIRECT block dev
```

关键原则：

1. **单写入口**：只有 `ldbfsd` 打开块设备并执行元数据提交；
2. **客户端无本地真相**：客户端可缓存读结果，但元数据真相只在 daemon；
3. **所有修改操作线性化**：create/append/rename/unlink 在 daemon 内串行提交并分配全局 `LSN`。

这样无需 FUSE，也能在用户态实现跨进程一致视图。

### 16.2 可见性保证（Read-Your-Writes + 跨进程有序可见）

每个变更事务在 daemon 中经历：

1. 执行数据落盘与元数据提交；
2. 生成递增 `commit_lsn`；
3. 返回给发起方；
4. 通过通知通道（eventfd/UDS 消息）广播“目录/文件版本已更新”。

客户端维护 `seen_lsn`：

- 对发起方：收到 ACK（含 `commit_lsn`）后，后续读至少在该 LSN 上执行（read-your-writes）；
- 对其他进程：在收到通知并将本地缓存推进到该 LSN 后可见；
- 若客户端要“强一致读”，可在请求里携带 `min_lsn`，daemon 等待直到 `current_lsn >= min_lsn` 再返回。

### 16.2.1 daemon 跨进程通信方式推荐

在 Linux 上，针对本场景（同机、多进程、低延迟、需要事件通知与可选零拷贝），推荐优先级如下：

1. **Unix Domain Socket（UDS）作为控制面主通道（首选）**
   - 优点：
     - API 简单（`sendmsg/recvmsg`），可靠字节流或报文语义；
     - 无需网络栈路由，延迟低；
     - 支持 `SCM_RIGHTS` 传 fd，便于共享 memfd/eventfd；
     - 权限与身份校验容易（文件路径权限、`SO_PEERCRED`）。
   - 适合：元数据请求、锁/租约、`WAIT_LSN`、失效通知控制消息。

2. **共享内存（`memfd`/`shm_open`）作为数据面（可选增强）**
   - 优点：减少大块 `READ`/`APPEND` 数据在 socket 上的拷贝；
   - 方式：
     - 通过 UDS 下发共享 ring buffer 的 fd；
     - 控制消息走 UDS，数据 payload 走共享内存槽位；
     - 用 `eventfd` 或 futex 做“槽位可用/已完成”通知。
   - 适合：高吞吐顺序 I/O 或大 value 场景。

3. **eventfd 作为通知面（建议与 UDS 组合）**
   - 优点：语义简单、开销低，适合唤醒等待 `commit_lsn` 推进的客户端；
   - 适合：`invalidate`/`commit_lsn advanced` 的批量唤醒。

不建议作为主方案：

- TCP loopback：可用但多一层网络协议栈复杂度；
- 管道/FIFO：双向与多路复用能力弱；
- 仅信号量/共享内存：工程复杂，调试困难，协议演进成本高。

**落地建议（默认配置）**：

- `UDS(stream or seqpacket) + protobuf/flatbuffer 消息 + eventfd 通知`；
- 当 `READ/APPEND` 平均 payload 较大（如 >64KiB）时，启用 `UDS 控制面 + memfd 共享数据面` 混合模式。

### 16.3 IPC 协议建议（最小集合）

请求字段：`op, path/inode, offset, len, min_lsn, client_id, req_id`
返回字段：`status, data/bytes, file_version, commit_lsn`

最小操作：

- 元数据：`OPEN, STAT, READDIR, RENAME, UNLINK, MKDIR`
- 数据：`APPEND, READ, FSYNC, CLOSE`
- 同步：`WAIT_LSN(lsn)`
- 订阅：`SUBSCRIBE(dir|inode)`（接收失效/版本推进通知）

### 16.4 客户端缓存与失效机制

建议缓存分层：

- `name -> inode` 目录项缓存；
- inode 属性缓存（size/version/state）；
- 热数据块缓存（可选）。

失效策略：

1. daemon 对变更对象发送 `invalidate(inode|dir, new_version, lsn)`；
2. 客户端若发现 `cached_version < new_version`，则丢弃缓存并回源；
3. READ 可带 `if_version==x` 做轻量校验，失败则重取元数据。

### 16.5 锁与并发控制（多进程）

- daemon 内：
  - 全局提交锁（保证 commit_lsn 全序）；
  - inode 级 rwlock（并行读、串行同 inode append）；
- 客户端侧：
  - 不做分布式锁真相，仅做请求合并与本地互斥优化；
- `LockFile/UnlockFile`：
  - 由 daemon 维护租约锁（owner=client_id, ttl）；
  - client 异常退出由心跳超时自动回收。

### 16.6 append-only 下的并发规则

由于只支持 append，简化规则如下：

1. 同一文件同一时刻仅一个 append writer（由 daemon 赋予写租约）；
2. 不同文件 append 可并行；
3. 读者总是按 `logical_size@lsn` 读取，永远不会读到“size 前移但数据缺失”的状态；
4. `fsync` 返回的 `commit_lsn` 可作为跨进程可见性的同步点。

### 16.7 进程故障与 daemon 故障

- **客户端崩溃**：
  - daemon 感知连接断开，释放该 client 的锁与未完成请求上下文；
- **daemon 崩溃重启**：
  - 按第 7/15 节恢复元数据；
  - 提升 `epoch`，客户端检测 epoch 变化后强制清空本地缓存并重新 OPEN；
  - 通过 `epoch + lsn` 防止旧响应污染新会话。

### 16.8 为什么这比“多进程直接各自操作块设备”更好

若多个进程都直接写块设备（无中心协调），会遇到：

- 元数据竞争更新（size/extent 覆盖）；
- 缓存不可见（彼此不知道对方何时提交）；
- 难以实现全局线性化与可靠锁回收。

因此，不使用 FUSE 时，**“单 daemon + 多 client IPC”是最实际的多进程可见性方案**。


---

## 17. 不用 daemon，仅靠共享内存能否实现多进程可见性？

可以实现“可见性”，但要区分两个层次：

1. **缓存可见性**（进程 A 修改后，进程 B 能看见）
2. **提交一致性**（崩溃后元数据/数据仍保持一致）

仅共享内存天然只能解决第一层；第二层仍需要一套跨进程提交协议与持久化顺序控制。

### 17.1 可行架构：Shared-Memory Metadata + 多进程协作提交

可将 inode 表、目录项索引、free extent 索引放到 `shm`（`shm_open + mmap(MAP_SHARED)`）中：

- 所有进程映射同一份内存元数据；
- 每个 inode 带 `version` 与 `seq`；
- 全局有 `commit_lsn` 与 `epoch`；
- 进程间通过 `futex/pthread_mutex(PTHREAD_PROCESS_SHARED)` 同步。

这样在运行态可做到“写入后立即对其他进程可见”。

### 17.2 核心难点：没有 daemon 时，谁负责线性化提交？

无 daemon 表示不存在天然“单写入口”，需要在共享内存里显式建立：

- **全局提交锁**：一次只允许一个进程执行“数据落盘 -> 元数据提交”；
- **每 inode append 锁**：同一文件只能单 writer append；
- **提交日志槽（ring）**：记录 `txid/lsn/op/crc`，供其他进程追平视图；
- **租约与心跳**：持锁进程崩溃时由其他进程接管并回收锁。

否则会出现多个进程并发更新 `size/tail_extent` 的覆盖问题。

### 17.3 基于 sector 原子 TailMeta 的无 daemon 提交流程（append-only）

在你前面提出的约束下（append-only + sector 原子），可采用：

1. 获取 inode append 锁 + 全局提交锁；
2. 追加写数据块到 data area；
3. `fdatasync(data_fd)`；
4. 原子写 `TailMeta` sector（同 sector 内含 `logical_size + tail_extent + version + crc`）；
5. `fdatasync(meta_fd)`；
6. 在共享内存将 inode `size/version/lsn` 原子前移；
7. 释放锁并 `futex_wake` 通知等待者。

这样可保证：

- 其他进程看到的新 `size` 一定对应已持久化的数据；
- 崩溃后恢复可通过 `TailMeta` 的 `version+crc` 选取最后有效状态。

### 17.4 可见性如何做得“强”

共享内存下建议定义两种读：

- **普通读**：读取当前本地可见 `inode.version`；
- **屏障读**：携带 `min_lsn`，若 `global_lsn < min_lsn` 则等待条件变量/futex，直到追平。

这与 daemon 模型里的 `WAIT_LSN` 语义等价，可提供跨进程 read-your-writes。

### 17.5 需要额外注意的问题

1. **进程崩溃持锁**：必须用 robust mutex 或租约超时机制；
2. **内存一致性模型**：更新 inode 可见字段前后需发布/获取屏障（C++ `atomic_thread_fence`）；
3. **共享内存损坏**：运行态 shm 不是持久化真相，重启后仍应以磁盘 `TailMeta`/checkpoint 重建；
4. **升级兼容**：共享内存布局需版本化（`shm_layout_version`），否则不同二进制无法互通。

### 17.6 结论

- **可以**：不使用 daemon、仅共享内存，也能实现多进程“操作可见性”。
- **前提**：必须补上跨进程锁、提交线性化、故障接管、LSN 屏障读等机制。
- **工程取舍**：
  - daemon 方案实现更清晰、故障域集中；
  - 纯共享内存方案省掉常驻进程，但并发与故障处理复杂度更高。


### 17.7 共享内存锁在 crash 后如何释放（关键）

这是纯共享内存方案最容易踩坑的点。推荐采用“**robust mutex + 租约 + fencing token**”三层机制，而不是只靠一种手段。

#### A. 第一层：robust 进程共享互斥锁（立即检测 owner 死亡）

对共享内存中的关键锁（全局提交锁、inode append 锁）使用：

- `pthread_mutexattr_setpshared(PTHREAD_PROCESS_SHARED)`
- `pthread_mutexattr_setrobust(PTHREAD_MUTEX_ROBUST)`

行为：

1. 进程 A 持锁时崩溃；
2. 进程 B `pthread_mutex_lock` 返回 `EOWNERDEAD`；
3. B 进入“修复态”，检查并修复该锁保护的数据结构；
4. 修复完成后调用 `pthread_mutex_consistent`，再解锁。

这保证“锁”本身不会永久遗失。

#### B. 第二层：租约（lease）+ 心跳（防止假活锁）

仅 robust mutex 仍不够，因为持锁进程可能没死但长期卡住（如 D-state I/O 挂起）。

建议每把逻辑锁附加：

- `owner_pid`
- `owner_start_time`（防 PID 复用）
- `lease_expire_mono_ns`
- `lock_epoch`

持锁进程需周期性续租；等待者若超时：

1. 先 `kill(pid, 0)` + 启动时间校验 owner 是否同一进程；
2. 若 owner 已死或租约过期，触发“抢占恢复流程”；
3. 进入恢复者角色，执行未完成事务判定（见 C）。

#### C. 第三层：fencing token（防止脑裂写入）

锁回收后必须防止“旧 owner 复活线程”继续写盘。

做法：

- 每次授予写锁时分配单调递增 `fencing_token`；
- 所有真正落盘前都校验“我持有的 token == 当前 token”；
- token 过期者即使仍在运行，也只能失败退出，不得提交。

这样能阻断 stale writer。

#### D. 崩溃恢复时如何处理“半提交”

若持锁进程在以下区间崩溃：

- 数据已 `fdatasync(data_fd)`，`TailMeta` 未更新；
- 或 `TailMeta` 更新了，但共享内存 inode `size/version` 尚未前移。

恢复者规则：

1. 以磁盘 `TailMeta(version, crc)` 为准；
2. 回放/重建共享内存 inode 可见字段；
3. 将未完成事务标记 aborted（或补齐提交，二选一但要固定策略）；
4. 推进 `global_lsn` 并广播一次全局 invalidate。

即：**共享内存不是真相源，磁盘元数据才是。**

#### E. 推荐锁回收状态机

`LOCKED -> OWNER_DEAD_DETECTED -> RECOVERING -> CONSISTENT -> UNLOCKED`

- `OWNER_DEAD_DETECTED`：来自 `EOWNERDEAD` 或租约超时；
- `RECOVERING`：单恢复者持有“恢复权”；
- `CONSISTENT`：已完成元数据修复并 `pthread_mutex_consistent`；
- 其他等待者在 `RECOVERING` 期间只能等待，不可并发修复。

#### F. 工程建议

1. 将“恢复逻辑”做成幂等函数，可重复执行；
2. 为每次锁接管记录审计日志（owner、token、lsn、耗时）；
3. 压测故障注入：
   - 持锁后立刻 `SIGKILL`；
   - `data_fdatasync` 后 `SIGKILL`；
   - `TailMeta` 写后、shm 发布前 `SIGKILL`；
4. 若检测到底层 FS/平台不支持 robust 语义，降级到“单 daemon 模式”更稳妥。

结论：

- 在共享内存方案里，**仅靠互斥锁不足以保证 crash 后正确释放与接管**；
- robust mutex 解决“锁遗失”，lease 解决“卡死持锁”，fencing 解决“过期写者”；
- 三者结合，才能在无 daemon 的多进程场景下实现可恢复的锁语义。
