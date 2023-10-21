# NLU 训练数据

NLU 训练数据存储了有关用户消息的结构化信息。

NLU（自然语言理解）的目标是从用户消息中提取结构化信息。这通常包括用户的[意图](glossary.md#intent)和他们消息包含的[实体](glossary.md#entity)。你可以将[正则表达式](nlu-training-data.md#regular-expressions)和[查找表](nlu-training-data.md#lookup-tables)等额外信息添加到训练数据中，来帮助模型正确的识别意图和实体。

## 训练样本 {#training-examples}

NLU 训练数据由按意图分类的示例用户消息组成。为了更容易使用你的意图，请使用用户想要通过该意图完成的内容相关的内容对其命名，保证名称全部为小写字母，并避免使用空格和特殊符号。

!!! info "注意"

    `/` 符号为保留的分割符，用于将[检索意图](glossary.md#retrieval-intent)与响应文本标识符分隔。确保不要在意图的名称中使用它。

## 实体 {#entities}

[实体](glossary.md#entity)是用户消息中的结构化信息。要使实体提取工作，你需要指定训练数据来训练机器学习模型，或者你需要定义[正则表达式](nlu-training-data.md#regular-expressions-for-entity-extraction)来使用基于字符模式的 [`RegexEntityExtractor`](components.md#regexentityextractor) 提取实体。

在决定你需要提取哪些实体时，请考虑你的对话机器人需要哪些信息来实现其用户目标。用户可能会提供任何用户目标都不需要的信息，你并不需要从中提取实体。

有关如何在训练数据中标注实体，请参见[训练数据格式](training-data-format.md)。

## 同义词 {#synonyms}

同义词将提取的实体映射到提取的文本之外的值。当用户有多种表述同一事物的方式时，可以使用同义词。考虑提取实体的最终目标，并从那里找出哪些值应该被认为是等效的。

假设你有一个用户查找用户余额的 `account` 实体。一种可能的账户类型为“credit”。用户可能将他们的“credit”账户表述为“credit account”和“credit card account”。

在这种情况下，你可以将“credit card account”和“credit account”定义为“credit”的同义词：

```yaml
nlu:
- synonym: credit
  examples: |
    - credit card account
    - credit account
```

然后，如果这些短语中的任何一个被提取为一个实体，它将被映射为 `credit` 值。

!!! info "提供训练样本"

    同义词映射发生在提取实体之后。这意味着你的训练样本应包括同义词示例（`credit card account` 和 `credit account`），以便模型学会将它们识别为实体并用 `credit` 替换它们。

有关如何在训练数据中包含同义词的详细信息，请参见[训练数据格式](training-data-format.md)。

## 正则表达式 {#regular-expressions}

你可以结合流水线中的 [`RegexFeaturizer`](components.md#regexfeaturizer) 和 [`RegexEntityExtractor`](components.md#regexentityextractor) 组件使用正则表达式来改进意图分类和实体提取。

### 用于意图分类的正则表达式 {#regular-expressions-for-intent-classification}

你可以通过在流水线中包含 `RegexFeaturizer` 组件来使用正则表达式改进意图分类。使用 `RegexFeaturizer` 时，正则表达式不会作为对意图进行分类的规则。它仅提供意图分类器用于学习意图分类模式的功能。目前，所有意图分类器均可以使用可用的正则表达式功能。

正则表达式的名称是一个用户可读的描述。它可以帮助你记住正则表达式的用途，它是相应模式特征的标题。其不必匹配任何意图或实体名称。一个用于“help”请求的正则表达式如下所示：

```yaml
nlu:
- regex: help
  examples: |
    - \bhelp\b
```

被匹配的意图可以是 `greet`、`help_me`、`assistance` 或其他任何东西。

尽量以匹配尽可能少的单词的方式创建正则表达式。例如，使用 `\bhelp\b` 而不是 `help.*`，因为后者可能匹配整个消息，而第一个仅匹配一个单词。

!!! info "提供训练样本"

    `RegexFeaturizer` 为意图分类器提供特征，但它不直接预测意图。提供足够多的包含正则表达式的样本，以便意图分类器可以学习使用正则表达式特征。

### 用于实体提取的正则表达式 {#regular-expressions-for-entity-extraction}

如果实体具有确定性结构，你可以通过以下两种方法使用正则表达式：

#### 将正则表达式作为特征 {#regular-expressions-as-features}

你可以使用正则表达式为 NLU 流水线中的 [`RegexFeaturizer`](components.md#regexfeaturizer) 组件创建特征。

将正则表达式与 `RegexFeaturizer` 一起使用时，正则表达式的名称无关紧要。使用 `RegexFeaturizer` 时，正则表达式提供一种特征，可以帮助模型学习意图/实体和符合正则表达式的输入之间的关联。

!!! info "提供训练样本"

    `RegexFeaturizer` 为实体提取器提供特征，但它不直接预测实体。提供足够多的包含正则表达式的样本，以便意图分类器可以学习使用正则表达式特征。

目前仅 `CRFEntityExtractor` 和 `DIETClassifier` 组件支持用于实体提取的正则表达式特征。其他实体提取器，例如 `MitieEntityExtractor` 或 `SpacyEntityExtractor` 不会使用生成的特征，并且它们的存在不会改善这些提取器的实体识别。

#### 用于基于规则实体提取的正则表达式 {#regular-expressions-for-rule-based-entity-extraction}

你可以使用 NLU 流水线中的 [`RegexEntityExtractor`](components.md#regexentityextractor) 组件将正则表达式用于基于规则的实体提取。

使用 `RegexEntityExtractor` 时，正则表达式的名称应与你要提取的实体名称匹配。例如，你可以通过在训练数据中包含如下正则表达式和至少两个标注的样本来提取 10-12 位的账号：

```yaml
nlu:
- regex: account_number
  examples: |
    - \d{10,12}
- intent: inform
  examples: |
    - my account number is [1234567891](account_number)
    - This is my account number [1234567891](account_number)
```

每当用户消息包含 10-12 位数字序列时，它将被提取为 `account_number` 实体。`RegexEntityExtractor` 不需要训练样本来学习提取实体，但你至少需要两个标注的样本，以便 NLU 模型可以在训练时将其注册为实体。

### 查找表 {#lookup-tables}

查找表是用于生成不区分大小写的正则表达式模式的单词列表。它们的使用方式与使用[正则表达式](nlu-training-data.md#regular-expressions)的方式相同，与流水线中的 [`RegexFeaturizer`](components.md#regexfeaturizer) 和 [`RegexEntityExtractor`](components.md#regexentityextractor) 组件结合使用。

你可以使用查找表来帮助提取具有一组已知可能值的实体。确保查找表足够详尽。例如，要提取国家名称，你可以添加包含所有国家的查找表：

```yaml
nlu:
- lookup: country
  examples: |
    - Afghanistan
    - Albania
    - ...
    - Zambia
    - Zimbabwe
```

在使用带有 `RegexFeaturizer` 的查找表时，请为你要详匹配的意图或实体提供足够的样本，以便模型可以学习使用生成的正则表达式为特征。在使用带有 `RegexEntityExtractor` 的查找表时，至少提供两个标注的实体样本，以便 NLU 模型可以在训练时将其注册为实体。

## 实体角色和分组 {#entities-roles-and-groups}

将词语标注为自定义实体可以让你在训练数据中定义一些概念。例如：你可以通过标注来识别城市：

```
I want to fly from [Berlin]{"entity": "city"} to [San Francisco]{"entity": "city"} .
```

但是，有时候你希望向实体添加更多详细信息。

例如，要构建一个预定航班的对话机器人，其需要知道上例中的两个城市哪个是出发城市，哪个是目的城市。`Berlin` 和 `San Francisco` 均为城市，但它们在信息中扮演着不同的角色。为了区分不同的角色，除了实体标签之外，你还可以分配一个角色标签。

```
- I want to fly from [Berlin]{"entity": "city", "role": "departure"} to [San Francisco]{"entity": "city", "role": "destination"}.
```

你还可以通过在实体标签旁边指定一个 `group` 标签来对不同的实体进行分组。例如，`group` 标签可用于定义不同的顺序。在如下示例中，`group` 标签指定了哪些配料与哪些披萨搭配，以及每个披萨的大小。

```
Give me a [small]{"entity": "size", "group": "1"} pizza with [mushrooms]{"entity": "topping", "group": "1"} and
a [large]{"entity": "size", "group": "2"} [pepperoni]{"entity": "topping", "group": "2"}
```

有关如何在训练数据中定义具有角色和组的实体的详细信息，请参见[训练数据格式](training-data-format.md#entities)。

提取器返回的实体对象将包括检测到的角色/组标签。

```json
{
  "text": "Book a flight from Berlin to SF",
  "intent": "book_flight",
  "entities": [
    {
      "start": 19,
      "end": 25,
      "value": "Berlin",
      "entity": "city",
      "role": "departure",
      "extractor": "DIETClassifier",
    },
    {
      "start": 29,
      "end": 31,
      "value": "San Francisco",
      "entity": "city",
      "role": "destination",
      "extractor": "DIETClassifier",
    }
  ]
}
```

!!! info "注意"

    目前仅 [`DIETClassifier`](components.md#dietclassifier) 和 [`CRFEntityExtractor`](components.md#crfentityextractor) 支持实体角色和组。

为了使用具有角色和组的实体正确训练你的模型，请确保为实体和角色或组标签的每个组合包含足够的训练样本。为了使模型能够泛化，请确保在训练样本中包含一些变化。例如，应该包括一些 `fly TO y FROM x` 的样本，而不仅仅只有 `fly FROM x TO y`。

要从具有特定角色/组的实体填充槽，你需要为槽定义一个 `from_entity` [槽映射](domain.md#slot-mappings)并指定所需的角色/组。例如：

```yaml
entities:
   - city:
       roles:
       - departure
       - destination

slots:
  departure:
    type: any
    mappings:
    - type: from_entity
      entity: city
      role: departure
  destination:
    type: any
    mappings:
    - type: from_entity
      entity: city
      role: destination
```

### 影响对话预测的实体角色和组 {#entity-roles-and-groups-influencing-dialogue-predictions}

如果你想按角色或组影响对话预测，则需要修改你的故事来包含所需的角色或组标签。你还需要在[领域文件](domain.md#entities)中列出实体的相应角色和组。

假设你想根据用户的位置输出不同的句子。例如，如果用户刚从伦敦抵达，你可能想问一下去伦敦的行程如何。但是，如果用户正在前往马德里的途中，你可能希望该用户有一个愉快的旅途。你可以通过如下两个故事实现此目的：

```yaml
stories:
- story: The user just arrived from another city.
  steps:
    - intent: greet
    - action: utter_greet
    - intent: inform_location
      entities:
        - city: London
          role: from
    - action: utter_ask_about_trip

- story: The user is going to another city.
  steps:
    - intent: greet
    - action: utter_greet
    - intent: inform_location
      entities:
        - city: Madrid
          role: to
    - action: utter_wish_pleasant_stay
```

## BILOU 实体标记 {#bilou-entity-tagging}

[DIETClassifier](components.md#dietclassifier) 和 [CRFEntityExtractor](components.md#crfentityextractor) 具有 `BILOU_flag` 选项，其表示为用于机器学习模型在处理实体是的一种标记模式。`BILOU` 是 Beginning、Inside、Last、Outside 和 Unit-length 的缩写。

例如，一个训练样本如下：

```yaml
[Alex]{"entity": "person"} is going with [Marty A. Rick]{"entity": "person"} to [Los Angeles]{"entity": "location"}.
```

首先拆分为一个标记的列表。然后机器学习模型根据 `BILOU_flag` 选项的值应用如下所示的标记模式：

| 标记    | `BILOU_flag = true` | `BILOU_flag = false` |
| :------ | :------------------ | :------------------- |
| alex    | U-person            | person               |
| is      | O                   | O                    |
| going   | O                   | O                    |
| with    | O                   | O                    |
| marty   | B-person            | person               |
| a       | I-person            | person               |
| rick    | L-person            | person               |
| to      | O                   | O                    |
| los     | B-location          | location             |
| angeles | L-location          | location             |

与普通标记模式相比，BILOU 标记模式更丰富。在预测实体时，它可以有助于提高机器学习模型的性能。

!!! info "不一致的 BILOU 标签"

    当 `BILOU_flag` 选项设置为 `True` 时，模型可能会预测不一致的 BILOU 标签，例如 `B-person`、`I-location`、`L-person`。开源 Rasa 使用一些启发式方法来清理不一致的 BILOU 标签。例如，`B-person`、`I-location`、`L-person` 改为 `B-person`、`I-person`、`L-person`。
