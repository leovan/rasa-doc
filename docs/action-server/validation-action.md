# 槽验证动作

了解如何在 Rasa SDK 中实现 `ValidationAction` 类。

Rasa SDK 中有两个帮助类，用于执行自定义槽提取和验证：

- `ValidationAction`：用于提取和验证可以在表单上下文之外设置或更新的槽的自定义动作的基类。
- `FormValidationAction`：用于提取和验证仅在表单上下文中设置的槽的自定义动作的基类。

为了实现自定义槽提取和验证逻辑，你可以选择继承 `ValidationAction` 或 `FormValidationAction` 类，具体取决于你要设置或更新槽的上下文。

## `ValidationAction` 类 {#validationaction-class}

你可以扩展 Rasa SDK 中的 `ValidationAction` 类来定义自定义提取和/或验证可以在表单上下文之外设置或更新的槽。

!!! note "注意"

    `ValidationAction` 旨在提取表单上下文之外的槽。它将忽略任何具有在[槽映射 `conditions`](/domain/#mapping-conditions)下指定形式的槽的提取和验证方法。当指定的表单处于活动状态或没有表单处于活动状态时，它不会运行这些方法。请扩展 [`FormValidationAction`](/action-server/validation-action/#formvalidationaction-class) 类来仅在表单的上下文中应用自定义槽映射。

### 如何继承 `ValidationAction` {#how-to-subclass-validationaction}

首先，你必须将此动作的名称 `action_validate_slot_mappings` 添加到领域 `actions` 列表中。请注意，你不需要在扩展 `ValidationAction` 的自定义动作中实现 `name` 方法，因为这已经实现。如果你覆盖 `name` 方法，自定义验证动作将不会运行，因为原始名称是硬编码在默认动作 `action_extract_slots` 对动作服务器进行的调用中。

你应该只创建一个 `ValidationAction` 子类，它应该根据你的用例包含不同槽的所有提取和验证方法。

使用此选项，你无需在[自定义槽映射](/domain/#custom-slot-mappings)中指定 `action` 键，因为默认动作 [`action_extract_slots`](/default-actions/#action_extract_slots) 会自动运行 `action_validate_slot_mappings`，如果领域的 `action` 部分存在。

#### 使用预定义映射验证槽 {#validation-of-slots-with-predefined-mappings}

要使用预定义的映射验证槽，你必须编写名为 `validate_<slot_name>` 的函数。

在如下示例中，仅当提取的值是字符串类型时，槽位置的值才大写：

```python
from typing import Text, Any, Dict

from rasa_sdk import Tracker, ValidationAction
from rasa_sdk.executor import CollectingDispatcher
from rasa_sdk.types import DomainDict


class ValidatePredefinedSlots(ValidationAction):
    def validate_location(
        self,
        slot_value: Any,
        dispatcher: CollectingDispatcher,
        tracker: Tracker,
        domain: DomainDict,
    ) -> Dict[Text, Any]:
        """Validate location value."""
        if isinstance(slot_value, str):
            # validation succeeded, capitalize the value of the "location" slot
            return {"location": slot_value.capitalize()}
        else:
            # validation failed, set this slot to None
            return {"location": None}
```

#### 提取自定义槽映射 {#extraction-of-custom-slot-mappings}

要定义自定义提取代码，请使用自定义槽映射为每个槽编写一个 `extract_<slot_name>` 方法。

如下示例显示了一个自定义动作的实现，该动作提取槽 `count_of_insults` 来追踪用户的态度。

```python
from typing import Dict, Text, Any

from rasa_sdk import Tracker
from rasa_sdk.executor import CollectingDispatcher
from rasa_sdk.forms import ValidationAction


class ValidateCustomSlotMappings(ValidationAction):
    async def extract_count_of_insults(
        self, dispatcher: CollectingDispatcher, tracker: Tracker, domain: Dict
    ) -> Dict[Text, Any]:
        intent_of_last_user_message = tracker.get_intent_of_latest_message()
        current_count_of_insults = tracker.get_slot("count_of_insults")
        if intent_of_last_user_message == "insult":
           current_count_of_insults += 1

        return {"count_of_insults": current_count_of_insults}
```

### `ValidationAction` 类实现 {#validationaction-class-implementation}

`ValidationAction` 是 `Action` Rasa SDK 类和抽象 Python `ABC` 类的子类。因此，该类实现了从 `Action` 继承的 `name` 和 `run` 方法。此外，`ValidationAction` 实现了更专业的方法，这些方法将在 `run` 方法中调用：

- `get_extraction_events`：使用可用的 `extract_<slot name>` 方法提取自定义槽。
- `get_validation_events`：通过为每个槽调用可用的 `validate_<slot name>` 方法来验证槽。
- `required_slots`：返回验证动作应填充的槽。

#### 方法 {#methods}

`ValidationAction.name`

定义动作的名称，必须编码为 `action_validate_slot_mappings`。

返回：

动作名称

返回类型：

`str`

`ValidationAction.run`

```python
async ValidationAction.run(dispatcher, tracker, domain)
```

`run` 方法通过调用 `get_extraction_events` 方法执行 `extract_<slot name>` 方法中定义的自定义提取代码，然后使用返回的事件更新追踪器。`run` 方法还将通过 `get_validation_events` 方法执行 `validate_<slot name>` 方法中定义的自定义验证代码，并将返回的事件添加到追踪器。

参数：

- `dispatcher`：用于将消息发送回用户的分发器。使用 `dispatcher.utter_message()` 或任何其他 `rasa_sdk.executor.CollectingDispatcher` 方法。请参阅[分发器的文档](/action-server/sdk-dispatcher/)。
- `tracker`：当前用户的状态追踪器。你可以使用 `tracker.get_slot(slot_name)` 访问槽值，最新的用户消息是 `tracker.latest_message.text` 和任何其他 `rasa_sdk.Tracker` 属性。请参[阅追踪器文档](/action-server/sdk-tracker/)。
- `domain`：对话机器人的领域。

返回：

`rasa_sdk.events.Event` 实例的列表。请参阅[事件文档](/action-server/sdk-events/)。

返回类型：

`List[Dict[str, Any]]`

`ValidationAction.required_slots`

```python
async ValidationAction.required_slots(domain_slots, dispatcher, tracker, domain)
```

`required_slots` 方法将返回 `domain_slots`，它是在领域中映射的所有槽名称的列表，不包括任何带有条件的槽映射。`domain_slots` 由 `domain_slots` 方法返回，它只接受 `Domain` 作为参数。

返回：

`Text` 类型的槽名称列表。

`ValidationAction.get_extraction_events`

```python
async ValidationAction.get_extraction_events(dispatcher, tracker, domain)
```

`get_extraction_events` 方法将通过 `required_slots` 方法调用收集槽名称列表，然后遍历每个槽名称来运行 `extract_<slot name>` 方法（如果有）。

返回：

`rasa_sdk.events.SlotSet` 实例的列表。请参阅 [`SlotSet` 事件的文档](/action-server/sdk-events/#slotset)。

`ValidationAction.get_validation_events`

```python
async ValidationAction.get_validation_events(dispatcher, tracker, domain)
```

`get_validation_events` 方法将收集槽名称列表来通过 `required_slots` 方法调用进行验证。然后它将通过 `tracker.slots_to_validate` 调用获取最近设置的槽及其值的映射。循环遍历最近提取的槽的映射，它将检查槽是否在 `required_slots` 中，然后运行 `validate_<slot name>` 方法（如果该槽可用）。

返回：

`rasa_sdk.events.SlotSet` 实例的列表。请参阅 [`SlotSet` 事件文档](/action-server/sdk-events/#slotset)。

## `FormValidationAction` 类 {#formvalidationaction-class}

`FormValidationAction` 自定义动作仅在其名称中指定的表单被激活时才会运行。如果某些自定义槽映射只应在表单的上下文中提取和验证，则自定义动作应继承自 `FormValidationAction` 而不是 `ValidationAction`。

由于扩展 `FormValidationAction` 的自定义动作仅在表单处于活动状态时在每个用户轮次上运行，因此在这种情况下不需要使用映射条件。

要了解有关如何实现此类的更多信息，请参阅表单[高级用法](/forms/#advanced-usage)。

### `FormValidationAction` 类实现 {#formvalidationaction-class-implementation}

`FormValidationAction` 是 `ValidationAction` Rasa SDK 类和抽象 Python `ABC` 类的子类。`FormValidationAction` 继承了 `ValidationAction` 类的大部分方法，但它覆盖了 `name` 和 `domain_slots` 方法，实现了一个新方法 `next_requested_slot` 并扩展了 `run` 方法的实现。

#### 方法 {#methods-1}

`FormValidationAction.name`

如果对话机器人自定义动作子类 `FormValidationAction` 未返回遵循如下命名规定的自定义名称：`validate_<form name>`，则 `name` 方法将引发 `NotImplementedError` 异常。

`FormValidationAction.required_slots`

`required_slots` 方法将返回 `domain_slots`，它是表单的 `domain_slots` 中包含的所有槽名称的列表。`domain_slots` 由 `domain_slots` 方法返回，它只接受 `Domain` 作为参数。

`FormValidationAction.next_requested_slot`

仅当 `required_slots` 方法被自定义动作子类 `FormValidationAction` 覆盖时，方法 `next_requested_slot` 才会将 `REQUESTED_SLOT` 的值设置为下一个未设置的槽。

如果用户没有覆盖 `required_slots`，那么我们将让开源 Rasa 中的 `FromAction` 请求下一个槽，并且该方法将返回 `None`。

该方法所需的参数有：

- [分发器](/action-server/sdk-dispatcher/)
- [追踪器](/action-server/sdk-tracker/)
- 机器人的领域

`FormValidationAction.run`

`ValidationAction.run` 方法的原始实现被扩展为添加对 `next_requested_slot` 方法的调用。`next_requested_slot` 方法调用的输出（如果不为 `None`）被添加到 `run` 方法返回的事件列表中。
