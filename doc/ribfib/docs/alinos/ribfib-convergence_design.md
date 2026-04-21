# 项目设计

本项目为 SONiC RIB/FIB 路由收敛处理功能。通过在 fpmsyncd 的 FIB 块中引入反向遍历（backwalk）基础设施，在 NHT 事件触发时快速修复受影响的 NHG，实现前缀无关收敛（PIC），将流量中断窗口从 O(前缀数) 降低至 O(NHG 深度)。

## 路由收敛处理

### 核心架构决策

1. **FIB 位于 fpmsyncd 内**：经 Routing Working Group 多轮讨论，FIB 功能放在 APPDB 之前的 fpmsyncd 中，而非 orchagent 之后。优势：减少不必要的硬件编程、避免冗余 APPDB 写入、保持 APPDB 对象一致性。fpmsyncd 复杂度增加是已知代价。

2. **双阶段遍历**：
   - Phase 1（PIC Core）：从 resolved NHG ID 出发，沿 Zebra NHG 链的 "dependents"（resolve through）关系反向遍历，禁用失败路径并更新 APPDB。
   - Phase 2（PIC Edge）：通过 NEXTHOP->SONIC NHG ID 表查找，更新包含失败 nexthop 的 SONiC NHG（gateway NHG）。Phase 2 当前延后实现。
   - **SRv6 VPN PIC Edge 收敛依赖 Phase 2 才能生效。**

3. **ID 查找而非指针**：NHG 间依赖关系通过 ID 查找而非直接指针维护，提高代码健壮性。

4. **可插拔的 walk/prune spec**：backwalk 框架通过回调函数定义遍历行为和剪枝条件，便于未来扩展。

5. **同步执行**：backwalk 在 fpmsyncd 事件循环中同步执行。收敛期间阻塞其他事件处理是可接受的 -- 快速修复本身就是优先任务。并发 NHT 事件被序列化处理，后续 backwalk 可见前次修改的状态。

6. **遍历方向与状态传播方向分离**：backwalk 沿 "dependents" 边向上遍历（决定访问哪些节点），walk spec 内部读取 "depends" 方向的子节点状态（传播禁用状态）。DFS 序保证子节点先于父节点被访问。

### 数据模型

**核心表：**

| 表 | 索引 | 用途 |
|----|------|------|
| SONiC Zebra NHG 表 | Zebra NHG ID | 存储 zebra_dplane_ctx + SONiC 元数据，维护 NHG 依赖图 |
| NEXTHOP KEY -> Zebra NHG ID 映射 | Nexthop 地址 | Warm reboot 时映射新旧 NHG ID |
| SONiC NHG 表 | SONiC NHG ID | 存储 gateway NHG（PIC NHG），用于 SRv6 VPN |
| NEXTHOP -> SONIC NHG ID 表 | Nexthop 地址 | PIC Edge 快速查找 |

**新增字段：**
- `RIBNHGEntry::m_resolved_enable_group`：跟踪每个 resolved 成员的启用/禁用状态。NHG Install/Update 事件重置为全部启用；backwalk 期间标记失败路径为禁用。实现可选 `unordered_map<uint32_t, bool>` 或 `std::set<uint32_t>`（仅存储禁用 ID），实现阶段评估。

### 接口定义

| 函数 | 输入 | 输出 | 职责 |
|------|------|------|------|
| `fib_nhg_trigger_node_quick_fixup` | nexthop_addr, resolved_nhg_id, original_nhg_id | void | NHT 事件入口，初始化 walking ctx 并启动 backwalk |
| `fib_nhg_back_walk` | nhg_id, walking_ctx, depth | void | 递归反向遍历 NHG 依赖图，含 null 安全检查和深度限制 |
| `fib_nhg_walk_spec_for_node_quick_fixup` | RIBNHGEntry, ctx | bool | 判断节点是否受影响并禁用失败路径 |
| `fib_nhg_prune_spec_for_node_quick_fixup` | RIBNHGEntry, walk_result | bool | 判断是否停止继续遍历 |
| `RIBNHGEntry::getNextHopGroupFields` | - | string fields | 生成 APPDB 输出，排除禁用路径 |

### 关键行为规则

- **起始节点行为**：walk_spec 对起始节点（resolved_nhg_id）通常返回 false（它是触发点，非更新目标）。backwalk 无论 walk_spec 结果如何都继续遍历其 dependents。
- **单路径 NHG**：如果 NHG 仅有一条路径且需要禁用，跳过 APPDB 写入（不写空 NHG）。此行为不劣于无 PIC 现状。
- **FRR 权威性**：FRR 重新收敛后发送的 NHG 更新会完全覆盖 quick fixup 状态（m_resolved_enable_group 重置）。快速修复是临时优化，FRR 收敛结果是最终状态。
- **部分更新优于无更新**：backwalk 中 getEntry() 返回 null 或 writeToDB() 失败时，记录警告并继续遍历，不终止。

### 约束条件

- backwalk 使用 visited_node_set 防止重复访问
- 防御性递归深度限制（1000 层），超过时记录警告
- SRv6 VPN 场景中 backwalk 在 gateway NHG 处停止（prune spec）
- 快速修复具有幂等性，重复 NHT 事件产生相同结果
- 依赖 FRR 提供 resolve-through/resolve-via 信息和 NHT 触发
- VRF 隔离：需验证 Zebra NHG ID 在跨 VRF 场景中的全局唯一性

### 当前限制

1. Phase 2（PIC Edge / SONiC NHG 表遍历）尚未实现 -- SRv6 VPN PIC Edge 收敛依赖此功能
2. original_nhg_id 参数保留但未使用
3. 无 CLI 可视化 backwalk 状态
4. 依赖 FRR Phase 1 变更完成
5. 单路径 NHG 失败时流量持续黑洞直到 FRR 收敛
6. 部分 backwalk 失败时已更新的 NHG 不回滚（渐进改善策略）
7. 防御性递归深度限制 1000 层

### 性能评估

- backwalk 时间复杂度：O(N * M)，N = 受影响 NHG 数，M = 平均成员数
- 10K NHG / 平均 4 成员 -> 约 100ms（APPDB 写入为瓶颈）
- m_resolved_enable_group 内存开销：10K NHG * 4 成员 * 4 字节 ≈ 160KB

### 日志规范

- NOTICE：FIB 块初始化完成
- DEBUG：每次 backwalk 触发（nexthop 地址、NHG ID、访问节点数）
- WARNING：getEntry() 返回 null、递归深度超限

## 变更记录

| 日期 | 功能 | 变更说明 |
|------|------|---------|
| 2026-04-16 | 路由收敛处理 | 新增：backwalk 基础设施、快速修复机制、PIC Core/Edge 双阶段设计 |
| 2026-04-16 | 路由收敛处理（探索细化） | 更新：起始节点行为、null 安全、深度限制、并发 NHT 处理、Phase 2 测试范围、VRF 隔离验证、日志规范 |
