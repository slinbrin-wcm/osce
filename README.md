# osce
osce考试系统
一个完整的OSCE（客观结构化临床考试）考核系统通常包含以下核心服务模块：

1. 用户与权限管理服务

用户服务：考生、考官、管理员账号管理

认证服务：登录认证、单点登录、设备认证

权限服务：角色权限控制、操作权限验证

会话服务：登录状态管理、设备绑定管理

2. 考试编排服务

考试计划服务：考试场次安排、时间表管理

考站编排服务：考站顺序、轮转规则、时间控制

考生分配服务：考生分组、考站分配、轮转调度

考试模板服务：标准化考试模板管理

3. 考站执行服务

考站任务服务：当前考站状态、考生信息、倒计时

评分表服务：评分标准、评分项、权重配置

计时服务：考站倒计时、考试总计时、超时处理

多媒体服务：视频录制、音频采集、文件上传

4. 评分与评估服务

评分采集服务：实时评分提交、评分项验证

评分计算服务：自动计分、权重计算、总分统计

评估分析服务：成绩分析、通过率统计、能力评估

成绩单服务：成绩单生成、打印、导出

5. 设备与终端服务

设备管理服务：PAD设备注册、绑定、状态监控

终端应用服务：考官PAD应用、考生终端应用

设备同步服务：数据同步、状态同步、配置同步

离线缓存服务：网络异常时的本地缓存

6. 事件与消息服务

消息中间件：事件发布/订阅、实时通知

事件总线服务：系统事件路由、事件处理

通知服务：推送通知、短信通知、邮件通知

日志服务：操作日志、系统日志、审计日志

7. 数据与同步服务

数据同步服务：多端数据同步、冲突解决

数据归档服务：考试数据归档、历史数据管理

备份服务：数据备份、恢复

报表服务：统计报表、分析报表

8. 监控与管理服务

系统监控服务：服务健康检查、性能监控

异常告警服务：系统异常告警、业务异常告警

配置管理服务：系统配置、业务参数配置

运维管理服务：系统维护、数据清理

9. 第三方集成服务

考生信息集成：与教务系统、医院HIS系统对接

身份认证集成：与统一身份认证平台对接

视频监控集成：与监控系统对接

成绩发布集成：与成绩管理系统对接

10. 安全与合规服务

数据加密服务：数据传输加密、存储加密

访问控制服务：IP白名单、访问频率控制

合规审计服务：操作审计、数据变更审计

隐私保护服务：考生隐私数据保护

# OSCE考核系统多考官多设备架构设计

针对一个考站分配多考官和多设备的情况，需要建立考官序号与设备序号的精确对应关系。以下是完整的服务架构和交互流程：

核心服务模块
1. 考站编排服务

考站配置模块：定义考站考官数量、角色（主考/副考）

考官分配模块：分配具体考官到考站，指定考官序号

设备分配模块：分配设备到考站，指定设备序号

绑定关系管理：建立考官序号↔设备序号的映射关系

2. 设备管理服务

设备注册模块：设备唯一标识、设备类型、设备能力

设备分组模块：按考站分组设备，分配设备序号

设备状态监控：实时监控设备在线状态、电量、网络

设备绑定验证：验证设备与考官的绑定关系

3. 考官管理服务

考官信息管理：考官资质、专业领域、角色权限

考官考站分配：分配考官到具体考站，指定考官序号

考官设备绑定：建立考官与设备的动态绑定关系

考官状态跟踪：考官登录状态、评分状态、位置状态

4. 评分协同服务

评分任务分配：将评分项分配给不同考官

评分一致性校验：多考官评分的一致性检查

评分权重计算：不同考官评分的权重分配

最终成绩合成：综合多考官评分计算最终成绩

5. 实时协同服务

状态同步模块：多设备间状态实时同步

事件广播模块：考试事件向所有设备广播

冲突解决模块：处理多设备操作冲突

协同控制模块：主考官对副考官的控制权限

考官序号与设备序号绑定机制
绑定关系数据结构

```mermaid
graph TD
    A[开始考试配置] --> B[考试基础信息]
    
    subgraph "核心业务配置"
        B --> C[场次与时段结构]
        C --> D[考试方式与规则]
        D --> E[考试安全配置]
        E --> F[考试时钟配置]
        F --> G[调度模式与策略]
        G --> H[语音叫号配置]
        H --> I[技能考题型配置]
        I --> J[项目依赖关系配置]
        J --> K[多场次调度配置]
    end
    
    K --> L[项目配置1:参考对象]
    L --> M[项目配置2:考站资源]
    M --> N[配置考站:考官/SP资源]
    
    N --> O{基础配置验证}
    O -->|通过| P[安排考生名单]
    O -->|不通过| Q[返回修改配置]
    
    P --> R[排考及调整]
    R --> S{发布考试配置}
    
    S -->|发布成功| T[完成考试配置]
    
    T --> U[编辑路径指引<br>（可选，可随时配置）]
    
    style U stroke-dasharray: 5,5
```

```mermaid
sequenceDiagram
    participant ExamMgr as 考试管理服务(调度引擎)
    participant Scheduler as 调度器(轮转核心)
    participant StationA as 考站A
    participant StationB as 考站B
    participant PAD_A as PAD(A)
    participant PAD_B as PAD(B)
    participant Score as 评分服务
    participant MQ as 消息中间件
    participant Retry as 重试/补偿服务

    Note over ExamMgr, MQ: 阶段1：初始化（多考站并行启动）
    ExamMgr->>Scheduler: 加载排考规则(考生队列+考站矩阵)
    Scheduler->>MQ: 发布初始轮转计划(多考站并行)
    
    MQ->>StationA: 分配考生A1
    MQ->>StationB: 分配考生B1
    
    StationA->>PAD_A: 下发考生+评分表
    StationB->>PAD_B: 下发考生+评分表

    Note over ExamMgr, MQ: 阶段2：跨站轮转（核心循环）
    loop 每一轮轮转（全局调度）
        Scheduler->>MQ: 发布轮转指令(跨站流转)
        
        MQ->>StationA: A1 → 下一站(B)
        MQ->>StationB: B1 → 下一站(A)
        
        StationA->>MQ: 发布考生离站事件
        StationB->>MQ: 发布考生离站事件
        
        MQ->>StationA: 推送新考生
        MQ->>StationB: 推送新考生
        
        StationA->>PAD_A: 更新评分表
        StationB->>PAD_B: 更新评分表
    end

    Note over PAD_A, Score: 阶段3：评分（异步解耦）
    PAD_A->>Score: 提交评分
    PAD_B->>Score: 提交评分
    Score->>MQ: 发布评分完成事件
    MQ->>Scheduler: 更新整体进度

    Note over PAD_A, Retry: 阶段4：延迟容错（掉线/超时）
    alt PAD掉线 / 网络异常
        PAD_A--xMQ: 心跳丢失
        
        MQ->>Scheduler: 上报异常
        Scheduler->>Retry: 标记异常考站/考生
        
        Retry->>MQ: 发布重试任务(重新下发评分表)
        MQ->>PAD_A: 恢复任务
        
    else 超时未评分
        Scheduler->>Retry: 标记超时
        
        Retry->>Score: 锁定当前评分状态
        Retry->>MQ: 发布补偿任务
    end

    Note over ExamMgr, Retry: 阶段5：补考机制（关键）
    Scheduler->>Retry: 汇总异常考生列表
    
    Retry->>ExamMgr: 生成补考名单
    ExamMgr->>Scheduler: 插入补考队列(低优先级或独立批次)
    
    Scheduler->>MQ: 发布补考调度
    
    MQ->>StationA: 分配补考考生
    StationA->>PAD_A: 下发补考任务

    Note over ExamMgr, MQ: 阶段6：收敛与结束
    Scheduler->>MQ: 发布结束信号
    
    MQ->>StationA: 停止接收任务
    MQ->>StationB: 停止接收任务
    
    Score->>ExamMgr: 汇总最终成绩
```

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

    Note over Admin, Binding: 阶段1：管理员配置多考官考站
    Admin->>+StationMgr: 1. 创建多考官考站<br/>考站ID:915, 考官数:2
    StationMgr-->>-Admin: 2. 考站创建成功
    
    Admin->>+ExaminerMgr: 3. 分配考官到考站<br/>考官A(序号1), 考官B(序号2)
    ExaminerMgr-->>-Admin: 4. 考官分配成功
    
    Admin->>+DevMgr: 5. 分配设备到考站<br/>设备PAD-001(序号1), PAD-002(序号2)
    DevMgr-->>-Admin: 6. 设备分配成功
    
    Admin->>+Binding: 7. 建立绑定关系<br/>考官1↔设备1, 考官2↔设备2
    Binding-->>-Admin: 8. 绑定关系建立成功

    Note over PAD1, Binding: 阶段2：设备启动与绑定验证
    par 设备并行启动
        PAD1->>+DevMgr: 9. 设备启动注册<br/>设备ID:PAD-001
        DevMgr->>Binding: 10. 查询设备绑定关系
        Binding-->>DevMgr: 11. 返回绑定信息<br/>考站:915, 设备序号:1, 考官序号:1
        DevMgr-->>-PAD1: 12. 返回设备凭证与绑定信息
        
        PAD2->>+DevMgr: 13. 设备启动注册<br/>设备ID:PAD-002
        DevMgr->>Binding: 14. 查询设备绑定关系
        Binding-->>DevMgr: 15. 返回绑定信息<br/>考站:915, 设备序号:2, 考官序号:2
        DevMgr-->>-PAD2: 16. 返回设备凭证与绑定信息
    end

    Note over PAD1, Binding: 阶段3：考官登录与双重验证
    par 考官并行登录
        PAD1->>+Auth: 17. 考官A登录<br/>账号密码 + 设备凭证
        Auth->>ExaminerMgr: 18. 验证考官考站分配
        ExaminerMgr-->>Auth: 19. 考官A分配在考站915, 序号1
        Auth->>Binding: 20. 验证设备绑定关系
        Binding-->>Auth: 21. 设备PAD-001绑定考官A, 序号匹配
        Auth-->>-PAD1: 22. 登录成功<br/>返回考官信息与角色
        
        PAD2->>+Auth: 23. 考官B登录<br/>账号密码 + 设备凭证
        Auth->>ExaminerMgr: 24. 验证考官考站分配
        ExaminerMgr-->>Auth: 25. 考官B分配在考站915, 序号2
        Auth->>Binding: 26. 验证设备绑定关系
        Binding-->>Auth: 27. 设备PAD-002绑定考官B, 序号匹配
        Auth-->>-PAD2: 28. 登录成功<br/>返回考官信息与角色
    end

    Note over PAD1, Binding: 阶段4：绑定异常处理场景
    rect rgb(255, 200, 200)
        Note over PAD1, PAD2: 场景：考官A尝试在设备2登录
        PAD2->>Auth: 29. 考官A在设备2登录尝试
        Auth->>ExaminerMgr: 30. 验证考官考站分配
        ExaminerMgr-->>Auth: 31. 考官A分配在考站915, 序号1
        Auth->>Binding: 32. 验证设备绑定关系
        Binding-->>Auth: 33. 设备PAD-002绑定考官B, 不匹配考官A
        
        alt 自动重新绑定
            Auth->>+Admin: 34. 请求重新绑定<br/>考官A↔设备2
            Admin->>Binding: 35. 更新绑定关系
            Binding->>PAD1: 36. 通知原设备解绑
            PAD1-->>PAD1: 37. 清理本地会话
            Binding-->>Admin: 38. 绑定更新成功
            Admin-->>Auth: 39. 重新绑定批准
            Auth-->>PAD2: 40. 登录成功(新绑定)
        else 拒绝重新绑定
            Auth-->>PAD2: 41. 登录失败:设备不匹配<br/>请使用指定设备登录
        end
    end

    Note over PAD1, Binding: 阶段5：考试协同执行
    par 多考官协同评分
        PAD1->>StationMgr: 42. 获取考站任务<br/>考官序号:1
        StationMgr-->>PAD1: 43. 返回考站详情与评分表
        
        PAD2->>StationMgr: 44. 获取考站任务<br/>考官序号:2
        StationMgr-->>PAD2: 45. 返回考站详情与评分表
    end
    
    loop 考生评分过程
        Note over PAD1, PAD2: 考生入场，开始评分
        PAD1->>Binding: 46. 开始评分<br/>考官1准备就绪
        PAD2->>Binding: 47. 开始评分<br/>考官2准备就绪
        
        par 独立评分
            PAD1->>Scoring: 48. 提交评分项1<br/>考官序号:1
            PAD2->>Scoring: 49. 提交评分项2<br/>考官序号:2
        end
        
        Scoring->>Consensus: 50. 检查评分一致性
        Consensus-->>Scoring: 51. 一致性检查结果
        
        alt 评分一致
            Scoring->>StationMgr: 52. 更新考生评分状态
        else 评分不一致
            Scoring->>PAD1: 53. 通知评分差异
            Scoring->>PAD2: 54. 通知评分差异
            PAD1->>PAD2: 55. 发起协同讨论
            PAD2-->>PAD1: 56. 确认讨论结果
            PAD1->>Scoring: 57. 提交最终一致评分
        end
    end
```

```mermaid
sequenceDiagram
    participant ExamMgr as 考试管理服务(调度引擎)
    participant Scheduler as 调度器(轮转核心)
    participant StationA as 考站A
    participant StationB as 考站B
    participant PAD_A as PAD(A)
    participant PAD_B as PAD(B)
    participant Score as 评分服务
    participant MQ as 消息中间件
    participant Retry as 重试/补偿服务

    Note over ExamMgr, MQ: 阶段1：初始化（多考站并行启动）
    ExamMgr->>Scheduler: 加载排考规则(考生队列+考站矩阵)
    Scheduler->>MQ: 发布初始轮转计划(多考站并行)
    
    MQ->>StationA: 分配考生A1
    MQ->>StationB: 分配考生B1
    
    StationA->>PAD_A: 下发考生+评分表
    StationB->>PAD_B: 下发考生+评分表

    Note over ExamMgr, MQ: 阶段2：跨站轮转（核心循环）
    loop 每一轮轮转（全局调度）
        Scheduler->>MQ: 发布轮转指令(跨站流转)
        
        MQ->>StationA: A1 → 下一站(B)
        MQ->>StationB: B1 → 下一站(A)
        
        StationA->>MQ: 发布考生离站事件
        StationB->>MQ: 发布考生离站事件
        
        MQ->>StationA: 推送新考生
        MQ->>StationB: 推送新考生
        
        StationA->>PAD_A: 更新评分表
        StationB->>PAD_B: 更新评分表
    end

    Note over PAD_A, Score: 阶段3：评分（异步解耦）
    PAD_A->>Score: 提交评分
    PAD_B->>Score: 提交评分
    Score->>MQ: 发布评分完成事件
    MQ->>Scheduler: 更新整体进度

    Note over PAD_A, Retry: 阶段4：延迟容错（掉线/超时）
    alt PAD掉线 / 网络异常
        PAD_A--xMQ: 心跳丢失
        
        MQ->>Scheduler: 上报异常
        Scheduler->>Retry: 标记异常考站/考生
        
        Retry->>MQ: 发布重试任务(重新下发评分表)
        MQ->>PAD_A: 恢复任务
        
    else 超时未评分
        Scheduler->>Retry: 标记超时
        
        Retry->>Score: 锁定当前评分状态
        Retry->>MQ: 发布补偿任务
    end

    Note over ExamMgr, Retry: 阶段5：补考机制（关键）
    Scheduler->>Retry: 汇总异常考生列表
    
    Retry->>ExamMgr: 生成补考名单
    ExamMgr->>Scheduler: 插入补考队列(低优先级或独立批次)
    
    Scheduler->>MQ: 发布补考调度
    
    MQ->>StationA: 分配补考考生
    StationA->>PAD_A: 下发补考任务

    Note over ExamMgr, MQ: 阶段6：收敛与结束
    Scheduler->>MQ: 发布结束信号
    
    MQ->>StationA: 停止接收任务
    MQ->>StationB: 停止接收任务
    
    Score->>ExamMgr: 汇总最终成绩
```

## 多考官协同评分模式
1. 独立评分模式:考官1评分 → 考官2评分 → 系统自动计算平均分
2. 主副考官模式:主考官评分(权重70%) + 副考官评分(权重30%) = 最终成绩
3. 共识评分模式:考官1评分 → 考官2评分 → 差异超过阈值 → 考后由管理员在成绩编辑中修订

设备故障处理流程

设备离线检测：监控服务发现设备离线

自动切换：将考官切换到备用设备

数据同步：将原设备数据同步到新设备

通知管理员：报告设备故障，需要维护

数据一致性保证
1. 实时同步机制

操作日志同步：所有评分操作实时同步到所有设备

状态广播：考生状态、计时状态实时广播

冲突检测：检测多设备同时操作同一数据

2. 最终一致性策略

本地优先：网络异常时本地操作优先

异步同步：网络恢复后异步同步数据

版本控制：使用版本号解决数据冲突

人工干预：无法自动解决的冲突提示人工处理
