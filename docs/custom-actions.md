# 自定义动作

自定义动作可以运行所需的任何代码，包括 API 调用、数据库查询等。它们可以开灯、将事件添加到日历中、检查用户的银行余额或能想象到的任何其他内容。

有关如何实现自定义动作的详细信息，请参见 [SDK 文档](/action-server/running-action-server)。想在故事中使用的任何自定义动作都应该添加到[领域](/domain)的 `actions` 部分中。

当对话引擎预测要执行的自定义动作时，它将调用动作服务，并带有如下信息：

```json
{
  "next_action": "string",
  "sender_id": "string",
  "tracker": {
    "conversation_id": "default",
    "slots": {},
    "latest_message": {},
    "latest_event_time": 1537645578.314389,
    "followup_action": "string",
    "paused": false,
    "events": [],
    "latest_input_channel": "rest",
    "active_loop": {},
    "latest_action": {},
  },
"domain": {
    "config": {},
    "session_config": {},
    "intents": [],
    "entities": [],
    "slots": {},
    "responses": {},
    "actions": [],
    "forms": {},
    "e2e_actions": []
  },
  "version": "version"
}
```

你的动作服务应该使用事件和响应列表进行响应：

```json
{
  "events": [{}],
  "responses": [{}]
}
```
