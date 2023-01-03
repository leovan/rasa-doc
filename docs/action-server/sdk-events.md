# 事件

在内部，Rasa 对话表示为事件列表。Rasa SDK 为每个事件提供类，并负责将事件类的实例转换为格式正确的事件负载。

本页面是关于 `rasa_sdk` 中的事件类。无论使用 `rasa_sdk` 还是其他动作服务器，事件及其底层负载的副作用都是相同的。有关事件的副作用、其底层负载和 Rasa 中的类的详细信息，它被转换为查看所有动作服务器的事件文档（每个小结也有对应链接）。

!!! tip "重要事件"

    在 Rasa SDK 动作服务器中编写的所有事件都需要从 `rasa_sdk.events` 导入。

## 事件类 {#event-classes}

### SlotSet {#slotset}

```python
rasa_sdk.events.SlotSet(
    key: Text,
    value: Any = None,
    timestamp: Optional[float] = None
)
```

底层事件：[`slot`](/action-server/events/#slot)

参数：

- `key`：要设置的槽的名称。
- `value`：将槽设置为的值。数据类型必须与槽的类型匹配。
- `timestamp`：可选的事件时间戳。

示例：

```python
evt = SlotSet(key = "name", value = "Mary")
```

### AllSlotsReset {#allslotsreset}

```python
rasa_sdk.events.AllSlotsReset(timestamp: Optional[float] = None)
```

底层事件：[`reset_slots`](/action-server/events/#reset_slots)

参数：

- `timestamp`：可选的事件时间戳。

示例：

```python
evt = AllSlotsReset()
```

### ReminderScheduled {#reminderscheduled}

```python
rasa_sdk.events.ReminderScheduled(
    intent_name: Text,
    trigger_date_time: datetime.datetime,
    entities: Optional[Union[List[Dict[Text, Any]], Dict[Text, Text]]] = None,
    name: Optional[Text] = None,
    kill_on_user_message: bool = True,
    timestamp: Optional[float] = None,
)
```

底层事件：[`reminder`](/action-server/events/#reminder)

参数：

- `intent_name`：提醒器将触发的意图
- `trigger_date_time`：应触发动作执行的日期
- `entities`：根据意图发送的实体
- `name`：提醒器的 ID。如果有多个具有相同 ID 的提醒器，则只会运行最后一个。
- `kill_on_user_msg`：用户消息是否会在触发时间之前中止提醒。
- `timestamp`：可选的事件时间戳。

示例：

```python
from datetime import datetime

evt = ReminderScheduled(
    intent_name = "EXTERNAL_dry_plant",
    trigger_date_time = datetime(2020, 9, 15, 0, 36, 0, 851609),
    entities = [{"name": "plant","value":"orchid"}],
    name = "remind_water_plants",
)
```

### ReminderCancelled {#remindercancelled}

```python
ReminderCancelled(
    name: Optional[Text] = None,
    intent_name: Optional[Text] = None,
    entities: Optional[Union[List[Dict[Text, Any]], Dict[Text, Text]]] = None,
    timestamp: Optional[float] = None,
)
```

底层事件：[`cancel_reminder`](/action-server/events/#cancel_reminder)

参数：

- `name`：提醒器的 ID
- `intent_name`：提醒器将触发的意图
- `entities`：根据意图发送的实体
- `timestamp`：可选的事件时间戳。

示例：

```python
evt = ReminderCancelled(name = "remind_water_plants")
```

### ConversationPaused {#conversationpaused}

```python
ConversationPaused(timestamp: Optional[float] = None)
```

底层事件：[`pause`](/action-server/events/#pause)

参数：

- `timestamp`：可选的事件时间戳。

示例：

```python
evt = ConversationPaused()
```

### ConversationResumed {#conversationresumed}

```python
ConversationResumed(timestamp: Optional[float] = None)
```

底层事件：[`resume`](/action-server/events/#resume)

参数：

- `timestamp`：可选的事件时间戳。

示例：

```python
evt = ConversationResumed()
```

### FollowupAction {#followupaction}

```python
FollowupAction(
    name: Text,
    timestamp: Optional[float] = None
)
```

底层事件：[`followup`](/action-server/events/#followup)

参数：

- `name`：将执行的后续动作名称。
- `timestamp`：可选的事件时间戳。

示例：

```python
evt = FollowupAction(name = "action_say_goodbye")
```

### UserUtteranceReverted {#userutterancereverted}

```python
UserUtteranceReverted(timestamp: Optional[float] = None)
```

底层事件：[`rewind`](/action-server/events/#rewind)

参数：

- `timestamp`：可选的事件时间戳。

示例：

```python
evt = UserUtteranceReverted()
```

### ActionReverted {#actionreverted}

```python
ActionReverted(timestamp: Optional[float] = None)
```

底层事件：[`undo`](/action-server/events/#undo)

参数：

- `timestamp`：可选的事件时间戳。

示例：

```python
evt = Restarted()
```

### Restarted {#restarted}

```python
Restarted(timestamp: Optional[float] = None)
```

底层事件：[`restart`](/action-server/events/#restart)

参数：

- `timestamp`：可选的事件时间戳。

示例：

```python
evt = Restarted()
```

### SessionStarted {#sessionstarted}

```python
SessionStarted(timestamp: Optional[float] = None)
```

底层事件：[`session_started`](/action-server/events/#session_started)

参数：

- `timestamp`：可选的事件时间戳。

示例：

```python
evt = SessionStarted()
```

### UserUttered {#useruttered}

```python
UserUttered(
    text: Optional[Text],
    parse_data: Optional[Dict[Text, Any]] = None,
    timestamp: Optional[float] = None,
    input_channel: Optional[Text] = None,
)
```

底层事件：[`user`](/action-server/events/#user)

参数：

- `text`：用户消息的文本
- `parse_data`：用户消息的解析数据。这通常由 NLU 填充。
- `input_channel`：接受消息的频道
- `timestamp`：可选的事件时间戳。

示例：

```python
evt = UserUttered(text = "Hallo bot")
```

### BotUttered {#botuttered}

```python
BotUttered(
    text: Optional[Text] = None,
    data: Optional[Dict[Text, Any]] = None,
    metadata: Optional[Dict[Text, Any]] = None,
    timestamp: Optional[float] = None,
)
```

底层事件：[`bot`](/action-server/events/#bot)

参数：

- `text`：对话机器人发送给用户的文本
- `data`：对话机器人响应的任何非文本元素。数据结构与 [API 规范](https://rasa.com/docs/rasa/pages/action-server-api/)中给出的 `responses` 结构相匹配。
- `metadata`：任意 KV 元数据
- `timestamp`：可选的事件时间戳

示例：

```python
evt = BotUttered(text = "Hallo user")
```

### ActionExecuted {#actionexecuted}

```python
ActionExecuted(
    action_name,
    policy=None,
    confidence: Optional[float] = None,
    timestamp: Optional[float] = None,
)
```

底层事件：[`action`](/action-server/events/#action)

参数：

- `action_name`：被调用的动作名称
- `policy`：用于预测动作的策略
- `confidence`：预测动作的置信度
- `timestamp`：可选的事件时间戳

示例：

```python
evt = ActionExecuted("action_greet_user")
```
