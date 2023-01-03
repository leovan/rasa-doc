# 上下文对话

考虑上下文通常是提供良好用户体验的关键。此页面是创建上下文对话模式的指南。

在上下文对话中，对话中上一步之外的内容在接下来应该发生的事情中发挥作用。例如，如果用户问“How many?”，仅从消息中并不清楚用户在问什么。在对话机器人说“You've got mail!”的上下文中，响应应该为“You have five letters in your mailbox”。在关于未付账单的对话中，响应应该为“You have three overdue bills”。对话机器人需要知道上一个动作才能选择下一个动作。

要创建上下文感知的对话机器人，你需要定义对话历史记录如何影响下一个响应。

例如，如果用户询问音乐会对话机器人如何开始，对话机器人会根据他们是否喜欢音乐做出不同的反应：

与喜欢音乐的用户的对话：

<figure markdown>
  ![](/images/contextual-conversations/user-likes-music.png){ width="600" }
  <figcaption>喜欢音乐的用户</figcaption>
</figure>

与不喜欢音乐的用户的对话：

<figure markdown>
  ![](/images/contextual-conversations/user-does-not-like-music.png){ width="600" }
  <figcaption>不喜欢音乐的用户</figcaption>
</figure>

## 创建上下文对话模式的指南 {#step-by-step-guide-on-creating-contextual-conversation-patterns}

### 定义槽 {#defining-slots}

[槽](/domain/#slots)是对话机器人的记忆。槽存储了对话机器人需要参考的信息片段，并可以根据 `slot_was_set` 事件指导对话流。不同[类型的槽](/domain/#slot-types)都有自己的方式影响对话流。

在音乐会聊天机器人示例中，`likes_music` 槽是一个布尔类型槽。如果值为 `true`，则对话机器人会发送一条介绍消息。如果值为 `false`，则对话机器人会发生不同的消息。你可以在领域内定义一个槽及其类型：

```yaml title='domain.yml'
slots:
  likes_music:
    type: bool
    mappings:
    - type: custom
```

### 创建故事 {#defining-slots}

[故事](/stories/)是对话应该如何进行的样本。在上面的示例中，音乐会对话机器人对喜欢音乐的用户和不喜欢音乐的用户的响应是不同的，因为有两个故事：

```yaml title='stories.yml'
stories:
  - story: User likes music
    steps:
    - intent: how_to_get_started
    - action: utter_get_started
    - intent: affirm
    - action: action_set_music_preference
    - slot_was_set:
      - likes_music: True
    - action: utter_awesome

  - story: User doesn't like music
    steps:
    - intent: how_to_get_started
    - action: utter_get_started
    - intent: deny
    - action: action_set_music_preference
    - slot_was_set:
      - likes_music: False
    - action: utter_goodbye
```

这些故事根据用户的意图（`affirm` 或 `deny`）而有所不同。根据用户的意图，自定义动作设置了一个进一步引导对话的槽。

### 配置 TEDPolicy {#configuring-the-tedpolicy}

除了添加故事来考虑上下文之外，机器学习策略还可以帮助你的模型泛化至未曾见过的对话路径。重要的是要了解使用机器学习策略并不意味着放弃对对话机器人的控制。如果基于规则的策略能做出预测，则该预测将始终具有更高的[策略优先级](/policies/#policy-priority)，并将预测下一个动作。

TEDPolicy 用于处理非预期的用户行为。例如，在下面的对话中（提取自[Rasa X/Enterprise](https://rasa.com/docs/rasa-enterprise/user-guide/review-conversations/) 中的一段对话）：

```yaml
stories:
- story: Story from conversation with a2baab6c83054bfaa8d598459c659d2a on November 28th 2019
  steps:
  - checkpoint: STORY_START
  - intent: greet
  - action: action_greet_user
  - slot_was_set:
    - shown_privacy: true
  - intent: ask_whoisit
  - action: action_chitchat
  - intent: ask_whatspossible
  - action: action_chitchat
  - intent: telljoke
  - action: action_chitchat
  - intent: how_to_get_started
    entities:
    - product: enterprise
  - slot_was_set:
    - product: enterprise
  - action: utter_explain_enterprise
  - action: utter_also_explain_nlucore
  - intent: affirm
  - action: utter_explain_nlu
  - action: utter_explain_core
  - action: utter_direct_to_step2
```

在这里我们可以看到用户首先完成了一些闲聊任务，然后最终问询他如何开始使用 Rasa X/Enterprise。TEDPolicy 正确预测应该向用户解释 Rasa X/Enterprise，然后将他们带入入门路径，而不需要先询问所有问题。

由于机器学习策略已经泛化到这种情况，你应该将此故事添加到训练数据中，来不断改建对话机器人并帮助模型在未来得到更好的泛化能力。[Rasa X/Enterprise](https://rasa.com/docs/rasa-enterprise/) 是一个可以帮助你改进对话机器人并使其更具有情景性的工具。

#### 设置 `max_history` {#setting-max_history}

通常，只有一定数据的上下文与对话机器人相关。[`max_history`](/policies/#max-history) 是 Rasa 对话管理的一个超参数，它控制模型可以查看对话中多少步骤以决定下一步要采取的行动。

在下面的故事中，用户连续三次寻求帮助。前两次，机器人发送相同的消息，但第三次，它将其将给了人工。

```yaml title='stories.yml'
stories:
  - story: user persists in asking for help
    steps:
    - intent: help
    - action: utter_help
    - intent: help
    - action: utter_help
    - intent: help
    - action: action_human_handoff
```

为了让模型学习这种模式，对话机器人至少需要知道前四个步骤，即 `max_history` 为 4。如果 `max_history` 为 3，则模型没有足够的上下文来查看用户已经发送了两次帮助请求，并且不会预测移交人工的动作。

你可以在配置文件设置 `max_history` 并将其传递给策略设置。例如：

```yaml title='config.yml'
policies:
  - name: "TEDPolicy"
    max_history: 5
```

你需要将 `max_history` 设置的足够大，以确保对话机器人对下一步做什么做出预测时有足够的上下文。更多有关信息，请参见[特征提取器](/policies/#featurizers)文档。

## 总结 {#summary}

以下是使对话机器人可以进行上下文对话的总结：

- 为上下文对话编写故事
- 使用槽存储以供后续使用的上下文信息
- 根据对话机器人所需的上下文量为策略设置恰当的 `max_history`
- 使用 TEDPolicy 泛化未见过的对话路径
