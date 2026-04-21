<!-- sonic-dev-v1.0 -->
<SONIC_PROJECT_CONTEXT>

<!-- Path Variables (install.sh will replace these on deployment) -->
<!-- CHECKLIST_PATH: extra_info/designed_checklist.md -->
<!-- TEMPLATE_PATH: extra_info/testcase_designed_template.md -->

<EXTREMELY_IMPORTANT>

## SONiC 功能开发判断

绑定了 Step 0 钩子的 skill（propose、explore、brainstorming）在执行时，必须确认当前是否为 SONiC 功能开发：

1. **检查会话上下文**：如果当前会话已经通过 `/alinos.propose` 或其他 skill 确认了"是，SONiC 功能开发"，则**直接沿用**该判断，无需再次询问。

2. **首次判断**（仅在会话中尚未确认时）：使用 **AskUserQuestion** 工具询问：
   - 问题: "是否 SONiC 功能开发?"
   - 选项: "是，SONiC 功能开发" / "否，非 SONiC 功能"

3. 如果判断结果为 **"否"**：跳过本文件所有绑定规则，直接进入标准 skill 流程。

4. 如果判断结果为 **"是"**：继续执行下方对应 skill 的绑定规则。

此判断具有**最高优先级**，在任何 skill 步骤之前执行。

</EXTREMELY_IMPORTANT>

---

## `/alinos.propose` 绑定规则

执行 `/alinos.propose` 时，在 SONiC 判断通过后（选择"是"），必须在标准步骤之前执行以下流程：

### Step P1: 选择设计模版

使用 **AskUserQuestion** 工具询问：
- 问题: "选择设计模版"
- 选项: "SONiC 社区 HLD（英文 13 章）" / "团队内部设计（中文 + checklist + 测试模版）"

### Step P2a: 社区 HLD 路径（英文）

如果选择"SONiC 社区 HLD"：
1. 按本文件「HLD 模版」章节的 13 章结构生成英文 HLD 文档
2. 自检：13 章节是否完整

### Step P2b: 团队内部设计路径（中文）

如果选择"团队内部设计"：

1. 使用 **Read** 工具读取 `CHECKLIST_PATH`（见路径变量声明）
   - 文件不存在 → 使用 **AskUserQuestion** 警告用户，跳过 checklist 步骤
2. 输出**自检清单覆盖表**（必填，不可跳过）：

| 编号 | 自检项 | 是否涉及 | 分析说明 |
|------|--------|---------|---------|
| UI-1 | CLI/Show 命令 | 是/否 | 具体分析 |
| UI-2 | YANG Model 适配 | 是/否 | 具体分析 |
| ... | （共 32 行，每项一行） | ... | ... |

3. 使用 **Read** 工具读取 `TEMPLATE_PATH`（见路径变量声明）
   - 文件不存在 → 使用 **AskUserQuestion** 警告用户，跳过测试模版步骤
4. 按模版章节结构生成**功能测试/UT 设计**（包含 TC-XXX 用例格式和 TC-ERR-XXX 异常测试格式）
5. 使用 **TodoWrite** 工具为"是否涉及=是"的 checklist 项创建跟踪任务

### Step P3: 合规自检

输出以下自检清单（所有路径均须执行）：

```
## 合规自检
- [ ] 自检清单覆盖表行数 = checklist 文件条目数（当前 32 项）
- [ ] 所有"是否涉及=是"的项有具体分析，非空泛描述
- [ ] 输出文档包含模版要求的所有必填章节
- [ ] 功能测试设计涵盖了 SCENARIO-1~15 中所有标记为"是"的场景
```

完成后，继续执行标准 propose 流程的剩余步骤。

---

## `/alinos.explore` 绑定规则

执行 `/alinos.explore` 时，在 SONiC 判断通过后（选择"是"），在标准苏格拉底六维探索**完成后**，增加以下 phase：

### Phase Extra: SONiC 自检清单校对

1. 使用 **Read** 工具读取 `CHECKLIST_PATH`
2. 输出**自检清单覆盖表**（格式同 propose 绑定规则，32 行）
3. 使用 **TodoWrite** 工具为"是否涉及=是"的项创建跟踪
4. 对比 propose 阶段的覆盖表，检查是否有新增"应涉及"的项
5. 未覆盖但应涉及的项 → 标记为 gap → 对 gap 项进行苏格拉底审查
6. 输出合规自检清单（同 propose Step P3）

---

## `/alinos.brainstorming` 绑定规则

执行 `/alinos.brainstorming` 时，在 SONiC 判断通过后（选择"是"），在标准流程的"探索项目上下文"步骤后，增加：

1. 使用本文件「SONiC 架构概览」章节的 8 类组件分类，逐类确认当前功能的影响范围
2. 输出**组件影响分析表**：

| 组件类别 | 是否影响 | 影响说明 |
|---------|---------|---------|
| 控制面 (SWSS) | 是/否 | |
| 路由 | 是/否 | |
| 数据面 | 是/否 | |
| 管理面 | 是/否 | |
| 基础设施 | 是/否 | |
| 平台 | 是/否 | |
| 监控/遥测 | 是/否 | |
| 构建/测试 | 是/否 | |

3. 使用 **Read** 工具读取 `CHECKLIST_PATH`，输出自检清单覆盖表（32 行）
4. 输出合规自检清单

---

## SONiC 架构概览

### 组件分类（8 类）

| 类别 | 组件 |
|------|------|
| 控制面 (SWSS) | orchagent, syncd, *mgrd (portmgrd, intfmgrd, vlanmgrd, buffermgrd 等) |
| 路由 | FRR (BGP, OSPF, IS-IS), zebra, route maps, fpmsyncd, bgpcfgd |
| 数据面 | SAI, ASIC SDK, kernel datapath (TC, iptables) |
| 管理面 | sonic-cli (AliNOS CLI), sonic-utilities, REST API, gNMI, SNMP |
| 基础设施 | swss-common, sonic-py-common, Redis DBs, Yang models |
| 平台 | BSP, platform plugins, PDK, thermal/fan/PSU 控制 |
| 监控/遥测 | sflow, telemetry container, gNMI streaming, counters |
| 构建/测试 | sonic-buildimage, pytest, GTest, vs-image, sonic-mgmt, SAI-PTF |

### 核心数据流

```
AliNOS CLI (sonic-cli) -> sonic-utilities -> CONFIG_DB -> *mgrd -> APP_DB -> orchagent -> ASIC_DB -> syncd -> SAI -> ASIC
                                                                                                          |
                                                                     STATE_DB <- orchagent <- ASIC_DB (状态回读)
```

**CLI 完整链路**: AliNOS CLI（sonic-cli 仓库）是独立的用户配置界面，部分命令调用 sonic-utilities 进行实际的配置写入 CONFIG_DB。开发 CLI 功能时需同时关注 sonic-cli 和 sonic-utilities 两个仓库。

### Redis DB 用途

| DB | 用途 |
|----|------|
| CONFIG_DB | 用户配置持久化 |
| APP_DB | 应用层数据（*mgrd 写入，orchagent 消费） |
| ASIC_DB | ASIC 编程状态（orchagent 写入，syncd 消费） |
| STATE_DB | 系统运行状态 |
| COUNTERS_DB | 统计计数器 |
| FLEX_COUNTER_DB | 灵活计数器配置 |

---

## DC 网络架构

### 设备角色体系

| 角色 | 说明 | 典型位置 |
|------|------|---------|
| ASW (ToR) | 接入交换机，连接服务器 | 机柜顶 |
| PSW (Leaf) | 汇聚交换机 | Pod 内 |
| DSW (Spine) | 核心交换机 | Pod 间 |
| MA | 管理接入 | 带外管理 |
| DCC | 数据中心核心 | DC 间 |
| MC | 管理核心 | 管理网核心 |

### 配置下发链路

```
集中配置平台 → GRPC/Telemetry → 设备 Telemetry 模块 → Config DB
```

**Telemetry 白名单机制**: 通过 GRPC 下发的配置必须在设备的 Telemetry 白名单中显式声明，否则会被拒绝。新功能如需通过 Telemetry 下发配置，必须同步申请加入白名单。

### 功能上线要求

- 所有功能必须具备**回退能力**（功能级别关闭，配置清理，状态清理，资源释放）
- 配置区分**基础配置**（启动时加载）和**动态配置**（运行时修改）

---

## SONiC 代码审查清单

执行 /alinos.code-review 时逐项检查：

**Redis DB 交互**
- [ ] 使用正确的 Table 类型（Producer/Consumer/Subscriber）
- [ ] key 格式符合 SONiC 约定
- [ ] 写入顺序不导致处理异常
- [ ] DB 操作原子性

**内存安全（C++）**
- [ ] RAII 和智能指针，无裸 new/delete
- [ ] 异常安全：task 处理中异常不泄漏资源
- [ ] 无 buffer overflow 风险

**向后兼容**
- [ ] Config DB schema 兼容旧版本配置
- [ ] Yang model 变更不破坏现有校验
- [ ] API 变更不影响现有调用方

**平台可移植性**
- [ ] 不依赖特定 ASIC vendor 行为
- [ ] 多 ASIC 平台已考虑
- [ ] 平台相关代码在 platform/ 层隔离

**测试覆盖**
- [ ] 单元测试覆盖核心逻辑路径
- [ ] 集成测试覆盖 Redis DB 交互路径
- [ ] 功能测试方法已设计（若涉及端到端行为）

**Warm/Fast Reboot**
- [ ] 变更是否影响 warm-reboot 流程
- [ ] 状态恢复逻辑是否正确

---

## SONiC 调试工具速查

| 场景 | 命令/工具 |
|------|----------|
| Redis DB 查看 | `redis-cli -n <db_id>`, `sonic-db-cli` |
| 进程日志 | `journalctl -u <service>`, `/var/log/syslog` |
| SAI 计数器 | `show interfaces counters`, `portstat` |
| 配置验证 | `sonic-cfggen -d --print-data`, `show runningconfiguration` |
| 系统状态 | `show system-health`, `show services` |
| 路由状态 | `show ip route`, `vtysh -c "show ..."` |
| 平台信息 | `show platform summary`, `show environment` |

---

## HLD 模版（上游贡献用）

上游 SONiC 社区 HLD 文档必须包含以下章节：

1. Revision / 修订历史
2. Scope / 范围
3. Definitions & Abbreviations / 术语定义
4. Overview / 概述
5. Requirements / 需求
6. Architecture Design / 架构设计
7. High-Level Design / 高层设计（按组件细分）
8. SAI API / SAI 接口变更
9. Configuration and Management / 配置管理（CLI, Yang, REST, gNMI）
10. Warmboot and Fastboot Design Impact / 重启影响
11. Restrictions/Limitations / 限制
12. Testing Requirements/Design / 测试需求和设计
13. Open/Action Items / 待解决事项

---

## 仓库级开发规则

### 适用仓库：sonic-swss
- orchagent 特性开发（C++）、*mgrd 守护进程
- 每个功能变更必须配套单元/集成测试，作为同一 PR 的一部分
- 集成测试在 tests/ 目录，使用 vs-image 运行
- C++ 新功能用 GTest，Python 脚本用 pytest
- C++ 内存管理：RAII、智能指针、避免裸 new/delete
- orchagent 中的 task/event 处理必须考虑异常安全
- ProducerStateTable 写入顺序影响 orchagent 处理逻辑
- SAI 属性设置的错误处理（SAI_STATUS_SUCCESS 检查）

### 适用仓库：sonic-swss-common
- 作为基础库，任何变更必须配套 GTest 单元测试
- 接口变更需考虑所有下游依赖的兼容性
- Python 绑定变更同步配套 pytest 测试
- API 向后兼容性（下游组件依赖此库）
- 线程安全（多个守护进程并发使用）

### 适用仓库：sonic-sairedis
- SAI 接口变更必须配套 GTest 单元测试
- 使用 vslib (Virtual SAI) 进行本地测试验证
- 序列化/反序列化逻辑必须覆盖正常和异常输入
- syncd 重启和 warm-reboot 逻辑的安全性

### 适用仓库：sonic-cli
- AliNOS CLI 用户配置界面，独立于 sonic-utilities
- CLI 命令通过调用 sonic-utilities 完成实际配置写入
- 新增 CLI 命令需同步评估 sonic-utilities 侧是否需要配套修改
- CLI 参数校验和帮助文本需在 sonic-cli 侧实现
- show 命令的数据源确认（直接读 DB 还是调用 utilities）

### 适用仓库：sonic-utilities
- CLI 命令基于 click 框架
- 新 CLI 命令必须配套 pytest 单元测试
- CLI 输出格式一致性（tabulate 格式、JSON 输出）
- Yang model 校验覆盖所有配置路径
- 错误信息对用户友好，包含操作建议

### 适用仓库：sonic-buildimage
- 修改子模块代码后需同步更新 sonic-buildimage 的子模块引用
- PR 需要在子模块仓库和 sonic-buildimage 分别提交
- Dockerfile.j2 变更须考虑镜像大小影响
- platform/ 变更须确认不影响其他平台

---

## L3 输出格式示例（BGP advertise-delay 场景）

以下为团队内部设计路径的理想输出片段，用于格式对齐参考：

### 自检清单覆盖表（片段）

| 编号 | 自检项 | 是否涉及 | 分析说明 |
|------|--------|---------|---------|
| UI-1 | CLI/Show 命令 | 是 | 新增 `bgp advertise-delay-onstartup` 配置命令和 `show bgp summary` 状态显示 |
| UI-2 | YANG Model 适配 | 是 | 新增 BGP_GLOBAL_PARAMETERS 表的 advertise_delay_onstartup 字段 |
| UI-3 | Telemetry 白名单 | 否 | 本功能通过 CLI 直接配置，不通过 Telemetry 下发 |
| SPEC-1 | 功能点与原始需求 | 是 | 4 个功能点：仅重启触发、基于首个 Peer UP 计时、与 update-delay 兼容、loopback 放通 |
| SPEC-5 | 基础配置与动态修改 | 是 | 基础配置，设备启动时加载，不支持运行时动态修改 |
| SCENARIO-14 | Reboot / 打补丁 | 是 | 设备 reboot 后配置持久化，timer 重新触发 |

### 功能测试概览表（片段）

| 用例编号 | 用例标题 | 覆盖功能点 | 通过状态 |
|---------|---------|-----------|---------|
| TC-001 | advertise-delay-onstartup 基本功能验证 | 功能点 1,2 | ⬜ |
| TC-ERR-001 | 进程异常重启后 advertise-delay 状态恢复 | SCENARIO-1 | ⬜ |

---

## 团队基础设施

<!-- 团队安装后根据实际情况修改以下内容 -->
- Virtual switch (vs-image): 可用
- 物理测试床: 可用
- 多 ASIC 测试床: 可用
- CI pipeline: [填写 CI 系统信息]

</SONIC_PROJECT_CONTEXT>
