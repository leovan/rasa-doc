# 动作

当 Rasa 对话机器人调用自定义动作时，它会向动作服务器发送请求。Rasa 只知道请求响应中返回的任何事件和响应，由动作服务器根据 Rasa 提供的动作名称调用正确的代码。

为了更好地理解 Rasa 调用自定义动作时会发生什么，请考虑如下示例：

你已将天气机器人部署到 Facebook 和 Slack。用户可以使用 `ask_weather` 意图来询问天气。如果用户指定了 `location`，则会有一个槽位置。`action_tell_weather` 动作将使用 API 获取天气预报，如果用户未指定，则使用默认位置。该操作会将 `temperature` 设置为天气预报的最高温度。返回的消息将根据他们使用的频道有所不同。

## 自定义动作输入 {#custom-action-input}

你的动作服务器从 Rasa 服务器接受如下负载：

```json
{
    "next_action": "action_tell_weather",
    "sender_id": "2687378567977106",
    "tracker": {
        "sender_id": "2687378567977106",
        "slots": {
            "location": null,
            "temperature": null
        },
        "latest_message": {
            "text": "/ask_weather",
            "intent": {
                "name": "ask_weather",
                "confidence": 1
            },
            "intent_ranking": [
                {
                    "name": "ask_weather",
                    "confidence": 1
                }
            ],
            "entities": []
        },
        "latest_event_time": 1599850576.655345,
        "followup_action": null,
        "paused": false,
        "events": [
            {
                "event": "action",
                "timestamp": 1599850576.654908,
                "name": "action_session_start",
                "policy": null,
                "confidence": null
            },
            {
                "event": "session_started",
                "timestamp": 1599850576.654916
            },
            {
                "event": "action",
                "timestamp": 1599850576.654928,
                "name": "action_listen",
                "policy": null,
                "confidence": null
            },
            {
                "event": "user",
                "timestamp": 1599850576.655345,
                "text": "/ask_weather",
                "parse_data": {
                    "text": "/ask_weather",
                    "intent": {
                        "name": "ask_weather",
                        "confidence": 1
                    },
                    "intent_ranking": [
                        {
                            "name": "ask_weather",
                            "confidence": 1
                        }
                    ],
                    "entities": []
                },
                "input_channel": "facebook",
                "message_id": "3f2f2317dada4908b7a841fd3eab6bf9",
                "metadata": {}
            }
        ],
        "latest_input_channel": "facebook",
        "active_form": {},
        "latest_action_name": "action_listen"
    },
    "domain": {
        "config": {
            "store_entities_as_slots": true
        },
        "session_config": {
            "session_expiration_time": 60,
            "carry_over_slots_to_new_session": true
        },
        "intents": [
            {
                "greet": {
                    "use_entities": true
                }
            },
            {
                "ask_weather": {
                    "use_entities": true
                }
            }
        ],
        "entities": [],
        "slots": {
            "location": {
                "type": "rasa.core.slots.UnfeaturizedSlot",
                "initial_value": null,
                "auto_fill": true
            },
            "temperature": {
                "type": "rasa.core.slots.UnfeaturizedSlot",
                "initial_value": null,
                "auto_fill": true
            }
        },
        "responses": {
            "utter_greet": [
                {
                    "text": "Hey! How are you?"
                }
            ]
        },
        "actions": [
            "action_tell_weather",
            "utter_greet"
        ],
        "forms": []
    },
    "version": "2.0.0"
}
```

### `next_action` {#next_action}

`next_action` 字段告诉动作服务器要运行什么动作。动作不必作为类的实现，但必须可以按名称调用。

在示例中，动作服务器应运行 `action_tell_weather` 动作。

### `sender_id` {#sender_id}

`sender_id` 告诉你进行对话的用户的唯一 ID。其格式因输入频道而异。它告诉你有关用户的内容，也屈居于输入频道以及频道如何识别用户。

在示例中，`sender_id` 不用于任何事情。

### `tracker` {#tracker}

`tracker` 包含有关对话的信息，包括历史记录和所有槽的记录：

- `sender_id`：与有效负载顶层可用的相同 `sender_id`
- `slots`：对话机器人领域中的每个槽及其当前时间的值
- `latest_message`：最新消息的属性
- `latest_event_time`：最后一个事件添加到追踪器的时间戳
- `followup_action`：调用的动作是强制的后续动作
- `paused`：对话当前是否暂停
- `events`：所有先前[事件](/action-server/events/)的列表
- `latest_input_channel`：接收到最后一条用户消息的输入频道
- `active_form`：当前活动表单的名称，如果有
- `latest_action_name`：机器人执行的最后一个动作的名称

在示例中，自定义动作使用 `location` 槽的值（如果已设置）来获取天气预报。它还会检查 `latest_input_channel` 属性并格式化消息负载，以便它在 Facebook Messenger 中正确显示。

### `domain` {#domain}

`domain` 是 `domain.yml` 文件的 JSON 表示形式。自定义动作不太可能引用其内容，因为他们是静态的并且不指示对话的状态。

### `version` {#version}

这是 Rasa 服务器的版本。自定义动作也不太可能引用此内容，但如果动作服务器仅与某些 Rasa 版本兼容，你可能会在验证步骤中使用它。

## 自定义动作输出 {#custom-action-output}

Rasa 服务器需要一个 `events` 和 `responses` 字典作为对自定义动作调用的响应。

### `events` {#events}

[事件](/action-server/events/)表示动作服务器如何影响对话。在示例中，自定义动作应将最高温度存储在 `temperature` 槽中，因此它需要返回一个 [`slot` 事件](/action-server/events/#slot)。要设置槽并不执行任何其他操作，响应负载应如下所示：

```json
{
    "events": [
        {
            "event": "slot",
            "timestamp": null,
            "name": "temperature",
            "value": "30"
        }
    ],
    "responses": []
}
```

请注意，事件将按照列出的顺序应用于追踪器。对于 `slot` 事件，顺序无关紧要，但对于其他事件类型不是。

### `responses` {#responses}

响应可以是[富响应文档](/responses/#rich-responses)中描述的任何响应类型。有关预期格式，请参阅 [API 规范](https://rasa.com/docs/rasa/pages/action-server-api/)的响应示例。

在示例案例中，你希望向用户发送包含天气预报的消息。要发送常规文本消息，响应负载将如下所示：

```json
{
    "events": [
        {
            "event": "slot",
            "timestamp": null,
            "name": "temperature",
            "value": "30"
        }
    ],
    "responses": [
        {
            "text": "This is your weather forecast!"
        }
    ]
}
```

但是，当想利用频道的特定功能，由于 `latest_input_channel` 是 Facebook，因此可以添加一个带有自定义负载的响应，该负载将根据 Facebook 的 API 规范呈现为媒体消息。然后，响应负载应如下所示：

```json
{
    "events": [
        {
            "event": "slot",
            "timestamp": null,
            "name": "temperature",
            "value": "30"
        }
    ],
    "responses": [
        {
            "text": "This is your weather forecast!"
        },
        {
            "attachment": {
                "type": "template",
                "payload": {
                    "template_type": "media",
                    "elements": [
                        {
                            "media_type": "weather_forcast.gif",
                            "attachment_id": "<id from facebook upload endpoint>"
                        }
                    ]
                }
            }
        }
    ]
}
```

当这个响应被发送回 Rasa 服务器时，Rasa 会将 `slot` 事件和两个响应应用到追踪器，并将两个消息都返回给用户。

## 特殊动作类型 {#special-action-types}

在某些情况下会自动触发一些特殊的动作类型，即[默认动作](/default-actions/)和[槽验证动作](/slot-validation-actions/)。这些特殊动作类型具有预定义的命名约定，必须遵循这些约定以保持自动触发行为。

可以通过实现具有完全相同名称的自定义动作来自定义默认动作。请参阅有关[默认动作的文档](/default-actions/)来了解每个动作的预期行为。

槽验证动作在每个用户轮次运行，具体取决于表单是否处于活动状态。当表单不活动时应该运行的槽验证动作必须名为 `action_validate_slot_mappings`。当表单处于活动状态时应该运行的槽验证动作必须名为 `validate_<form name>`。这些动作应该只返回 `SlotSet` 事件，并且分别表现得像 Rasa SDK [`ValidationAction` 类](/action-server/validation-action/#validationaction-class-implementation)和 [`FormValidationAction` 类](/action-server/validation-action/#formvalidationaction-class-implementation)。
