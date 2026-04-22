# 一、OSCE系统总体架构
🎯 “以调度引擎为核心，基于事件驱动的OSCE分布式考试系统”
核心三层：
接入层 → 业务服务层 → 调度与事件层 → 数据层

# 二、完整系统架构图
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

# 三、架构关键设计
## 1、调度引擎 = 系统大脑
职责：
轮转调度（跨站）
多考站并行控制
补考插入
异常处理触发

👉 本质：Scheduler = 状态机 + 队列 + 时间驱动
## 多考官多设备绑定
核心抽象：

考站
 ├── 考官序号 (1,2,3)
 └── 设备序号 (1,2,3)

绑定关系：
ExaminerIndex ↔ DeviceIndex

👉 强约束：登录必须“双因子匹配”（人 + 设备）

## 评分协同
支持三种模式：
| 模式   | 特点      |
| ---- | ------- |
| 独立评分 | 自动汇总    |
| 主副考官 | 权重计算    |
| 共识评分 | 冲突→协同解决 |

## 事件驱动（MQ是命脉）
所有核心动作都必须事件化：

考生入场
轮转调度
评分提交
考试开始/结束

👉 原则：

❌ 不允许“直接调用驱动流程”
✅ 必须通过 MQ 解耦

## 延迟容错
分三层：

✅ 设备层
心跳检测
自动重连
✅ 业务层
超时锁评分
标记异常
✅ 系统层
Retry补偿服务
幂等机制

## 重考机制
异常考生 → Retry收集 → ExamMgr生成补考计划 → Scheduler重新编排
👉 支持：插队、独立批次、单站补考

## 数据一致性
实时一致（弱一致）
MQ广播状态
最终一致（强保证）
评分入库
补偿重试

👉 技术手段：版本号、幂等ID、去重机制

# 🧠 图一：考生全生命周期状态机
```mermaid
stateDiagram-v2
    [*] --> 待考

    待考 --> 候考: 完成签到/入场
    候考 --> 在站: 调度分配考站
    
    在站 --> 评分中: 考试开始
    评分中 --> 已完成: 正常提交评分
    
    %% 异常分支
    在站 --> 异常: 设备故障/考官缺失
    评分中 --> 异常: 超时/中断
    
    异常 --> 补考待排: 标记异常考生
    
    补考待排 --> 补考中: 插入补考调度
    补考中 --> 已完成: 补考完成
    
    %% 终态
    已完成 --> [*]

    %% 强制结束
    在站 --> 已完成: 强制结束
    评分中 --> 已完成: 自动收卷
```

# 高并发分布式调度架构
```mermaid
graph TD

    %% 接入层
    subgraph 接入层
        PAD1[PAD集群]
        ADMIN[管理后台]
    end

    %% 网关
    GATEWAY[API网关]

    %% 调度集群
    subgraph 调度集群
        SCHED1[调度器1]
        SCHED2[调度器2]
        SCHED3[调度器3]
    end

    %% 分片规则
    SHARD[调度分片规则<br>按考场/楼层/场次]

    %% MQ
    subgraph MQ集群
        MQ1[Partition1]
        MQ2[Partition2]
        MQ3[Partition3]
    end

    %% 业务服务
    STATION[考站服务集群]
    SCORE[评分服务集群]

    %% 缓存 & DB
    REDIS[(Redis集群)]
    DB[(分库分表DB)]

    %% 容错
    RETRY[补偿服务]

    %% 连接关系
    PAD1 --> GATEWAY
    ADMIN --> GATEWAY

    GATEWAY --> SCHED1
    GATEWAY --> SCHED2
    GATEWAY --> SCHED3

    SCHED1 --> SHARD
    SCHED2 --> SHARD
    SCHED3 --> SHARD

    SHARD --> MQ1
    SHARD --> MQ2
    SHARD --> MQ3

    MQ1 --> STATION
    MQ2 --> STATION
    MQ3 --> STATION

    STATION --> SCORE

    SCORE --> MQ1
    SCORE --> MQ2
    SCORE --> MQ3

    STATION --> REDIS
    SCORE --> REDIS

    REDIS --> DB

    MQ1 --> RETRY
    MQ2 --> RETRY
    MQ3 --> RETRY
```

# 考站全生命周期状态机
```mermaid
stateDiagram-v2
    [*] --> 初始化

    初始化 --> 就绪: 设备完成认证 / 考官到位
    就绪 --> 待命: 等待调度指令

    待命 --> 分配考生: 接收调度(考生+轮次)
    分配考生 --> 候考中: 考生进入考站/等待开始

    候考中 --> 考试中: 到达开始时间 / 收到开始指令
    考试中 --> 评分中: 考试结束 / 进入评分阶段

    评分中 --> 已完成: 所有评分提交完成

    %% 正常轮转
    已完成 --> 待命: 等待下一轮调度

    %% 异常分支
    就绪 --> 异常: 设备故障 / 考官缺席
    待命 --> 异常: 调度异常
    候考中 --> 异常: 考生缺考
    考试中 --> 异常: 中断/掉线
    评分中 --> 异常: 超时未提交

    %% 异常恢复
    异常 --> 恢复中: 重试/人工介入
    恢复中 --> 待命: 恢复成功

    %% 不可恢复
    异常 --> 关闭: 严重故障

    %% 考试结束
    待命 --> 已关闭: 收到考试结束指令
    已完成 --> 已关闭: 最后一轮结束

    已关闭 --> [*]
```
