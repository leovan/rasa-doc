# 自定义图组件

你可以使用自定义 NLU 组件和策略扩展开源 Rasa。本页面提供了有关如何开发自定义图组件的指南。

开源 Rasa 提供了各种开箱即用的 [NLU 组件](/components)和[策略](/policies)，通过自定义图组件可以从头对其进行自定义或创建自己的组件。

要将你的自定义图组件同开源 Rasa 一起使用，它必须满足一下条件：

- 必须实现 [`GraphComponent` 接口](/custom-graph-components/#the-graphcomponent-interface)。
- 必须与使用的模型配置[注册](/custom-graph-components/#registering-graph-components-with-the-model-configuration)。
- 必须在[配置文件](/custom-graph-components/#using-custom-components-in-your-model-configuration)中使用。
- 必须使用类型注释。开源 Rasa 使用类型注释来验证你的模型配置。不允许[前向引用](https://www.python.org/dev/peps/pep-0484/#forward-references)。如果使用的是 Python 3.7，可以通过 [`from __future__ import annotations`](https://www.python.org/dev/peps/pep-0563/#enabling-the-future-behavior-in-python-3-7) 来摆脱前向引用。

## 图组件 {#graph-components}

开源 Rasa 使用传入的[模型配置](/model-configuration)来构建[有向无环图](https://en.wikipedia.org/wiki/Directed_acyclic_graph)。此图描述了模型配置中的项目之间的依赖关系以及数据在它们之间的流动方式。这主要有两个好处：

- 开源 Rasa 可以使用计算图来优化模型的执行。这方面的示例是有效缓存训练步骤或并行执行独立步骤。
- 开源 Rasa 可以灵活地表示不同的模型架构。只要图保持非循环，开源 Rasa 理论上可以根据模型配置将任何数据传递给任何图组件，而无需将底层软件架构与使用的模型架构联系起来。

当将模型配置转换为计算图时，[策略](/policies)和 [NLU 组件](/components)成为该图中的节点。虽然模型配置中的策略和 NLU 组件之间存在区别，但是当他们被放置在图中时，区别就被抽象掉了。此时策略和 NLU 组件成为抽象图组件。在实践中，这由 [`GraphComponent`](/custom-graph-components#the-graphcomponent-interface) 接口表示：策略和 NLU 组件都必须继承该接口，才能与 Rasa 的图兼容并可执行。

<figure markdown>
  ![](/images/custom-graph-components/graph-architecture.png){ width="600" }
  <figcaption>图架构</figcaption>
</figure>

## 开始 {#getting-started}

在开始之前，你需要决定要实现自定义 [NLU 组件](/components)还是[策略](/policies)。如果实现自定义策略，我们建议继承 `rasa.core.policies.policy.Policy` 类，该类实现了 `GraphComponent` 接口。

```python
from rasa.core.policies.policy import Policy
from rasa.engine.recipes.default_recipe import DefaultV1Recipe

# TODO: Correctly register your graph component
@DefaultV1Recipe.register(
    [DefaultV1Recipe.ComponentType.POLICY_WITHOUT_END_TO_END_SUPPORT], is_trainable=True
)
class MyPolicy(Policy):
    ...
```

如果要实现自定义 NLU 组件，可以从如下框架开始：

```python
from typing import Dict, Text, Any, List

from rasa.engine.graph import GraphComponent, ExecutionContext
from rasa.engine.recipes.default_recipe import DefaultV1Recipe
from rasa.engine.storage.resource import Resource
from rasa.engine.storage.storage import ModelStorage
from rasa.shared.nlu.training_data.message import Message
from rasa.shared.nlu.training_data.training_data import TrainingData

# TODO: Correctly register your component with its type
@DefaultV1Recipe.register(
    [DefaultV1Recipe.ComponentType.INTENT_CLASSIFIER], is_trainable=True
)
class CustomNLUComponent(GraphComponent):
    @classmethod
    def create(
        cls,
        config: Dict[Text, Any],
        model_storage: ModelStorage,
        resource: Resource,
        execution_context: ExecutionContext,
    ) -> GraphComponent:
        # TODO: Implement this
        ...

    def train(self, training_data: TrainingData) -> Resource:
        # TODO: Implement this if your component requires training
        ...

    def process_training_data(self, training_data: TrainingData) -> TrainingData:
        # TODO: Implement this if your component augments the training data with
        #       tokens or message features which are used by other components
        #       during training.
        ...

        return training_data

    def process(self, messages: List[Message]) -> List[Message]:
        # TODO: This is the method which Rasa Open Source will call during inference.
        ...
        return messages
```

阅读如下部分了解如何解决上述示例中的 `TODO` 以及需要在自定义组件中实现哪些其他方法。

!!! info "自定义分词器"

    如果你创建自定义分词器，你应该继承 `rasa.nlu.tokenizers.tokenizer.Tokenizer` 类。`train` 和 `process` 方法已经实现，因此仅需要重写 `tokenize` 方法。

## `GraphComponent` 接口 {#the-graphcomponent-interface}

要使用开源 Rasa 运行自定义的 NLU 组件或策略，它必须实现 `GraphComponent` 接口。

```python
from __future__ import annotations
from abc import ABC, abstractmethod
from typing import List, Type, Dict, Text, Any, Optional

from rasa.engine.graph import ExecutionContext
from rasa.engine.storage.resource import Resource
from rasa.engine.storage.storage import ModelStorage


class GraphComponent(ABC):
    """Interface for any component which will run in a graph."""

    @classmethod
    def required_components(cls) -> List[Type]:
        """Components that should be included in the pipeline before this component."""
        return []

    @classmethod
    @abstractmethod
    def create(
        cls,
        config: Dict[Text, Any],
        model_storage: ModelStorage,
        resource: Resource,
        execution_context: ExecutionContext,
    ) -> GraphComponent:
        """Creates a new `GraphComponent`.

        Args:
            config: This config overrides the `default_config`.
            model_storage: Storage which graph components can use to persist and load
                themselves.
            resource: Resource locator for this component which can be used to persist
                and load itself from the `model_storage`.
            execution_context: Information about the current graph run.

        Returns: An instantiated `GraphComponent`.
        """
        ...

    @classmethod
    def load(
        cls,
        config: Dict[Text, Any],
        model_storage: ModelStorage,
        resource: Resource,
        execution_context: ExecutionContext,
        **kwargs: Any,
    ) -> GraphComponent:
        """Creates a component using a persisted version of itself.

        If not overridden this method merely calls `create`.

        Args:
            config: The config for this graph component. This is the default config of
                the component merged with config specified by the user.
            model_storage: Storage which graph components can use to persist and load
                themselves.
            resource: Resource locator for this component which can be used to persist
                and load itself from the `model_storage`.
            execution_context: Information about the current graph run.
            kwargs: Output values from previous nodes might be passed in as `kwargs`.

        Returns:
            An instantiated, loaded `GraphComponent`.
        """
        return cls.create(config, model_storage, resource, execution_context)

    @staticmethod
    def get_default_config() -> Dict[Text, Any]:
        """Returns the component's default config.

        Default config and user config are merged by the `GraphNode` before the
        config is passed to the `create` and `load` method of the component.

        Returns:
            The default config of the component.
        """
        return {}

    @staticmethod
    def supported_languages() -> Optional[List[Text]]:
        """Determines which languages this component can work with.

        Returns: A list of supported languages, or `None` to signify all are supported.
        """
        return None

    @staticmethod
    def not_supported_languages() -> Optional[List[Text]]:
        """Determines which languages this component cannot work with.

        Returns: A list of not supported languages, or
            `None` to signify all are supported.
        """
        return None

    @staticmethod
    def required_packages() -> List[Text]:
        """Any extra python dependencies required for this component to run."""
        return []
```

### `create` {#create}

`create` 方法用于在训练期间实例化图组件，其必须被重写。开源 Rasa 在调用该方法时会传递如下参数：

- `config`：组件的默认配置，会与模型配置文件中提供给图组件的配置合并。
- `model_storage`：可以使用它来持久化和加载图组件。有关其用法的更多信息，请参见[模型持久化](/custom-graph-components/#model-persistence)部分。
- `resource`：`model_storage` 中组件的唯一标识符。有关其用法的更多信息，请参见[模型持久化](/custom-graph-components/#model-persistence)部分。
- `execution_context`：其提供了有关当前执行模式的附加信息。
- `model_id`：推理期间使用的模型的唯一标识符。此参数在训练期间为 `None`。
- `should_add_diagnostic_data`：如果为 `True`，则应在实际预测之上将额外的诊断元数据添加到图组件的预测中。
- `is_finetuning`：如果为 `True`，则可以使用[微调](/command-line-interface#incremental-training)来训练图组件。
- `graph_schema`：`graph_schema` 表述了用于训练对话机器人或使用它进行预测的计算图。
- `node_name`：`node_name` 是图模式中步骤的唯一标识符，由调用的图组件实现。

### `load` {#load}

`load` 方法用于在推理期间实例化图组件。此方法的默认实现调用 `create` 方法。如果图组件[将数据保留为训练的一部分](/custom-graph-components/#model-persistence)，建议重写该方法。有关各参数的描述，请参见 [`create`](/custom-graph-components/#create)。

### `get_default_config` {#get_default_config}

`get_default_config` 方法返回图组件的默认配置。其默认实现返回一个空字典，这意味着图组件没有任何配置。开源 Rasa 将在运行时使用配置文件中值更新默认配置。

### `supported_languages` {#supported_languages}

`supported_languages` 方法指定图组件支持哪些语言。开源 Rasa 将使用模型配置文件中的 `language` 键来验证图组件是否适用于指定[语言](https://en.wikipedia.org/wiki/List_of_ISO_639-1_codes)。如果图组件返回 `None`（默认实现），则表明图组件支持所有不属于 `not_supported_languages` 的语言。

示例：

- `[]`：图组件不支持任何语言。
- `None`：支持所有语言，除了 `not_supported_languages` 中定义的语言。
- `["en"]`：图组件只支持英语。

### `not_supported_languages` {#not_supported_languages}

`not_supported_languages` 方法指定图组件不支持哪些语言。开源 Rasa 将使用模型配置文件中的 `language` 键来验证图组件是否适用于指定[语言](https://en.wikipedia.org/wiki/List_of_ISO_639-1_codes)。如果图组件返回 `None`（默认实现），则表明它支持在 `supported_languages` 中指定的所有语言。

示例：

- `None` 或 `[]`：支持 `supported_languages` 中指定的所有语言。
- `["en"]`：图形组件可以使用除英语之外的任何语言。

### `required_packages` {#required_packages}

`required_packages` 方法指示需要安装哪些额外的 Python 包才能使用此图组件。如果在运行时找不到所需的库，开源 Rasa 将在执行期间引发错误。默认情况下，此方法返回一个空列表，这意味着图组件没有任何额外的依赖项。

示例：

- `[]`：使用此图形组件不需要额外的包。
- `["spacy"]`：需要安装 spacy 包才能使用此图组件。

## 模型持久化 {#model-persistence}

一些图组件在训练期间需要持久化数据，这些数据应该在推理时可供图组件使用。一个典型的用例是存储模型权重。为此，开源 Rasa 为图组件的 `create` 和 `load` 方法提供了 `model_storage` 和 `resource` 参数，如下面代码片段所示。`model_storage` 提供对来自所有图组件数据的访问。该 `resource` 允许你唯一地标识图组件在模型存储中的位置。

```python hl_lines="14 15 24 25"
from __future__ import annotations

from typing import Any, Dict, Text

from rasa.engine.graph import GraphComponent, ExecutionContext
from rasa.engine.storage.resource import Resource
from rasa.engine.storage.storage import ModelStorage

class MyComponent(GraphComponent):
    @classmethod
    def create(
        cls,
        config: Dict[Text, Any],
        model_storage: ModelStorage,
        resource: Resource,
        execution_context: ExecutionContext,
    ) -> MyComponent:
        ...

    @classmethod
    def load(
        cls,
        config: Dict[Text, Any],
        model_storage: ModelStorage,
        resource: Resource,
        execution_context: ExecutionContext,
        **kwargs: Any
    ) -> MyComponent:
        ...
```

### 写入模型存储 {#writing-to-the-model-storage}

如下代码片段说明了如何将图组件的数据写入模型存储。要在训练后保留图组件，`train` 方法需要访问 `model_storage` 和 `resource` 的值。因此，你应该在初始化时存储 `model_storage` 和 `resource` 的值。

图组件的 `train` 方法必须返回 `resource` 的值，以便开源 Rasa 可以缓存训练中的训练结果。`self._model_storage.write_to(self._resource)` 上下文管理器提供了一个目录路径，你可以在其中保存图组件所需的任何数据。

```python
from __future__ import annotations
import json
from typing import Optional, Dict, Any, Text

from rasa.engine.graph import GraphComponent, ExecutionContext
from rasa.engine.storage.resource import Resource
from rasa.engine.storage.storage import ModelStorage
from rasa.shared.nlu.training_data.training_data import TrainingData

class MyComponent(GraphComponent):

    def __init__(
        self,
        model_storage: ModelStorage,
        resource: Resource,
        training_artifact: Optional[Dict],
    ) -> None:
        # Store both `model_storage` and `resource` as object attributes to be able
        # to utilize them at the end of the training
        self._model_storage = model_storage
        self._resource = resource

    @classmethod
    def create(
        cls,
        config: Dict[Text, Any],
        model_storage: ModelStorage,
        resource: Resource,
        execution_context: ExecutionContext,
    ) -> MyComponent:
        return cls(model_storage, resource, training_artifact=None)

    def train(self, training_data: TrainingData) -> Resource:
        # Train your graph component
        ...

        # Persist your graph component
        with self._model_storage.write_to(self._resource) as directory_path:
            with open(directory_path / "artifact.json", "w") as file:
                json.dump({"my": "training artifact"}, file)

        # Return resource to make sure the training artifacts
        # can be cached.
        return self._resource
```

### 从模型存储中读取 {#reading-from-the-model-storage}

开源 Rasa 将调用图组件的 `load` 方法来实例化它从而进行推理。你可以使用 `self._model_storage.read_from(resource)` 上下文管理器来获取保存图组件数据的目录路径。使用提供的路径可以加载持久化数据并使用它初始化图组件。请注意，如果没有为给定 `resource` 找到持久化数据，`model_storage` 将抛出 `ValueError`。

```python
from __future__ import annotations
import json
from typing import Optional, Dict, Any, Text

from rasa.engine.graph import GraphComponent, ExecutionContext
from rasa.engine.storage.resource import Resource
from rasa.engine.storage.storage import ModelStorage

class MyComponent(GraphComponent):

    def __init__(
        self,
        model_storage: ModelStorage,
        resource: Resource,
        training_artifact: Optional[Dict],
    ) -> None:
        self._model_storage = model_storage
        self._resource = resource

    @classmethod
    def load(
        cls,
        config: Dict[Text, Any],
        model_storage: ModelStorage,
        resource: Resource,
        execution_context: ExecutionContext,
        **kwargs: Any,
    ) -> MyComponent:
        try:
            with model_storage.read_from(resource) as directory_path:
                with open(directory_path / "artifact.json", "r") as file:
                    training_artifact = json.load(file)
                    return cls(
                        model_storage, resource, training_artifact=training_artifact
                    )
        except ValueError:
            # This allows you to handle the case if there was no
            # persisted data for your component
            ...
```

## 使用模型配置注册图组件 {#registering-graph-components-with-the-model-configuration}

要使图组件可用于开源 Rasa，你可能必须使用一种配方注册图组件。开源 Rasa 使用配方将模型配置内容转换为可执行[图](/custom-graph-components#graph-components)。目前，开源 Rasa 支持 `default.v1` 和实验性的 `graph.v1` 配方。对于 `graph.v1` 配方，你需要使用 `DefaultV1Recipe.register` 装饰器注册图组件：

```python
from rasa.engine.graph import GraphComponent
from rasa.engine.recipes.default_recipe import DefaultV1Recipe


@DefaultV1Recipe.register(
    component_types=[DefaultV1Recipe.ComponentType.INTENT_CLASSIFIER],
    is_trainable=True,
    model_from="SpacyNLP",
)
class MyComponent(GraphComponent):
    ...
```

开源 Rasa 使用 `register` 饰器提供的信息和图组件在配置文件中的位置来规划图组件及其所需数据的执行。`DefaultV1Recipe.register` 装饰器允许你指定如下详细信息：

- `component_types`：这指定了图组件在对话机器人中的用途。可以指定为多种类型（例如：如果图组件既是意图分类器又是实体提取器）：
    - `ComponentType.MODEL_LOADER`：[语言模型](/components#language-models)的组件类型。这种类型的图组件为其他图组件的 `train`、`process_training_data` 和 `process` 方法提供预训练模型，如果它们指定了 `model_from=<model loader name>`。此图组件在训练和推理期间运行。开源 Rasa 将使用图组件的 `provide` 方法来检索应该提供给依赖图组件的模型。
    - `ComponentType.MESSAGE_TOKENIZER`：[分词器](/components#tokenizers)的组件类型。此图组件在训练和推理期间运行。如果指定了 `is_trainable=True`，开源 Rasa 将使用图组件的 `train` 方法。开源 Rasa 将使用 `process_training_data` 对训练数据样本进行分词，并在推理过程中使用 `process` 对消息进行分词。
    - `ComponentType.MESSAGE_FEATURIZER`：[特征化器](/components#featurizers)的组件类型。此图组件在训练和推理期间运行。如果指定了 `is_trainable=True`，开源 Rasa 将使用图组件的 `train` 方法。开源 Rasa 将使用 `process_training_data` 对训练数据样本进行特征化，并在推理过程中使用 `process` 对消息进行特征化。
    - `ComponentType.INTENT_CLASSIFIER`：[意图分类器](/components#intent-classifiers)的组件类型。如果 `is_trainable=True`，此图组件仅在训练期间运行。图组件始终在推理期间运行。如果指定了 `is_trainable=True`，开源 Rasa 将使用图组件的 `train` 方法。开源 Rasa 将在推理过程中使用 `process` 对消息的意图进行分类。
    - `ComponentType.ENTITY_EXTRACTOR`：[实体提取器](/components#entity-extractors)的组件类型。如果 `is_trainable=True`，此图组件仅在训练期间运行。图组件始终在推理期间运行。如果指定了 `is_trainable=True`，开源 Rasa 将使用图组件的 `train` 方法。开源 Rasa 将在推理过程中使用 `process` 来提取实体。
    - `ComponentType.POLICY_WITHOUT_END_TO_END_SUPPORT`：不需要额外端到端功能的[策略](/rasa/policies)的组件类型（更多信息请参见[端到端训练](/stories#end-to-end-training)）。如果 `is_trainable=True`，此图组件仅在训练期间运行。图组件始终在推理期间运行。如果指定了 `is_trainable=True`，开源 Rasa 将使用图组件的 `train` 方法。开源 Rasa 将使用图形组件的 `predict_action_probabilities` 来预测应该在对话中运行的下一个动作。
    - `ComponentType.POLICY_WITH_END_TO_END_SUPPORT`：需要额外端到端功能的[策略](/rasa/policies)的组件类型（更多信息请参见[端到端训练](/stories#end-to-end-training)）。端到端特征作为预计算参数传递到图组件的 `train` 和 `predict_action_probabilities` 中。如果 `is_trainable=True`，此图组件仅在训练期间运行。图组件始终在推理期间运行。如果指定了 `is_trainable=True`，开源 Rasa 将使用图组件的 `train` 方法。开源 Rasa 将使用图形组件的 `predict_action_probabilities` 来预测应该在对话中运行的下一个动作。
- `is_trainable`：指定图组件是否需要在处理其他依赖图组件的训练数据之前或在它可以进行预测之前进行自训练。
- `model_from`：指定是否需要为图组件的 `train`、`process_training_data` 和 `process` 方法提供预训练的[语言模型](/components#language-models)。这些方法必须支持 `model` 参数才能接收语言模型。请注意，你仍然需要确保提供此模型的图组件是模型配置的一部分。一个常见的用例是如果你想将 [SpacyNLP](/components#spacynlp) 语言模型暴露给你的其他 NLU 组件。

## 在模型配置中使用自定义组件 {#using-custom-components-in-your-model-configuration}

你可以像使用[模型配置](/model-configuration)中的任何其他 NLU 组件或策略一样使用自定义图组件。唯一的变化是你必须指定完整的模块名称而不是仅指定类名称。完整的模块名称取决于你的模块相对于指定 [PYTHONPATH](https://docs.python.org/3/using/cmdline.html#envvar-PYTHONPATH) 的位置。默认情况下，开源 Rasa 会将运行 CLI 的目录添加到 `PYTHONPATH` 中。如果例如从 `/Users/<user>/my-rasa-project` 运行 CLI 并且模块 `MyComponent` 位于 `/Users/<user>/my-rasa-project/custom_components/my_component.py` 中，则模块路径为 `custom_components.my_component.MyComponent`。除了 `name` 条目外所有内容都将作为 `config` 传递给组件。

```yaml title="config.yml" hl_lines="5 6"
recipe: default.v1
language: en
pipeline:
# other NLU components
- name: your.custom.NLUComponent
  setting_a: 0.01
  setting_b: string_value

policies:
# other dialogue policies
- name: your.custom.Policy
```

## 实现提示 {#implementation-hints}

### 消息元数据 {#message-metadata}

当[在训练数据中为意图样本定义元数据](/training-data-format#training-examples)时，NLU 组件可以在处理期间访问意图元数据和意图样本元数据：

```python
# in your component class

    def process(self, message: Message, **kwargs: Any) -> None:
        metadata = message.get("metadata")
        print(metadata.get("intent"))
        print(metadata.get("example"))
```

### 稀疏和稠密消息特征 {#sparse-and-dense-message-features}

如果你创建自定义消息特征化器，你可以返回两种不同的特征：序列特征和句子特征。序列特征是一个（$\text{词条数量} \times \text{特征维度}$）大小的矩阵，即该矩阵包含序列中每个词条的特征向量。句子特征由一个（$1 \times \text{特征维度}$）表示。

## 自定义组件示例 {#examples-of-custom-components}

### 稠密消息特征化器 {#dense-message-featurizer}

以下是使用预训练模型的稠密[消息特征化器](/components#featurizers)的示例：

```python
import numpy as np
import logging
from bpemb import BPEmb
from typing import Any, Text, Dict, List, Type

from rasa.engine.recipes.default_recipe import DefaultV1Recipe
from rasa.engine.graph import ExecutionContext, GraphComponent
from rasa.engine.storage.resource import Resource
from rasa.engine.storage.storage import ModelStorage
from rasa.nlu.featurizers.dense_featurizer.dense_featurizer import DenseFeaturizer
from rasa.nlu.tokenizers.tokenizer import Tokenizer
from rasa.shared.nlu.training_data.training_data import TrainingData
from rasa.shared.nlu.training_data.features import Features
from rasa.shared.nlu.training_data.message import Message
from rasa.nlu.constants import (
    DENSE_FEATURIZABLE_ATTRIBUTES,
    FEATURIZER_CLASS_ALIAS,
)
from rasa.shared.nlu.constants import (
    TEXT,
    TEXT_TOKENS,
    FEATURE_TYPE_SENTENCE,
    FEATURE_TYPE_SEQUENCE,
)


logger = logging.getLogger(__name__)


@DefaultV1Recipe.register(
    DefaultV1Recipe.ComponentType.MESSAGE_FEATURIZER, is_trainable=False
)
class BytePairFeaturizer(DenseFeaturizer, GraphComponent):
    @classmethod
    def required_components(cls) -> List[Type]:
        """Components that should be included in the pipeline before this component."""
        return [Tokenizer]

    @staticmethod
    def required_packages() -> List[Text]:
        """Any extra python dependencies required for this component to run."""
        return ["bpemb"]

    @staticmethod
    def get_default_config() -> Dict[Text, Any]:
        """Returns the component's default config."""
        return {
            **DenseFeaturizer.get_default_config(),
            # specifies the language of the subword segmentation model
            "lang": None,
            # specifies the dimension of the subword embeddings
            "dim": None,
            # specifies the vocabulary size of the segmentation model
            "vs": None,
            # if set to True and the given vocabulary size can't be loaded for the given
            # model, the closest size is chosen
            "vs_fallback": True,
        }

    def __init__(
        self,
        config: Dict[Text, Any],
        name: Text,
    ) -> None:
        """Constructs a new byte pair vectorizer."""
        super().__init__(name, config)
        # The configuration dictionary is saved in `self._config` for reference.
        self.model = BPEmb(
            lang=self._config["lang"],
            dim=self._config["dim"],
            vs=self._config["vs"],
            vs_fallback=self._config["vs_fallback"],
        )

    @classmethod
    def create(
        cls,
        config: Dict[Text, Any],
        model_storage: ModelStorage,
        resource: Resource,
        execution_context: ExecutionContext,
    ) -> GraphComponent:
        """Creates a new component (see parent class for full docstring)."""
        return cls(config, execution_context.node_name)

    def process(self, messages: List[Message]) -> List[Message]:
        """Processes incoming messages and computes and sets features."""
        for message in messages:
            for attribute in DENSE_FEATURIZABLE_ATTRIBUTES:
                self._set_features(message, attribute)
        return messages

    def process_training_data(self, training_data: TrainingData) -> TrainingData:
        """Processes the training examples in the given training data in-place."""
        self.process(training_data.training_examples)
        return training_data

    def _create_word_vector(self, document: Text) -> np.ndarray:
        """Creates a word vector from a text. Utility method."""
        encoded_ids = self.model.encode_ids(document)
        if encoded_ids:
            return self.model.vectors[encoded_ids[0]]

        return np.zeros((self.component_config["dim"],), dtype=np.float32)

    def _set_features(self, message: Message, attribute: Text = TEXT) -> None:
        """Sets the features on a single message. Utility method."""
        tokens = message.get(TEXT_TOKENS)

        # If the message doesn't have tokens, we can't create features.
        if not tokens:
            return None

        # We need to reshape here such that the shape is equivalent to that of sparsely
        # generated features. Without it, it'd be a 1D tensor. We need 2D (n_utterance, n_dim).
        text_vector = self._create_word_vector(document=message.get(TEXT)).reshape(
            1, -1
        )
        word_vectors = np.array(
            [self._create_word_vector(document=t.text) for t in tokens]
        )

        final_sequence_features = Features(
            word_vectors,
            FEATURE_TYPE_SEQUENCE,
            attribute,
            self._config[FEATURIZER_CLASS_ALIAS],
        )
        message.add_features(final_sequence_features)
        final_sentence_features = Features(
            text_vector,
            FEATURE_TYPE_SENTENCE,
            attribute,
            self._config[FEATURIZER_CLASS_ALIAS],
        )
        message.add_features(final_sentence_features)

    @classmethod
    def validate_config(cls, config: Dict[Text, Any]) -> None:
        """Validates that the component is configured properly."""
        if not config["lang"]:
            raise ValueError("BytePairFeaturizer needs language setting via `lang`.")
        if not config["dim"]:
            raise ValueError(
                "BytePairFeaturizer needs dimensionality setting via `dim`."
            )
        if not config["vs"]:
            raise ValueError("BytePairFeaturizer needs a vector size setting via `vs`.")
```

### 稀疏特征化器 {#sparse-message-featurizer}

以下是训练新模型的稀疏[消息特征化器](/components#featurizers)的示例：

```python
import logging
from typing import Any, Text, Dict, List, Type

from sklearn.feature_extraction.text import TfidfVectorizer
from rasa.engine.recipes.default_recipe import DefaultV1Recipe
from rasa.engine.graph import ExecutionContext, GraphComponent
from rasa.engine.storage.resource import Resource
from rasa.engine.storage.storage import ModelStorage
from rasa.nlu.featurizers.sparse_featurizer.sparse_featurizer import SparseFeaturizer
from rasa.nlu.tokenizers.tokenizer import Tokenizer
from rasa.shared.nlu.training_data.training_data import TrainingData
from rasa.shared.nlu.training_data.features import Features
from rasa.shared.nlu.training_data.message import Message
from rasa.nlu.constants import (
    DENSE_FEATURIZABLE_ATTRIBUTES,
    FEATURIZER_CLASS_ALIAS,
)
from joblib import dump, load
from rasa.shared.nlu.constants import (
    TEXT,
    TEXT_TOKENS,
    FEATURE_TYPE_SENTENCE,
    FEATURE_TYPE_SEQUENCE,
)

logger = logging.getLogger(__name__)


@DefaultV1Recipe.register(
    DefaultV1Recipe.ComponentType.MESSAGE_FEATURIZER, is_trainable=True
)
class TfIdfFeaturizer(SparseFeaturizer, GraphComponent):
    @classmethod
    def required_components(cls) -> List[Type]:
        """Components that should be included in the pipeline before this component."""
        return [Tokenizer]

    @staticmethod
    def required_packages() -> List[Text]:
        """Any extra python dependencies required for this component to run."""
        return ["sklearn"]

    @staticmethod
    def get_default_config() -> Dict[Text, Any]:
        """Returns the component's default config."""
        return {
            **SparseFeaturizer.get_default_config(),
            "analyzer": "word",
            "min_ngram": 1,
            "max_ngram": 1,
        }

    def __init__(
        self,
        config: Dict[Text, Any],
        name: Text,
        model_storage: ModelStorage,
        resource: Resource,
    ) -> None:
        """Constructs a new tf/idf vectorizer using the sklearn framework."""
        super().__init__(name, config)
        # Initialize the tfidf sklearn component
        self.tfm = TfidfVectorizer(
            analyzer=config["analyzer"],
            ngram_range=(config["min_ngram"], config["max_ngram"]),
        )

        # We need to use these later when saving the trained component.
        self._model_storage = model_storage
        self._resource = resource

    def train(self, training_data: TrainingData) -> Resource:
        """Trains the component from training data."""
        texts = [e.get(TEXT) for e in training_data.training_examples if e.get(TEXT)]
        self.tfm.fit(texts)
        self.persist()
        return self._resource

    @classmethod
    def create(
        cls,
        config: Dict[Text, Any],
        model_storage: ModelStorage,
        resource: Resource,
        execution_context: ExecutionContext,
    ) -> GraphComponent:
        """Creates a new untrained component (see parent class for full docstring)."""
        return cls(config, execution_context.node_name, model_storage, resource)

    def _set_features(self, message: Message, attribute: Text = TEXT) -> None:
        """Sets the features on a single message. Utility method."""
        tokens = message.get(TEXT_TOKENS)

        # If the message doesn't have tokens, we can't create features.
        if not tokens:
            return None

        # Make distinction between sentence and sequence features
        text_vector = self.tfm.transform([message.get(TEXT)])
        word_vectors = self.tfm.transform([t.text for t in tokens])

        final_sequence_features = Features(
            word_vectors,
            FEATURE_TYPE_SEQUENCE,
            attribute,
            self._config[FEATURIZER_CLASS_ALIAS],
        )
        message.add_features(final_sequence_features)
        final_sentence_features = Features(
            text_vector,
            FEATURE_TYPE_SENTENCE,
            attribute,
            self._config[FEATURIZER_CLASS_ALIAS],
        )
        message.add_features(final_sentence_features)

    def process(self, messages: List[Message]) -> List[Message]:
        """Processes incoming message and compute and set features."""
        for message in messages:
            for attribute in DENSE_FEATURIZABLE_ATTRIBUTES:
                self._set_features(message, attribute)
        return messages

    def process_training_data(self, training_data: TrainingData) -> TrainingData:
        """Processes the training examples in the given training data in-place."""
        self.process(training_data.training_examples)
        return training_data

    def persist(self) -> None:
        """
        Persist this model into the passed directory.

        Returns the metadata necessary to load the model again. In this case; `None`.
        """
        with self._model_storage.write_to(self._resource) as model_dir:
            dump(self.tfm, model_dir / "tfidfvectorizer.joblib")

    @classmethod
    def load(
        cls,
        config: Dict[Text, Any],
        model_storage: ModelStorage,
        resource: Resource,
        execution_context: ExecutionContext,
    ) -> GraphComponent:
        """Loads trained component from disk."""
        try:
            with model_storage.read_from(resource) as model_dir:
                tfidfvectorizer = load(model_dir / "tfidfvectorizer.joblib")
                component = cls(
                    config, execution_context.node_name, model_storage, resource
                )
                component.tfm = tfidfvectorizer
        except (ValueError, FileNotFoundError):
            logger.debug(
                f"Couldn't load metadata for component '{cls.__name__}' as the persisted "
                f"model data couldn't be loaded."
            )
        return component

    @classmethod
    def validate_config(cls, config: Dict[Text, Any]) -> None:
        """Validates that the component is configured properly."""
        pass
```

### NLU 元学习器 {#nlu-meta-learners}

!!! note "高级用例"

    NLU 元学习器是一个高级用例。以下部分仅在有一个根据先前分类器的输出学习参数的组件时才相关。对于手动设置参数或逻辑的组件，你可以创建一个 `is_trainable=False` 的组件，而不用担心前面的分类器。

NLU 元学习器是意图分类器或实体提取器，它们使用其他训练好的意图分类器或实体提取器的预测并尝试改进其结果。元学习器的一个例子是一个组件，它对两个先前的意图分类器的输出进行平均，或者是一个后备分类器，它根据意图分类器对训练样本的置信度来设置它的阈值。

从概念上讲，要构建可训练的回退分类器，首先需要将该回退分类器创建为自定义组件：

```python
from typing import Dict, Text, Any, List

from rasa.engine.graph import GraphComponent, ExecutionContext
from rasa.engine.recipes.default_recipe import DefaultV1Recipe
from rasa.engine.storage.resource import Resource
from rasa.engine.storage.storage import ModelStorage
from rasa.shared.nlu.training_data.message import Message
from rasa.shared.nlu.training_data.training_data import TrainingData
from rasa.nlu.classifiers.fallback_classifier import FallbackClassifier


@DefaultV1Recipe.register(
    [DefaultV1Recipe.ComponentType.INTENT_CLASSIFIER], is_trainable=True
)
class MetaFallback(FallbackClassifier):

    def __init__(
        self,
        config: Dict[Text, Any],
        model_storage: ModelStorage,
        resource: Resource,
        execution_context: ExecutionContext,
    ) -> None:
        super().__init__(config)

        self._model_storage = model_storage
        self._resource = resource

    @classmethod
    def create(
        cls,
        config: Dict[Text, Any],
        model_storage: ModelStorage,
        resource: Resource,
        execution_context: ExecutionContext,
    ) -> FallbackClassifier:
        """Creates a new untrained component (see parent class for full docstring)."""
        return cls(config, model_storage, resource, execution_context)

    def train(self, training_data: TrainingData) -> Resource:
        # Do something here with the messages
        return self._resource
```

接下来，需要创建一个自定义的意图分类器，它也是一个特征化器，因为分类器的输出需要被下游的另一个组件使用。对于自定义意图分类器组件，还需要定义应如何将其预测添加到指定 `process_training_data` 方法的消息数据中。确保不要覆盖意图的真实标签。下面是一个模板，展示了如何为此目的对 DIET 进行继承：

```python
from rasa.engine.recipes.default_recipe import DefaultV1Recipe
from rasa.shared.nlu.training_data.training_data import TrainingData
from rasa.nlu.classifiers.diet_classifier import DIETClassifier


@DefaultV1Recipe.register(
    [DefaultV1Recipe.ComponentType.INTENT_CLASSIFIER,
     DefaultV1Recipe.ComponentType.ENTITY_EXTRACTOR,
     DefaultV1Recipe.ComponentType.MESSAGE_FEATURIZER], is_trainable=True
)
class DIETFeaturizer(DIETClassifier):

    def process_training_data(self, training_data: TrainingData) -> TrainingData:
        # classify and add the attributes to the messages on the training data
        return training_data
```
