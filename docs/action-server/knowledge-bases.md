# 知识库动作

在开源机器人框架 Rasa 中使用 `ActionQueryKnowledgeBase` 可以在对话中利用知识库中的信息。

!!! caution "注意"

    此功能是实验性的。我们通过社区反馈引入了这一实验性功能，因此鼓励大家进行尝试。但是，相关功在未来可能会发生变更或删除。如果有任何正面或负面反馈，可以在 [Rasa 论坛](https://forum.rasa.com/)上进行分享。

基于知识的的动作使你能够处理如下类型的对话：

<figure markdown>
  ![](/images/action-server/knowledge-base-example.png)
  <figcaption>知识库示例</figcaption>
</figure>

对话式 AI 中的一个常见问题是，用户不仅通过名称来引用某些对象，而且还是用诸如“the first one”或“it”之类的引用术语。我们需要跟踪提供的信息，ß将这些提及解析为正确的对象。

此外，用户可能希望在对话期间获得有关对象的详细信息，例如：餐厅是否有户外座位，或者它有多贵。为了响应这些用户请求，需要有关餐厅领域的知识。由于信息可能会发生变化，因此硬编码信息不是好的解决方案。

为了应对上述挑战，Rasa 可以与知识库集成。要使用此集成，可以创建一个从 `ActionQueryKnowledgeBase` 继承的自定义动作，这是一个预先编写的自定义动作，其中包含查询知识库中的对象及其属性的逻辑。

可以在 `examples/knowledgebasebot`（[知识库机器人](https://github.com/RasaHQ/rasa/tree/main/examples/knowledgebasebot/)）中找到完整的示例，以及在下面实现此定义动作的说明。

## 使用 `ActionQueryKnowledgeBase` {#using-actionqueryknowledgebase}

### 创建一个知识库 {#create-a-knowledge-base}

用于回答用户请求的数据将存储在知识库中。知识库可用于存储复杂的数据结构。我们建议从使用 `InMemoryKnowledgeBase` 开始。一旦你想处理大量数据，可以切换到自定义知识库（请参见[创建自己的知识库](/action-server/knowledge-bases#create-a-knowledge-base)）。

要初始化 `InMemoryKnowledgeBase`，你需要在 JSON 文件中提供数据。如下示例包含有关餐馆和酒店的数据。JSON 结构应该包含每个对象类型的键，即 `restaurant` 和 `hotel`。每个对象类型都映射到一个对象列表，这里我们有一个包含 3 家餐厅的列表和一个包含 3 家酒店的列表。

```json
{
    "restaurant": [
        {
            "id": 0,
            "name": "Donath",
            "cuisine": "Italian",
            "outside-seating": true,
            "price-range": "mid-range"
        },
        {
            "id": 1,
            "name": "Berlin Burrito Company",
            "cuisine": "Mexican",
            "outside-seating": false,
            "price-range": "cheap"
        },
        {
            "id": 2,
            "name": "I due forni",
            "cuisine": "Italian",
            "outside-seating": true,
            "price-range": "mid-range"
        }
    ],
    "hotel": [
        {
            "id": 0,
            "name": "Hilton",
            "price-range": "expensive",
            "breakfast-included": true,
            "city": "Berlin",
            "free-wifi": true,
            "star-rating": 5,
            "swimming-pool": true
        },
        {
            "id": 1,
            "name": "Hilton",
            "price-range": "expensive",
            "breakfast-included": true,
            "city": "Frankfurt am Main",
            "free-wifi": true,
            "star-rating": 4,
            "swimming-pool": false
        },
        {
            "id": 2,
            "name": "B&B",
            "price-range": "mid-range",
            "breakfast-included": false,
            "city": "Berlin",
            "free-wifi": false,
            "star-rating": 1,
            "swimming-pool": false
        },
    ]
}
```

一旦在 JSON 文件（例如 `data.json`）中定义了数据，就可以使用该数据文件创建你的 `InMemoryKnowledgeBase`，该数据将被传递给查询知识库动作。

知识库中的每个对象都应至少具有 `name` 和 `id` 字段来使用默认实现。如果没有，你将不得不[自定义你的 `InMemoryKnowledgeBase`](/action-server/knowledge-bases#customizing-the-inmemoryknowledgebase)。

### 定义 NLU 数据 {#define-the-nlu-data}

在这个部分：

- 我们将引入一个新的意图 `query_knowledge_base`
- 我们将标注 `mention` 实体，以便模型检测到对象的间接提及，例如：“the first one”
- 我们将广泛使用同义词

为了让对话机器人了解用户想要从知识库中检索信息，需要定义一个新意图。我们将其称为 `query_knowledge_base`。

我们可以将 `ActionQueryKnowledgeBase` 能够处理的请求分为两类：(1). 用户想要获取特定类型的对象列表；(2). 用户想要了解对象的某个属性。意图应该包含这两个请求的多种变体。

```yaml
nlu:
- intent: query_knowledge_base
  examples: |
    - what [restaurants]{"entity": "object_type", "value": "restaurant"} can you recommend?
    - list some [restaurants]{"entity": "object_type", "value": "restaurant"}
    - can you name some [restaurants]{"entity": "object_type", "value": "restaurant"} please?
    - can you show me some [restaurants]{"entity": "object_type", "value": "restaurant"} options
    - list [German](cuisine) [restaurants]{"entity": "object_type", "value": "restaurant"}
    - do you have any [mexican](cuisine) [restaurants]{"entity": "object_type", "value": "restaurant"}?
    - do you know the [price range]{"entity": "attribute", "value": "price-range"} of [that one](mention)?
    - what [cuisine](attribute) is [it](mention)?
    - do you know what [cuisine](attribute) the [last one]{"entity": "mention", "value": "LAST"} has?
    - does the [first one]{"entity": "mention", "value": "1"} have [outside seating]{"entity": "attribute", "value": "outside-seating"}?
    - what is the [price range]{"entity": "attribute", "value": "price-range"} of [Berlin Burrito Company](restaurant)?
    - what about [I due forni](restaurant)?
    - can you tell me the [price range](attribute) of [that restaurant](mention)?
    - what [cuisine](attribute) do [they](mention) have?
```

上述示例仅显示了与餐厅领域相关的示例。你应该将知识库中存在的每种对象类型的示例添加到相同的 `query_knowledge_base` 意图中。

除了为每种查询类型添加各种训练样本外，还需要在训练样本中指定和标注如下实体：

- `object_type`：当训练样本从知识库中引用特定对象类型时，应该将对象类型标记为实体。使用[同义词](/training-data-format#synonyms)进行映射，例如 `restaurants` 到 `restaurant`，将正确的对象类型列为知识库中的键。
- `mention`：如果用户通过“the first one”，“that one”或“it”来引用一个对象，你应该将这些术语标记为 `mention`。还可以使用同义词将一些提及映射到符号。你可以在[解决提及](/action-server/knowledge-bases#resolve-mentions)中了解这一点。
- `attribute`：知识库中定义的所有属性名称都应标识为 NLU 数据中的 `attribute` 。同样，使用同义词将属性名称的变体映射到知识库中使用的变体。

请记住将这些实体添加到你的领域文件中（作为实体和槽）：

```yaml
entities:
  - object_type
  - mention
  - attribute

slots:
  object_type:
    type: any
    influence_conversation: false
    mappings:
    - type: from_entity
      entity: object_type
  mention:
    type: any
    influence_conversation: false
    mappings:
    - type: from_entity
      entity: mention
  attribute:
    type: any
    influence_conversation: false
    mappings:
    - type: from_entity
      entity: attribute
```

### 创建一个动作来查询知识库 {#create-an-action-to-query-your-knowledge-base}

要创建自己的知识库动作，需要继承 `ActionQueryKnowledgeBase` 并将知识库传递给 `ActionQueryKnowledgeBase` 的构造函数。

```python
from rasa_sdk.knowledge_base.storage import InMemoryKnowledgeBase
from rasa_sdk.knowledge_base.actions import ActionQueryKnowledgeBase

class MyKnowledgeBaseAction(ActionQueryKnowledgeBase):
    def __init__(self):
        knowledge_base = InMemoryKnowledgeBase("data.json")
        super().__init__(knowledge_base)
```

每当创建 `ActionQueryKnowledgeBase` 时，都需要将 `KnowledgeBase` 传递给构造函数。它可以是 `InMemoryKnowledgeBase` 或自己的 `KnowledgeBase` 实现（请参阅[创建自己的知识库](/action-server/knowledge-bases#create-a-knowledge-base)）。你只能从一个知识库中提取信息，因为不支持同时使用多个知识库。

这是此动作的全部代码。动作的名称为 `action_query_knowledge_base`，不要忘记将其添加到领域文件中：

```yaml
actions:
- action_query_knowledge_base
```

!!! note "注意"

    如果覆盖默认动作名称 `action_query_knowledge_base`，则需要将如下三个为特征化槽添加到领域文件中：`knowledge_base_objects`、`knowledge_base_last_object` 和 `knowledge_base_last_object_type`。这些槽由 `ActionQueryKnowledgeBase` 在内部使用。如果你保留默认动作名称，则会自动为你添加这些槽。

你还需要确保将故事添加到故事文件中，其中包括意图 `query_knowledge_base` 和动作 `action_query_knowledge_base`。例如：

```yaml
stories:
- story: knowledge base happy path
  steps:
  - intent: greet
  - action: utter_greet
  - intent: query_knowledge_base
  - action: action_query_knowledge_base
  - intent: goodbye
  - action: utter_goodbye
```

需要做的最后一件事是在领域文件中定义 `utter_ask_rephrase` 响应。如果该动作不知道如何处理用户的请求，它将使用此响应来要求用户重新措辞。例如，将如下响应添加到领域文件：

```yaml
responses:
  utter_ask_rephrase:
  - text: "Sorry, I'm not sure I understand. Could you rephrase it?"
  - text: "Could you please rephrase your message? I didn't quite get that."
```

添加所有相关部分后，该动作现在就可以查询数据库了。

## 如何工作 {#how-it-works}

`ActionQueryKnowledgeBase` 会查看在请求中提取的实体以及之前设置的槽，来决定查询什么。

### 查询知识库中的对象 {#query-the-knowledge-base-for-objects}

为了查询任何类型对象的知识库，用户的请求需要包含对象类型。如下为一个例子：

```
Can you please name some restaurants?
```

此问题包括感兴趣的对象类型：“restaurant”。机器人需要获取该实体来制定查询，否则该动作将不知道用户对哪些对象感兴趣。

当用户类似说：

```
What Italian restaurant options in Berlin do I have?
```

用户想要获得：(1). 有意大利美食 (2). 位于柏林的餐馆列表。如果 NER 在用户的请求中检测到这些属性，则动作将使用这些属性来过滤在知识库中找到的餐馆。

为了让对话机器人检测到这些属性，你需要在 NLU 数据中将“Italian”和“Berlin”标记为实体”

```yaml
intents:
- intent: query_knowledge_base
  examples: |
    - What [Italian](cuisine) [restaurant](object_type) options in [Berlin](city) do I have?.
```

属性名称“cuisine”和“city”应该与知识库中使用的名称相同。你还需要将它们作为实体和槽添加到领域文件中。

### 在知识库中查询对象属性 {#query-the-knowledge-base-for-an-attribute-of-an-object}

如果用户想要获取有关对象的特定信息，则请求应包括感兴趣的对象和属性。例如，如果用户提出如下问题：

```
What is the cuisine of Berlin Burrito Company?
```

用户想要获得餐厅“Berlin Burrito Company”（感兴趣的对象）的“cuisine”（感兴趣的属性）。

应将感兴趣的属性和对象标记为 NLU 训练数据中的实体：

```yaml
intents:
- intent: query_knowledge_base
  examples: |
    - What is the [cuisine](attribute) of [Berlin Burrito Company](restaurant)?
```

确保将对象类型“restaurant”作为实体和槽添加到领域文件中。

### 解析提及 {#resolve-mentions}

按照上述示例，用户可能并不总是用他们的名字来指代餐馆。用户可以通过名称来引用感兴趣的对象，例如“Berlin Burrito Company”（对象的表示字符串），或者他们可能通过提及引用先前列出的对象，例如：

```
What is the cuisine of the second restaurant you mentioned?
```

我们的动作能够将这些提及解析为知识库中的实际对象。更具体的说，它可以解析两种提及类型：(1). 序数提及，例如：“the first one”；(2). 例如“it”或“that one”的提及。

#### 序数提及

当用户通过它在列表中的位置来引用一个对象时，它被称为序数提及。如下为一个示例：

```
用户：What restaurants in Berlin do you know?
对话机器人：Found the following objects of type 'restaurant': 1: I due forni 2: PastaBar 3: Berlin Burrito Company
用户：Does the first one have outside seating?
```

用户使用“the first one”一词来指代“I due forni”。其他序数提及可能包括“the second one”，“the last one”，“any”或“3”。

当向用户呈现对象列表时，通常使用序数提及。为了将这些提及解析为实际对象，我们使用在 `KnowledgeBase` 类中设置的序数提及映射。默认映射如下所示：

```json
{
    "1": lambda l: l[0],
    "2": lambda l: l[1],
    "3": lambda l: l[2],
    "4": lambda l: l[3],
    "5": lambda l: l[4],
    "6": lambda l: l[5],
    "7": lambda l: l[6],
    "8": lambda l: l[7],
    "9": lambda l: l[8],
    "10": lambda l: l[9],
    "ANY": lambda l: random.choice(l),
    "LAST": lambda l: l[-1],
}
```

序数提及映射将字符串（例如“1”）映射到列表中的对象，例如 `lambda l: l[0]` 表示索引为 `0` 的对象。

例如，由于序数提及映射不包含“the first one”条目，因此使用[实体同义词](/training-data-format#synonyms)将 NLU 数据中的“the first one”映射到“1”很重要：

```yaml
intents:
- intent: query_knowledge_base
  examples: |
    - Does the [first one]{entity: "mention", value": 1} have [outside seating]{entity: "attribute", value": "outside-seating"}
```

NER 将“first one”检测为提及实体，但将“1”放入提及槽。因此，我们的动作可以将提及槽与序数提及映射一起使用，来将“first one”解析为实际对象“I due forni”。

你可以通过在知识库实现上调用 `set_ordinal_mention_mapping()` 函数来覆盖序数提及映射（请参阅[自定义 `InMemoryKnowledgeBase`](/action-server/knowledge-bases#customizing-the-inmemoryknowledgebase)）。

#### 其他提及

查看如下对话：

```
用户：What is the cuisine of PastaBar?
对话机器人：PastaBar has an Italian cuisine.
用户：Does it have wifi?
对话机器人：Yes.
用户：Can you give me an address?
```

在“Does it have wifi?”的问题中，用户通过“it”这个词来指代“PastaBar”。如果 NER 检测到“it”作为实体提及，知识库操作会将其解析为对话中最后提到的对象“PastaBar”。

在下一个输入中，用户间接引用对象“PastaBar”，而不是明确提及它。知识库动作将检测用户想要获取特定属性的值，在本例中为地址。如果 NER 未检测到提及或对象，则该动作假定用户指的是最近提及的对象“PastaBar”。

你可以通过在初始化动作时将 `use_last_object_mention` 设置为 `False` 来禁用此行为。

## 自定义 {#customization}

### 自定义 `ActionQueryKnowledgeBase` {#customizing-actionqueryknowledgebase}

如果你想自定义对话机器人对用户说的内容，可以覆盖 `ActionQueryKnowledgeBase` 的如下两个函数：

- `utter_objects()`
- `utter_attribute_value()`

当用户请求对象列表时使用 `utter_objects()`。一旦对话机器人从知识库中检索到对象，它将默认使用一条消息响应用户，格式如下：

```
Found the following objects of type 'restaurant': 1: I due forni 2: PastaBar 3: Berlin Burrito Company
```

或者如果没有找到对象：

```
I could not find any objects of type 'restaurant'.
```

如果你想更改话语格式，可以在动作中覆盖 `utter_objects()` 方法。

函数 `utter_attribute_value()` 确定当用户询问有关对象的特定信息时对话机器人会说什么。

如果在知识库中找到感兴趣的属性，对话机器人将用如下话语进行响应：

```
'Berlin Burrito Company' has the value 'Mexican' for attribute 'cuisine'.
```

如果没有找到所请求属性的值，对话机器人将响应：

```
Did not find a valid value for attribute 'cuisine' for object 'Berlin Burrito Company'.
```

如果要更改对话机器人话语，可以覆盖 `utter_attribute_value()` 方法。

!!! note "注意"

    博客上有一个关于如何在自定义动作中使用知识库的[教程](https://blog.rasa.com/integrating-rasa-with-knowledge-bases/)。本教程详细解释了 `ActionQueryKnowledgeBase` 背后的实现。

### 创建自己的知识库动作 {#creating-your-own-knowledge-base-actions}

`ActionQueryKnowledgeBase` 应允许你轻松开始将知识库集成到你的动作中，但是，该动作只能处理两种用户请求：

- 用户想要从知识库中获取对象列表
- 用户想要获取特定对象的属性值

该动作无法比较对象或考虑知识库中对象之间的关系。此外，在对话中解析对最后提到的对象的任何提及可能并不总是最佳的。

如果你想处理更复杂的用力，可以编写自己的自定义动作。我们在 `rasa_sdk.knowledge_base.utils`（[代码](https://github.com/RasaHQ/rasa-sdk/tree/main/rasa_sdk/knowledge_base/)）中添加了一些帮助函数来帮助你实现自己的解决方案。我们建议使用 `KnowledgeBase` 接口以便你仍可以将 `ActionQueryKnowledgeBase` 与新的自定义动作一起使用。

如果你编写了解决上述用例之一或新用例的知识库动作，请务必在[论坛](https://forum.rasa.com/)上告诉我们。

### 自定义 `InMemoryKnowledgeBase` {#customizing-the-inmemoryknowledgebase}

`InMemoryKnowledgeBase` 类继承 `KnowledgeBase`。你可以通过覆盖如下函数来自定义 `InMemoryKnowledgeBase`：

- `get_key_attribute_of_object()`：为了追踪用户上次谈论的对象，我们将键属性的值存储在特定的槽中。每个对象都应该有一个唯一的键属性，类似于关系数据库中的主键。默认情况下，每个对象类型的键属性名称都设置为 `id`。你可以通过调用 `set_key_attribute_of_object()` 覆盖特定对象类型的键属性的名称。
- `get_representation_function_of_object()`：让我们关注如下餐厅：

    ```json
    {
        "id": 0,
        "name": "Donath",
        "cuisine": "Italian",
        "outside-seating": true,
        "price-range": "mid-range"
    }
    ```

    当用户要求对话机器人列出所有意大利餐厅时，它不需要餐厅的所有详细信息。相反，你希望提供一个有意义的名称来标识餐厅，在大多数情况下，对象的名称就可以。函数 `get_representation_function_of_object()` 返回一个将上述餐厅对象映射到其名称的 lambda 函数。

    ```python
    lambda obj: obj["name"]
    ```

    每当对话机器人谈论特定对象时，都会使用此方法，以便为用户提供一个有意义的对象名称。

    默认情况下，lambda 函数返回对象的 `name` 属性的值。如果你的对象没有 `name` 属性，或者对象的 `name` 不明确，则应通过调用 `set_representation_function_of_object()` 为该对象设置一个新的 lambda 函数。

    ```json
    {
        "1": lambda l: l[0],
        "2": lambda l: l[1],
        "3": lambda l: l[2],
        "4": lambda l: l[3],
        "5": lambda l: l[4],
        "6": lambda l: l[5],
        "7": lambda l: l[6],
        "8": lambda l: l[7],
        "9": lambda l: l[8],
        "10": lambda l: l[9],
        "ANY": lambda l: random.choice(l),
        "LAST": lambda l: l[-1],
    }
    ```

    你可以通过调用 `set_ordinal_mention_mapping()` 函数来覆盖它。如果你想了解有关如何使用此映射的更多信息，请参见[解析提及](/action-server/knowledge-bases#resolve-mentions)。

有关使用 `set_representation_function_of_object()` 方法覆盖对象类型“hotel”的默认表示的 `InMemoryKnowledgeBase` 的示例实现，请参见[示例对话机器人](https://github.com/RasaHQ/rasa/blob/main/examples/knowledgebasebot/actions/actions.py)。`InMemoryKnowledgeBase` 本身的实现可以在 [rasa-sdk](https://github.com/RasaHQ/rasa-sdk/tree/main/rasa_sdk/knowledge_base/) 包中找到。

### 创建自己的知识库 {#creating-your-own-knowledge-base}

如果你有更多数据，或者如果你想使用更复杂的数据结构，例如涉及不同对象之间的关系，可以创建自己的知识库实现。只要继承 `KnowledgeBase` 并实现 `get_objects()`、`get_object()` 和 `get_attributes_of_object()` 方法。[知识库代码](https://github.com/RasaHQ/rasa-sdk/tree/main/rasa_sdk/knowledge_base/)提供了有关这些方法应该做什么的更多信息。

你还可以通过调整[自定义 `InMemoryKnowledgeBase`](/action-server/knowledge-bases#customizing-the-inmemoryknowledgebase) 部分中提到的方法来进一步自定义你的知识库。

!!! note "注意"

    我们写了一篇[博客](https://blog.rasa.com/set-up-a-knowledge-base-to-encode-domain-knowledge-for-rasa/)来解释如何建立自己的知识库。
