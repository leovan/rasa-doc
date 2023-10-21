# 事件

Rasa 中的对话被表示为一系列事件。自定义动作可以通过在对动作服务器请求的响应中返回事件来影响对话过程。

并非所有事件通常都由自定义动作返回，因为它们是由 Rasa 自动追踪（例如用户消息）。其他事件只有在自定义动作返回时才能被追踪。

## 事件类型 {#event-types}

### `slot` {#slot}

在追踪器上设置一个槽。它可以将槽设置为一个值，或者通过将槽的值设置为 `null` 来重置槽。

自动追踪：

- 当槽被同名实体填充时。

需要自定义动作来设置任何不由实体自动填充的槽。

JSON：

```json
{
    "event": "slot",
    "name": "departure_airport",
    "value": "BER"
}
```

参数：

- `name`：要设置的槽的名称。
- `value`：将槽设置为的值。数据类型必须与槽的[类型](../domain.md#slot-types)匹配。

Rasa 类：`rasa.core.events.SlotSet`

### `reset_slots` {#reset_slots}

将追踪器上的所有槽充值为 `null`。

自动追踪：从不

JSON：

```json
{
    "event": "reset_slots"
}
```

Rasa 类：`rasa.core.events.AllSlotsReset`

### `reminder` {#reminder}

安排在未来某个时间触发的意图。

自动追踪：从不

JSON：

```json
{
  "event": "reminder",
  "intent": "my_intent",
  "entities": {"entity1": "value1", "entity2": "value2"},
  "date_time": "2018-09-03T11:41:10.128172",
  "name": "my_reminder",
  "kill_on_user_msg": true,
}
```

参数：

- `intent`：提醒器将触发的意图
- `entities`：根据意图发送的实体
- `date_time`：应触发动作执行的日期。应为 UTC 或包含时区。
- `name`：提醒器的 ID。如果有多个具有相同 ID 的提醒器，则只会运行最后一个。
- `kill_on_user_msg`：用户消息是否会在触发时间之前中止提醒。

Rasa 类：`rasa.core.events.ReminderScheduled`

### `cancel_reminder` {#cancel_reminder}

取消预定的提醒器。与提供的参数匹配的所有提醒器都将被取消。

自动追踪：从不

JSON：

```json
{
  "event": "cancel_reminder",
  "name": "my_reminder",
  "intent": "my_intent",
  "entities": [
        {"entity": "entity1", "value": "value1"},
        {"entity": "entity2", "value": "value2"},
    ],
  "date_time": "2018-09-03T11:41:10.128172",
}
```

参数：

- `intent`：提醒器将触发的意图
- `entities`：根据意图发送的实体
- `date_time`：应触发动作执行的日期。应为 UTC 或包含时区。
- `name`：提醒器的 ID。如果有多个具有相同 ID 的提醒器，则只会运行最后一个。

### `pause` {#pause}

阻止对话机器人响应用户消息。对话将保持暂停状态，并且在明确[恢复](#resume)对话之前不会预测任何动作。

自动追踪：从不

JSON：

```json
{
    "event": "pause"
}
```

Rasa 类：`rasa.core.events.ConversationPaused`

### `resume` {#resume}

恢复之前暂停的对话。将此事件添加到追踪器后，对话机器人将再次开始预测动作。它不会预测对话暂停时收到的用户消息的动作。

自动追踪：从不

JSON：

```json
{
    "event": "resume"
}
```

Rasa 类：`rasa.core.events.ConversationResumed`

### `followup` {#followup}

强制采取后续动作，绕过动作预测。

自动追踪：从不

JSON：

```json
{
    "event": "followup",
    "name": "my_action"
}
```

参数：

- `name`：将执行的后续动作名称。

Rasa 类：`rasa.core.events.FollowupAction`

### `rewind` {#rewind}

恢复最后一个用户消息的所有副作用，并从追踪器中删除最后一个用户事件。

自动追踪：从不

JSON：

```json
{
    "event": "rewind"
}
```

Rasa 类：`rasa.core.events.UserUtteranceReverted`

### `undo` {#undo}

撤销最后一个机器人动作的所有副作用，并从追踪器中删除最后一个对话机器人动作。

自动追踪：从不

JSON：

```json
{
    "event": "undo"
}
```

Rasa 类：`rasa.core.events.ActionReverted`

### `restart` {#restart}

重置追踪器。`restart` 事件后，将没有对话历史记录和重启记录。

自动追踪：

- 当 `/restart` 默认意图被触发时。

JSON：

```json
{
    "event": "restart"
}
```

Rasa 类：`rasa.core.events.Restarted`

### `session_started` {#session_started}

通过重置追踪器并运行默认动作 `ActionSessionStart` 来开始新的对话。默认情况下，此动作会将现有的 `SlotSet` 事件转移到新的对话会话中。你可以在领域文件中 `session_config` 下配置此行为。

自动追踪：

- 每当用户第一次开始与对话机器人对话时。
- 每当会话过期的（在领域中指定 `session_expiration_time` 之后），用户恢复他们的对话。

使用 [`restart`](#restart) 事件重新启动对话不会自动导致 `session_started` 事件。

JSON：

```json
{
    "event": "session_started"
}
```

Rasa 类：`rasa.core.events.SessionStarted`

### `user` {#user}

用户向机器人发送一条消息。

自动追踪：

- 当用户向机器人发送消息时。

此事件通常不会由自定义动作返回。

JSON：

```json
{
    "event": "user",
    "text": "Hey",
    "parse_data": {
        "intent": {
            "name": "greet",
            "confidence": 0.9
        },
        "entities": []
    },
    "metadata": {},
}
```

参数：

- `text`：用户消息的文本
- `parse_data`：用户消息的解析数据。这通常由 NLU 填充。
- `metadata`：用户消息附带的任意元数据

Rasa 类：`rasa.core.events.UserUttered`

### `bot` {#bot}

对话机器人向用户发送的一条消息。

自动追踪：

- 每当自定义动作返回响应时
- 每当响应直接发送给用户而不是由自定义动作返回时（例如 `utter_actions`）

此事件通常不会由自定义动作显式返回，而是返回响应。

JSON：

```json
{
    "event": "bot",
    "text": "Hey there!",
    "data": {}
}
```

参数：

- `text`：对话机器人发送给用户的文本
- `data`：对话机器人响应的任何非文本元素。数据结构与 [API 规范](https://rasa.com/docs/rasa/pages/action-server-api/){:target="_blank"}中给出的 `responses` 结构相匹配。

Rasa 类：`rasa.core.events.BotUttered`

### `action` {#action}

记录对话机器人调用的动作。仅记录动作本身，动作创建的事件在应用时会单独记录。

自动追踪：

- 调用任何动作（包括自定义动作和响应），即使该动作未成功执行。

此事件通常不会由自定义动作显式返回。

JSON：

```json
{
    "event": "action",
    "name": "my_action"
}
```

参数：

- `name`：被调用的动作名称

Rasa 类：`rasa.core.events.ActionExecuted`
