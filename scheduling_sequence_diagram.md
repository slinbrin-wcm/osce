# 考站多考官协同评分时序图

```sequence
Alice->Bob: 提交作业
Bob->Alice: 确认收到
Bob->Charlie: 转发作业
Charlie->Bob: 确认收到
Charlie->David: 请求反馈
David->Charlie: 提供反馈
Charlie->Bob: 返回反馈
Bob->Alice: 提供最终评分
```