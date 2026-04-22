# OSCE考试系统项目说明文档

## 1. 项目概述

### 1.1 项目背景
OSCE（客观结构化临床考试）是医学教育中广泛使用的临床能力评估方法。本系统旨在构建一个**分布式、高并发、高可靠**的数字化OSCE考核平台，支持大规模、多考站、多考官的复杂考核场景。

### 1.2 核心目标
- **标准化**：统一考核流程与评分标准
- **高效性**：支持考生自动轮转、多考官协同
- **可靠性**：保障考试过程稳定，具备完善的异常处理与重考机制
- **可扩展**：支持未来功能模块与第三方系统集成

### 1.3 系统范围
涵盖从考试编排、考生调度、考站执行、多考官评分到成绩汇总的全流程数字化管理。

## 2. 总体架构设计

### 2.1 架构愿景
**“以调度引擎为核心，基于事件驱动的分布式考试系统”**。

### 2.2 核心架构图
系统采用清晰的分层架构，实现关注点分离和解耦。
```mermaid
graph TD
    %% ===== 接入层 =====
    subgraph 接入层
        PAD[考官PAD]
        CAND[考生终端]
        ADMIN[管理后台]
    end

    %% ===== 网关层 =====
    subgraph 网关层
        GATEWAY[API网关]
        AUTH[认证服务]
    end

    %% ===== 调度核心层 =====
    subgraph 调度核心
        EXAM[考试管理服务]
        SCHED[调度引擎]
        RULE[排考规则引擎]
    end

    %% ===== 业务服务层 =====
    subgraph 核心业务服务
        USER[用户与权限服务]
        STATION[考站任务服务]
        SCORE[评分服务]
        EXAMINER[考官管理服务]
        DEVICE[设备管理服务]
        BIND[绑定关系服务]
        COLLAB[评分协同服务]
    end

    %% ===== 事件驱动层 =====
    subgraph 事件系统
        MQ[消息中间件]
        EVENT[事件总线]
    end

    %% ===== 保障与容错 =====
    subgraph 保障机制
        RETRY[补偿/重试服务]
        OFFLINE[离线缓存服务]
        MONITOR[监控告警]
    end

    %% ===== 数据层 =====
    subgraph 数据层
        DB[(业务数据库)]
        CACHE[(缓存Redis)]
        LOG[(日志/审计)]
    end

    %% ===== 外部系统 =====
    subgraph 第三方
        HIS[HIS/教务系统]
        SSO[统一认证]
    end

    %% ===== 连接关系 =====
    PAD --> GATEWAY
    CAND --> GATEWAY
    ADMIN --> GATEWAY

    GATEWAY --> AUTH
    AUTH --> USER
    AUTH --> BIND

    GATEWAY --> STATION
    GATEWAY --> SCORE

    EXAM --> SCHED
    SCHED --> RULE

    SCHED --> MQ
    MQ --> STATION
    MQ --> SCORE
    MQ --> COLLAB

    STATION --> PAD
    SCORE --> MQ

    DEVICE --> BIND
    EXAMINER --> BIND

    COLLAB --> SCORE

    MQ --> RETRY
    RETRY --> SCHED

    PAD --> OFFLINE
    OFFLINE --> MQ

    MONITOR --> DEVICE
    MONITOR --> MQ

    STATION --> DB
    SCORE --> DB
    USER --> DB

    DB --> CACHE
    MQ --> LOG

    USER --> SSO
    EXAM --> HIS
```

### 2.3 核心设计理念
1.  **调度引擎为大脑**：集中管控全流程状态与流转。
2.  **事件驱动解耦**：通过消息中间件(MQ)连接各服务，实现异步、松耦合通信。
3.  **设备与考官强绑定**：确保评分责任到人，操作可追溯。
4.  **分层容错机制**：从设备、业务到系统层，均有完备的异常处理与恢复策略。

## 3. 核心组件详解

### 3.1 调度引擎 (Scheduler)
**职责**：系统的指挥中心。
- **轮转调度**：根据排考规则，驱动考生在多个考站间流转。
- **并发控制**：管理多考站并行考核。
- **异常调度**：触发并管理补考、重考流程。
- **时间驱动**：严格控制各考站计时与全局考试进度。

**本质**：一个由**状态机、队列和时间器**组成的核心控制器。

### 3.2 多考官多设备绑定机制
为支持一个考站内多位考官独立评分，需建立精确的考官与设备映射关系。

**核心抽象**：
```
考站
 ├── 考官列表 [(序号1, 考官A), (序号2, 考官B)]
 └── 设备列表 [(序号1, 设备PAD-001), (序号2, 设备PAD-002)]
```
**绑定关系**：`考官序号` ↔ `设备序号`

**工作流程**：管理员在后台配置考站时，即建立考官与设备的静态绑定。设备启动和考官登录时需进行双重验证。
```mermaid
sequenceDiagram
    participant Admin as 管理员后台
    participant StationMgr as 考站编排服务
    participant ExaminerMgr as 考官管理服务
    participant DevMgr as 设备管理服务
    participant PAD1 as 考官1PAD<br>(设备序号1)
    participant PAD2 as 考官2PAD<br>(设备序号2)
    participant Auth as 认证服务
    participant Binding as 绑定管理服务

    Note over Admin, Binding: 阶段1：管理员配置
    Admin->>StationMgr: 1. 创建多考官考站
    Admin->>ExaminerMgr: 2. 分配考官及序号
    Admin->>DevMgr: 3. 分配设备及序号
    Admin->>Binding: 4. 建立绑定关系(考官1↔设备1, 考官2↔设备2)

    Note over PAD1, Auth: 阶段2：设备启动与登录验证
    PAD1->>DevMgr: 5. 启动注册
    DevMgr->>Binding: 6. 查询绑定信息
    Binding-->>DevMgr: 7. 返回绑定(考站915, 序号1)
    DevMgr-->>PAD1: 8. 返回设备凭证
    PAD1->>Auth: 9. 考官A登录(账号+设备凭证)
    Auth->>Binding: 10. 验证绑定关系
    Binding-->>Auth: 11. 验证通过(序号匹配)
    Auth-->>PAD1: 12. 登录成功
```

**强约束**：考官登录时必须通过 **“双因子匹配”**（考官身份 + 指定设备），否则拒绝登录或触发重新绑定流程。

### 3.3 评分协同服务
支持三种多考官评分协同模式，适应不同考核场景：

| 模式 | 机制 | 应用场景 |
| :--- | :--- | :--- |
| **独立评分** | 考官独立评分，系统自动计算平均分或总和。 | 常规OSCE站，多位考官视角互补。 |
| **主副考官** | 主考官评分占高权重（如70%），副考官评分占低权重（如30%），系统加权计算最终成绩。 | 教学评估，强调主考官的判断。 |
| **共识评分** | 考官独立评分，若差异超过预设阈值，系统提示冲突，可在线讨论或由管理员仲裁后确定最终成绩。 | 高利害考试，确保评分公平公正。 |

### 3.4 事件驱动与消息中间件(MQ)
**原则**：所有核心业务流程的驱动（如考生入场、轮转、评分提交）必须通过事件发布/订阅模式，禁止服务间直接调用。
- **优势**：解耦服务，提高系统可扩展性和容错性。
- **关键事件**：
    - `Examinee.Assigned` (考生分配到站)
    - `Station.Rotate` (考站轮转)
    - `Scoring.Submitted` (评分提交)
    - `Exam.Start/End` (考试开始/结束)

### 3.5 延迟与容错机制
系统设计了三层容错保障：

| 层级 | 机制 | 目标 |
| :--- | :--- | :--- |
| **设备层** | 心跳检测、断线重连、离线缓存 | 保障单设备在弱网/断网下的基本操作与数据暂存。 |
| **业务层** | 操作超时锁定、异常状态标记、本地优先策略 | 防止业务流程因单点故障而阻塞，保证流程可继续。 |
| **系统层** | 补偿/重试服务(Retry)、幂等性设计、异步数据同步 | 最终数据一致性，自动或半自动恢复异常中断的业务。 |

**重考机制流程**：
1.  调度器或监控服务检测到异常（如设备故障、超时）。
2.  标记异常考生，并收集至`Retry`服务。
3.  `Retry`服务将异常考生名单提交考试管理服务，生成补考计划。
4.  调度器将补考计划插入调度队列（支持插队或独立批次）。
5.  执行补考流程。

## 4. 核心业务流程与状态机

### 4.1 考生全生命周期状态机
```mermaid
stateDiagram-v2
    [*] --> 待考
    待考 --> 候考: 完成签到/入场
    候考 --> 在站: 调度分配考站
    在站 --> 评分中: 考试开始
    评分中 --> 已完成: 正常提交评分
    在站 --> 异常: 设备故障/考官缺失
    评分中 --> 异常: 超时/中断
    异常 --> 补考待排: 标记异常考生
    补考待排 --> 补考中: 插入补考调度
    补考中 --> 已完成: 补考完成
    在站 --> 已完成: 强制结束
    评分中 --> 已完成: 自动收卷
    已完成 --> [*]
```

### 4.2 考站全生命周期状态机
```mermaid
stateDiagram-v2
    [*] --> 初始化
    初始化 --> 就绪: 设备/考官到位
    就绪 --> 待命: 等待调度
    待命 --> 分配考生: 接收调度指令
    分配考生 --> 候考中: 考生进入
    候考中 --> 考试中: 到达开始时间
    考试中 --> 评分中: 考试结束
    评分中 --> 已完成: 评分提交完成
    已完成 --> 待命: 等待下一轮
    就绪 --> 异常: 设备故障
    待命 --> 异常: 调度异常
    考试中 --> 异常: 中断/掉线
    异常 --> 恢复中: 重试/人工介入
    恢复中 --> 待命: 恢复成功
    异常 --> 关闭: 严重故障
    待命 --> 已关闭: 收到结束指令
    已完成 --> 已关闭: 最后一轮结束
    已关闭 --> [*]
```

## 5. 高并发与分布式部署架构
为支持大规模、多考场同时考试，系统采用分布式集群设计。
```mermaid
graph TD
    subgraph 接入层
        PAD1[PAD集群]
        ADMIN[管理后台]
    end
    GATEWAY[API网关]
    subgraph 调度集群
        SCHED1[调度器1]
        SCHED2[调度器2]
        SCHED3[调度器3]
    end
    SHARD[调度分片规则<br>按考场/场次]
    subgraph MQ集群
        MQ1[Partition1]
        MQ2[Partition2]
        MQ3[Partition3]
    end
    STATION[考站服务集群]
    SCORE[评分服务集群]
    REDIS[(Redis集群)]
    DB[(分库分表DB)]
    RETRY[补偿服务]

    PAD1 --> GATEWAY
    GATEWAY --> SCHED1 & SCHED2 & SCHED3
    SCHED1 & SCHED2 & SCHED3 --> SHARD
    SHARD --> MQ1 & MQ2 & MQ3
    MQ1 & MQ2 & MQ3 --> STATION
    STATION --> SCORE
    SCORE --> MQ1 & MQ2 & MQ3
    STATION & SCORE --> REDIS
    REDIS --> DB
    MQ1 & MQ2 & MQ3 --> RETRY
```
- **调度分片**：不同考场或考试场次由不同的调度器实例负责，实现水平扩展。
- **服务集群**：考站服务、评分服务等无状态业务服务可横向扩展。
- **数据分区**：数据库采用分库分表，缓存使用Redis集群，以承载高并发读写。

## 6. 核心服务模块列表
系统由以下微服务构成，每个服务职责单一：
1.  **用户与权限管理服务**：账号、认证、角色权限。
2.  **考试编排服务**：考试计划、考站编排、考生分配、模板管理。
3.  **考站执行服务**：考站任务、评分表、计时、多媒体采集。
4.  **评分与评估服务**：评分采集、计算、分析、成绩单。
5.  **设备与终端服务**：PAD设备管理、终端应用、离线缓存。
6.  **事件与消息服务**：消息中间件、事件总线、通知、日志。
7.  **数据与同步服务**：数据同步、归档、备份、报表。
8.  **监控与管理服务**：系统监控、告警、配置、运维。
9.  **第三方集成服务**：与教务、HIS、统一认证等系统对接。
10. **安全与合规服务**：数据加密、访问控制、审计、隐私保护。

## OSCE考试系统时序图 - 多考官考站调度流程
### 1. 多考官考站调度时序图
```mermaid
sequenceDiagram
    participant Station as 考站A
    participant Scheduler as 调度器
    participant MQ as 消息队列
    participant Candidate as 考生终端
    participant PadMaster as 主考官PAD
    participant PadSub1 as 副考官1PAD
    participant PadSub2 as 副考官2PAD
    participant Scoring as 评分服务
    participant Consensus as 评分一致性服务

    Note over Station: 阶段1：考站调度申请
    Station->>Station: 1. 检查考站状态\n设备就绪\n考官就位
    Station->>Scheduler: 2. 调度申请请求\n考站ID, 可用容量
    Scheduler->>Scheduler: 3. 检查考生队列\n考站资源
    Scheduler->>MQ: 4. 发布考生分配事件\n考生ID, 考站ID, 考官信息

    MQ-->>Station: 5. 推送考生分配事件
    MQ->>Candidate: 6. 推送考生信息\n考站位置, 准备时间
    MQ->>PadMaster: 7. 推送考生信息\n考生ID, 考试信息
    MQ->>PadSub1: 8. 推送考生信息\n考生ID, 考试信息
    MQ->>PadSub2: 9. 推送考生信息\n考生ID, 考试信息

    Note over Station: 阶段2：考生入场与准备
    Candidate->>Candidate: 10. 考生签到确认
    Candidate->>Station: 11. 考生就位通知
    PadMaster->>PadMaster: 12. 主考官准备就绪
    PadSub1->>PadSub1: 13. 副考官1准备就绪
    PadSub2->>PadSub2: 14. 副考官2准备就绪
    PadMaster->>Station: 15. 主考官就绪通知
    PadSub1->>Station: 16. 副考官1就绪通知
    PadSub2->>Station: 17. 副考官2就绪通知

    Station->>Station: 18. 验证考官就绪状态
    Station->>Candidate: 19. 发送考试开始指令
    Station->>PadMaster: 20. 发送考试开始指令
    Station->>PadSub1: 21. 发送考试开始指令
    Station->>PadSub2: 22. 发送考试开始指令

    Note over Station: 阶段3：考试执行与多考官评分
    Candidate->>Candidate: 23. 考生开始考试
    PadMaster->>PadMaster: 24. 主考官开始观察评分
    PadSub1->>PadSub1: 25. 副考官1开始观察评分
    PadSub2->>PadSub2: 26. 副考官2开始观察评分

    par 考试进行中
        Candidate->>Candidate: 27. 执行考试任务
        PadMaster->>PadMaster: 28. 记录评分项
        PadSub1->>PadSub1: 29. 记录评分项
        PadSub2->>PadSub2: 30. 记录评分项
    end

    Note over Station: 阶段4：考试结束与评分提交
    Candidate->>Candidate: 31. 考试时间结束
    Candidate->>Station: 32. 考试完成通知
    Station->>PadMaster: 33. 进入评分阶段指令
    Station->>PadSub1: 34. 进入评分阶段指令
    Station->>PadSub2: 35. 进入评分阶段指令

    par 多考官评分
        PadMaster->>Scoring: 36. 提交主考官评分\n考官ID:1, 评分项, 分数
        PadSub1->>Scoring: 37. 提交副考官1评分\n考官ID:2, 评分项, 分数
        PadSub2->>Scoring: 38. 提交副考官2评分\n考官ID:3, 评分项, 分数
    end

    Scoring->>Consensus: 39. 检查评分一致性\n考官1 vs 考官2 vs 考官3

    alt 评分一致
        Consensus-->>Scoring: 40. 评分一致，计算平均分
    else 评分不一致
        Consensus-->>Scoring: 41. 评分差异超过阈值\n触发协同讨论
        Scoring->>MQ: 42. 发布评分不一致事件
        MQ->>PadMaster: 43. 通知评分差异
        MQ->>PadSub1: 44. 通知评分差异
        MQ->>PadSub2: 45. 通知评分差异
        PadMaster->>PadSub1: 46. 发起在线讨论
        PadMaster->>PadSub2: 47. 发起在线讨论
        PadSub1-->>PadMaster: 48. 确认讨论结果
        PadSub2-->>PadMaster: 49. 确认讨论结果
        PadMaster->>Scoring: 50. 提交最终协商评分
    end

    Note over Station: 阶段5：评分完成与调度确认
    Scoring->>MQ: 51. 发布评分完成事件\n考生ID, 考站ID, 最终分数
    MQ->>Scheduler: 52. 评分完成事件处理
    Scheduler->>Scheduler: 53. 更新全局考试进度\n考生状态:已完成
    Scheduler->>Station: 54. 考站任务完成确认
    Scheduler->>MQ: 55. 发布考生离站事件
    MQ->>Candidate: 56. 推送离站指令\n下一考站信息
    Station->>Station: 57. 考站状态复位\n准备接待下一位考生
```
### 2. 关键流程说明
2.1 多考官协同调度特点

并行通知机制：调度器通过消息队列同时通知考生和所有考官

就绪状态验证：考站需等待所有考官确认就绪才能开始考试

同步开始指令：考站向所有参与方发送同步的开始指令

2.2 多考官评分流程

独立评分提交：各考官独立观察并提交评分

一致性检查：评分服务自动检查多考官评分一致性

协同解决机制：评分不一致时触发在线讨论流程

最终评分计算：基于协同结果计算最终成绩

2.3 状态流转控制

考生状态：待考 → 在考 → 评分中 → 完成

考站状态：就绪 → 占用 → 评分中 → 可用

考官状态：就绪 → 评分中 → 完成

### 3. 核心消息事件
| 序号 | 事件 | 触发方 | 接收方 | 作用 |
|------|------|--------|--------|------|
| 4 | 考生分配事件 | 调度器 | 消息队列 | 启动考站调度流程 |
| 19-22 | 考试开始指令 | 考站 | 考生/考官 | 同步开始考试 |
| 51 | 评分完成事件 | 评分服务 | 消息队列 | 结束当前考站任务 |
| 55 | 考生离站事件 | 调度器 | 消息队列 | 触发轮转调度 |
### 4. 异常处理流程
```mermaid
sequenceDiagram
    participant Station as 考站A
    participant Scheduler as 调度器
    participant MQ as 消息队列
    participant Pad as 考官PAD
    participant Retry as 重试服务

    Note over Station,Scheduler: 场景：考官设备离线
    Station->>Pad: 1. 发送开始指令
    Note right of Pad: 网络异常/设备离线
    Station-->>Pad: 2. 指令发送失败
    Station->>MQ: 3. 发布设备异常事件
    MQ->>Scheduler: 4. 设备异常事件处理
    Scheduler->>Scheduler: 5. 标记设备异常状态
    
    alt 有备用设备
        Scheduler->>Station: 6. 分配备用设备
        Station->>MQ: 7. 发送设备切换指令
        MQ->>Pad: 8. 切换至备用设备
    else 无备用设备
        Scheduler->>Scheduler: 9. 计算异常处理方案
        alt 启用单考官模式
            Scheduler->>Station: 10. 启用单考官模式<br/>主考官继续评分
        else 延迟考试
            Scheduler->>Station: 11. 暂停当前考试
            Scheduler->>Retry: 12. 生成重试任务
            Retry->>MQ: 13. 发布延迟考试事件
        end
    end
```

### 5. 调度器状态管理
```mermaid
stateDiagram-v2
    [*] --> 空闲
    空闲 --> 调度中: 接收调度请求
    调度中 --> 分配考生: 找到合适考生
    分配考生 --> 等待就绪: 推送考生信息
    等待就绪 --> 考试中: 所有考官就绪
    考试中 --> 评分中: 考试时间结束
    评分中 --> 评分完成: 所有评分提交
    评分完成 --> 空闲: 考生离站
    评分完成 --> 异常: 评分不一致/超时
    异常 --> 空闲: 异常处理完成
    异常 --> 等待就绪: 重新分配考生
```

## 7. 总结
本OSCE系统设计以**调度引擎**和**事件驱动**为核心，通过**多考官-设备绑定机制**确保评规范，利用**分层容错**和**重考机制**保障考试流程的鲁棒性。采用**分布式架构**支持高并发场景，并通过清晰的服务划分与状态机设计，构建了一个灵活、可靠、可扩展的数字化临床技能考核平台。
