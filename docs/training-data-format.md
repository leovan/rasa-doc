# 训练数据格式

本页描述了进入 Rasa 对话机器人的不同类型的训练数据以及这些训练数据的结构。

## 概述 {#overview}

开源 Rasa 使用 [YAML](https://yaml.org/spec/1.2/spec.html) 作为一种统一且可扩展的方式来管理所有训练数据，包括 NLU 数据、故事和规则。

你可以将训练数据拆分为任意数量的 YAML 文件，每个文件可以包含 NLU 数据、故事和规则的任意组合。训练数据解析器使用顶级键来确定训练数据类型。

[领域](/glossary/#domain)使用与训练数据相同的 YAML 格式，也可以跨多个文件拆分或合并到一个文件中。领域包括[响应](/responses/)和[表单](/forms/)的定义。有关如何格式化领域文件的信息，请参见[领域的文档](/domain/)。

!!! tips "遗留格式"

    在寻找 Markdown 数据格式？其在 Rasa 3.0 中已被删除，但你仍可以参考 [Markdown NLU 数据](https://legacy-docs-v1.rasa.com/nlu/training-data-format/)和 [Markdown 故事](https://legacy-docs-v1.rasa.com/core/stories/)的文档。如果你仍有 Markdown 格式的训练数据，那么推荐的方法是使用 Rasa 2.x 将数据从 Markdown 转换为 YAML。[迁移指南](/migration-guide/#training-data-files)解释了如何执行该操作。

### 高层级结构 {#high-level-structure}

每个文件可以包含一个或多个带有对应训练数据的键。一个文件可以包含多个键，但每个键在单个文件中只能出现一次。可用的键有：

- `version`
- `nlu`
- `stories`
- `rules`

你应该在所有 YAML 训练数据文件中指定 `version` 键。如果你未在训练数据中指定 `version` 键，Rasa 会假定你使用已安装的开源 Rasa 版本支持的最新训练数据格式规范。开源 Rasa 版本高于你机器上安装的版本的训练数据文件将被跳过。目前，Rasa 3.x 的最新训练数据格式规范是 3.1。

### 示例 {#example}

如下为一个简短的示例，其将所有训练数据保存在一个文件中：

```yaml
version: "3.1"

nlu:
- intent: greet
  examples: |
    - Hey
    - Hi
    - hey there [Sara](name)

- intent: faq/language
  examples: |
    - What language do you speak?
    - Do you only handle english?

stories:
- story: greet and faq
  steps:
  - intent: greet
  - action: utter_greet
  - intent: faq
  - action: utter_faq

rules:
- rule: Greet user
  steps:
  - intent: greet
  - action: utter_greet
```

要指定测试故事，你需要将其放在一个单独的文件中：

```yaml title="tests/test_stories.yml"
stories:
- story: greet and ask language
- steps:
  - user: |
      hey
    intent: greet
  - action: utter_greet
  - user: |
      what language do you speak
    intent: faq/language
  - action: utter_faq
```

[测试故事](/training-data-format/#test-stories)使用与故事训练数据相同的格式，并应放在带有 `test_` 前缀的单独文件中。

!!! tips "`|` 符号"

    如上例所示，`user` 和 `examples` 键后跟 `|`（管道）符号。在 YAML 中，`|` 标识具有保留缩进的多行字符串。这有助于在训练样本中保留其他特殊符号（例如 `"`，`'` 和训练样本中的其他符号）可用。

## NLU 训练数据 {#nlu-training-data}

[NLU](/glossary/#nlu) 训练数据由按[意图](/glossary/#intent)分类的用户话语样本组成。训练样本还可以包括[实体](/glossary/#entity)。实体是从用户信息中提取的结构化信息。你还可以在训练数据中添加额外的信息，例如正则表达式和查找表，来帮助模型正确的识别意图和实体。

NLU 训练数据在 `nlu` 键下定义。可以在此键下添加的项目有：

- 按用户意图分组的[训练样本](/training-data-format/#training-examples)，例如可选带标注的[实体](/training-data-format/#entities)

    ```yaml
    nlu:
    - intent: check_balance
      examples: |
        - What's my [credit](account) balance?
        - What's the balance on my [credit card account]{"entity":"account","value":"credit"}
    ```

- [同义词](/training-data-format/#synonyms)

    ```yaml
    nlu:
    - synonym: credit
      examples: |
        - credit card account
        - credit account
    ```

- [正则表达式](/training-data-format/#regular-expressions)

    ```yaml
    nlu:
    - regex: account_number
      examples: |
        - \d{10,12}
    ```

- [查找表](/training-data-format/#lookup-tables)

    ```yaml
    nlu:
    - lookup: banks
      examples: |
        - JPMC
        - Comerica
        - Bank of America
    ```

### 训练样本 {#training-examples}

训练样本按[意图](/glossary/#intent)分组并列在 `examples` 键下。通常会在每一行列出一个样本，如下所示：

```yaml
nlu:
- intent: greet
  examples: |
    - hey
    - hi
    - whats up
```

但是，如果有自定的 NLU 组件并且需要样本元数据，也可以使用扩展格式：

```yaml
nlu:
- intent: greet
  examples:
  - text: |
      hi
    metadata:
      sentiment: neutral
  - text: |
      hey there!
```

`metadata` 键可以包含任意键值数据，这些数据与样本相关联并可被 NLU 管道中的组件访问。在上面的示例中，情感元数据可以被管道中的自定义组件用于情感分析。

你还可以在意图级别指定此元数据：

```yaml
nlu:
- intent: greet
  metadata:
    sentiment: neutral
  examples:
  - text: |
      hi
  - text: |
      hey there!
```

在这种情况下，`metadata` 键的内容将传递给每个意图样本。

如果你想指定[检索意图](/glossary/#retrieval-intent)，则 NLU 样本如下：

```yaml
nlu:
- intent: chitchat/ask_name
  examples: |
    - What is your name?
    - May I know your name?
    - What do people call you?
    - Do you have a name for yourself?

- intent: chitchat/ask_weather
  examples: |
    - What's the weather like today?
    - Does it look sunny outside today?
    - Oh, do you mind checking the weather for me please?
    - I like sunny days in Berlin.
```

所有检索意图都添加了一个后缀用于标识对话机器人的特定响应键。在上面的例子中，`ask_name` 和 `ask_weather` 是后缀。后缀与检索意图名称由 `/` 分隔。

!!! note "`/` 的特殊含义"

    如上例所示，`/` 作为保留的分隔符用于将检索意图与其关联的响应键分开。确保不要在意图的名称中使用它。

### 实体 {#entities}

[实体](/glossary/#entity)是从用户消息中提取的结构化信息。

在训练样本中实体采用实体名称进行标注。除了实体名称之外，你还可以使用[同义词](/nlu-training-data/#synonyms)、[角色和分组](/nlu-training-data/#entities-roles-and-groups)来标注实体。

在训练样本中，实体标注如下所示：

```yaml
nlu:
- intent: check_balance
  examples: |
    - how much do I have on my [savings](account) account
    - how much money is in my [checking]{"entity": "account"} account
    - What's the balance on my [credit card account]{"entity":"account","value":"credit"}
```

标注一个实体的完整语法可能为：

```
[<entity-text>]{"entity": "<entity name>", "role": "<role name>", "group": "<group name>", "value": "<entity synonym>"}
```

在这种表示中，`role`，`group` 和 `value` 关键字是可选的。`value` 字段表示同义实体。要了解标签角色和分组的用途，请参见[实体角色和分组](/nlu-training-data/#entities-roles-and-groups)部分。

### 同义词 {#synonyms}

同义词通过将提取的实体映射到一个值而非提取的文字来规范化训练数据。你可以使用以下格式的同义词：

```yaml
nlu:
- synonym: credit
  examples: |
    - credit card account
    - credit account
```

你还可以通过指定实体的值在训练样本中定义同义词：

```yaml
nlu:
- intent: check_balance
  examples: |
    - how much do I have on my [credit card account]{"entity": "account", "value": "credit"}
    - how much do I owe on my [credit account]{"entity": "account", "value": "credit"}
```

有关同义词的更多信息，请参见 [NLU 训练数据](/nlu-training-data/#synonyms) 页面。

### 正则表达式 {#regular-expressions}

你可以使用 [`RegexFeaturizer`](/components/#regexfeaturizer) 和 [`RegexEntityExtractor`](/components/#regexentityextractor) 组件用正则表达式来改进意图分类和实体提取。

定义正则表达式的格式如下：

```yaml
nlu:
- regex: account_number
  examples: |
    - \d{10,12}
```

这里 `account_number` 是正则表达式的名称。当用作 `RegexFeaturizer` 的特征时，正则表达式的名称无关紧要。使用 `RegexEntityExtractor` 时，正则表达式的名称应与你要提取实体的名称匹配。

有关合适以及如何将正则表达式与每个组件一起使用，请参见 [NLU 训练数据](/nlu-training-data/#synonyms) 页面。

### 查找表 {#lookup-tables}

查找表是用于生成不区分大小写的正则表达式模式的单词列表。格式如下：

```yaml
nlu:
- lookup: banks
  examples: |
    - JPMC
    - Bank of America
```

当你在训练数据中提供查找表时，该表的内容将组合成一个大的正则表达式。此正则表达式用于检查每个训练样本以查看它是否包含查找表中词条的匹配项。

查找表正则表达式的处理方式与直接在训练数据中指定正则表达式相同，可以与 [`RegexFeaturizer`](/components/#regexfeaturizer) 和 [`RegexEntityExtractor`](/components/#regexentityextractor) 一起使用。查找表的名称受制于与正则表达式功能名称相同的约束。

有关查找表的更多信息，请参见 [NLU 训练数据](/nlu-training-data/#synonyms) 页面。

## 对话训练数据 {#conversation-training-data}

故事和规则都是用户和对话机器人之间对话的表示。它们用于训练对话管理模型。[故事](/stories/)用于训练机器学习模型，来识别对话中的模式并泛化至未见过的对话路径。[规则](/rules/)描述了应该始终遵循相同路径并用于训练 [RulePolicy](/policies/#rule-policy) 的小段对话。

### 故事 {#stories}

故事由以下部分组成：

- `story`：故事的名称。名称可以为任意值，不用于训练。你可以将其用作故事的可读参考。
- `metadata`：任意且可选，不用于训练。你可以使用它来存储有关数据的相关信息，例如：作者。
- `steps` 列表：构成故事的用户消息和动作。

例如：

```yaml
stories:
- story: Greet the user
  metadata:
    author: Somebody
    key: value
  steps:
  # list of steps
  - intent: greet
  - action: utter_greet
```

每个步骤可为如下之一：

- 一个[用户消息](/training-data-format/#user-messages)，由意图和实体表示。
- 一个 [`or` 语句](/training-data-format/#or-statement)，包含两个或多个用户消息。
- 一个[动作](/training-data-format/#actions)。
- 一个[表单](/training-data-format/#forms)。
- 一个设置事件的[槽](/training-data-format/#slots)。
- 一个[检查点](/training-data-format/#checkpoints)，其将故事与另一个故事相连。

#### 用户消息 {#user-messages}

所有用户消息都会指定 `intent:` 键和一个可选的 `entities:` 键。

在编写故事时，你不必处理用户发送的消息的具体内容。相反，你可以利用 NLU 管道的输出，其使用意图和实体的组合来表示与用户可能发送的具有相同含义的所有可能消息。

用户消息遵循如下格式：

```yaml hl_lines="4 5 6"
stories:
- story: user message structure
  steps:
  - intent: intent_name  # Required
    entities:  # Optional
    - entity_name: entity_value
  - action: action_name
```

例如，要表达 `I want to check my credit balance` 这句话，其中 `credit` 为一个实体：

```yaml hl_lines="4 5 6"
stories:
- story: story with entities
  steps:
  - intent: account_balance
    entities:
    - account_type: credit
  - action: action_credit_account_balance
```

此处包含实体很重要，因为策略会根据意图和实体的组合来学习预测下一个动作（但是，你可以使用 [`use_entities`](/training-data-format/#entities) 属性改变此行为）。

#### 动作 {#actions}

对话机器人执行的所有动作都使用 `action:` 键指定，后跟动作的名称。在编写故事时，你会遇到两种类型的动作：

- [响应](/domain/#responses)：以 `utter_` 开头并向用户发送特定消息。例如：

```yaml hl_lines="5"
stories:
- story: story with a response
  steps:
  - intent: greet
  - action: utter_greet
```

- [自定义动作](/custom-actions/)：以 `actions_` 开头，运行任意代码并发送任意数量的消息（或不发送）。

```yaml hl_lines="5"
stories:
- story: story with a custom action
  steps:
  - intent: feedback
  - action: action_store_feedback
```

#### 表单 {#forms}

[表单](/glossary/#form)是一种特定类型的自定义动作，其中包含在一组所需槽并要求用户提供此信息上的循环逻辑。你可以在领域的 `forms` 部分[定义一个表单](/forms/#defining-a-form)。一旦定义，你应该指定表单的[预期路径](/glossary/#happy--unhappy-paths)作为[规则](/forms/)。你应该在故事中包含表单的中断或其他非预期路径，以便模型可以泛化至未见过的对话序列。作为故事的一个步骤，表单采用以下格式：

```yaml
stories:
- story: story with a form
  steps:
  - intent: find_restaurant
  - action: restaurant_form                # Activate the form
  - active_loop: restaurant_form           # This form is currently active
  - active_loop: null                      # Form complete, no form is active
  - action: utter_restaurant_found
```

`action` 步骤激活表单后开始在所需的槽上执行循环。`active_loop: restaurant_form` 步骤表示当前有一个活动表单。与 `slot_was_set` 步骤相似，`form` 步骤不会将表单设置为活动状态，而是指示它应该已经被激活。同样，`active_loop: null` 步骤表示在采取后续步骤之前不应该激活任何表单。

表单可以被中断并保持活动状态。在这种情况下，中断应该出现在 `action: <form to activate>` 步骤之后，然后是 `active_loop: <active form>` 步骤。表单的中断可能如下所示：

```yaml
stories:
- story: interrupted food
  steps:
    - intent: request_restaurant
    - action: restaurant_form
    - intent: chitchat
    - action: utter_chitchat
    - active_loop: restaurant_form
    - active_loop: null
    - action: utter_slots_values
```

#### 槽 {#slots}

槽事件通过 `slot_was_set:` 键进行指定，同时带有槽的名称和可选的槽值。

[槽](/domain/#slots)充当机器人的记忆。槽由默认动作 [`action_extract_slots`](/default-actions/#action_extract_slots) 根据领域中指定的[槽映射](/domain/#slot-mappings)设置，或由自定义动作设置。它们在 `slot_was_set` 步骤中被故事引用。例如：

```yaml hl_lines="5 6"
stories:
- story: story with a slot
  steps:
  - intent: celebrate_bot
  - slot_was_set:
    - feedback_value: positive
  - action: utter_yay
```

这意味着故事要求 `feedback_value` 槽的值为 `positive`，对话才能够继续。

是否需要包含槽的值取决于[槽类型](/domain/#slot-types)以及该值是否可以或应该影响对话。如果值无关紧要，例如：`text` 槽，你可以只列出槽的名称：

```yaml hl_lines="5 6"
stories:
- story: story with a slot
  steps:
  - intent: greet
  - slot_was_set:
    - name
  - action: utter_greet_user_by_name
```

默认情况下，任何槽的初始值为 `null`，你可以使用它来检查槽是否未设置：

```yaml hl_lines="5 6"
stories:
- story: French cuisine
  steps:
  - intent: inform
  - slot_was_set:
    - cuisine: null
```

!!! tips "槽如何工作"

    故事并不设置槽。如果槽映射适用，则槽必须由默认动作 `action_extract_slots` 设置，或者在 `slot_was_set` 步骤之前由自定义动作设置。

#### 检查点 {#checkpoints}

检查点使用 `checkpoint:` 键指定，可以在故事的开头或结尾。

检查点是将故事连接在一起的方式。它们可以是故事的第一步，也可以是最后一步。如果是故事的最后一步，则该故事将与其他故事相连，该故事以训练模型时的同名检查点开始。如下是一个检查点结尾的故事示例，以及以相同检查点开始的故事：

```yaml
stories:
- story: story_with_a_checkpoint_1
  steps:
  - intent: greet
  - action: utter_greet
  - checkpoint: greet_checkpoint

- story: story_with_a_checkpoint_2
  steps:
  - checkpoint: greet_checkpoint
  - intent: book_flight
  - action: action_book_flight
```

故事开头的检查点可以以设置的槽为条件，例如：

```yaml hl_lines="6 7 8"
stories:
- story: story_with_a_conditional_checkpoint
  steps:
  - checkpoint: greet_checkpoint
    # This checkpoint should only apply if slots are set to the specified value
    slot_was_set:
    - context_scenario: holiday
    - holiday_name: thanksgiving
  - intent: greet
  - action: utter_greet_thanksgiving
```

检查点可以帮助简化训练数据并减少其中的冗余，但不要过度使用它们。使用大量检查点会很快让故事难以理解。如果在不同的故事中经常重复一系列步骤，则使用它们是有意义的，但是没有检查点的故事更容易阅读和编写。

#### 或语句 {#or-statement}

`or` 步骤是以相同方式处理多个意图或槽事件的方法，同时无需为每个意图编写单独的故事。例如，如果你要求用户确认某事，你可能希望以相同的方式处理 `affirm` 和 `thankyou` 意图。带有 `or` 步骤的故事将在训练时转换为多个单独的故事。例如，以下故事将在训练时转换为两个故事：

```yaml hl_lines="6 7 8"
stories:
- story: story with OR
  steps:
  - intent: signup_newsletter
  - action: utter_ask_confirm
  - or:
    - intent: affirm
    - intent: thanks
  - action: action_signup_newsletter
```

你还可以将 `or` 语句与槽事件一起使用。如下意味着故事需要 `name` 槽被设置的值为 `joe` 或 `bob`。这个故事将在训练时转换为两个故事。

```yaml hl_lines="6 7 8"
stories:
- story:
  steps:
  - intent: greet
  - action: utter_greet
  - intent: tell_name
  - or:
    - slot_was_set:
        - name: joe
    - slot_was_set:
        - name: bob
  # ... next actions
```

同检查点一样，或语句可能很有用，但如果你使用很多，最好考虑重组你的领域和/或意图。

!!! danger "不要过度使用"

    过度使用这些功能（检查点和或语句）会减慢训练速度。

### 规则 {#rules}

规则列在 `rules` 键下，看起来同故事类似。规则也有一个 `steps` 键，其中包含与故事相同的步骤列表。规则还可以包含 `conversation_started` 和 `conditions` 键。这些用于指定规则适用的条件。

一个带有条件的规则如下所示：

```yaml
rules:
- rule: Only say `hey` when the user provided a name
  condition:
  - slot_was_set:
    - user_provided_name: true
  steps:
  - intent: greet
  - action: utter_greet
```

有关编写规则的更多信息，请参见[规则](/rules/#writing-a-rule)。

## 测试故事 {#test-stories}

测试故事用于检查一个消息是否被正确的进行分类和动作预测。

测试故事使用与[故事](/training-data-format/#stories)相同的格式，除了用户消息步骤可以包含一个 `user` 用来指定用户消息的实际文本标注实体。如下是一个测试故事的例子：

```yaml
stories:
- story: A basic end-to-end test
  steps:
  - user: |
     hey
    intent: greet
  - action: utter_ask_howcanhelp
  - user: |
     show me [chinese]{"entity": "cuisine"} restaurants
    intent: inform
  - action: utter_ask_location
  - user: |
     in [Paris]{"entity": "location"}
    intent: inform
  - action: utter_ask_price
```

了解更多有关测试的信息，请参见[测试对话机器人](/testing-your-assistant/)。

## 端到端训练 {#end-to-end-training}

!!! info "2.2 新功能"

    端到端训练是一项实验性功能。我们引入实验性功能以从社区中获取反馈，因此鼓励大家进行尝试。但是，该功能未来可能会发生更改或删除。如果你有任何反馈，请在 [Rasa 论坛](https://forum.rasa.com/)上与我们分享。

通过[端到端训练](/stories/#end-to-end-training)，你不必处理 NLU 管道提取的消息的特定意图。相反，你可以使用 `user` 键将用户消息的文本直接放在故事中。

这些端到端的用户消息遵循如下格式：

```yaml hl_lines="4"
stories:
- story: user message structure
  steps:
    - user: the actual text of the user message
    - action: action_name
```

此外，你可以添加可由 [TED 策略](/policies/#ted-policy)提取的实体标签。实体标签的语法与 [NLU 训练数据](/training-data-format/#entities)中的语法相同。例如，以下故事包含用户消息 `I can always go for sushi`。通过使用 NLU 训练数据中的语法 `[sushi](cuisine)`，你可以将 `sushi` 标记为 `cuisine` 类型的一个实体。

```yaml hl_lines="4"
stories:
- story: story with entities
  steps:
  - user: I can always go for [sushi](cuisine)
  - action: utter_suggest_cuisine
```

同样，你可以将对话机器人消息直接放在故事中，方法是使用 `bot` 键，后面为机器人的消息文本。

包含机器人消息的故事示例如下：

```yaml hl_lines="7"
stories:
- story: story with an end-to-end response
  steps:
  - intent: greet
    entities:
    - name: Ivan
  - bot: Hello, a person with a name!
```

你还可以设置一个混合的端到端的故事：

```yaml
stories:
- story: full end-to-end story
  steps:
  - intent: greet
    entities:
    - name: Ivan
  - bot: Hello, a person with a name!
  - intent: search_restaurant
  - action: utter_suggest_cuisine
  - user: I can always go for [sushi](cuisine)
  - bot: Personally, I prefer pizza, but sure let's search sushi restaurants
  - action: utter_suggest_cuisine
  - user: Have a beautiful day!
  - action: utter_goodbye
```

Rasa 端到端训练与标准 Rasa 完全集成。这意味着你可以混合故事，其中一些步骤由动作或意图定义，其他步骤由用户消息或对话机器人响应直接定义。
