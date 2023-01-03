# 规则

规则是一种用于训练对话机器人的对话管理模型的训练数据。规则描述了应该始终遵循的相同路径的简短对话。

不要过度使用规则。规则非常适合处理小型的特定对话模式，但与[故事](/stories/)不同，规则没有能力泛化至未见过的对话路径。结合规则和故事，可以使对话机器人变得更加鲁棒并能够处理真实的用户行为。

如果你无法决定是编写故事还是编写规则来实现某种行为，请参见[编写对话数据](/writing-stories/)。

有关 Rasa 对话机器人实现规则的其他示例，请参见[规则示例对话机器人](https://github.com/RasaHQ/rasa/tree/main/examples/rules)。

## 编写规则 {#writing-a-rule}

在开始编写规则之前，你必须确保将[规则策略](/policies/#rule-policy)添加到模型配置中：

```yaml
policies:
- ... # Other policies
- name: RulePolicy
```

可以将规则添加到训练数据的 `rules` 部分。

要表明规则可以在对话中的任何点使用，请从启动会话的意图开始，然后添加对话机器人应执行的动作来响应该意图。

```yaml
rules:
- rule: Say `hello` whenever the user sends a message with intent `greet`
  steps:
  - intent: greet
  - action: utter_greet
```

此示例规则适用于对话开始以及用户决定在正在进行的对话中发送带有 `greet` 意图的消息时。

仅作为规则出现在训练数据中而不出现在故事中的对话轮次将在预测时被 `TEDPolicy` 等仅机器学习策略忽略。

```yaml
rules:
- rule: Say `hello` whenever the user sends a message with intent `greet`
  steps:
  - intent: greet
  - action: utter_greet

stories:
- story: story to find a restaurant
  steps:
  - intent: find_restaurant
  - action: restaurant_form
  - action: utter_restaurant_found
```

例如，如果如上所述定义了问候规则并且不将其添加到任何故事中，则在 `RulePolicy` 预测 `utter_greet` 之后，`TEDPolicy` 将进行预测，就像没有发生 `greet` 和 `utter_greet` 轮次一样。

### 用于对话开始的规则 {#rules-for-the-conversation-start}

要编写仅适用于对话开始的规则，请在规则中添加一个 `conversation_start: true`。

```yaml
rules:
- rule: Say `hello` when the user starts a conversation with intent `greet`
  conversation_start: true
  steps:
  - intent: greet
  - action: utter_greet
```

如果用户稍后在对话中发送带有 `greet` 的意图的消息，规则将不匹配。

### 有条件的规则 {#rules-with-conditions}

条件描述了为适用规则而必须满足的要求。为此，请在 `condition` 键下添加有关先前对话的任何信息：

```yaml
rules:
- rule: Only say `hello` if the user provided a name
  condition:
  - slot_was_set:
    - user_provided_name: true
  steps:
  - intent: greet
  - action: utter_greet
```

你可以在条件下包含的可能信息包括 `slot_was_set` 事件和 `active_loop` 事件。

### 在规则结束跳过等待用户输入 {#skip-waiting-for-user-input-at-the-end-of-a-rule}

默认情况下，规则将在完成最后一步后等待下一条用户消息：

```yaml
rules:
- rule: Rule which will wait for user message when it was applied
  steps:
  - intent: greet
  - action: utter_greet
  # - action: action_listen
  # Every rule implicitly includes a prediction for `action_listen` as last step.
  # This means that Rasa Open Source will wait for the next user message.
```

如果你想将下一个动作预测交给另一个故事或规则，请将 `wait_for_user_input: false` 添加到你的规则中：

```yaml
rules:
- rule: Rule which will not wait for user message once it was applied
  steps:
  - intent: greet
  - action: utter_greet
  wait_for_user_input: false
```

这表明对话机器人应该在等待更多用户输入之前执行另一个动作。

### 规则和表格 {#rules-and-forms}

当[表单](/forms/)处于活动状态时，对话机器人将根据表单的定义方式进行预测，而忽略规则。如果出现以下情况，规则将再次可用：

- 表单填充了所有必须的槽
- 表单拒绝执行（更多详细信息，请参见[处理预期路径](/forms/#writing-stories--rules-for-unhappy-form-paths)）
