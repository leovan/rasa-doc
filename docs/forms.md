# 表单

最常见的对话模式之一就是从用户那里收集一些信息以便做某事（预定餐厅、调用 API、搜索数据库等）。这也称为槽填充。

## 用法 {#usage}

要在开源 Rasa 中使用表单，你需要确保将[规则策略](policies.md#rule-policy)添加到策略配置中。例如：

```yaml
policies:
- name: RulePolicy
```

### 定义表单 {#defining-a-form}

通过将表单添加到[领域](domain.md)中的 `forms` 部分来定义表单。表单的名称也是你可以在[故事](stories.md)或[规则](rules.md)中用于处理表单执行的动作的名称。你需要为必需的 `required_slots` 键指定槽名称列表。

如下示例表单 `restaurant_form` 将填充 `cuisine` 和 `num_people` 槽。

```yaml
entities:
- cuisine
- number
slots:
  cuisine:
    type: text
    mappings:
    - type: from_entity
      entity: cuisine
  num_people:
    type: any
    mappings:
    - type: from_entity
      entity: number
forms:
  restaurant_form:
    required_slots:
        - cuisine
        - num_people
```

可以在 `ignored_intents` 键下为整个表单定义要忽略的意图列表。在 `ignored_intents` 下列出的意图将被添加到每个槽映射的 `not_intent` 键中。

例如，如果你不希望在 `chitchat` 意图时填写表单的所需槽，那么需要定义如下内容（在表单名称之后和 `ignored_intents` 意图关键字下）：

```yaml
entities:
- cuisine
- number
slots:
  cuisine:
    type: text
    mappings:
    - type: from_entity
      entity: cuisine
  num_people:
    type: any
    mappings:
    - type: from_entity
      entity: number
forms:
  restaurant_form:
    ignored_intents:
    - chitchat
    required_slots:
        - cuisine
        - num_people
```

一旦表单动作第一次被调用，表单就会被激活并提示用户输入下一个所需的槽。它通过查找名为 `utter_ask_<form_name>_<slot_name>` 或 `utter_ask_<slot_name>`（如果未找到前者）的[响应](responses.md)来执行此动作。确保在领域文件中为每个必须的槽定义这些响应。

### 激活表单 {#activating-a-form}

要激活表单，你需要添加一个[故事](stories.md)或[规则](rules.md)，它描述了对话机器人应该何时运行表单。在特定意图触发表单的情况下，可以使用如下示例规则：

```yaml
rules:
- rule: Activate form
  steps:
  - intent: request_restaurant
  - action: restaurant_form
  - active_loop: restaurant_form
```

!!! info "注意"

    `active_loop: restaurant_form` 步骤指示应在运行 `restaurant_form` 后激活表单。

### 停用表单 {#deactivating-a-form}

填写完所有必须的槽后，表单将自动停用。你可以使用规则或故事来描述对话机器人在表单结束时的行为。如果不添加适用的故事或规则，对话机器人将在表单完成后自动接收下一条用户消息。如下示例在表单 `your_form` 填满所有必须的槽后立即运行 `utter_submit` 和 `utter_slots_values`。

```yaml
rules:
- rule: Submit form
  condition:
  # Condition that form is active.
  - active_loop: restaurant_form
  steps:
  # Form is deactivated
  - action: restaurant_form
  - active_loop: null
  - slot_was_set:
    - requested_slot: null
  # The actions we want to run when the form is submitted.
  - action: utter_submit
  - action: utter_slots_values
```

用户可能希望尽早脱离表单。请参见[为非预期的表单路径编写故事/规则](forms.md#writing-stories--rules-for-unhappy-form-paths)，了解如何为此案例编写故事或规则。

### 槽映射 {#slot-mappings}

!!! warning "3.0 版本发生改变"

    从 3.0 版本开始，[槽映射](domain.md#slot-mappings)在领域的 `slots` 部分中定义。此更改允许在多个表单中重复使用相同的槽映射，从而消除不必要的重复。请按照[迁移指南](migration-guide.md)更行对话机器人。

    请特别注意[映射条件](domain.md#mapping-conditions)和[唯一实体映射](domain.md#unique-from_entity-mapping-matching)约束的作用。

### 为非预期的表单路径编写故事/规则 {#writing-stories--rules-for-unhappy-form-paths}

用户不会总是回复要求他们提供的信息。通常，用户会提出问题、闲聊、改变主意，或者以其他方式偏离预期的路径。

当表单处于活动状态时，如果用户的输入未填充请求的槽，则表单动作的执行将被拒绝，即表单将自动引发 `ActionExecutionRejection`。以下是表单将引发 `ActionExecutionRejection` 的特定场景：

- 请求一个槽，但用户没有用他们的最后一条信息填充槽，并且没有定义用于[验证槽](forms.md#validating-form-input)或[提取槽](forms.md#custom-slot-mappings)的自定义动作。
- 请求一个槽，但用于[验证槽](forms.md#validating-form-input)或[提取槽](forms.md#custom-slot-mappings)的自定义动作未返回任何 `SlotSet` 事件。

要有意拒绝表单执行，还可以返回 `ActionExecutionRejected` 事件作为自定义验证或槽映射的一部分。

要处理可能导致表单执行被拒绝的情况，你可以编写包含预期中断的规则或故事。例如，如果你希望用户与对话机器人闲聊，可以添加一个规则来处理这个问题：

```yaml
rules:
- rule: Example of an unhappy path
  condition:
  # Condition that form is active.
  - active_loop: restaurant_form
  steps:
  # This unhappy path handles the case of an intent `chitchat`.
  - intent: chitchat
  - action: utter_chitchat
  # Return to form after handling the `chitchat` intent
  - action: restaurant_form
  - active_loop: restaurant_form
```

在某些情况下，用户可能会在表单动作的中间改变主意并决定不继续他们的初始请求。在这种情况下，对话机器人应该获取要求的槽。

使用 `action_deactivate_loop` 默认动作可以优雅地处理此类情况，该动作将停用表单并重置请求的槽。此类对话示例故事如下所示：

```yaml
stories:
- story: User interrupts the form and doesn't want to continue
  steps:
  - intent: request_restaurant
  - action: restaurant_form
  - active_loop: restaurant_form
  - intent: stop
  - action: utter_ask_continue
  - intent: stop
  - action: action_deactivate_loop
  - active_loop: null
```

强烈建议使用[交互式学习](writing-stories.md#using-interactive-learning)来构建这些规则和故事。如果手动编写这些规则/故事可能会错过重要的事情。

## 进阶用法 {#advanced-usage}

使用[自定义动作](actions.md#custom-actions)可完全自定义表单。

### 校验表单输入 {#validating-form-input}

从用户输入中提取槽值后，可以验证提取的槽。默认情况下，开源 Rasa 仅在请求槽后验证是否填充了任何槽。

可以通过[自定义动作](actions.md#custom-actions) `validate_<form_name>` 来验证任何提取的槽。确保将此动作添加到领域的 `actions` 部分：

```yaml
actions:
- validate_restaurant_form
```

执行表单时，它将在每个用户轮流验证最新填充的槽后运行自定义动作。

此自定义动作可以扩展 `FormValidationAction` 类来简化验证提取槽的过程。在这种情况下，你需要为每个提取的槽编写名为 `validate_<slot_name>` 的函数。

如下示例显示了一个自定义动作的实现，该动作验证名为 `cuisine` 的槽是否有效。

```python
from typing import Text, List, Any, Dict

from rasa_sdk import Tracker, FormValidationAction
from rasa_sdk.executor import CollectingDispatcher
from rasa_sdk.types import DomainDict


class ValidateRestaurantForm(FormValidationAction):
    def name(self) -> Text:
        return "validate_restaurant_form"

    @staticmethod
    def cuisine_db() -> List[Text]:
        """Database of supported cuisines"""

        return ["caribbean", "chinese", "french"]

    def validate_cuisine(
        self,
        slot_value: Any,
        dispatcher: CollectingDispatcher,
        tracker: Tracker,
        domain: DomainDict,
    ) -> Dict[Text, Any]:
        """Validate cuisine value."""

        if slot_value.lower() in self.cuisine_db():
            # validation succeeded, set the value of the "cuisine" slot to value
            return {"cuisine": slot_value}
        else:
            # validation failed, set this slot to None so that the
            # user will be asked for the slot again
            return {"cuisine": None}
```

你还可以扩展 `Action` 类并使用 `tracker.slots_to_validate` 检索提取的槽，以完全自定义验证过程。

### 自定义槽映射 {#custom-slot-mappings}

!!! warning "3.0 版本发生改变"

    提供给 `FormValidationAction` 的 `required_slots` 方法的 `slots_mapped_in_domain` 参数已经被 `domain_slots` 参数替换，请将自定义动作更新为新的参数名称。

如果预定义的[槽映射](domain.md#slot-mappings)都不适用你的用例，可以使用[自定义动作](actions.md#custom-actions) `validate_<form_name>` 编写自己的提取代码。开源 Rasa 将在表单运行时触发此动作。

如果使用的是 Rasa SDK，我们建议你扩展提供的 `FormValidationAction`。使用 `FormValidationAction` 时，需要三个步骤提取自定义槽：

1. 为应该以自定义方式映射的每个槽定义一个方法 `extract_<slot_name>`。
2. 在领域文件中，对于表单的 `required_slots`，列出所有必须的槽，包括预定义和自定义映射。

此外，可以重写 `required_slots` 方法来添加动态请求的槽，可以在[动态表单行为](forms.md#dynamic-form-behavior)部分获取更多信息。

!!! info "注意"

    如果在领域文件的 `slots` 部分添加了一个带有自定义映射的槽，你只想通过扩展 `FormValidationAction` 的自定义动作在表单的上下文中对其进行验证，请确保此槽具有自定义类型的映射并且槽名称包含在表单的 `required_slots` 中。

如下示例显示了一个表单的实现，该表单以自定义方式提取槽 `outdoor_seating`，以及使用预定义映射的槽。`extract_outdoor_seating` 方法根据关键字 `outdoor` 是否出现在最后一个用户消息中来设置槽 `outdoor_seating`。

```python
from typing import Dict, Text, List, Optional, Any

from rasa_sdk import Tracker
from rasa_sdk.executor import CollectingDispatcher
from rasa_sdk.forms import FormValidationAction


class ValidateRestaurantForm(FormValidationAction):
    def name(self) -> Text:
        return "validate_restaurant_form"

    async def extract_outdoor_seating(
        self, dispatcher: CollectingDispatcher, tracker: Tracker, domain: Dict
    ) -> Dict[Text, Any]:
        text_of_last_user_message = tracker.latest_message.get("text")
        sit_outside = "outdoor" in text_of_last_user_message

        return {"outdoor_seating": sit_outside}
```

默认情况下，`FormValidationAction` 会自动将 [`requested_slot`](forms.md#the-requested_slot-slot) 设置为 `required_slots` 中指定的第一个未填充的槽。

### 动态表单行为 {#dynamic-form-behavior}

默认情况下，开源 Rasa 将在领域文件中为表单列出的槽中请求下一个空槽。如果使用[自定义槽映射](forms.md#custom-slot-mappings)和 `FormValidationAction`，它将要求 `required_slots` 方法返回第一个空槽。如果 `required_slots` 中的所有槽都已经填满，则表单将被停用。

如果需要，可以动态更新表单的所需槽。例如，当你需要根据前一个槽的填充方式获得更多详细信息或想要更改请求槽的顺序时，这很有用。

如果你使用 Rasa SDK，我们建议使用 `FormValidationAction` 并覆盖 `required_slots` 以适应动态行为。你应该为每个不使预定义映射的槽实现一个方法 `extract_<slot name>`，如[自定义槽映射](forms.md#custom-slot-mappings)中所述。如下示例将询问用户是否想坐在阴凉处或阳光下，以防他们想说坐在外面。

```python
from typing import Text, List, Optional

from rasa_sdk.forms import FormValidationAction

class ValidateRestaurantForm(FormValidationAction):
    def name(self) -> Text:
        return "validate_restaurant_form"

    async def required_slots(
        self,
        domain_slots: List[Text],
        dispatcher: "CollectingDispatcher",
        tracker: "Tracker",
        domain: "DomainDict",
    ) -> List[Text]:
        additional_slots = ["outdoor_seating"]
        if tracker.slots.get("outdoor_seating") is True:
            # If the user wants to sit outside, ask
            # if they want to sit in the shade or in the sun.
            additional_slots.append("shade_or_sun")

        return additional_slots + domain_slots
```

相反，如果想在特定条件下从领域文件中定义表单的 `required_slots` 中删除一个槽，应该将 `domain_slots` 复制到一个新变量并将更改应用于该新变量，而不是直接修改 `domain_slots`。直接修改 `domain_slots` 可能会导致意外行为。例如：

```python
from typing import Text, List, Optional

from rasa_sdk.forms import FormValidationAction

class ValidateBookingForm(FormValidationAction):
    def name(self) -> Text:
        return "validate_booking_form"

    async def required_slots(
        self,
        domain_slots: List[Text],
        dispatcher: "CollectingDispatcher",
        tracker: "Tracker",
        domain: "DomainDict",
    ) -> List[Text]:
        updated_slots = domain_slots.copy()
        if tracker.slots.get("existing_customer") is True:
            # If the user is an existing customer,
            # do not request the `email_address` slot
            updated_slots.remove("email_address")

        return updated_slots
```

### `requested_slot` 槽 {#the-requested_slot-slot}

`requested_slot` 槽作为 [`text`](domain.md#text-slot) 类型的槽自动添加到领域中。在对话期间，`requested_slot` 的值将被忽略。如果要更改此行为行为，则需要将 `requested_slot` 作为分类槽添加到领域文件中，并将 `influence_conversation` 设置为 `true`。如果想以不同方式处理非预期路径，则可能需要执行此动作，具体取决于用户当前询问的槽。例如，如果用户用另一个问题来回答机器人的一个问题，例如“why do you need to know that?”。对这种解释意图的响应取决于在故事中所处的位置。在餐厅案例中，故事将如下所示：

```yaml
stories:
- story: explain cuisine slot
  steps:
  - intent: request_restaurant
  - action: restaurant_form
  - active_loop: restaurant
  - slot_was_set:
    - requested_slot: cuisine
  - intent: explain
  - action: utter_explain_cuisine
  - action: restaurant_form
  - active_loop: null

- story: explain num_people slot
  steps:
  - intent: request_restaurant
  - action: restaurant_form
  - active_loop: restaurant
  - slot_was_set:
    - requested_slot: cuisine
  - slot_was_set:
    - requested_slot: num_people
  - intent: explain
  - action: utter_explain_num_people
  - action: restaurant_form
  - active_loop: null
```

同样，强烈建议使用[交互式学习](writing-stories.md#using-interactive-learning)来构建这些故事。

### 使用自定义动作请求下一个槽 {#using-a-custom-action-to-ask-for-the-next-slot}

一旦表单确定用户接下来必须填写哪个槽，它将执行 `utter_ask_<form_name>_<slot_name>` 或 `utter_ask_<slot_name>` 动作来要求用户提供必要的信息。如果常规话术不够，可以使用自定义动作 `action_ask_<form_name>_<slot_name>` 或 `action_ask_<slot_name>` 来请求下一个槽。

```python
from typing import Dict, Text, List

from rasa_sdk import Tracker
from rasa_sdk.events import EventType
from rasa_sdk.executor import CollectingDispatcher
from rasa_sdk import Action


class AskForSlotAction(Action):
    def name(self) -> Text:
        return "action_ask_cuisine"

    def run(
        self, dispatcher: CollectingDispatcher, tracker: Tracker, domain: Dict
    ) -> List[EventType]:
        dispatcher.utter_message(text="What cuisine?")
        return []
```

如果槽有多个询问选项，Rasa 将按照如下顺序排列优先级：

1. `action_ask_<form_name>_<slot_name>`
2. `utter_ask_<form_name>_<slot_name>`
3. `action_ask_<slot_name>`
4. `utter_ask_<slot_name>`
