# 生成 NLU 数据

NLU（自然语言理解）是开源 Rasa 的一部分，其用于意图分类、实体提取和响应检索。

NLU 接受诸如“I am looking for a French restaurant in the center of town” 并返回如下结构数据：

```json
{
  "intent": "search_restaurant",
  "entities": {
    "cuisine": "French",
    "location": "center"
  }
}
```

构建 NLU 模型是困难的，构建可用于生产环境的模型更是难上加难。以下是设计 NLU 训练数据和流水线来充分利用机器人的一些技巧。

## 用于 NLU 的 CDD {#conversation-driven-development-for-nlu}

对话驱动开发（CDD）意味着让真实用户对话指导开发。对于构建一个出色的 NLU 模型，这意味着两个关键的事情：

### 收集真实数据 {#gather-real-data}

在构建 NLU 训练数据时，开发人员有时会尝试使用文本生成工具或模板来快速增加训练数据样本量。这是一个不好的方式，原因如下：

- 首先，合成数据和用户真实发送给对话机器人的消息看起来并不一样，因此模型会表现欠佳。
- 其次，通过对合成数据进行训练和测试，会欺骗你自己认为模型实际上表现良好，且并不会注意到重大问题。

请记住，如果你使用脚本生成训练数据，模型唯一能学习到的就是如何对脚本进行逆向工程。

为了避免这些问题，收集尽可能多的真实用户数据作为训练数据是一个不错的选择。真实的用户消息可能很混乱，包括拼写错误，而且与你的意图的理想示例相去甚远。但请记住，这些是才是需要模型进行预测的消息。对话机器人最初总是会犯错的，但是对用户数据进行训练和评估的过程会使模型在现实场景中得到更有效的泛化。

### 尽早同测试用户分享 {#share-with-test-users-early}

为了收集真实数据，需要获取真实的用户消息。对话机器人开发者仅能够给出有限的样本，同时用户总是会说出乎意料的话。这意味着你应该尽早与开发团队之外的测试用户共享对话机器人。更多有关信息请参见 [CDD 指南](conversation-driven-development.md)。

## 避免意图混淆 {#avoiding-intent-confusion}

意图是指利用从训练样本中提取的字符或词级别特征的分类，具体取决于添加到 NLU 流水线中的[特征提取器](components.md)。当不同的意图包含以相似方式排序的单词时，这会给意图分类器造成混淆。

### 实体拆分与意图 {#splitting-on-entities-vs-intents}

当你希望对话机器人根据用户提供的信息做出不同响应时容易发生意图混淆。例如：“How do I migrate to Rasa from IBM Watson?”和“I want to migrate from Dialogflow.”。

由于这些消息每条都会触发不同的响应，最初可能会为每种消息创建单独的意图，例如：`watson_migration` 和 `dialogflow_migration`。但是这些意图可能有相同的目的（迁移到 Rasa），并且可能会有类似的描述，这可能会导致模型混淆这些意图。

为了避免意图混淆，应该将这些训练样本合并为单个 `migration` 意图，将响应设置为依赖一个 `product` 槽中的实体值。这样也会使得当没有实体时更容易处理，例如：“How do I migrate to Rasa?”。例如：

```yaml
stories:
- story: migrate from IBM Watson
  steps:
    - intent: migration
      entities:
      - product
    - slot_was_set:
      - product: Watson
    - action: utter_watson_migration

- story: migrate from Dialogflow
  steps:
    - intent: migration
      entities:
      - product
    - slot_was_set:
      - product: Dialogflow
    - action: utter_dialogflow_migration

- story: migrate from unspecified
  steps:
    - intent: migration
    - action: utter_ask_migration_product
```

## 改进实体识别 {#improving-entity-recognition}

在开源 Rasa 中，你可以在训练数据中定义自定义实体并对其进行标注，来让模型学习识别他们。开源 Rasa 还提供了提取预训练实体的组件以及其他形式的训练数据来帮助模型识别和处理实体。

### 预训练实体提取器 {#pre-trained-entity-extractors}

姓名、地址、城市等常见实体需要大量训练数据才能让 NLU 模型获得有效的泛化。

开源 Rasa 为预训练提取提供了两个不错的选项：[`SpacyEntityExtractor`](components.md#spacyentityextractor) 和 [`DucklingEntityExtractor`](components.md#ducklingentityextractor)。由于这些提取器已经在大量数据上进行了预训练，因此可以利用它们在无需对训练数据标注的情况下提取支持类型的实体。

### 正则表达式 {#regexes}

正则表达式可用于对结构化模式的信息提取，例如：5 位的美国邮政编码。正则表达式模式可以用于生成 NUL 模型学习的特征，或作为用于直接实体匹配的方法。更多信息请参见[正则表达式特征](generating-nlu-data.md)。

### 查找表 {#lookup-tables}

查找表类似正则表达式模式，用于检查训练数据中是否存在查找表词条。与正则表达式类似，查找表可以用于为模型提供特征以改进实体识别，或用于基于匹配的实体识别。查找表的应用示例包括冰激淋口味、瓶装水品牌、甚至袜子长度风格（参见[查找表](training-data-format.md#lookup-tables)）。

### 同义词 {#synonyms}

在训练数据中添加同义词有助于将某些实体值映射到一个规范化实体。但是，同义词并不意味着可以提高模型实体识别能力，其对 NUL 性能也没有任何影响。

同义词的一个很好的用例是规范化属于不同组的实体。例如，在询问用户他们感兴趣的保险单的对话机器人中，他们可能会回答“my truck”、“a car”或“I drive o batmobile”。将 `truck`、`car` 和 `batmobile` 映射到标准化的 `auto` 是一个不错的主意，这样处理逻辑仅需要考虑一个更小的可能性集合（参见[同义词](training-data-format.md#synonyms)）。

## 应对特殊问题 {#handling-edge-cases}

### 拼写错误 {#misspellings}

遇到拼写错误是不可避免的，因此对话机器人需要一种有效的方法来处理这个问题。请注意，我们的目标不是纠正拼写错误，而是正确识别意图和实体。出于这个原因，虽然拼写检查器似乎是一个显而易见的解决方案，但通过调整特征提取器和训练数据通常足以解决拼写错误。

添加一个字符级别特征提取器可以通过考虑词的部分信息而非整个词来有效地防御拼写错误。可以通过用于 `CountVectorsFeaturizer` 的 `char_wb` 分析器将字符级别特征提取器添加到流水线中，例如：

```yaml
pipeline:
# <other components>
- name: CountVectorsFeaturizer
  analyze: char_wb
  min_ngram: 1
  max_ngram: 4
# <other components>
```

除了字符级别特征提取器外，你还可以将常见的拼写错误添加到训练数据中。

### 定义超出范围的意图 {#defining-an-out-of-scope-intent}

在对话机器人中定义 `out_of_scope` 意图来捕获超过机器人知识领域外的用户信息是一个不错的主意。当识别出 `out_of_scope` 意图时，你可以回复诸如“I'm not sure how to handle that, here are some things you can ask me...”之类的信息来引导用户获得可以支持的能力。

## 发布更新 {#shipping-updates}

像对待代码一样对待你的数据。就像你永远不会在没有审核的情况下发布代码更新一样，你应该仔细审查对训练数据的更新，因为它会对模型的性能产生重大影响。

使用 Github 或 Bitbucket 等版本控制系统来追踪数据更改，并在必要的时候回滚更新。

确保为你的 NLU 模型构建测试，来[评估](testing-your-assistant.md)训练数据和超参数时的模型性能。在 Jenkins 或 Git Workflow 等[持续集成流水线](setting-up-ci-cd.md)中自动执行这些测试，来简化开发流程并确保发布高质量的更新。
