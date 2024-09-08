# 闲聊和 FAQ

FAQ 对话机器人是最容易构建的对话机器人，通常也是大家构建的第一种对话机器人。本页面是处理 FAQ 和闲聊等非上下文问题所需的概念和训练数据的指南。

[FAQ](glossary.md#faqs) 和[闲聊](glossary.md#chitchat)是对话机器人使用一组固定的消息进行响应，同时无论之前发生什么对话机器人都应始终以相同的方式进行回答的两种情况。例如，在接下来的对话中，每个问题都可以在对话的任何时候被问到，答案与用户之前所说的内容无关。

<div class="md-chat"><div class="chat-container"><div class="chat-input chat-item stack-xs"><p class="chat-bubble"><span class="sr-only">User: </span><span class="content">What's your name</span></p></div><div class="chat-output chat-item stack-xs"><p class="chat-bubble"><span class="sr-only">Bot: </span><span class="content">My name is Sara!</span></p></div><div class="chat-input chat-item stack-xs"><p class="chat-bubble"><span class="sr-only">User: </span><span class="content">Which languages can I build assistants in?</span></p></div><div class="chat-output chat-item stack-xs"><p class="chat-bubble"><span class="sr-only">Bot: </span><span class="content">You can use Rasa to build assistants in any language you want!</span></p></div><div class="chat-input chat-item stack-xs"><p class="chat-bubble"><span class="sr-only">User: </span><span class="content">What’s Rasa X/Enterprise?</span></p></div><div class="chat-output chat-item stack-xs"><p class="chat-bubble"><span class="sr-only">Bot: </span><span class="content">Rasa X/Enterprise is a tool to learn from real conversations and improve your assistant.</span></p></div></div><figcaption>闲聊</figcaption></div>

## 用于 FAQ 和闲聊的响应选择器使用指南 {#step-by-step-guide-on-using-response-selector-for-faqs-and-chitchat}

为了处理 FAQ 和闲聊你需要一个基于规则的对话管理策略（[RulePolicy](policies.md#rule-policy)）和一个简单的方式来对问题做出合适的响应（[ResponseSelector](components.md#responseselector)）。

### 更新配置 {#updating-the-configuration}

对于 FAQ 和闲聊，你希望对话机器人对于每次问到的相同类型问题都以相同的方式做出回应。[规则](rules.md)可以用来达成这个效果。要使用规则，你需要在配置文件中将 [RulePolicy](policies.md#rule-policy) 添加到策略里：

```yaml title='config.yml'
policies:
# other policies
- name: RulePolicy
```

之后，将 ResponseSelector 包含在配置文件中的 NLU 流水线中。ResponseSelector 需要一个特征提取器和意图分类器才能工作，因此它应该位于流水线中这些组件之后，例如：

```yaml title='config.yml'
pipeline:
  - name: WhitespaceTokenizer
  - name: RegexFeaturizer
  - name: LexicalSyntacticFeaturizer
  - name: CountVectorsFeaturizer
  - name: CountVectorsFeaturizer
    analyzer: char_wb
    min_ngram: 1
    max_ngram: 4
  - name: DIETClassifier
    epochs: 100
  - name: EntitySynonymMapper
  - name: ResponseSelector
    epochs: 100
```

默认情况下，ResponseSelector 将为所有检索意图构建一个检索模型，要分别为 FAQ 和闲聊设置响应，需要使用多个 ResponseSelector 组件来指定 `retrieval_intent` 键：

```yaml title='config.yml'
pipeline:
# Other components
- name: ResponseSelector
  epochs: 100
  retrieval_intent: faq
- name: ResponseSelector
  epochs: 100
  retrieval_intent: chitchat
```

### 定义检索意图和 ResponseSelector {#defining-retrieval-intents-and-the-responseselector}

考虑一个包含 20 种不同 FAQ 的示例，尽管每个问题都表示为一个单独的意图，但所有 FAQ 意图在对话中都以相同的方式进行处理。对于每个 FAQ 意图，对话机器人会根据提出的问题检索正确的响应。

你可以使用一个简单的动作，例如 `utter_faq`，通过一个将它们汇总成为一个单独的名为 `faq` 的[检索意图](glossary.md#retrieval-intent)来处理所有 FAQ，而不是编写 20 条规则。

单一的动作使用 [ResponseSelector](components.md#responseselector) 的输出来为用户询问的特定 FAQ 返回正确的响应。

### 创建规则 {#creating-rules}

针对每个检索意图只需要编写一个规则。包含在检索意图下的所有意图都将以相同的方式进行处理。动作的名称以 `utter_` 开头并以检索意图的名称结果。编写用于响应 FAQ 和闲聊的的规则：

```yaml title='rules.yml'
rules:
  - rule: respond to FAQ
    steps:
    - intent: faq
    - action: utter_faq
  - rule: respond to chitchat
    steps:
    - intent: chitchat
    - action: utter_chitchat
```

`utter_faq` 和 `utter_chitchat` 动作将使用 ResponseSelector 的预测来返回实际的响应消息。

### 更新 NLU 训练数据 {#updating-the-nlu-training-data}

用于 ResponseSelector 的 NLU 训练样本同常规训练样本一样，只是他们的名称必须参考被分组的检索意图：

```yaml title='nlu.yml'
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

请确保在领域文件中包含添加的 `chitchat` 意图：

```yaml title='domain.yml'
intents:
# other intents
- chitchat
```

### 定义响应 {#defining-the-responses}

ResponseSelector 的响应遵循检索意图相同的命名约定。除此之外，它们还可以有同常规对话机器人[响应](domain.md#responses)的所有特征。对于上面列出的闲聊意图，响应可以如下所示：

```yaml title='domain.yml'
responses:
  utter_chitchat/ask_name:
  - image: "https://i.imgur.com/zTvA58i.jpeg"
    text: Hello, my name is Retrieval Bot.
  - text: I am called Retrieval Bot!
  utter_chitchat/ask_weather:
  - text: Oh, it does look sunny right now in Berlin.
    image: "https://i.imgur.com/vwv7aHN.png"
  - text: I am not sure of the whole week but I can see the sun is out today.
```

## 总结 {#summary}

完成如下操作后，就可以训练对话机器人并进行测试了！

- 在 `config.yml` 文件中将 RulePolicy 添加至策略中，将 ResponseSelector 添加到流水线中。
- 添加至少一个用于响应 FAQ 或闲聊的规则。
- 为 FAQ 或闲聊意图添加样本。
- 为 FAQ 或闲聊意图田间响应。
- 在领域中更新意图。

现在，你的对话机器人应该能够正确且一致地响应 FAQ 或闲聊了，即使这些插入语发生在对话机器人正在帮助用户完成另一项任务时。
