# 默认动作

默认动作是默认内置到对话管理器中的动作。其中大部分是根据某些对话情况自动预测的。你可能需要自定义这些来个性化对话机器人。

这些动作中的每一个都有一个默认行为，在下面的部分中进行了描述。为了覆盖此默认行为，请编写一个[自定义动作](/custom-actions/)，其 `name()` 方法返回默认动作相同的名称：

```python
class ActionRestart(Action):

  def name(self) -> Text:
      return "action_restart"

  async def run(
      self, dispatcher, tracker: Tracker, domain: Dict[Text, Any]
  ) -> List[Dict[Text, Any]]:

      # custom behavior

      return [...]
```

将此动作添加到领域文件的 `actions` 部分，以便对话机器人知道使用自定义定义而不是默认定义：

```yaml
actions:
  - action_restart
```

!!! caution "注意"

    将此动作添加到领域文件后，使用 `rasa train --force` 重新训练模型。否则 Rasa 不会知道你已经改变了任何东西，并且可能会跳过重新训练你的对话模型。

## `action_listen` {#action_listen}

此动作被预测用于发出信号让向对话机器人不应执行任何动作并等待下一个用户输入。

## `action_restart` {#action_restart}

此动作会重置整个对话历史记录，包括在此期间设置的任何槽。

如果模型配置中包含 [`RulePolicy`](/rules/)，它可以由用户在对话中通过发送 `/restart` 消息来触发。如果你在领域中定义了一个 `utter_restart` 响应，它也会被发送给用户。

## `action_session_start` {#action_session_start}

此动作会启动一个新的对话会话，并在如下情况下执行：

- 每次新对话开始时。
- 用户在领域[会话配置](/domain/#session-configuration)中的 `session_expiration_time` 参数定义的一段时间内处于非活动状态后。
- 当用户在对话期间发送 `/session_start` 消息时。

该动作将重置对话跟踪器，但默认情况下不会清除任何已设置的槽。

### 自定义 {#customization}

会话开始动作的默认行为是提取所有现有槽并将它们转移到下一个会话中。假设不想保留所有的槽，而只想保留用户名和他们的电话号码。为此，你需要使用如下所示的自定动作覆盖 `action_session_start`：

```python
from typing import Any, Text, Dict, List
from rasa_sdk import Action, Tracker
from rasa_sdk.events import SlotSet, SessionStarted, ActionExecuted, EventType


class ActionSessionStart(Action):
    def name(self) -> Text:
        return "action_session_start"

    @staticmethod
    def fetch_slots(tracker: Tracker) -> List[EventType]:
        """Collect slots that contain the user's name and phone number."""

        slots = []
        for key in ("name", "phone_number"):
            value = tracker.get_slot(key)
            if value is not None:
                slots.append(SlotSet(key=key, value=value))
        return slots

    async def run(
      self, dispatcher, tracker: Tracker, domain: Dict[Text, Any]
    ) -> List[Dict[Text, Any]]:

        # the session should begin with a `session_started` event
        events = [SessionStarted()]

        # any slots that should be carried over should come after the
        # `session_started` event
        events.extend(self.fetch_slots(tracker))

        # an `action_listen` should be added at the end as a user message follows
        events.append(ActionExecuted("action_listen"))

        return events
```

如果要访问与触发会话开始的用户消息一起发送的元数据，可以访问特殊槽 `session_started_metadata`：

```python
from typing import Any, Text, Dict, List
from rasa_sdk import Action, Tracker
from rasa_sdk.events import SessionStarted, ActionExecuted


class ActionSessionStart(Action):
    def name(self) -> Text:
        return "action_session_start"

    async def run(
      self, dispatcher, tracker: Tracker, domain: Dict[Text, Any]
    ) -> List[Dict[Text, Any]]:
        metadata = tracker.get_slot("session_started_metadata")

        # Do something with the metadata
        print(metadata)

        # the session should begin with a `session_started` event and an `action_listen`
        # as a user message follows
        return [SessionStarted(), ActionExecuted("action_listen")]
```

## `action_default_fallback` {#action_default_fallback}

此动作撤销最后一次用户对话机器人交互并发送 `utter_default` 响应（如果已定义）。如果启用了此[回退机制](/fallback-handoff/)，则它是由低动作预测置信度触发的。

## `action_deactivate_loop` {#action_deactivate_loop}

此动作会停用活动循环并重置请求的槽。这在[处理表单中的非预期路径](/forms/#writing-stories--rules-for-unhappy-form-paths)时使用。

!!! note "注意"

    如果希望重置所有槽，我们建议使用自定义动作，在表单停用后返回 `AllSlotsReset` 事件。

## `action_two_stage_fallback` {#action_two_stage_fallback}

这是一个用于处理低 NLU 置信度的回退循环。更多信息请参见[处理低 NLU 置信度](/fallback-handoff/#nlu-fallback)的信息。

## `action_default_ask_affirmation` {#action_default_ask_affirmation}

`action_two_stage_fallback` 循环将使用此动作。它要求用户确认他们消息的意图。可以自定义此动作来使得特定用例更加个性化。

## `action_default_ask_rephrase` {#action_default_ask_rephrase}

如果用户拒接 `action_default_ask_affirmation` 意图，则 `action_two_stage_fallback` 循环使用此动作。它要求用户重新表述他们的信息。

## `action_back` {#action_back}

此动作撤销上一次用户与对话机器人的交互。如果配置了 [RulePolicy](/policies/#rule-policy)，它可以由用户通过向对话机器人发送 `/back` 消息来触发。

## 表单动作 {#form-action}

默认情况下，开源 Rasa 使用 `FormAction` 来处理[表单逻辑](/forms/)。你可以通过将带有表单名称的自定义动作添加到领域中来使用自定义动作覆盖此默认动作。覆盖表单的默认动作只能是从开源 Rasa 1.0 迁移到 2.0 过程中使用。

## `action_unlikely_intent` {#action_unlikely_intent}

开源 Rasa 通过 [`UnexpecTEDIntentPolicy](/policies/#unexpected-intent-policy) 触发 `action_unlikely_intent`。你可以通过调整 `UnexpecTEDIntentPolicy` 的 [`tolerance`](/policies/#unexpected-intent-policy) 参数来控制预测此动作的频率。

### 自定义 {#customization-1}

你可以自定义对话机器人的行为来配置触发 `action_unlikely_intent` 后应该发生的情况。例如，可以使用规则触发移交人工来进行跟进：

```yaml
- rule: trigger human handoff with action_unlikely_intent
  steps:
    - action: action_unlikely_intent
    - action: ask_human_handoff
    - intent: affirm
    - action: trigger_human_handoff
```

或者，你也可以通过将 `action_unlikely_intent` 添加到领域中的动作列表并实现自定义行为来覆盖其行为作为[自定义动作](/custom-actions/)：

```python
class ActionUnlikelyIntent(Action):
    def name(self) -> Text:
        return "action_unlikely_intent"

    async def run(
        self, dispatcher, tracker: Tracker, domain: Dict[Text, Any],
    ) -> List[Dict[Text, Any]]:

        # Implement custom logic here
        return []
```

!!! note "注意"

    由于 `action_unlikely_intent` 可以在推理期间的任何对话步骤触发，所有仅在故事数据上训练的策略，例如：`TEDPolicy`、`UnexpecTEDIntentPolicy` 和 `MemoizationPolicy` 在进行预测时都会忽略它在跟踪器中的存在。但是，`RulePolicy` 考虑了它的存在，因此[对话行为是可自定义的](/default-actions/#customization-1)。

!!! note "注意"

    `action_unlikely_intent` 不能包含在训练故事中。它只能添加到规则中。

## `action_extract_slots` {#action_extract_slots}

此动作在每个用户轮次之后，下一个对话机器人预测和执行之前运行。`action_extract_slots` 循环遍历每个领域槽的[槽映射](/domain/#slot-mappings)，以便使用从最新用户消息中提取的信息在整个对话中设置或更新槽。

如果 `action_extract_slots` 找到[自定义槽映射](/domain/#custom-slot-mappings)，它将首先检查是否通过 `action` 键在映射中定义了自定义动作，然后运行它。

应用所有槽映射后，`action_extract_slots` 将运行自定义验证动作 `action_validate_slot_mappings`（如果在领域动作中存在）。否则它将立即返回已经提取的槽。

请注意，槽映射或槽验证使用的自定义动作应仅返回 `SlotSet` 或 `BotUttered` 类型的事件。任何其他类型的事件都是不允许的，并且在更新追踪器时将被忽略。

默认操作 `action_extract_slots` 替换了之前由 `FormAction` 执行的槽提取。如果希望根据从触发表单的意图中提取的信息设置槽，则必须显式指定不包含 `conditions` 键的映射。仅当指定的表单处于活动状态时，才会应用带有条件的槽映射。`action_extract_slots` 直接在每个用户消息之后运行，因此这在激活表单之前。因此，应该应用于触发表单的用户消息的映射不得指定条件，否则表单将在激活后重新请求槽。

!!! note "注意"

    如果 `action_default_fallback` 是对话机器人预测和执行的下一个动作，这将导致 `UserUtteranceReverted` 事件，该事件将取消设置先前在最后一个用户轮次中填充的槽。
