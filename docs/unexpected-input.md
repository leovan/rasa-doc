# 处理非预期输入

在构建对话机器人时，用户会说出意料之外的话。本页面是用于处理非预期输入的指南。

非预期输入是指偏离定义的[预期路径](glossary.md#happy--unhappy-paths)，例如：

- 用户在沟通订阅的途中离开，然后回来说“hi!”
- 当对话机器人询问他们的电子邮箱时，用户问“Why do you need to know that?”

本页面是用于处理对话机器人仍在领域内的非预期输入方法的指南。根据尝试处理的非预期输入类型，所描述的部分或全部方法可能适用。本指南不是关于消除用户输入歧义和处理超出范围的问题，对于这些情况，请参见[回退和移交人工](fallback-handoff.md)。

## 用户插入语 {#user-interjections}

有两种非预期输入：一般性插入语和上下文插入语。一般性插入语是无论上下文如何都应该得到相同响应的干扰中断。如果你已经对意图定义了响应规则，则无需对这种干扰中断执行任何其他操作。[FAQ 和闲聊](chitchat-faqs.md)是常见的一般性插入语。上下文插入语的响应取决于对话上下文。例如，如果用户问“Why do you need that?”，则答案取决于机器人刚刚问询的内容。

### 上下文插入语 {#contextual-interjections}

处理上下文插入语和处理[上下文对话](contextual-conversations.md)类似。

上下文插入语的一种常见情况是在填写[表单](forms.md)时，用户会问“Why do you need to know that?”或“Can you explain that?”。响应会根据槽的不同而不同。例如：

<figure markdown>
  ![](images/unexpected-input/contextual-interjection.png){ width="600" }
  <figcaption>上下文插入语</figcaption>
</figure>

由于我们希望 `requested_slot` 影响对话，因此我们需要将 `requested_slot` 槽的属性 `influence_conversation` 设置为 `true`，并为其分配分类类型：

```yaml title='domain.yml'
slots:
  requested_slot:
    type: categorical
    values:
      - cuisine
      - num_people
      - outdoor_seating
      - preferences
      - feedback
    influence_conversation: true
    mappings:
    - type: custom
```

这意味着对话模型在进行预测时会关注槽的值（更多信息请参见[槽如何影响对话机器人行为](domain.md#slots-and-conversation-behavior)）。

你可以根据 `requested_slot` 的值编写特定响应插入语的故事，例如：

```yaml title='stories.yml'
stories:
- story: cuisine interjection
  steps:
  - intent: request_restaurant
  - action: restaurant_form
  - active_loop: restaurant_form
  - slot_was_set:
    - requested_slot: cuisine
  - intent: explain
  - action: utter_explain_cuisine
  - action: restaurant_form

- story: number of people interjection
  steps:
  - intent: request_restaurant
  - action: restaurant_form
  - active_loop: restaurant_form
  - slot_was_set:
    - requested_slot: num_people
  - intent: explain
  - action: utter_explain_num_people
  - action: restaurant_form
```

## 总结 {#summary}

如何处理非预期输入取决于响应是否是上下文敏感的。

对于一般性插入语：

- 定义单轮交互规则
- 使用 ResponseSelector 处理 [FAQ 和闲聊插入语](chitchat-faqs.md)

对于上下文插入语：

- 将 `requested_slot` 设置为一个分类槽（用于表单）
- 为插入语的特定上下文响应编写故事，在可用的情况下使用槽值
