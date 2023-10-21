# 响应

响应是对话机器人发送给用户的消息。响应通常只有文本，但也可以包括图像和按钮等内容。

## 定义响应 {#defining-responses}

响应位于领域文件或单独的 `responses.yml` 文件的 `responses` 键下。每个响应名称都应该以 `utter_` 开头。例如，你可以在响应 `utter_greet` 和 `utter_bye` 下添加问候和再见的响应。

```yaml title="domain.yml" hl_lines="5-8"
intents:
  - greet

responses:
  utter_greet:
  - text: "Hi there!"
  utter_bye:
  - text: "See you!"
```

如果在对话机器人中使用[检索意图](glossary.md#retrieval-intent)，还需要为对话机器人对这些意图的回复添加响应：

```yaml hl_lines="5 6 8 9"
intents:
  - chitchat

responses:
  utter_chitchat/ask_name:
  - text: Oh yeah, I am called the retrieval bot.

  utter_chitchat/ask_weather:
  - text: Oh, it does look sunny right now in Berlin.
```

!!! info "注意"

    注意检索意图的响应名称的特殊格式。名称都以 `utter_` 开头，然后是检索意图的名称（此处为 `chitchat`），最后是指定不同 `response` 键的后缀（此处为 `ask_name` 和 `ask_weather`）。请参见 [NLU 训练样本文档](training-data-format.md#training-examples)来了解更多信息。

### 在响应中使用变量 {#using-variables-in-responses}

你可以使用变量将信息插入到响应中。在响应中，变量用大括号括起来。例如，如下的 `name` 变量：

```yaml title="domain.yml" hl_lines="3"
responses:
  utter_greet:
  - text: "Hey, {name}. How are you?"
```

当使用 `utter_greet` 响应时，Rasa 会自动使用名为 `name` 的槽中找到值来填充变量。如果槽值不存在或为空，则该变量将填充为 `None`。

填充变量的另一种方法是在[自定义动作](custom-actions.md)中。在自定义动作代码中，你可以为响应提供值来填充特定变量。如果将 Rasa SDK 用于动作服务，可以将变量的值作为关键字参数传递给 [`dispatcher.utter_message`](action-server/sdk-dispatcher.md)：

```python hl_lines="3"
dispatcher.utter_message(
    template="utter_greet",
    name="Sara"
)
```

如果你使用不同的自定义动作服务，可以通过向服务返回的响应添加额外参数来提供值：

```json hl_lines="8"
{
  "events":[
    ...
  ],
  "responses":[
    {
      "template":"utter_greet",
      "name":"Sara"
    }
  ]
}
```

### 响应变体 {#response-variations}

为给定的响应提供多种响应变体以供选择可以使得对话机器人的回复更有趣：

```yaml title="domain.yml" hl_lines="3 4"
responses:
  utter_greet:
  - text: "Hey, {name}. How are you?"
  - text: "Hey, {name}. How is your day going?"
```

在上例中，当 `utter_greet` 被预测为下一个动作时，Rasa 将随机选择两个响应变体中的一个来使用。

#### 用于响应的 ID {#ids-for-responses}

!!! info "3.6 版本新特性"

    你现在可以为任意响应设置 ID。当你想要使用 NLG 服务器生成响应时会非常有用。ID 的类型为字符串。

带有 ID 的相应示例如下：

```yaml title="domain.yml" hl_lines="3 4"
responses:
  utter_greet:
  - id: "greet_1"
    text: "Hey, {name}. How are you?"
  - id: "greet_2"
    text: "Hey, {name}. How is your day going?"
```

### 特定频道的响应变体 {#channel-specific-response-variations}

要根据用户连接的频道指定不同的响应变体，请使用特定于频道的响应变体。

在如下示例中，`channel` 键的第一个响应变体特定于 `slack` 频道，而第二个变体则不是特定于通道：

```yaml title="domain.yml" hl_lines="4"
responses:
  utter_ask_game:
  - text: "Which game would you like to play on Slack?"
    channel: "slack"
  - text: "Which game would you like to play?"
```

!!! info "注意"

    确保 `channel` 键与输入频道的 `name()` 方法返回的值相匹配。如果你使用的是内置频道，此值也将与 `credentials.yml` 文件中使用的频道名称相匹配。

当对话机器人在给定响应名称下查找合适的响应变体时，它会首先尝试从当前频道的特定频道变体中进行选择。如果不存在此类变体，则对话机器人将从非频道特定的响应变体中进行选择。

在上述示例中，第二个响应变体没有指定频道，对话机器人可以将其用于除 `slack` 之外的所有频道。

!!! warning "警告"

    对于每个响应，至少有一个没有 `channel` 键的响应变体。这使得对话机器人在所有环境中都可以正确的响应，例如在新的频道、命令行和交互式学习中。

### 条件响应变体 {#conditional-response-variations}

也可以使用条件响应变体基于一个或多个槽值来选择特定响应变体。条件响应变体在领域或响应 YAML 文件中定义，类似于标准响应变体，但具有额外的 `condition` 键。此键指定 `name` 和 `value` 槽约束的列表。

当在对话期间触发响应时，将根据当前对话状态检查每个条件响应变化的约束。如果所有约束槽值都等于当前对话状态的对应槽值，则响应变体可以被对话机器人使用。

!!! info "注意"

    对话状态槽和约束槽值的比较是由相等 `==` 运算符执行的，它也需要槽值的类型匹配。例如，如果将约束指定为 `value: true`，则需要用布尔值 `true` 填充插槽，而不是字符串 `"true"`。

在下面的示例中，将定义一个具有一个约束的条件响应变体，即 `logged_in` 槽设置为 `true`：

```yaml title="domain.yml"
slots:
  logged_in:
    type: bool
    influence_conversation: False
    mappings:
    - type: custom
  name:
    type: text
    influence_conversation: False
    mappings:
    - type: custom

responses:
  utter_greet:
    - condition:
        - type: slot
          name: logged_in
          value: true
      text: "Hey, {name}. Nice to see you again! How are you?"
    - text: "Welcome. How is your day going?"
```

```yaml title="stories.yml"
stories:
- story: greet
  steps:
  - action: action_log_in
  - slot_was_set:
    - logged_in: true
  - intent: greet
  - action: utter_greet
```

在上述示例中，只要执行 `utter_greet` 动作并将 `logged_in` 槽设置为 `true`，就会使用第一个响应变体（`"Hey, {name}. Nice to see you again! How are you?"`）。没有条件的第二个变体将被视为默认值，并在 `logged_in` 不等于 `true` 时使用。

!!! warning "警告"

    强烈建议始终提供没有条件的默认响应变体，以防止没有条件响应与已填充槽匹配的情况发生。

在对话期间，Rasa 将从所有满足约束的条件响应变体中进行选择。如果有多个符合条件的条件响应变体，Rasa 将随机选择一个。例如，考虑如下响应：

```yaml title="domain.yml"
responses:
  utter_greet:
    - condition:
        - type: slot
          name: logged_in
          value: true
      text: "Hey, {name}. Nice to see you again! How are you?"

    - condition:
        - type: slot
          name: eligible_for_upgrade
          value: true
      text: "Welcome, {name}. Did you know you are eligible for a free upgrade?"
    - text: "Welcome. How is your day going?"
```

如果 `logged_in` 和 `eligible_for_upgrade` 都设置为 `true`，那么第一个和第二个响应变体都可以使用，并且将由对话机器人以相同的概率选择。

你可以继续使用特定于频道的响应变体以及条件响应变体，如下例所示：

```yaml title="domain.yml"
slots:
  logged_in:
    type: bool
    influence_conversation: False
    mappings:
    - type: custom
  name:
    type: text
    influence_conversation: False
    mappings:
    - type: custom

responses:
  utter_greet:
    - condition:
        - type: slot
          name: logged_in
          value: true
      text: "Hey, {name}. Nice to see you again on Slack! How are you?"
      channel: slack
    - text: "Welcome. How is your day going?"
```

Rasa 将按以下顺序优先选择响应：

1. 匹配频道的条件响应。
2. 匹配频道的默认响应。
3. 未匹配频道的条件响应。
4. 未匹配频道的默认响应。

## 富响应 {#rich-responses}

可以通过添加视觉和交互元素来丰富响应。很多频道支持多种类型的元素：

### 按钮 {#buttons}

如下是使用按钮的响应示例：

```yaml title="domain.yml" hl_lines="4-8"
responses:
  utter_greet:
  - text: "Hey! How are you?"
    buttons:
    - title: "great"
      payload: "/mood_great"
    - title: "super sad"
      payload: "/mood_sad"
```

按钮列表中的每个按钮都应该有两个键：

- `title`：按钮上显示的文本。
- `payload`：单击按钮时从用户发送给助手的消息。

如果希望按钮将实体也传递给对话机器人：

```yaml title="domain.yml" hl_lines="4-8"
responses:
  utter_greet:
  - text: "Hey! Would you like to purchase motor or home insurance?"
    buttons:
    - title: "Motor insurance"
      payload: '/inform{{"insurance":"motor"}}'
    - title: "Home insurance"
      payload: '/inform{{"insurance":"home"}}'
```

可以通过如下方式传递多个实体：

```
'/intent_name{{"entity_type_1":"entity_value_1", "entity_type_2": "entity_value_2"}}'
```

!!! info "使用按钮绕过 NLU"

    你可以使用按钮绕过 NLU 预测并触发特定意图和实体。

    以 `/` 开头的消息直接发送到 `RegexInterpreter`，它需要简写的 `/intent{entities}` 格式的 NLU 输入。在上述示例中，如果用户单击按钮，则用户输入将直接分类为 `mood_great` 或 `mood_sad` 意图。

    你可以使用如下各式包含意图传递给 `RegexInterpreter` 的实体：

    `/inform{"ORG":"Rasa", "GPE":"Germany"}`

    `RegexInterpreter` 将根据意图对上述信息进行分类，并提取分别为 `ORG` 和 `GPE` 类型的实体 `Rasa` 和 `Germany`。

!!! info "在 domain.yml 中转义花括号"

    你需要在 `domain.yml` 中使用双花括号编写 `/intent{entities}` 简写响应，以便对话机器人不会将其视为[响应中的变量](responses.md#using-variables-in-responses)并在花括号内插入内容。

!!! warning "检查频道"

    请记住，如何显示定义的按钮取决于输出频道的实现。例如，某些频道可以提供的按钮数量有限，查看 `概念 > 频道连接器` 下的频道文档，了解特定于频道的限制。

### 图片 {#images}

你可以通过在 `image` 键下提供图像的 URL 来将图像添加到响应中：

```yaml title="domain.yml" hl_lines="3"
  utter_cheer_up:
  - text: "Here is something to cheer you up:"
    image: "https://i.imgur.com/nGF1K8f.jpg"
```

### 自定义输出载荷 {#custom-output-payloads}

你可以使用 `custom` 键将任意输出发送到输出频道。输出频道接受存储在 `custom` 键下的对象作为 JSON 有效载荷。

如下是如何将[日期选择器](https://api.slack.com/reference/block-kit/block-elements#datepicker)发送到 [Slack 输出频道](connectors/slack.md)的示例：

```yaml title="domain.yml" hl_lines="3-14"
responses:
  utter_take_bet:
  - custom:
      blocks:
      - type: section
        text:
          text: "Make a bet on when the world will end:"
          type: mrkdwn
        accessory:
          type: datepicker
          initial_date: '2019-05-21'
          placeholder:
            type: plain_text
            text: Select a date
```

## 在对话中使用响应 {#using-responses-in-conversations}

### 将响应作为动作调用 {#calling-responses-as-actions}

如果响应的名称以 `utter_` 开头，则可以直接将响应用作动作，而不会在领域的 `actions` 部分中列出。将响应添加到领域中：

```yaml title="domain.yml"
responses:
  utter_greet:
  - text: "Hey! How are you?"
```

你可以在故事中使用相同的响应作为动作：

```yaml title="stories.yml" hl_lines="5"
stories:
- story: greet user
  steps:
  - intent: greet
  - action: utter_greet
```

当 `utter_greet` 动作运行时，它会将响应中的消息发送回用户。

!!! info "改变响应"

    如果你想更改文本或响应的任何其他部分，需要重新训练对话机器人，然后才能获取这些更改。

### 从自定义动作调用响应 {#calling-responses-from-custom-actions}

你可以使用响应从自定义动作中生成响应消息。如果你使用 Rasa SDK 作为动作服务，可以使用调度程序生成响应消息，例如：

```python title="actions.py" hl_lines="8"
from rasa_sdk.interfaces import Action

class ActionGreet(Action):
    def name(self):
        return 'action_greet'

    def run(self, dispatcher, tracker, domain):
        dispatcher.utter_message(template="utter_greet")
        return []
```

如果你使用不同的自定义动作服务，服务应返回如下 JSON 来调用 `utter_greet` 响应：

```json hl_lines="5"
{
  "events":[],
  "responses":[
    {
      "template":"utter_greet"
    }
  ]
}
```
