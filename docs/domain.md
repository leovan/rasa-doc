# 领域

领域定义了对话机器人的全部操作。它指定对话机器人的意图、实体、槽、响应、表单和动作。它还定义了对话会话的配置。

如下是一个取自 [concertbot](https://github.com/RasaHQ/rasa/tree/main/examples/concertbot) 完整的领域实例：

```yaml
version: "3.1"

intents:
  - affirm
  - deny
  - greet
  - thankyou
  - goodbye
  - search_concerts
  - search_venues
  - compare_reviews
  - bot_challenge
  - nlu_fallback
  - how_to_get_started

entities:
  - name

slots:
  concerts:
    type: list
    influence_conversation: false
    mappings:
    - type: custom
  venues:
    type: list
    influence_conversation: false
    mappings:
    - type: custom
  likes_music:
    type: bool
    influence_conversation: true
    mappings:
    - type: custom

responses:
  utter_greet:
    - text: "Hey there!"
  utter_goodbye:
    - text: "Goodbye :("
  utter_default:
    - text: "Sorry, I didn't get that, can you rephrase?"
  utter_youarewelcome:
    - text: "You're very welcome."
  utter_iamabot:
    - text: "I am a bot, powered by Rasa."
  utter_get_started:
    - text: "I can help you find concerts and venues. Do you like music?"
  utter_awesome:
    - text: "Awesome! You can ask me things like \"Find me some concerts\" or \"What's a good venue\""

actions:
  - action_search_concerts
  - action_search_venues
  - action_show_concert_reviews
  - action_show_venue_reviews
  - action_set_music_preference

session_config:
  session_expiration_time: 60  # value in minutes
  carry_over_slots_to_new_session: true
```

## 多个领域文件 {#multiple-domain-files}

领域可以定义为单个 YAML 文件，也可以拆分为目录中的多个文件。当拆分为多个文件时，领域内容可被读取并自动合并在一起。

你可以通过运行如下命令在[命令行界面](/command-line-interface/#rasa-train)中训练具有拆分领域文件的模型：

```shell
rasa train --domain path_to_domain_directory
```

## 意图 {#intents}

领域文件中的 `intents` 键列出了 [NLU 数据](/nlu-training-data/)和[对话训练数据](/training-data-format/#conversation-training-data)中使用的所有意图。

### 忽略某些意图的实体 {#ignoring-entities-for-certain-intents}

要忽略某些意图的所有实体，你可以将 `use_entities: []` 参数添加到领域文件中的意图，如下所示：

```yaml
intents:
  - greet:
      use_entities: []
```

要忽略某些实体或显式地仅考虑某些实体，你可以使用如下语法：

```yaml
intents:
- greet:
    use_entities:
      - name
      - first_name
- farewell:
    ignore_entities:
      - location
      - age
      - last_name
```

对于任何单一意图，只能是 `use_entities` 或 `ignore_entities`。

这些意图的排除实体将不被特征化，因此不会影响下一个动作预测。当你有一个不关心被提取的实体的意图时，这很有用。

如果你在没有 `use_entities` 或 `ignore_entities` 参数的情况下列出意图，实体将被正常特征化。

也可以通过将实体本身的 `influence_conversation` 标识设置为 `false` 来忽略所有意图的实体。更多详细信息请参见[实体部分](/domain/#entities)。

排除的意图实体将不被特征化，因此不会影响下一个动作预测。当你有一个不关心被提取的实体的意图时，这很有用。

如果你在没有此参数的情况下列出你的意图，并且没有将任何实体的 `influence_conversation` 标识设置为 `false`，则所有实体都将正常进行特征化。

!!! note "注意"

    如果你希望这些实体不通过槽影响动作预测，请为具有相同名称的槽设置 [`influence_conversation: false`](/domain/#slots-and-conversation-behavior)。

## 实体

!!! info "3.1 版本新功能"

    从 3.1 版本开始，你可以在实体下使用 `influence_conversation` 标识。该标识可以设置为 `false` 来表明不应针对任何意图对实体进行特征化。它是一种将实体添加到领域中每个意图的 `ignore_entities` 列表的简写语法。该标识是可选的，模型行为保持不变。

实体部分列出了可以由 NLU 管道中的任何[实体提取器](/components/)提取的所有实体。

例如：

```yaml
entities:
   - PERSON           # entity extracted by SpacyEntityExtractor
   - time             # entity extracted by DucklingEntityExtractor
   - membership_type  # custom entity extracted by DIETClassifier
   - priority         # custom entity extracted by DIETClassifier
```

当使用多个领域文件时，可以在任何领域文件中指定实体，并可以在任何领域文件中被任何意图使用或忽略。

如果使用[实体角色和分组](/nlu-training-data/#entities-roles-and-groups)功能，你还需要在本节中列出实体的角色和组。

例如：

```yaml
entities:
   - city:            # custom entity extracted by DIETClassifier
       roles:
       - from
       - to
   - topping:         # custom entity extracted by DIETClassifier
       groups:
       - 1
       - 2
   - size:            # custom entity extracted by DIETClassifier
       groups:
       - 1
       - 2
```

默认情况下，实体会影响动作预测。为了防止提取的实体影响特定意图的对话，你可以[忽略某些意图的实体](/domain/#ignoring-entities-for-certain-intents)。要忽略所有意图的实体，不必在每个意图的 `ignore_entities` 标志下列出，可以在实体下将 `influence_conversation` 设置为 `false`。

```yaml
entities:
- location:
    influence_conversation: false
```

此语法与将实体添加到领域中每个意图的 `ignore_entities` 列表具有相同的效果。

显式设置 `influence_conversation: true` 不会改变任何行为。这是默认设置。

## 槽 {#slots}

槽是对话机器人的记忆。它作为键值存储用于存储用户提供的信息（例如：家乡）以及收集的有关外部的信息（例如：数据库查询结果）。

槽在领域的 `slots` 部分中定义，包括它们的名称、[类型](/domain/#slot-types)和他们是否以及如何[影响对话机器人的行为](/domain/#slots-and-conversation-behavior)。以下示例定义了一个名为“slot_name”的槽，类型为 `text`，预定义槽映射 `from_entity`。

```yaml
slots:
  slot_name:
    type: text
    mappings:
    - type: from_entity
      entity: entity_name
```

### 槽和对话行为 {#slots-and-conversation-behavior}

你可以使用 `influence_conversation` 属性来指定槽是否影响对话。

如果你想在槽中存储信息而不影响对话，请在定义槽时设置 `influence_conversation: false`。

以下示例定义了一个 `age` 槽，它将存储有关用户年龄的信息，但不会影响对话的流程。这意味着对话机器人每次预测下一个动作时都会忽略槽的值。

```yaml
slots:
  age:
    type: text
    # this slot will not influence the predictions
    # of the dialogue policies
    influence_conversation: false
```

定义槽时，如果你省略了 `influence_conversation` 或将其设置为 `true`，则该槽将影响下一个动作预测，除非它的槽类型为 `any`。槽影响对话的方式将取决于[槽的类型](/domain/#slot-types)。

以下示例定义了一个影响对话的 `home_city` 槽。[`text` 槽](/domain/#text-slot)将根据槽是否具有值来影响对话机器人的行为。`text` 槽的特定值（例如：Bangalore 或 New York 或 Hong Kong）没有任何区别。

```yaml
slots:
  # this slot will influence the conversation depending on
  # whether the slot is set or not
  home_city:
    type: text
    influence_conversation: true
```

例如，考虑两个输入“What is the weather like?”和“What is the weather like in Bangalore?”。对话应该根据 `home_city` 槽是否由 NLU 自动设置而进行区分。如果槽已经设置，对话机器人可以预测 `action_forecast` 动作。如果没有设置槽，则需要先获取 `home_city` 信息才能预测天气。

### 槽类型 {#slot-types}

#### 文本类型槽 {#text-slot}

- 类型

    `text`

- 用途

    存储文本值

- 示例

    ```yaml
    slots:
       cuisine:
          type: text
          mappings:
          - type: from_entity
            entity: cuisine
    ```

- 描述

    如果 `influence_conversation` 设置为 `true`，对话机器人的行为将根据 `slot` 是否设置而改变。不同的文本不会进一步影响对话。这意味着如下两个故事是相等的：

    ```yaml
    stories:
    - story: French cuisine
      steps:
      - intent: inform
      - slot_was_set:
        - cuisine: french

    - story: Vietnamese cuisine
      steps:
      - intent: inform
      - slot_was_set:
        - cuisine: vietnamese
    ```

#### 布尔类型槽 {#boolean-slot}

- 类型

    `bool`

- 用途

    存储 `ture` 或 `false` 值。

- 示例

    ```yaml
    slots:
       is_authenticated:
          type: bool
          mappings:
          - type: custom
    ```

- 描述

    如果 `influence_conversation` 设置为 `true`，对话机器人的行为将根据槽是否为空、设置为 `true` 或设置为 `false` 而改变。请注意，空 `bool` 槽对对话的影响与设置为 `false` 的槽不同。

#### 分类型槽 {#categorical-slot}

- 类型

    `categorical`

- 用途

    存储 N 个可选值之一的槽。

- 示例

    ```yaml
    slots:
      risk_level:
         type: categorical
         values:
           - low
           - medium
           - high
         mappings:
           - type: custom
    ```

- 描述

    如果 `influence_conversation` 设置为 `true`，对话机器人的行为将根据槽的具体值而改变。这意味对话机器人的行为会有所不同，具体取决于上述示例中的槽值为 `low`、`medium` 还是 `high`。

    默认值 `__other__` 会自动添加到用户定义的值中。所有遇到的未在槽值中明确定义的值都映射到 `__other__`。`__other__` 不应用作用户定义的值。否则它将作为所有未见过值映射到的默认值。

#### 浮点类型槽 {#float-slot}

- 类型

    `float`

- 用途

    存储实数

- 示例

    ```yaml
    slots:
      temperature:
        type: float
        min_value: -100.0
        max_value:  100.0
        mappings:
        - type: custom
    ```

- 默认值

    `max_value=1.0`，`min_value=0.0`

- 描述

    如果 `influence_conversation` 设置为 `true`，对话机器人行为将根据 `slot` 的值而改变。如果该值介于 `min_value` 和 `max_value` 之间，则使用数字指定的值。所有低于 `min_value` 的值将被视为 `min_value`，所有高于 `max_value` 的值将被视为 `max_value`。因此，如果 `max_value` 设置为 `1`，则槽值在 `2` 和 `3.5` 之间没有区别。

#### 列表类型槽 {#list-slot}

- 类型

    `list`

- 用途

    存储列表值

- 示例

    ```yaml
    slots:
      shopping_items:
        type: list
        mappings:
        - type: from_entity
          entity: shopping_item
    ```

- 描述

    如果 `influence_conversation` 设置为 `true`，对话机器人的行为将根据列表是否为空而改变。存储在槽中的列表长度不会影响对话。只有列表长度是零还是非零才重要。

#### 任意类型槽 {#any-slot}

- 类型

    `any`

- 用途

    存储任意数据（可以为任意类型，例如：词典或列表）

- 示例

    ```yaml
    slots:
      shopping_items:
        type: any
        mappings:
        - type: custom
    ```

- 描述

    `any` 类型槽在对话期间总是被忽略。对于此槽类型，`influence_conversation` 属性不能设置为 `true`。如果要存储影响对话的自定义数据结构，请使用[自定义槽类型](/domain/#custom-slot-types)。

#### 自定义槽类型 {#custom-slot-types}

也许你的餐厅预定系统最多只能处理 6 人的预订。在这种情况下，你希望槽的值影响下一个选定的动作（而不仅仅是它是否已被指定）。你可以通过定义自定义槽类型来做这一点。

下面代码定义了一个名为 `NumberOfPeopleSlot` 的自定义槽类型。特征化定义了如何将槽的值转换为向量，以便开源 Rasa 机器学习模型可以处理它。`NumberOfPeopleSlot` 有三个可能的值，可以用长度为 `2` 的向量表示。

- `(0,0)`：未设置
- `(1,0)`：介于 1 和 6 之间
- `(0,1)`：多于 6

```python title="my_custom_slots.py"
from rasa.shared.core.slots import Slot

class NumberOfPeopleSlot(Slot):

    def feature_dimensionality(self):
        return 2

    def as_feature(self):
        r = [0.0] * self.feature_dimensionality()
        if self.value:
            if self.value <= 6:
                r[0] = 1.0
            else:
                r[1] = 1.0
        return r
```

你可以将自定义槽类型作为独立的 Python 模块实现，与自定义动作代码分开。将自定义槽的代码保存在名为 `__init__.py` 的空文件同级目录中，以便将其识别为 Python 模块。然后，你可以通过其他模块路径引用自定义槽类型。

例如，假设你已将上面的代码保存在 `addons/my_custom_slots.py` 中，这是一个与你的对话机器人项目相关的目录：

```
└── rasa_bot
    ├── addons
    │   ├── __init__.py
    │   └── my_custom_slots.py
    ├── config.yml
    ├── credentials.yml
    ├── data
    ├── domain.yml
    ├── endpoints.yml
```

则你的自定义槽类型的模块路径为 `addons.my_custom_slots.NumberOfPeopleSlot`。使用模块路径来应用领域文件中的自定义槽类型：

```yaml title="domain.yaml"
slots:
  people:
    type: addons.my_custom_slots.NumberOfPeopleSlot
    influence_conversation: true
    mappings:
    - type: custom
```

现在开源 Rasa 可以使用你的自定义槽类型，添加基于 `people` 槽值不同的训练故事。你可以为 `people` 槽值在 1 到 6 之间的情况编写一个故事，为大于 6 的值编写一个故事。你可以选择这些范围内的任何值来放入你的故事，因为它们都以相同的方式进行特征化（请参见上面的特征化情况）。

```yaml
stories:
- story: collecting table info
  steps:
  # ... other story steps
  - intent: inform
    entities:
    - people: 3
  - slot_was_set:
    - people: 3
  - action: action_book_table

- story: too many people at the table
  steps:
  # ... other story steps
  - intent: inform
    entities:
    - people: 9
  - slot_was_set:
    - people: 9
  - action: action_explain_table_limit
```

### 槽映射 {#slot-mappings}

!!! info "3.0 版本新功能"

    从 3.0 版本开始，槽映射在领域文件中 `slots` 部分中定义。此更改删除了通过自动填充设置槽的隐含机制，并用在每个用户消息后设置槽的新显示机制取而代之。你需要在 `domain.yml` 文件的 `slots` 部分为每个槽显式定义槽映射。如果你是从早期版本迁移，请参见[迁移指南](/migration-guide/#slot-mappings)来更新对话机器人。

开源 Rasa 带有 4 个预定义的映射，用于根据最新的用户消息填充槽。

除了预定义的映射，你还可以定义[自定义槽映射](/domain/#custom-slot-mappings)。所有自定义槽映射都应包含 `custom` 类型的映射。

槽映射被指定为领域文件中 `mappings` 键下的 YAML 字典列表。槽映射按照他们在领域中列出的顺序排列优先级。第一个找到的槽映射将用于填充槽。

默认行为是在每个用户消息之后应用槽映射，而不考虑对话上下文。要使槽映射仅在表单的上下文中使用，请参见[映射条件](/domain/#mapping-conditions)。在表单上下文中应用 `from_entity` 槽映射还有一个额外的默认限制。更多详细信息，请参阅[唯一 `from_entity` 映射匹配](/domain/#mapping-conditions)。

请注意，你还可以为可选参数 `intent` 和 `not_intent` 定义意图列表。

#### `from_entity` {#from_entity}

`from_entity` 槽映射根据提取的实体填充槽。需要如下参数：

- `entity`：用于填充槽的实体

如下参数是可选的，可用于进一步指定映射何时应用：

- `intent`：仅在预测此意图时应用映射。
- `not_intent`：预测此意图时不应用映射。
- `role`：仅当提取的实体具有此角色时才应用映射。
- `group`：仅当提取的实体数据该组时才应用映射。

```yaml
entities:
- entity_name
slots:
  slot_name:
    type: any
    mappings:
    - type: from_entity
      entity: entity_name
      role: role_name
      group: group name
      intent: intent_name
      not_intent: excluded_intent
```

#### 唯一 `from_entity` 映射匹配 {#unique-from_entity-mapping-matching}

在表单的上下文中应用 `from_entity` 槽映射存在限制。当表单处于活动状态时，仅当满足以下一个或多个条件时才会应用 `from_entity` 槽映射：

- 表单刚刚请求了带有 `from_entity` 映射的槽。
- 只有一个活动表单的 `required_slots` 具有特定的 `from_entity` 映射，包括提取实体的所有属性（即实体名称、角色、分组）。这称为表单的唯一实体映射。如果映射在 `required_slots` 列表中不是唯一的，则将忽略提取的实体。

存在此限制是为了防止表单使用相同的提取实体值填充多个必须的槽。

例如：在下面的例子中，一个实体 `date` 唯一地设置了 `arrival_date` 槽，一个具有 `from` 角色的 `city` 实体唯一地设置 `departure_city` 槽，同时一个具有 `to` 角色的 `city` 实体唯一地设置 `arrival_city` 槽，因此，即使没有请求这些槽，它们也可以用于填充相应的槽。但是，没有角色的 `city` 实体可以填充 `departure_city` 和 `arrival_city` 槽，这取决于请求哪个槽。因此，如果在请求 `arrival_date` 槽时提取 `city` 实体，它将被表单忽略。

```yaml
slots:
  departure_city:
    type: text
    mappings:
    - type: from_entity
      entity: city
      role: from
    - type: from_entity
      entity: city
  arrival_city:
    type: text
    mappings:
    - type: from_entity
      entity: city
      role: to
    - type: from_entity
      entity: city
  arrival_date:
    type: any
    mappings:
    - type: from_entity
      entity: date
forms:
  your_form:
    required_slots:
    - departure_city
    - arrival_city
    - arrival_date
```

请注意，唯一的 `from_entity` 映射约束不会阻止填充不在活动表单的 `required_slots` 中的槽。无论映射的唯一性如何，这些映射将照常应用。要将槽映射的适用性限制为特定形式，请参见[映射条件](/domain/#mapping-conditions)。

#### `from_text` {#from_text}

`from_text` 映射将使用最后一个用户消息的文本来填充 `slot_name` 槽。如果 `intent_name` 为 `None`，则无论意图名称如何，都会填充槽。否则，只有当用户的意图是 `intent_name` 时才会填充槽。

如果消息的意图是 `excluded_intent`，则槽映射将不适用。

```yaml
slots:
  slot_name:
    type: text
    mappings:
    - type: from_text
      intent: intent_name
      not_intent: excluded_intent
```

!!! note "注意"

    要在使用 `from_text` 槽映射时保持 2.x 版本样式的行为，你必须使用[映射条件](/domain/#mapping-conditions)，其中定义了 `active_loop` 和 `requested_slot` 键。

#### `from_intent` {#from_intent}

如果用户意图是 `intent_name`，则 `from_intent` 映射将使用 `my_value` 值填充 `slot_name` 槽。如果你选择不指定参数意图，则只要未在 `not_intent` 参数下列出该意图，则无论消息的意图如何，都将应用槽映射。

如下参数是必须的：

- `value`：填充 `slot_name` 槽的值

如下参数是可选的，可用于进一步指定映射何时应用：

- `intent`：仅在预测此意图时应用映射。
- `not_intent`：预测此意图时不应用映射。

请注意，如果你选择不定义参数意图，则只要未在 `not_intent` 参数下列出该意图，则无论消息的意图如何，都将应用槽映射。

```yaml
slots:
  slot_name:
    type: any
    mappings:
    - type: from_intent
      value: my_value
      intent: intent_name
      not_intent: excluded_intent
```

#### `from_trigger_intent` {#from_trigger_intent}

如果表单由具有 `intent_name` 意图的用户消息激活，则 `from_trigger_intent` 映射将使用 `my_value` 值填充 `slot_name` 槽。如果消息的意图是 `excluded_intent`，则槽映射将不适用。

```yaml
slots:
  slot_name:
    type: any
    mappings:
    - type: from_trigger_intent
      value: my_value
      intent: intent_name
      not_intent: excluded_intent
```

### 映射条件 {#mapping-conditions}

要仅在表单的上下文中应用槽映射，请在槽映射的 `conditions` 键中指定表单的名称。条件在 `active_loop` 键中列出映射适用的表单名称。

条件还可以包括请求槽的名称。如果 `requested_slot` 未提及，如果提取了相关信息，则将设置该槽，而不管表单正在请求哪个槽。

```yaml
slots:
  slot_name:
    type: text
    mappings:
    - type: from_text
      intent: intent_name
      conditions:
      - active_loop: your_form
        requested_slot: slot_name
      - active_loop: another_form
```

!!! note "注意"

    如果 `conditions` 不包含在槽映射中，则如论是否有任何表单处于活动状态，槽映射都将适用。只要在表单的 `required_slots` 中列出一个槽，当表单被激活时，如果它为空，表单就会提示输入该槽。

#### 自定义槽映射 {#custom-slot-mappings}

当预定义映射都不适合你的用例时，你可以使用[槽验证动作](/slot-validation-actions/)定义自定义槽映射。你必须将此槽映射定义为 `custom` 类型，例如：

```yaml
slots:
  day_of_week:
    type: text
    mappings:
    - type: custom
      action: action_calculate_day_of_week
```

你还可以使用 `custom` 槽映射来列出会话过程中将由任意自定义动作填充的槽，方法是列出类型而不列出特定动作。例如：

```yaml
slots:
  handoff_completed:
    type: boolean
    mappings:
    - type: custom
```

此槽不会在每个用户轮次时更新，但只会在预测为其返回 `SlotSet` 事件的自定义动作时更新。

#### 初始槽值 {#initial-slot-values}

你可以为领域文件中的槽提供初始值：

```yaml
slots:
  num_fallbacks:
    type: float
    initial_value: 0
    mappings:
    - type: custom
```

## 响应 {#responses}

响应是向用户发送消息而不运行任何自定代码或返回事件的动作。这些响应可以直接在领域文件的 `responses` 键下定义，并且可以包含丰富的内容，例如按钮和附件。有关响应以及如何定义它们的更多信息，请参见[响应](/responses/)。

## 表单 {#forms}

表单是一种特殊类型的动作，旨在帮助对话机器人从用户那里收集信息。在领域文件的 `forms` 键下定义表单。有关表单以及如何定义它们的更多信息，请参见[表单](/forms/)。

## 动作 {#actions}

[动作](/actions/)是对话机器人实际上可以做的事情。例如，一个动作可以是：

- 回复用户
- 调用外部 API
- 查询数据库
- 几乎任何东西

所有[自定义动作](/custom-actions/)都应该列在领域文件中，除了不需要列在 `actions:` 下的响应，因为他们已经列在 `responses:` 下。

## 会话配置 {#session-configuration}

对话会话表示对话机器人和用户之间的对话。对话会话可以通过 3 种方式开始：

- 用户开始与对话机器人对话
- 用户在一段可配置的不活动时间后发送他们的第一条消息
- 使用 `/session_start` 意图消息触发手动启动会话

你可以在 `session_config` 键下定义在领域中触发新对话会话的不活动时间段。

可用参数有：

- `session_expiration_time`：定义了以分钟为单位的不活动时间，之后将开始新会话。
- `carry_over_slots_to_new_session`：确定是否应该将现有的设置槽转移到新会话。

默认会话配置如下所示：

```yaml
session_config:
  session_expiration_time: 60  # value in minutes, 0 means infinitely long
  carry_over_slots_to_new_session: true  # set to false to forget slots between sessions
```

这意味着如果用户在 60 分钟不活动后发送他们的第一条消息，则会触发一个新的对话会话，并且任何现有的槽都将转移到新会话中。将 `session_expiration_time` 的值设置为 `0` 意味着会话不会结束（请注意，`action_session_start` 动作仍将在会话开始时触发）。

!!! note "注意"

    会话启动会触发默认动作 `action_session_start`。它的默认实现将所有现有槽移动到新会话中。请注意，所有对话都以 `action_session_start` 开始。例如，覆盖此动作可用于使用来自外部 API 调用的槽初始化跟踪器，或使用对话机器人消息开始对话。[自定义会话启动动作](/default-actions/#customization)文档可以向你展示如何做到这一点。

## 配置 {#config}

领域文件中的 `config` 键维护 `store_entities_as_slots` 参数。此参数仅在阅读故事并将其转变为跟踪器的情况下使用。如果参数设置为 `true`，故事中存在适用的实体，则会导致从实体隐式设置槽。当实体与 `from_entity` 槽映射匹配时，`store_entities_as_slots` 定义实体值是否应放置在该槽中。因此，此参数跳过在故事中手动添加显示 `slot_was_set` 步骤。默认情况下，此行为是打开的。

你可以通过将 `store_entities_as_slots` 设置为 `false` 来关闭此功能：

```yaml title="domain.yml"
config:
  store_entities_as_slots: false
```

!!! info "在寻找 `config.yml`"

    如果你正在寻找有关 `config.yml` 文件的信息，请参见有关[模型配置](/model-configuration/)的文档。
