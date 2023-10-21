# 模型配置

配置文件定义了模型将用于根据用户输入进行预测的组件和策略。

`recipe` 键允许不同类型的配置和模型架构。目前，支持 `default.v1` 和实验性的 `graph.v1`。

!!! info "3.5 版本新特性"

    配置文件现在一个新的强制键值 `assistant_id`，用于表示唯一的对话机器人标识。

`assistant_id` 键必须指定一个唯一值賴区分部署的多个对话机器人。对话机器人标识将于模型 ID 一起传递给每个事件的元数据。请注意，如果配置文件不包含此必需的键值或未修改默认值，则每次运行 `rasa train` 时都会生成一个随机的对话机器人名称并将其添加到配置中。

`language` 和 `pipeline` 键指定模型用于进行 NLU 预测的[组件](components.md)。`policies` 键定义模型用于预测下一步动作的[策略](policies.md)。

如果你不知道要选择哪些组件或策略，可以使用[建议配置](model-configuration.md#suggested-config)功能，该功能会推荐合理的默认值。

## 建议配置 {#suggested-config}

你可以将流水线和/或策略密钥保留在配置文件之外。当你运行 `rasa train` 时，建议配置功能将为缺少的键选择默认配置来训练模型。

确保在 `config.yml` 文件中使用 2 个字母的 ISO 语言代码指定 `language` 键。

示例 `config.yml` 文件：

```yaml
recipe: default.v1
assistant_id: example_bot
language: en

pipeline:
# will be selected by the Suggested Config feature

policies:
- name: MemoizationPolicy
- name: TEDPolicy
  max_history: 5
  epochs: 10
```

选择的配置也将作为注释写入 `config.yml` 文件，因此你可以查看使用了哪个配置。对于上面的示例，生成的文件可能看起来如下所示：

```yaml
recipe: default.v1
assistant_id: example_bot
language: en

pipeline:
# # No configuration for the NLU pipeline was provided. The following default pipeline was used to train your model.
# # If you'd like to customize it, uncomment and adjust the pipeline.
# # See https://rasa.com/docs/rasa/tuning-your-model for more information.
#   - name: WhitespaceTokenizer
#   - name: RegexFeaturizer
#   - name: LexicalSyntacticFeaturizer
#   - name: CountVectorsFeaturizer
#   - name: CountVectorsFeaturizer
#     analyzer: char_wb
#     min_ngram: 1
#     max_ngram: 4
#   - name: DIETClassifier
#     epochs: 100
#   - name: EntitySynonymMapper
#   - name: ResponseSelector
#     epochs: 100
#   - name: FallbackClassifier
#     threshold: 0.3
#     ambiguity_threshold: 0.1

policies:
- name: MemoizationPolicy
- name: TEDPolicy
  max_history: 5
  epochs: 10
```

如果你愿意，可以取消注释其中一个或两个键的建议配置并进行修改。请注意，这将再次训练时禁用此键的自动建议。只要你将配置注释掉并不为键指定任何配置，那么每当你训练新模型时都会建议使用默认配置。

!!! info "NLU 或仅对话模型"

    如果你运行 `rasa train nlu`，则只会自动选择 `pipeline` 的默认配置，如果你运行 `rasa train core`，则只会选择 `policies` 的默认配置。
