# 故事

故事是一种用于训练对话机器人对话管理模型的训练数据。故事可用于训练泛化到未见过的对话路径的模型。

## 格式 {#format}

故事是用户和智能对话机器人之间对话的表示，转换为特定格式，其中用户输入表示为意图（以及必要的实体），对话机器人的响应和动作表示为动作名称。

以下是 Rasa 故事格式的对话示例：

```yaml
stories:
- story: collect restaurant booking info  # name of the story - just for debugging
  steps:
  - intent: greet                         # user message with no entities
  - action: utter_ask_howcanhelp
  - intent: inform                        # user message with entities
    entities:
    - location: "rome"
    - price: "cheap"
  - action: utter_on_it                   # action that the bot should execute
  - action: utter_ask_cuisine
  - intent: inform
    entities:
    - cuisine: "spanish"
  - action: utter_ask_num_people
```

### 用户消息 {#user-messages}

在编写故事时，你不必处理用户发送消息的具体内容。相反，你可以利用 NLU 管道的输出，它允许你仅使用意图和实体的组合来表示可以表示相同含义的用户发送的所有可能消息。

包含实体也是很重的，因为策略会根据意图和实体的组合来学习预测下一个动作（但是，你可以使用 [use_entities](/domain#ignoring-entities-for-certain-intents) 属性来更改此行为）。

### 动作 {#actions}

对话机器人执行的所有动作，包括的[响应](/responses)都列在故事的 `action` 键下。

你可以通过将来自你的领域的响应作为一个动作列在故事中。同样，你可以通过在领域中的 `actions` 列表中包含自定义动作的名称来指示故事应该调用自定义动作。

### 事件 {#events}

在训练期间，开源 Rasa 不会调用动作服务器。这意味着你的对话机器人的对话管理模型不知道自定义动作将返回哪些事件。

因此，诸如设置槽或激活/停用表单等事件必须作为故事的一部分明确写出。更多详细信息请参见有关[事件](https://rasa.com/docs/action-server/events)的文档。

#### 槽事件 {#slot-events}

槽事件写在故事中的 `slot_was_set` 下。如果此槽设置在自定义动作中，请在自定义动作之后立即添加 `slot_was_set` 事件。如果你的自定义动作槽值重置为 `None`，则相应的事件如下所示：

```yaml hl_lines="6 7"
stories:
- story: set slot to none
  steps:
    # ... other story steps
    - action: my_custom_action
    - slot_was_set:
      - my_slot: null
```

#### 表单事件 {#form-events}

在处理故事中的表单时，需要记住以下三种事件：

- 表单动作事件（例如：`- action: restaurant_form`）在第一次启动表单时开始使用，并且在表单已经处于活动状态时恢复表单动作。
- 表单激活事件（例如：`- active_loop: restaurant_form`）在第一个表单动作事件之后立即使用。
- 表单停用事件（例如：`- active_loop: null`）用于停用表单。

!!! note "编写表单故事"

    为了避免忘记添加事件，推荐的编写这些故事的方法是使用[交互式学习](/writing-stories#using-interactive-learning)。

## 检查点和 OR 语句 {#checkpoints-and-or-statements}

如果需要，应该谨慎使用检查点和 OR 语句。通常可以通过[规则](/rasa/rules)或 [ResponseSelector](/components#responseselector) 来更好地实现所需要的。

### 检查点 {#checkpoints}

你可以使用检查点来模块化和简化你的训练数据。检查点可能很有用，但不要过度使用它们。使用大量检查点会很快使你的示例故事难以理解，并且会减慢训练速度。

如下是包含检查点的示例故事：

```yaml
stories:
- story: beginning of flow
  steps:
  - intent: greet
  - action: action_ask_user_question
  - checkpoint: check_asked_question

- story: handle user affirm
  steps:
  - checkpoint: check_asked_question
  - intent: affirm
  - action: action_handle_affirmation
  - checkpoint: check_flow_finished

- story: handle user deny
  steps:
  - checkpoint: check_asked_question
  - intent: deny
  - action: action_handle_denial
  - checkpoint: check_flow_finished

- story: finish flow
  steps:
  - checkpoint: check_flow_finished
  - intent: goodbye
  - action: utter_goodbye
```

!!! note "注意"

    与常规故事不同，检查点不限于从用户输入开始。只要在故事的正确位置插入检查点，第一个事件也可以是自定义动作或响应。

### OR 语句 {#or-statements}

另一种编写较短故事或以相同方式处理多个意图或槽时间的方法是使用 `affirm` 语句。例如，如果你要求用户确认某事，并且你希望以相同的方式对待 `affirm` 和 `thankyou` 意图。如下故事会在训练时转换成两个故事：

```yaml
stories:
- story:
  steps:
  # ... previous steps
  - action: utter_ask_confirm
  - or:
    - intent: affirm
    - intent: thankyou
  - action: action_handle_affirmation
```

你还可以将 `or` 语句与槽事件一起使用。如下意味着故事需要设置 `name` 槽的值，并且需为 `joe` 或 `bob`：

```yaml
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

`or` 语句可能很有用，但如果你大量使用，最好重组你的领域和/或意图。过度使用 OR 语句会减慢训练速度。

## 测试对话格式 {#test-conversation-format}

测试对话格式是一种将 NLU 数据和故事合并到一个文件中进行评估的格式。有关此格式的更多信息，请参见[测试你的对话机器人](/testing-your-assistant)。

!!! warning "仅用于测试"

    此格式仅用于测试，不能用于训练。

## 端到端训练 {#end-to-end-training}

!!! info "2.2 新功能"

    端到端训练是一项实验性功能。我们引入实验性功能以获得社区的反馈，因此鼓励大家进行尝试。但是，未来功能可能发生变更或删除。如果你有正面或负面反馈，请在 [Rasa 论坛](https://forum.rasa.com/)上与我们分享。

通过端到端训练，你不必处理由 NLU 管道提取的消息的特定意图或领域文件中的单独 `utter_` 响应。相反，你可以直接在故事中包含用户消息和/或机器人响应文本。有关如何编写端到端故事的详细说明，请参见[训练数据格式](/training-data-format#end-to-end-training)。

你可以将端到端格式的训练数据与具有指定意图和动作的标记训练数据混合。故事可以具有意图/动作定义的一些步骤以及由用户或对话机器人对话直接定义的其他步骤。

我们称其为端到端训练，是因为策略可以使用和预测真实的文本。对于端到端的用户输入，由 NLU 管道分类的意图和提取的实体则被忽略。

只有[规则策略](/policies#rule-policy)和 [TED 策略](/policies#ted-policy) 允许端到端训练。

- `RulePolicy` 在预测期间使用简单的字符串匹配。也就是说，基于用户文本的规则只有在规则中的用户文本字符串和预测期间的输入相同时才会匹配。
- `TEDPolicy` 通过额外的神经网络传递用户文本来创建文本的隐含表示。为了获取稳健的性能，你需要提供足够的训练故事来为端到端对话捕获各种用户文本。

Rasa 策略被训练用于下一个话语选择。创建 `utter_` 响应的唯一区别是 `TEDPolicy` 如何对对话机器人话语进行特征化。在 `utter_` 动作情况下，`TEDPolicy` 只能看到动作名称，而如果你使用 `bot` 键提供真实对话，`TEDPolicy` 将根据 NLU 配置将其作为文本输入进行特征化。如果在稍微不同的情况下出现类似的话语，这会有所帮助。然而，这也会使事情变得更难学习，因为不同的话语具有相似的文本这一事实会使得 `TEDPolicy` 更容易混淆这些话语。

端到端的训练在 `TEDPolicy` 中需要更多的参数，因此，训练端到端模型可能需要大量计算资源，具体取决于你的故事中有多少端到端轮次。
