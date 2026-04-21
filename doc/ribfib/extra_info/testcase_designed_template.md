# [功能名]-测试文档

<!-- 该文档统一放在 ALINOS 开发目录下，供运营同学指导测试、ENC 同学指导配置格式、
     架构配置模版同学新增配置参考。 -->

---

# 1 功能定义

<!-- 如果有设计文档，1.1/1.2 章节可在设计文档中体现，本文档此处提供设计文档链接即可。
     也可考虑增加简要的功能设计说明。 -->

## 1.1 功能描述

> _advertise-delay on startup 功能在 BGP 启动后，以第一个邻居建立为起点计时，在 timer 期间不对外发布路由，在 timer 后开始向外发布路由。解决 BGP 邻居和路由规格大的场景，同时处理 BGP 邻居路由发布和 BGP 邻居建立性能问题引发的 BGP 邻居无法稳定建连的问题_

> _原始需求链接_

> _【待填写：需求链接】_

## 1.2 功能分析

<!-- 此处需要和需求方反复对齐确认 -->

> _1. 仅在重启场景触发，后续 BGP 重连不重复触发_

> _2. advertise-delay-timer 的计时器基于第 1 个 Peer UP 时开始计时_

> _3. advertise-delay on startup 和 update-delay 需兼容使用_

> _4. loopback 地址需要能在 timer 期间放通，缩短设备托管时间_

---

# 2 使用范围

<!-- 明确功能的适用范围，与 designed_checklist.md 中 SPEC-3/SPEC-4 对应 -->

| 维度 | 内容 |
|------|------|
| **支持的角色** | _lsw_ |
| **支持的架构范围** | _lsw7.1_ |
| **支持的平台** | _th5_ |
| **支持的版本** | _7.3.2_ |

---

# 3 命令

## 3.1 CLI 命令

```
root@PSW1.ne558:~# cli
PSW1.ne558# configure terminal
PSW1.ne558(config)# bgp advertise-delay-onstartup 800
```

> _【待填写：按需补充命令说明】_

## 3.2 Config DB 配置

```json
{
    "BGP_GLOBAL_PARAMETERS": {
        "advertise_delay_onstartup": {
            "delay_timer": "800"
        }
    }
}
```

> **注意：**
>
> 1. `advertise_delay_map` 的配置同 `advertise_delay` 的使用
> 2. `advertise_delay_onstartup` 和 `advertise_delay` 使用冲突，二者不可同时存在
> 3. lsw 的场景下，根据路由量和 BGP neighbor 量，建议 `advertise_delay_onstartup` 配置时长为 800

---

# 4 测试过程

## 4.1 测试拓扑

> _【待填写：测试拓扑图或拓扑描述】_

## 4.2 测试基础配置

> _使用 lsw7.1 配置_

> _【待填写：基础配置内容或配置文件路径】_

## 4.3 功能测试

### 测试概览

<!-- 须涵盖 1.2 功能分析中的所有功能点，每个功能点至少一条用例 -->

| 用例编号 | 用例标题 | 覆盖功能点 | 通过状态 |
|---------|---------|-----------|---------|
| TC-001 | advertise-delay-onstartup 基本功能验证 | 功能点 1,2 | ⬜ |
| TC-002 | 与 update-delay 兼容性验证 | 功能点 3 | ⬜ |
| TC-003 | timer 期间 loopback 地址放通验证 | 功能点 4 | ⬜ |
| _TC-NNN_ | _【待填写】_ | | ⬜ |

### TC-001: advertise-delay-onstartup 基本功能验证

| 字段 | 内容 |
|------|------|
| **前置条件** | lsw7.1 架构，BGP 邻居未建立 |
| **测试步骤** | 1. 配置 `bgp advertise-delay-onstartup 800` 2. 建立第一个 BGP 邻居 3. 观察 timer 期间是否抑制路由发布 4. 等待 timer 超时后验证路由开始外发 |
| **预期结果** | timer 期间无路由发布，超时后路由正常外发 |
| **实际结果** | _【测试后填写】_ |
| **通过状态** | ⬜ Pass / ⬜ Fail |

### TC-002: 与 update-delay 兼容性验证

| 字段 | 内容 |
|------|------|
| **前置条件** | _【待填写】_ |
| **测试步骤** | _【待填写】_ |
| **预期结果** | _【待填写】_ |
| **实际结果** | _【测试后填写】_ |
| **通过状态** | ⬜ Pass / ⬜ Fail |

### TC-003: timer 期间 loopback 地址放通验证

| 字段 | 内容 |
|------|------|
| **前置条件** | _【待填写】_ |
| **测试步骤** | _【待填写】_ |
| **预期结果** | _【待填写】_ |
| **实际结果** | _【测试后填写】_ |
| **通过状态** | ⬜ Pass / ⬜ Fail |

<!-- 按需继续添加 TC-NNN -->

## 4.4 异常测试

<!-- 覆盖异常、边界、容错场景。
     对照 designed_checklist.md 中 SCENARIO-1~15 逐条评估是否适用。 -->

| 用例编号 | 用例标题 | 关联 SCENARIO | 通过状态 |
|---------|---------|--------------|---------|
| TC-ERR-001 | 进程异常重启后 advertise-delay 状态恢复 | SCENARIO-1 | ⬜ |
| TC-ERR-002 | timer 期间 BGP 邻居全部断开 | SCENARIO-9 | ⬜ |
| TC-ERR-003 | 设备 reboot 后功能恢复 | SCENARIO-14 | ⬜ |
| _TC-ERR-NNN_ | _【待填写】_ | | ⬜ |

### TC-ERR-001: 进程异常重启后 advertise-delay 状态恢复

| 字段 | 内容 |
|------|------|
| **触发条件** | BGP 进程在 advertise-delay timer 运行期间被 kill |
| **测试步骤** | 1. 配置 advertise-delay-onstartup 并建立 BGP 邻居 2. 在 timer 运行期间 kill BGP 进程 3. 等待进程自动重启 4. 验证 timer 和路由发布状态 |
| **预期行为** | 进程重启后 advertise-delay 重新生效，路由不会在 timer 期间发布 |
| **恢复验证** | _【测试后填写：功能是否完全恢复正常】_ |
| **通过状态** | ⬜ Pass / ⬜ Fail |

### TC-ERR-002: timer 期间 BGP 邻居全部断开

| 字段 | 内容 |
|------|------|
| **触发条件** | _【待填写】_ |
| **测试步骤** | _【待填写】_ |
| **预期行为** | _【待填写】_ |
| **恢复验证** | _【测试后填写】_ |
| **通过状态** | ⬜ Pass / ⬜ Fail |

### TC-ERR-003: 设备 reboot 后功能恢复

| 字段 | 内容 |
|------|------|
| **触发条件** | _【待填写】_ |
| **测试步骤** | _【待填写】_ |
| **预期行为** | _【待填写】_ |
| **恢复验证** | _【测试后填写】_ |
| **通过状态** | ⬜ Pass / ⬜ Fail |

<!-- 按需继续添加 TC-ERR-NNN -->

## 4.5 自动化测试

<!-- 追踪每条用例的自动化覆盖状态 -->

| 用例编号 | 自动化状态 | 自动化框架 | 脚本路径 | 备注 |
|---------|-----------|-----------|---------|------|
| TC-001 | ⬜ 已自动化 / ⬜ 待自动化 / ⬜ 不适用 | | | |
| TC-002 | ⬜ 已自动化 / ⬜ 待自动化 / ⬜ 不适用 | | | |
| TC-003 | ⬜ 已自动化 / ⬜ 待自动化 / ⬜ 不适用 | | | |
| TC-ERR-001 | ⬜ 已自动化 / ⬜ 待自动化 / ⬜ 不适用 | | | |
| TC-ERR-002 | ⬜ 已自动化 / ⬜ 待自动化 / ⬜ 不适用 | | | |
| TC-ERR-003 | ⬜ 已自动化 / ⬜ 待自动化 / ⬜ 不适用 | | | |

## 4.6 场景关联矩阵

<!-- 关联 designed_checklist.md 中 SCENARIO-1~15 条目，确保异常场景全覆盖。
     对每个 SCENARIO 条目标记"是否适用"，适用的必须有对应测试用例。 -->

| Checklist 条目 | 场景描述 | 是否适用 | 对应用例 | 备注 |
|---------------|---------|---------|---------|------|
| SCENARIO-1 | 进程异常重启后功能自动恢复 | ⬜ 是 / ⬜ 否 | TC-ERR-001 | |
| SCENARIO-2 | 接口 Down/Up | ⬜ 是 / ⬜ 否 | | |
| SCENARIO-3 | 子接口扩缩容 | ⬜ 是 / ⬜ 否 | | |
| SCENARIO-4 | 路由隔离 | ⬜ 是 / ⬜ 否 | | |
| SCENARIO-5 | LACP 隔离 / Shut 所有下连口 | ⬜ 是 / ⬜ 否 | | |
| SCENARIO-6 | BGP 扩缩容 | ⬜ 是 / ⬜ 否 | | |
| SCENARIO-7 | 路由震荡 | ⬜ 是 / ⬜ 否 | | |
| SCENARIO-8 | 接口震荡 | ⬜ 是 / ⬜ 否 | | |
| SCENARIO-9 | 邻居震荡 | ⬜ 是 / ⬜ 否 | TC-ERR-002 | |
| SCENARIO-10 | MAC 震荡 | ⬜ 是 / ⬜ 否 | | |
| SCENARIO-11 | BMC 掉电 | ⬜ 是 / ⬜ 否 | | |
| SCENARIO-12 | 磁盘更换 | ⬜ 是 / ⬜ 否 | | |
| SCENARIO-13 | 插拔光模块 | ⬜ 是 / ⬜ 否 | | |
| SCENARIO-14 | Reboot / 打补丁 | ⬜ 是 / ⬜ 否 | TC-ERR-003 | |
| SCENARIO-15 | Warm Boot | ⬜ 是 / ⬜ 否 | | |

---

# 5 Debug 命令

<!-- 请列出排查该功能所需的所有 show/debug 命令 -->

* 查看 advertise-delay-onstartup 状态

    > `show bgp summary` 存在 3 种显示格式：

    * 如果 advertise-delay-onstartup 未启动（BGP 实例不存在 peer 建立）
    * 如果 advertise-delay-onstartup 开始启动，但未超时（BGP 实例存在 peer 建立）
    * 如果 advertise-delay-onstartup 开始启动，且超时

* **未启动**

> ![image.png](https://alidocs.oss-cn-zhangjiakou.aliyuncs.com/res/8oLl95VQdNd70lap/img/fc92fb5f-5bd0-4970-9bf3-873a004b3f8f.png)
<!-- 请替换为实际截图或文本输出 -->

> 无 advertise delay 相关信息

* **启动未超时**

> ![image.png](https://alidocs.oss-cn-zhangjiakou.aliyuncs.com/res/8oLl95VQdNd70lap/img/26c12076-7460-432a-8842-c022076558bd.png)
<!-- 请替换为实际截图或文本输出 -->

> 显示用户 advertise-delay-onstartup 的配置
>
> 显示 BGP 实例第一个 peer 建立的时间点
>
> 如果 advertise-delay-onstartup timer 未超时，显示 in progress

* **启动且超时**

> ![image.png](https://alidocs.oss-cn-zhangjiakou.aliyuncs.com/res/8oLl95VQdNd70lap/img/f6f0921c-e44f-41a2-b93f-53cd0b04e9b6.png)
<!-- 请替换为实际截图或文本输出 -->

> 显示用户 advertise-delay-onstartup 的配置
>
> 显示 BGP 实例第一个 peer 建立的时间点
>
> 如果 advertise-delay-onstartup timer 超时，显示具体时间节点
>
> 同时路由开始外发

---

# 6 代码提交

> 提供 submodule 和 build image 的提交 PR

<!-- 请附上 code review 通过的截图或链接 -->

> _【待填写：PR 链接】_
