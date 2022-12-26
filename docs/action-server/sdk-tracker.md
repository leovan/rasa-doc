# 追踪器

`Tracker` 类代表一个 Rasa 对话追踪器。它允许你在自定义动作中访问机器人的记忆。你可以通过 `Tracker` 属性和方法获取有关过去事件和当前对话状态的信息。

## 属性 {#attributes}

如下内容可用作 `Tracker` 对象的属性：

- `sender_id`：与对话机器人聊天的人的唯一 ID。
- `slot`：在领域 `ref` 中定义的填充的槽列表。
- `latest_message`：包含最新消息属性的字典：意图、实体和文本。
- `events`：所有先前事件的列表。
- `active_loop`：当前活动循环的名称。
- `latest_action_name`：对话机器人执行的最后一个动作的名称。

## 方法 {#methods}

`Tracker` 提供的方法有：

### `Tracker.current_state` {#trackercurrent_state}

将当前追踪器状态作为对象返回。

返回类型：

`Dict[str, Any]`

### `Tracker.is_paused` {#trackeris_paused}

指示追踪器当前是否暂停。

返回类型：

`bool`

### `Tracker.get_latest_entity_values` {#trackerget_latest_entity_values}

获取在最新消息中为传递的实体类型和可选角色和组找到的实体值。如果你只对给定类型的第一个实体感兴趣，请使用：

```python
next(tracker.get_latest_entity_values(“my_entity_name”), None)
```

如果没有找到实体，则默认结果为 `None`。

参数：

- `entity_type`：感兴趣的实体类型
- `entity_role`：可选的感兴趣的实体角色
- `entity_group`：可选的感兴趣的实体组

返回：

实体值列表

返回类型：

`Iterator[str]`

### `Tracker.get_latest_input_channel` {#trackerget_latest_input_channel}

获取最新的 `UserUttered` 事件的 `input_channel` 的名称。

返回类型：

`Optional[str]`

### `Tracker.events_after_latest_restart` {#trackerevents_after_latest_restart}

在最近重新启动后返回事件列表。

返回类型：

`List[Dict]`

### `Tracker.get_slot` {#trackerget_slot}

检索槽的值。

参数：

- `key`：要检索值的槽的名称

返回类型：

`Optional[Any]`

### `Tracker.get_intent_of_latest_message` {#trackerget_intent_of_latest_message}

检索用户的最新意图。

参数：

- `skip_fallback_intent`（默认值：`True`）：可选择跳过 `nlu_fallback` 意图并返回下一个最高排名。

返回：

最新消息的意图（如果可用）。

返回类型：

`Optional[Text]`
