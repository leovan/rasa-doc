# 图配方

图配方为可执行图提供了更精细的配置。

!!! tip "默认配方或图配方"

    如果在现有模型上运行机器学习实验或消融研究，可能只需要图配方。对于大多数应用，我们建议从默认配方开始。

除了默认配方外，我们现在还支持图配方。图配方对如何构建执行图模式提供了更精细的控制。

!!! info "3.1 版本新特性"

    此功能是一项实验性功能。我们引入实验性功能以获得社区的反馈，因此鼓励大家进行尝试。但是，未来功能可能发生变更或删除。如果你有正面或负面反馈，请在 [Rasa 论坛](https://forum.rasa.com/)上与我们分享。

## 与默认配方的区别 {#differences-with-default-recipe}

默认配方和新的图配方之间存在一些差异。主要区别在于：

- 默认配方在配置文件中命名为 `default.v1`，图配方命名为 `graph.v1`。
- 默认配方提供易于使用的配方结构，而图配方更高级和强大。
- 默认配方更加固执己见，提供各种默认值，而图配方更加明确。
- 如果缺少 `config.yml` 的某些部分，默认配方可以自动配置自己并将使用默认值转储到文件中，而图配方则不会这样做，其认为所见即所得。使用图配方不会得到出乎意料的结果。
- 默认配方将图配置主要分为两部分：流水线和策略。这些也可以描述为 NLU 和核心（对话管理）部分。另一方面，对于图配方，训练（`train_schema`）和预测（`predict_schema`）之间存在分隔。

!!! tip "从头开始？"

    如果你不知道选择哪个配方，请使用默认配方快速引导你的项目。如果稍后发现需要更细粒度的控制，可以随时将配方更改为图配方。

## 图配置文件结构 {#graph-configuration-file-structure}

图配方共享具有相同含义的 `recipe` 和 `language` 键。相似之处仅限于此，因为图配方没有 `pipeline` 和 `policies` 键，但他们确实具有 `train_schema` 和 `predict_schema` 键，用于分别在训练和预测运行期间确定图节点。除此之外，NLU 和核心的目标节点可以用图配方明确指定，这些可以用 `nlu_target` 和 `core_target` 声明。如果省略了目标，则默认配方使用的节点名称将接管，他们分别是 nlu 和 core 的 `run_RegexMessageHandler` 和 `select_prediction`。

如下是一个示例图配方：

```yaml
# The config recipe.
# https://rasa.com/docs/rasa/model-configuration/
recipe: graph.v1

language: en

core_target: custom_core_target

nlu_target: custom_nlu_target

train_schema:
  nodes:
    # We skip schema_validator node (we only have this for DefaultV1Recipe
    # since we don't do validation for the GraphV1Recipe)
    finetuning_validator:
      needs:
        importer: __importer__
      uses: rasa.graph_components.validators.finetuning_validator.FinetuningValidator
      constructor_name: create
      fn: validate
      config:
        validate_core: true
        validate_nlu: true
      eager: false
      is_target: false
      is_input: true
      resource: null
    nlu_training_data_provider:
      needs:
        importer: finetuning_validator
      uses: rasa.graph_components.providers.nlu_training_data_provider.NLUTrainingDataProvider
      constructor_name: create
      fn: provide
      config:
        language: en
        persist: false
      eager: false
      is_target: false
      is_input: true
      resource: null
    domain_provider:
      needs:
        importer: finetuning_validator
      uses: rasa.graph_components.providers.domain_provider.DomainProvider
      constructor_name: create
      fn: provide_train
      config: { }
      eager: false
      is_target: true
      is_input: true
      resource: null
    domain_for_core_training_provider:
      needs:
        domain: domain_provider
      uses: rasa.graph_components.providers.domain_for_core_training_provider.DomainForCoreTrainingProvider
      constructor_name: create
      fn: provide
      config: { }
      eager: false
      is_target: false
      is_input: true
      resource: null
    story_graph_provider:
      needs:
        importer: finetuning_validator
      uses: rasa.graph_components.providers.story_graph_provider.StoryGraphProvider
      constructor_name: create
      fn: provide
      config:
        exclusion_percentage: null
      eager: false
      is_target: false
      is_input: true
      resource: null
    training_tracker_provider:
      needs:
        story_graph: story_graph_provider
        domain: domain_for_core_training_provider
      uses: rasa.graph_components.providers.training_tracker_provider.TrainingTrackerProvider
      constructor_name: create
      fn: provide
      config: { }
      eager: false
      is_target: false
      is_input: false
      resource: null
    train_MemoizationPolicy0:
      needs:
        training_trackers: training_tracker_provider
        domain: domain_for_core_training_provider
      uses: rasa.core.policies.memoization.MemoizationPolicy
      constructor_name: create
      fn: train
      config: { }
      eager: false
      is_target: true
      is_input: false
      resource: null

predict_schema:
  nodes:
    nlu_message_converter:
      needs:
        messages: __message__
      uses: rasa.graph_components.converters.nlu_message_converter.NLUMessageConverter
      constructor_name: load
      fn: convert_user_message
      config: {}
      eager: true
      is_target: false
      is_input: false
      resource: null
    custom_nlu_target:
      needs:
        messages: nlu_message_converter
        domain: domain_provider
      uses: rasa.nlu.classifiers.regex_message_handler.RegexMessageHandler
      constructor_name: load
      fn: process
      config: {}
      eager: true
      is_target: false
      is_input: false
      resource: null
    domain_provider:
      needs: {}
      uses: rasa.graph_components.providers.domain_provider.DomainProvider
      constructor_name: load
      fn: provide_inference
      config: {}
      eager: true
      is_target: false
      is_input: false
      resource:
        name: domain_provider
    run_MemoizationPolicy0:
      needs:
        domain: domain_provider
        tracker: __tracker__
        rule_only_data: rule_only_data_provider
      uses: rasa.core.policies.memoization.MemoizationPolicy
      constructor_name: load
      fn: predict_action_probabilities
      config: {}
      eager: true
      is_target: false
      is_input: false
      resource:
        name: train_MemoizationPolicy0
    rule_only_data_provider:
      needs: {}
      uses: rasa.graph_components.providers.rule_only_provider.RuleOnlyDataProvider
      constructor_name: load
      fn: provide
      config: {}
      eager: true
      is_target: false
      is_input: false
      resource:
        name: train_RulePolicy1
    custom_core_target:
      needs:
        policy0: run_MemoizationPolicy0
        domain: domain_provider
        tracker: __tracker__
      uses: rasa.core.policies.ensemble.DefaultPolicyPredictionEnsemble
      constructor_name: load
      fn: combine_predictions_from_kwargs
      config: {}
      eager: true
      is_target: false
      is_input: false
      resource: null
```

!!! info "图目标"

    对于 NLU，将使用默认目标名称 `run_RegexMessageHandler`，而对于核心（对话管理），如果省略，目标将称为 `select_prediction`。确保你的架构定义中具有相关名称的图节点。

    以类似的方式，请注意，第一个图节点所需的默认资源固定为 `__importer__`（表示配置、训练数据等）用于训练任务，对于预测认为则为 `__message__`（表示接收的消息）。确保你的第一个节点使用这些依赖项。

## 图节点配置 {#graph-node-configuration}

正如在上面示例中所见，图配方非常明确，你可以根据需要配置每个图节点。以下是一些键的含义解释：

- `needs`：你可以在此定义图节点需要哪些数据以及来自哪个父节点。键为数据名称，值为节点名称。

    ```yaml
    needs:
        messages: nlu_message_converter
    ```

    当前图节点需要由 `nlu_message_converter` 节点提供的消息。

- `uses`：你可以使用此键提供用于实例化此节点的类。请提供 Python 路径语法的完整路径，例如：

    ```yaml
    uses: rasa.graph_components.converters.nlu_message_converter.NLUMessageConverter
    ```

    不需要使用 Rasa 内部图组件类，你可以在此使用自己的组件。请参见[自定义图组件](custom-graph-components.md)页面来了解如何编写自己的图组件。

- `constructor_name`：这是用于实例化组件的构造函数。例如：

    ```yaml
    constructor_name: load
    ```

- `fn`：这是用于执行图组件的函数。例如：

    ```yaml
    fn: combine_predictions_from_kwargs
    ```

- `config`：你可以使用此键为组件提供任何配置参数。

    ```yaml
    config:
        language: en
        persist: false
    ```

- `eager`：这决定了组件是否应该在构建图时立即加载，或者是否应该等到运行时（称为延迟实例化）。通常我们总是在训练期间延迟实例化并在推理期间立即实例化（以避免第一次预测时较慢）。

    ```yaml
    eager: true
    ```

- `resource`：如果提供，则从该资源加载图节点，而不是从头开始实例化。例如用于加载经过训练的组件进行预测。

    ```yaml
    resource:
        name: train_RulePolicy1
    ```

- `is_target`：布尔值，如果为 `True`，则在 fingerprint 期间无法修剪此节点（尽管它可能会替换为缓存值）。例如用于训练结果的所有组件始终需要添加到模型存档中，以便在推理期间数据可用。

    ```yaml
    is_target: false
    ```

- `is_input`：布尔值。具有 `is_input` 的节点始终运行（在 fingerprint 运行期间也是）。这确保我们可以检测文件内容的变化。

    ```yaml
    is_input: false
    ```
