sequenceDiagram
    participant Admin as 管理员后台
    participant PAD as 考站PAD<br>(固定设备)
    participant Auth as 用户认证服务
    participant DevMgr as 设备管理服务
    participant ExamMgr as 考试管理服务
    participant Station as 考站任务服务
    participant Score as 评分管理服务
    participant MQ as 消息中间件
    participant Sync as 数据同步服务

    Note over Admin, Sync: 阶段0：管理员配置设备与考站绑定
    Admin->>+DevMgr: 1. 配置设备绑定关系<br>(设备ID:Pad-001, 考站:915)
    DevMgr-->>-Admin: 2. 绑定成功
    
    DevMgr->>Station: 3. 同步绑定关系<br>(设备Pad-001→考站915)
    DevMgr->>ExamMgr: 4. 同步设备考站映射

    Note over PAD, Sync: 阶段1：设备启动与自动认证
    PAD->>+DevMgr: 5. 设备启动，注册设备信息
    DevMgr->>DevMgr: 6. 验证设备合法性<br>检查绑定状态
    DevMgr-->>-PAD: 7. 返回设备凭证(DeviceToken)<br>及绑定的考站ID(915)
    
    PAD->>+Auth: 8. 无感登录<br>(DeviceToken)
    Auth->>DevMgr: 9. 验证设备凭证与绑定关系
    DevMgr-->>Auth: 10. 返回绑定的考官账号
    Auth-->>-PAD: 11. 自动登录成功<br>返回用户凭证与考官信息

    Note over PAD, Sync: 阶段2：获取考站任务与事件订阅
    PAD->>+Station: 12. 获取考站915任务<br>(考站ID来自设备绑定)
    Station->>ExamMgr: 13. 获取当前考试场次
    ExamMgr-->>Station: 14. 返回考试信息
    Station-->>-PAD: 15. 返回考站详情<br>当前考生、评分表、倒计时

    PAD->>MQ: 16. 订阅主题<br>(exam.*.station.915)
    MQ-->>PAD: 17. 订阅确认

    Note over PAD, Sync: 阶段3：考试开始与事件驱动
    ExamMgr->>MQ: 18. 发布"考试开始"事件
    MQ->>PAD: 19. 推送考试开始通知
    PAD->>Station: 20. 确认考站就绪
    Station->>MQ: 21. 发布"考生入场"事件
    MQ->>PAD: 22. 推送新考生信息

    Note over PAD, Sync: 阶段4：评分提交循环
    loop 对每个考生/每个评分项
        PAD->>+Score: 23. 提交评分
        Score->>MQ: 24. 发布"评分提交"事件
        MQ->>Station: 25. 更新考生状态
        MQ->>ExamMgr: 26. 更新考试进度
        Score-->>-PAD: 27. 提交成功
    end

    Note over PAD, Sync: 阶段5：考试结束与数据同步
    ExamMgr->>MQ: 28. 发布"考试结束"事件
    MQ->>PAD: 29. 推送考试结束通知
    MQ->>Station: 30. 停止考站任务
    MQ->>Score: 31. 停止评分接收
    
    Station->>+Sync: 32. 触发考站数据同步
    Score->>Sync: 33. 提交最终评分数据
    Sync->>ExamMgr: 34. 归档完整考试数据
    Sync-->>-Station: 35. 同步完成确认
    Sync-->>-PAD: 36. 数据同步完成
