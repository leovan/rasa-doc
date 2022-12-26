# 策略

对话机器人使用策略来决定在对话的每个步骤中要执行的动作。对话机器人可以同时使用机器学习和基于规则的策略。

通过在项目的 `config.yml` 中指定 `policies` 键来自定义对话机器人使用的策略。有不同的策略可供选择，你可以在单个配置中包含多个策略。如下是一个策略列表的示例：

```yaml title="config.yml"
recipe: default.v1
language:  # your language
pipeline:
  # - <pipeline components>

policies:
  - name: MemoizationPolicy
  - name: TEDPolicy
    max_history: 5
    epochs: 200
  - name: RulePolicy
```

!!! tip "从头开始？"

    如果你不知道要选择哪些策略，请在 `config.yml` 中完全忽略 `policies` 键。这样做[建议配置](/model-configuration#suggested-config)功能将为你提供默认策略。

## 动作选择 {#action-selection}

在每一轮，配置中定义的每个策略都会以一定的置信度预测下一个动作。有关每项策略如何做出决策的更多信息，请参见下面的策略说明。以最高置信度进行预测的策略决定了对话机器人的下一步动作。

!!! info "最大预测数量"

    默认情况下，对话机器人最多可以预测每条用户消息后的 10 个下一步动作。要更新此值，可以将环境变量 `MAX_NUMBER_OF_PREDICTIONS` 设置为所需的最大预测数量。

### 策略优先级 {#policy-priority}

如果两个策略以相同的置信度进行预测（例如，Memoization 和 Rule Policies 可能都以置信度 1 进行预测），则需要考虑策略的优先级。开源 Rasa 策略具有默认优先级，这些优先级已设置为确保在出现平局的情况下获得预测结果。它们看起来如下，数字越大优先级越高：

- 6 - `RulePolicy`
- 3 - `MemoizationPolicy` 或 `AugmentedMemoizationPolicy`
- 2 - `UnexpecTEDIntentPolicy`
- 1 - `TEDPolicy`

通常，不建议在配置中为每个优先级设置多个策略。如果有两个具有相同优先级的策略并且它们以相同的置信度进行预测，则将随机选择生成的动作。

如果你创建自己的策略，请使用这些优先级作为确定策略优先级的指南。如果你的策略是机器学习策略，它很可能具有优先级 1，与 `TEDPolicy` 相同。

!!! danger "覆盖策略优先级"

    所有策略优先级都可以通过策略配置中的 `priority` 参数进行配置，但我们不建议在自定义策略等特定情况之外更改它们。这样做可能会导致非预期和非期望的对话机器人行为。

## 机器学习策略 {#machine-learning-policies}

### TED Policy {#ted-policy}

Transformer 嵌入对话（TED）策略是一种用于下一步动作预测和实体识别的多任务架构。该架构由两个任务共享的多个 Transformer 编码器组成。在输入词条序列的对应用户序列 Transformer 编码输出上通过一个条件随机场（CRF）预测实体标签序列。对于下一个动作预测，对话 Transformer 编码器输出和系统动作标签被嵌入到同一个语义向量空间中。我们使用点积损失来最大化与目标标签的相似性，同时最小化与负样本的相似性。

如果你想了解有关模型的更多信息，请查看我们的[论文](https://arxiv.org/abs/1910.00486)和 [Youtube 频道](https://www.youtube.com/watch?v=j90NvurJI4I&list=PL75e0qA87dlG-za8eLI6t0_Pbxafk-cxb&index=14&ab_channel=Rasa)，其中我们详细地解释了模型的架构。

TED 策略架构包括如下步骤：

1. 拼接特征

    - 用户输入（用于意图和实体）或通过用户序列 Transformer 编码器处理用户文本。
    - 通过对话机器人序列 Transformer 编码器处理的前先系统动作或对话机器人消息。
    - 槽或活动的表单。

    每次进入对话 Transformer 之前输入向量都会先进入嵌入层。

2. 将输入向量的嵌入传给对话 Transformer 编码器。
3. 将稠密层应用于对话 Transformer 的输出来获取每次的对话嵌入。
4. 应用一个稠密层来获取每次系统动作的嵌入。
5. 计算对话嵌入和系统动作嵌入的相似度。这一步是基于 [StarSpace](https://arxiv.org/abs/1709.03856) 的概念。
6. 将用户序列 Transformer 的词条级别输出与每个步骤的对话 Transformer 编码器的输出拼接在一起。
7. 应用 CRF 算法来预测每个用户文本输入的上下文实体。

配置：

你可以使用 `config.yml` 文件将配置参数传递给 TEDPolicy。如果要微调模型，请先修改如下参数：

- `epochs`：此参数设置算法使用训练数据的次数（默认值：`1`）。一个 `epoch` 等于所有训练样本的一次前向传递和一次反向传递。有时模型需要更多的 `epoch` 才能很好的学习。有时更多的 `epochs` 不会影响性能。`epochs` 越少，模型训练得越快。配置示例如下：

    ```yaml title="config.yml"
    policies:
    - name: TEDPolicy
      epochs: 200
    ```

- `max_history`：此参数控制模型查看多个对话历史来决定下面要采取的动作。本策略的 `max_history` 默认值为 `None`，这意味着从会话重新启动以来的完整对话历史记录都将被考虑在内。如果你想限制模型只能看到一定数量的先前对话轮次，可以将 `max_history` 设置为一个有限值。请注意，你应该仔细选择 `max_history` 以便模型有足够的先前对话轮次来创建正确的预测。更多有关信息，请参见[特征化器](/policies#featurizers)。配置示例如下：

    ```yaml title="config.yml"
    policies:
    - name: TEDPolicy
      max_history: 8
    ```

- `number_of_transformer_layers`：此参数设置序列 Transformer 编码器使用的层数，用于用户序列 Transformer 编码器、动作和用于对话 Transformer 编码器的动作标签文本。（默认值：`text: 1, action_text: 1, label_action_text: 1, dialogue: 1`）。序列 Transformer 编码器的数量对应模型使用的 Transformer 块。
- `transformer_size`：此参数设置序列 Transformer 编码器层中的单元数，用于用户序列 Transformer 编码器、动作和用于对话 Transformer 编码器的动作标签文本。（默认值：`text: 128, action_text: 128, label_action_text: 128, dialogue: 128`）。从 Transformer 编码器输出的向量具有给定的 `transformer_size`。
- `connection_density`：此参数定义模型中所有前馈层设置为非零值的核权重的比例（默认值：`0.2`）。该值应介于 0 和 1 之间。如果将 `connection_density` 设置为 1，则不会将核权重设置为 0，该层则为标准的前馈层。不应该将 `connection_density` 设置为 0，因为这将导致所有核权重为 0，从而模型无法进行学习。
- `split_entities_by_comma`：此参数定义是否应将逗号分割的相邻实体视为一个或拆分。例如，一个 `ingredients` 类型实体“apple, banana”可以拆分为“apple”和“banana”。例如，一个 `address` 类型实体“Schönhauser Allee 175, 10119 Berlin”应被视为一个实体。

    可以全局设置为 `True` 或 `False`：

    ```yaml title="config.yml"
    policies:
      - name: TEDPolicy
        split_entities_by_comma: True
    ```

    或针对实体类型单独设置，例如：

    ```yaml title="config.yml"
    policies:
      - name: TEDPolicy
        split_entities_by_comma:
          address: False
          ingredients: True
    ```

- `constrain_similarities`：此参数设置为 `True` 是会在所有相似项上应用 sigmoid 交叉熵损失。这有助于将输入标签和负标签的相似度保持为较小的值。这应该有助于将模型更好的泛化到现实中的测试集。
- `model_confidence`：此参数允许用户配置在推理过程中如何计算置信度。它可以取两个值：

    - `softmax`：置信度在 `[0, 1]` 范围内（旧的行为和当前的默认值）。计算出的相似性使用 `softmax` 激活函数进行归一化。
    - `linear_norm`：置信度在 `[0, 1]` 范围内。计算的点积相似度使用线性函数进行归一化。

    请尝试使用 `linear_norm` 作为 `model_confidence` 的值。这应该更容易[处理低置信度的预测动作](/fallback-handoff#handling-low-action-confidence)。

- `use_gpu`：此参数定义是否使用 GPU（如果可用）进行训练。默认情况下，如果 GPU 可用（即 `use_gpu` 为 `True`），`TEDPolicy` 将在 GPU 上进行训练。要强制 `TEDPolicy` 仅使用 CPU 进行训练，请将 `use_gpu` 设置为 `False`。

上述配置参数是应该进行配置的，从而使模型可以拟合数据。同时，还有更多参数可以进行调整。

??? info "更多配置参数"

    ```
    +---------------------------------------+------------------------+--------------------------------------------------------------+
    | Parameter                             | Default Value          | Description                                                  |
    +=======================================+========================+==============================================================+
    | hidden_layers_sizes                   | text: []               | Hidden layer sizes for layers before the embedding layers    |
    |                                       | action_text: []        | for user messages and bot messages in previous actions       |
    |                                       | label_action_text: []  | and labels. The number of hidden layers is                   |
    |                                       |                        | equal to the length of the corresponding list.               |
    +---------------------------------------+------------------------+--------------------------------------------------------------+
    | dense_dimension                       | text: 128              | Dense dimension for sparse features to use after they are    |
    |                                       | action_text: 128       | converted into dense features.                               |
    |                                       | label_action_text: 128 |                                                              |
    |                                       | intent: 20             |                                                              |
    |                                       | action_name: 20        |                                                              |
    |                                       | label_action_name: 20  |                                                              |
    |                                       | entities: 20           |                                                              |
    |                                       | slots: 20              |                                                              |
    |                                       | active_loop: 20        |                                                              |
    +---------------------------------------+------------------------+--------------------------------------------------------------+
    | concat_dimension                      | text: 128              | Common dimension to which sequence and sentence features of  |
    |                                       | action_text: 128       | different dimensions get converted before concatenation.     |
    |                                       | label_action_text: 128 |                                                              |
    +---------------------------------------+------------------------+--------------------------------------------------------------+
    | encoding_dimension                    | 50                     | Dimension size of embedding vectors                          |
    |                                       |                        | before the dialogue transformer encoder.                     |
    +---------------------------------------+------------------------+--------------------------------------------------------------+
    | transformer_size                      | text: 128              | Number of units in user text sequence transformer encoder.   |
    |                                       | action_text: 128       | Number of units in bot text sequence transformer encoder.    |
    |                                       | label_action_text: 128 | Number of units in bot text sequence transformer encoder.    |
    |                                       | dialogue: 128          | Number of units in dialogue transformer encoder.             |
    +---------------------------------------+------------------------+--------------------------------------------------------------+
    | number_of_transformer_layers          | text: 1                | Number of layers in user text sequence transformer encoder.  |
    |                                       | action_text: 1         | Number of layers in bot text sequence transformer encoder.   |
    |                                       | label_action_text: 1   | Number of layers in bot text sequence transformer encoder.   |
    |                                       | dialogue: 1            | Number of layers in dialogue transformer encoder.            |
    +---------------------------------------+------------------------+--------------------------------------------------------------+
    | number_of_attention_heads             | 4                      | Number of self-attention heads in transformers.              |
    +---------------------------------------+------------------------+--------------------------------------------------------------+
    | unidirectional_encoder                | True                   | Use a unidirectional or bidirectional encoder                |
    |                                       |                        | for `text`, `action_text`, and `label_action_text`.          |
    +---------------------------------------+------------------------+--------------------------------------------------------------+
    | use_key_relative_attention            | False                  | If 'True' use key relative embeddings in attention.          |
    +---------------------------------------+------------------------+--------------------------------------------------------------+
    | use_value_relative_attention          | False                  | If 'True' use value relative embeddings in attention.        |
    +---------------------------------------+------------------------+--------------------------------------------------------------+
    | max_relative_position                 | None                   | Maximum position for relative embeddings.                    |
    +---------------------------------------+------------------------+--------------------------------------------------------------+
    | batch_size                            | [64, 256]              | Initial and final value for batch sizes.                     |
    |                                       |                        | Batch size will be linearly increased for each epoch.        |
    |                                       |                        | If constant `batch_size` is required, pass an int, e.g. `8`. |
    +---------------------------------------+------------------------+--------------------------------------------------------------+
    | batch_strategy                        | "balanced"             | Strategy used when creating batches.                         |
    |                                       |                        | Can be either 'sequence' or 'balanced'.                      |
    +---------------------------------------+------------------------+--------------------------------------------------------------+
    | epochs                                | 1                      | Number of epochs to train.                                   |
    +---------------------------------------+------------------------+--------------------------------------------------------------+
    | random_seed                           | None                   | Set random seed to any 'int' to get reproducible results.    |
    +---------------------------------------+------------------------+--------------------------------------------------------------+
    | learning_rate                         | 0.001                  | Initial learning rate for the optimizer.                     |
    +---------------------------------------+------------------------+--------------------------------------------------------------+
    | embedding_dimension                   | 20                     | Dimension size of dialogue & system action embedding vectors.|
    +---------------------------------------+------------------------+--------------------------------------------------------------+
    | number_of_negative_examples           | 20                     | The number of incorrect labels. The algorithm will minimize  |
    |                                       |                        | their similarity to the user input during training.          |
    +---------------------------------------+------------------------+--------------------------------------------------------------+
    | similarity_type                       | "auto"                 | Type of similarity measure to use, either 'auto' or 'cosine' |
    |                                       |                        | or 'inner'.                                                  |
    +---------------------------------------+------------------------+--------------------------------------------------------------+
    | loss_type                             | "cross_entropy"        | The type of the loss function, either 'cross_entropy'        |
    |                                       |                        | or 'margin'.                                                 |
    +---------------------------------------+------------------------+--------------------------------------------------------------+
    | ranking_length                        | 0                      | Number of top actions to include in prediction. Confidences  |
    |                                       |                        | of all other actions will be set to 0. Set to 0 to let the   |
    |                                       |                        | prediction include confidences for all actions.              |
    +---------------------------------------+------------------------+--------------------------------------------------------------+
    | renormalize_confidences               | False                  | Normalize the top predictions. Applicable only with loss     |
    |                                       |                        | type 'cross_entropy' and 'softmax' confidences.              |
    +---------------------------------------+------------------------+--------------------------------------------------------------+
    | maximum_positive_similarity           | 0.8                    | Indicates how similar the algorithm should try to make       |
    |                                       |                        | embedding vectors for correct labels.                        |
    |                                       |                        | Should be 0.0 < ... < 1.0 for 'cosine' similarity type.      |
    +---------------------------------------+------------------------+--------------------------------------------------------------+
    | maximum_negative_similarity           | -0.2                   | Maximum negative similarity for incorrect labels.            |
    |                                       |                        | Should be -1.0 < ... < 1.0 for 'cosine' similarity type.     |
    +---------------------------------------+------------------------+--------------------------------------------------------------+
    | use_maximum_negative_similarity       | True                   | If 'True' the algorithm only minimizes maximum similarity    |
    |                                       |                        | over incorrect intent labels, used only if 'loss_type' is    |
    |                                       |                        | set to 'margin'.                                             |
    +---------------------------------------+------------------------+--------------------------------------------------------------+
    | scale_loss                            | True                   | Scale loss inverse proportionally to confidence of correct   |
    |                                       |                        | prediction.                                                  |
    +---------------------------------------+------------------------+--------------------------------------------------------------+
    | regularization_constant               | 0.001                  | The scale of regularization.                                 |
    +---------------------------------------+------------------------+--------------------------------------------------------------+
    | negative_margin_scale                 | 0.8                    | The scale of how important it is to minimize the maximum     |
    |                                       |                        | similarity between embeddings of different labels.           |
    +---------------------------------------+------------------------+--------------------------------------------------------------+
    | drop_rate_dialogue                    | 0.1                    | Dropout rate for embedding layers of dialogue features.      |
    |                                       |                        | Value should be between 0 and 1.                             |
    |                                       |                        | The higher the value the higher the regularization effect.   |
    +---------------------------------------+------------------------+--------------------------------------------------------------+
    | drop_rate_label                       | 0.0                    | Dropout rate for embedding layers of label features.         |
    |                                       |                        | Value should be between 0 and 1.                             |
    |                                       |                        | The higher the value the higher the regularization effect.   |
    +---------------------------------------+------------------------+--------------------------------------------------------------+
    | drop_rate_attention                   | 0.0                    | Dropout rate for attention. Value should be between 0 and 1. |
    |                                       |                        | The higher the value the higher the regularization effect.   |
    +---------------------------------------+------------------------+--------------------------------------------------------------+
    | connection_density                    | 0.2                    | Connection density of the weights in dense layers.           |
    |                                       |                        | Value should be between 0 and 1.                             |
    +---------------------------------------+------------------------+--------------------------------------------------------------+
    | use_sparse_input_dropout              | True                   | If 'True' apply dropout to sparse input tensors.             |
    +---------------------------------------+------------------------+--------------------------------------------------------------+
    | use_dense_input_dropout               | True                   | If 'True' apply dropout to sparse features after they are    |
    |                                       |                        | converted into dense features.                               |
    +---------------------------------------+------------------------+--------------------------------------------------------------+
    | evaluate_every_number_of_epochs       | 20                     | How often to calculate validation accuracy.                  |
    |                                       |                        | Set to '-1' to evaluate just once at the end of training.    |
    +---------------------------------------+------------------------+--------------------------------------------------------------+
    | evaluate_on_number_of_examples        | 0                      | How many examples to use for hold out validation set.        |
    |                                       |                        | Large values may hurt performance, e.g. model accuracy.      |
    |                                       |                        | Keep at 0 if your data set contains a lot of unique examples |
    |                                       |                        | of dialogue turns.                                           |
    |                                       |                        | Set to 0 for no validation.                                  |
    +---------------------------------------+------------------------+--------------------------------------------------------------+
    | tensorboard_log_directory             | None                   | If you want to use tensorboard to visualize training         |
    |                                       |                        | metrics, set this option to a valid output directory. You    |
    |                                       |                        | can view the training metrics after training in tensorboard  |
    |                                       |                        | via 'tensorboard --logdir <path-to-given-directory>'.        |
    +---------------------------------------+------------------------+--------------------------------------------------------------+
    | tensorboard_log_level                 | "epoch"                | Define when training metrics for tensorboard should be       |
    |                                       |                        | logged. Either after every epoch ('epoch') or for every      |
    |                                       |                        | training step ('batch').                                     |
    +---------------------------------------+------------------------+--------------------------------------------------------------+
    | checkpoint_model                      | False                  | Save the best performing model during training. Models are   |
    |                                       |                        | stored to the location specified by `--out`. Only the one    |
    |                                       |                        | best model will be saved.                                    |
    |                                       |                        | Requires `evaluate_on_number_of_examples > 0` and            |
    |                                       |                        | `evaluate_every_number_of_epochs > 0`                        |
    +---------------------------------------+------------------------+--------------------------------------------------------------+
    | e2e_confidence_threshold              | 0.5                    | The threshold that ensures that end-to-end is picked only if |
    |                                       |                        | the policy is confident enough.                              |
    +---------------------------------------+------------------------+--------------------------------------------------------------+
    | featurizers                           | []                     | List of featurizer names (alias names). Only features        |
    |                                       |                        | coming from the listed names are used. If list is empty      |
    |                                       |                        | all available features are used.                             |
    +---------------------------------------+------------------------+--------------------------------------------------------------+
    | entity_recognition                    | True                   | If 'True' entity recognition is trained and entities are     |
    |                                       |                        | extracted.                                                   |
    +---------------------------------------+------------------------+--------------------------------------------------------------+
    | constrain_similarities                | False                  | If `True`, applies sigmoid on all similarity terms and adds  |
    |                                       |                        | it to the loss function to ensure that similarity values are |
    |                                       |                        | approximately bounded.                                       |
    |                                       |                        | Used only when `loss_type=cross_entropy`.                    |
    +---------------------------------------+------------------------+--------------------------------------------------------------+
    | model_confidence                      | "softmax"              | Affects how model's confidence for each action               |
    |                                       |                        | is computed. Currently, only one value is supported:         |
    |                                       |                        | 1. `softmax` - Similarities between input and action         |
    |                                       |                        | embeddings are post-processed with a softmax function,       |
    |                                       |                        | as a result of which confidence for all labels sum up to 1.  |
    +---------------------------------------+------------------------+--------------------------------------------------------------+
    | BILOU_flag                            | True                   | If 'True', additional BILOU tags are added to entity labels. |
    +---------------------------------------+------------------------+--------------------------------------------------------------+
    | split_entities_by_comma               | True                   | Splits a list of extracted entities by comma to treat each   |
    |                                       |                        | one of them as a single entity. Can either be `True`/`False` |
    |                                       |                        | globally, or set per entity type, such as:                   |
    |                                       |                        | ```                                                          |
    |                                       |                        | - name: TEDPolicy                                            |
    |                                       |                        |   split_entities_by_comma:                                   |
    |                                       |                        |     address: True                                            |
    |                                       |                        | ```                                                          |
    +---------------------------------------+------------------------+--------------------------------------------------------------+
    ```

!!! note "注意"

    除了上述配置参数外，TEDPolicy 预测性能和训练时间还受 `rass train` 命令的 `-augmentation` 参数影响。更多有关信息请参见[数据增强](/policies#data-augmentation)。

### UnexpecTED Intent Policy {#unexpected-intent-policy}

!!! info "2.8 版本新功能"

    此功能是一项实验性功能。我们引入实验性功能以获得社区的反馈，因此鼓励大家进行尝试。但是，未来功能可能发生变更或删除。如果你有正面或负面反馈，请在 [Rasa 论坛](https://forum.rasa.com/)上与我们分享。

`UnexpecTEDIntentPolicy` 可以帮助回顾对话并允许对话机器人对不太可能的用户轮次做出反应。它是一种辅助策略，只能与至少一个其他策略一起使用，因为它可以触发的动作是特殊的 [`action_unlikely_intent`](/default-actions#action_unlikely_intent) 动作。

`UnexpecTEDIntentPolicy` 具有与 [`TEDPolicy`](/policies#ted-policy) 相同的模型架构。区别在于任务级别。`UnexpecTEDIntentPolicy` 不是学习接下来要触发的最佳动作，而是从训练故事中学习给定对话上下文的用户最有可能表达的意图集合。它通过检查 NLU 预测的意图是否是最可能的意图，在推理时使用学习到的信息。如果在给定对话上下文的情况下，NLU 预测的意图确实可能发生，则 `UnexpectTEDIntentPolicy` 不会触发任何动作。否则，他会触发置信度为 `1.0` 的 [`action_unlikely_intent`](/default-actions#action_unlikely_intent)。

`UnexpecTEDIntentPolicy` 应被视为对 `TEDPolicy` 的辅助。期望 `TEDPolicy` 来改进对话机器人在训练数据中对期望处理的独特对话路径的覆盖，`UnexpecTEDIntentPolicy` 有助于从过去的对话中展现这些独特的对话路径。例如，如果你的训练数据中有如下故事：

```yaml title="stories.yml"
stories:
- story: book_restaurant_table
  steps:
  - intent: request_restaurant
  - action: restaurant_form
  - active_loop: restaurant_form
  - action: restaurant_form
  - active_loop: null
  - slot_was_set:
    - requested_slot: null
```

但是实际的对话可能会在表格内遇见你没有考虑到的插入语：

```yaml title="stories.yml" hl_lines="11 12 13"
stories:
- story: actual_conversation
  steps:
  - user: |
        I'm looking for a restaurant.
    intent: request_restaurant
  - action: restaurant_form
  - active_loop: restaurant_form
  - slot_was_set:
    - requested_slot: cuisine
  - user: |
        Does it matter? I want to be quick.
    intent: deny
```

一旦 `deny` 意图被触发，处理表单的策略将继续请求填写 `cuisine` 槽。因为训练故事并没有说应该区别对待这种情况。为了帮助确定此时可能缺少处理用户 `deny` 意图的特殊故事，`UnexpecTEDIntentPolicy` 可以在 `deny` 意图时候立即触发 `action_unlikely_intent` 动作。随后，你可以通过添加处理此特殊情况的新训练故事来改进你的对话机器人。

为了减少错误警告，`UnexpecTEDIntentPolicy` 在推理时有两种机制：

- `UnexpecTEDIntentPolicy` 的[优先级](/policies#policy-priority)有意保持低于所有[基于规则的策略](/policies#rule-based-policies)，因为对于 `TEDPolicy` 和 `UnexpecTEDIntentPolicy` 的新情况可能存在其他规则。
- 如果最后预测的意图不存在于任何训练故事中，则 `UnexpecTEDIntentPolicy` 不会预测 `action_unlikely_intent`，如果意图仅用于规则中，则可能会发生这种情况。

预测 `action_unlikely_intent`

`UnexpecTEDIntentPolicy` 在用户发出消息后立即调用，并且可以触发 `action_unlikely_intent` 或放弃（在这种情况下，其他策略将会预测动作）。为了确定是否应该触发 `action_unlikely_intent`，`UnexpecTEDIntentPolicy` 计算当前对话上下文中用户意图的分数，并检查该分数是否低于某个阈值分数。

这个阈值分数是通过收集 ML 模型在许多“负样本”上的输出来计算的。这些负样本是不正确的对话上下文和用户意图的组合。`UnexpecTEDIntentPolicy` 通过选择随机故事并将其与此时未出现过的意图随机配对，从训练数据中生成这些负样本。例如，如果你有一个训练故事：

```yaml title="stories.yml"
version: 2.0
stories:
- story: happy path 1
  steps:
  - intent: greet
  - action: utter_greet
  - intent: mood_great
  - action: utter_goodbye
```

和一个 `affirm` 意图，那么一个有效的负样本为：

```yaml title="negative_stories.yml" hl_lines="7"
version: 2.0
stories:
- story: negative example with affirm unexpected
  steps:
  - intent: greet
  - action: utter_greet
  - intent: affirm
```

在这里，`affirm` 意图是非预期的，因为它不会出现在所有训练故事的特定会话上下文中。对于每个意图，`UnexpecTEDIntentPolicy` 使用这些负样本示例来计算模型预测的分数范围。阈值分数是从这个分数范围中选取的，使得一定百分比的负样本的预测分数高于阈值分数，因此不会为它们触发 `action_unlikely_intent`。这个负样本比例可以通过 `tolerance` 参数控制。`tolerance` 越高，在 `UnexpecTEDIntentPolicy` 触发 `action_unlikely_intent` 动作之前，意图的分数越低（意图越不可能）。

配置：

你可以使用 `config.yml` 文件将配置参数传递给 `UnexpecTEDIntentPolicy`。如果要对模型性能进行调优，可以修改如下参数：

- `epochs`：此参数设置算法使用训练数据的次数（默认值：`1`）。一个 `epoch` 等于所有训练样本的一次前向传递和一次反向传递。有时模型需要更多的 `epoch` 才能很好的学习。有时更多的 `epochs` 不会影响性能。`epochs` 越少，模型训练得越快。配置示例如下：

    ```yaml title="config.yml"
    policies:
    - name: UnexpecTEDIntentPolicy
      epochs: 200
    ```

- `max_history`：此参数控制模型查看多个对话历史来决定下面要采取的动作。本策略的 `max_history` 默认值为 `None`，这意味着从会话重新启动以来的完整对话历史记录都将被考虑在内。如果你想限制模型只能看到一定数量的先前对话轮次，可以将 `max_history` 设置为一个有限值。请注意，你应该仔细选择 `max_history` 以便模型有足够的先前对话轮次来创建正确的预测。根据你的数据集，更高的 `max_history` 值可能会导致更频繁的预测 `action_unlikely_intent`，因为随着考虑更多对话上下文，唯一可能的对话路径数量会增加。类似地，更小的 `max_history` 值可能会导致 `action_unlikely_intent` 被触发的频率降低，但也可能是表明相应的对话路径是高度唯一的因此也是未预期的。建议将 `UnexpecTEDIntentPolicy` 的 `max_history` 与 `TEDPolicy` 的 `max_history` 设置为相同值。配置如下：

    ```yaml title="config.yml"
    policies:
    - name: UnexpecTEDIntentPolicy
      max_history: 8
    ```

- `ignore_intents_list`：此参数允许你将 `UnexpecTEDIntentPolicy` 配置不预测 `action_unlikely_intent` 的意图子集。当遇到生成太多错误警告时，你可能希望对于某些意图执行此操作。
- `tolerance`：容差参数是一个介于 `0.0` 到 `1.0`（包含）之间的数。它有助于调整在推理时[预测 `action_unlikely_intent`](/policies#prediction-of-action_unlikely_intent) 期间使用的阈值分数。

    `0.0` 意味着调整阈值分数以使得在训练期间 `0%` 遇见的负样本被预测为低于阈值分数的分数。因此，来自所有负样本的对话都将触发 `action_unlikely_intent` 动作。

    `0.1` 的容差意味着调整阈值分数以使得在训练期间 `10%` 遇见的负样本被预测为低于阈值分数的分数。

    `1.0` 的容差意味着阈值分数非常低，以至于 `UnexpecTEDIntentPolicy` 不会对它在训练期间遇到的任何负样本触发 `action_unlikely_intent`。

- `use_gpu`：此参数定义是否使用 GPU（如果可用）进行训练。默认情况下，如果 GPU 可用（即 `use_gpu` 为 `True`），`TEDPolicy` 将在 GPU 上进行训练。要强制 `UnexpecTEDIntentPolicy` 仅使用 CPU 进行训练，请将 `use_gpu` 设置为 `False`。

上述配置参数是应该进行配置的，从而使模型可以拟合数据。同时，还有更多参数可以进行调整。

??? info "更多配置参数"

    ```
    +---------------------------------------+------------------------+--------------------------------------------------------------+
    | Parameter                             | Default Value          | Description                                                  |
    +=======================================+========================+==============================================================+
    | hidden_layers_sizes                   | text: []               | Hidden layer sizes for layers before the embedding layers    |
    |                                       |                        | for user messages and bot messages in previous actions       |
    |                                       |                        | and labels. The number of hidden layers is                   |
    |                                       |                        | equal to the length of the corresponding list.               |
    +---------------------------------------+------------------------+--------------------------------------------------------------+
    | dense_dimension                       | text: 128              | Dense dimension for sparse features to use after they are    |
    |                                       | intent: 20             | converted into dense features.                               |
    |                                       | action_name: 20        |                                                              |
    |                                       | label_intent: 20       |                                                              |
    |                                       | entities: 20           |                                                              |
    |                                       | slots: 20              |                                                              |
    |                                       | active_loop: 20        |                                                              |
    +---------------------------------------+------------------------+--------------------------------------------------------------+
    | concat_dimension                      | text: 128              | Common dimension to which sequence and sentence features of  |
    |                                       |                        | different dimensions get converted before concatenation.     |
    +---------------------------------------+------------------------+--------------------------------------------------------------+
    | encoding_dimension                    | 50                     | Dimension size of embedding vectors                          |
    |                                       |                        | before the dialogue transformer encoder.                     |
    +---------------------------------------+------------------------+--------------------------------------------------------------+
    | transformer_size                      | text: 128              | Number of units in user text sequence transformer encoder.   |
    |                                       | dialogue: 128          | Number of units in dialogue transformer encoder.             |
    +---------------------------------------+------------------------+--------------------------------------------------------------+
    | number_of_transformer_layers          | text: 1                | Number of layers in user text sequence transformer encoder.  |
    |                                       | dialogue: 1            | Number of layers in dialogue transformer encoder.            |
    +---------------------------------------+------------------------+--------------------------------------------------------------+
    | number_of_attention_heads             | 4                      | Number of self-attention heads in transformers.              |
    +---------------------------------------+------------------------+--------------------------------------------------------------+
    | unidirectional_encoder                | True                   | Use a unidirectional or bidirectional encoder                |
    |                                       |                        | for `text`, `action_text`, and `label_action_text`.          |
    +---------------------------------------+------------------------+--------------------------------------------------------------+
    | use_key_relative_attention            | False                  | If 'True' use key relative embeddings in attention.          |
    +---------------------------------------+------------------------+--------------------------------------------------------------+
    | use_value_relative_attention          | False                  | If 'True' use value relative embeddings in attention.        |
    +---------------------------------------+------------------------+--------------------------------------------------------------+
    | max_relative_position                 | None                   | Maximum position for relative embeddings.                    |
    +---------------------------------------+------------------------+--------------------------------------------------------------+
    | batch_size                            | [64, 256]              | Initial and final value for batch sizes.                     |
    |                                       |                        | Batch size will be linearly increased for each epoch.        |
    |                                       |                        | If constant `batch_size` is required, pass an int, e.g. `8`. |
    +---------------------------------------+------------------------+--------------------------------------------------------------+
    | batch_strategy                        | "balanced"             | Strategy used when creating batches.                         |
    |                                       |                        | Can be either 'sequence' or 'balanced'.                      |
    +---------------------------------------+------------------------+--------------------------------------------------------------+
    | epochs                                | 1                      | Number of epochs to train.                                   |
    +---------------------------------------+------------------------+--------------------------------------------------------------+
    | random_seed                           | None                   | Set random seed to any 'int' to get reproducible results.    |
    +---------------------------------------+------------------------+--------------------------------------------------------------+
    | learning_rate                         | 0.001                  | Initial learning rate for the optimizer.                     |
    +---------------------------------------+------------------------+--------------------------------------------------------------+
    | embedding_dimension                   | 20                     | Dimension size of dialogue & system action embedding vectors.|
    +---------------------------------------+------------------------+--------------------------------------------------------------+
    | number_of_negative_examples           | 20                     | The number of incorrect labels. The algorithm will minimize  |
    |                                       |                        | their similarity to the user input during training.          |
    +---------------------------------------+------------------------+--------------------------------------------------------------+
    | ranking_length                        | 10                     | Number of top actions to normalize scores for. Applicable    |
    |                                       |                        | only with loss type 'cross_entropy' and 'softmax'            |
    |                                       |                        | confidences. Set to 0 to disable normalization.              |
    +---------------------------------------+------------------------+--------------------------------------------------------------+
    | scale_loss                            | True                   | Scale loss inverse proportionally to confidence of correct   |
    |                                       |                        | prediction.                                                  |
    +---------------------------------------+------------------------+--------------------------------------------------------------+
    | regularization_constant               | 0.001                  | The scale of regularization.                                 |
    +---------------------------------------+------------------------+--------------------------------------------------------------+
    | drop_rate_dialogue                    | 0.1                    | Dropout rate for embedding layers of dialogue features.      |
    |                                       |                        | Value should be between 0 and 1.                             |
    |                                       |                        | The higher the value the higher the regularization effect.   |
    +---------------------------------------+------------------------+--------------------------------------------------------------+
    | drop_rate_label                       | 0.0                    | Dropout rate for embedding layers of label features.         |
    |                                       |                        | Value should be between 0 and 1.                             |
    |                                       |                        | The higher the value the higher the regularization effect.   |
    +---------------------------------------+------------------------+--------------------------------------------------------------+
    | drop_rate_attention                   | 0.0                    | Dropout rate for attention. Value should be between 0 and 1. |
    |                                       |                        | The higher the value the higher the regularization effect.   |
    +---------------------------------------+------------------------+--------------------------------------------------------------+
    | use_sparse_input_dropout              | True                   | If 'True' apply dropout to sparse input tensors.             |
    +---------------------------------------+------------------------+--------------------------------------------------------------+
    | use_dense_input_dropout               | True                   | If 'True' apply dropout to sparse features after they are    |
    |                                       |                        | converted into dense features.                               |
    +---------------------------------------+------------------------+--------------------------------------------------------------+
    | evaluate_every_number_of_epochs       | 20                     | How often to calculate validation accuracy.                  |
    |                                       |                        | Set to '-1' to evaluate just once at the end of training.    |
    +---------------------------------------+------------------------+--------------------------------------------------------------+
    | evaluate_on_number_of_examples        | 0                      | How many examples to use for hold out validation set.        |
    |                                       |                        | Large values may hurt performance, e.g. model accuracy.      |
    |                                       |                        | Keep at 0 if your data set contains a lot of unique examples |
    |                                       |                        | of dialogue turns.                                           |
    +---------------------------------------+------------------------+--------------------------------------------------------------+
    | tensorboard_log_directory             | None                   | If you want to use tensorboard to visualize training         |
    |                                       |                        | metrics, set this option to a valid output directory. You    |
    |                                       |                        | can view the training metrics after training in tensorboard  |
    |                                       |                        | via 'tensorboard --logdir <path-to-given-directory>'.        |
    +---------------------------------------+------------------------+--------------------------------------------------------------+
    | tensorboard_log_level                 | "epoch"                | Define when training metrics for tensorboard should be       |
    |                                       |                        | logged. Either after every epoch ('epoch') or for every      |
    |                                       |                        | training step ('batch').                                     |
    +---------------------------------------+------------------------+--------------------------------------------------------------+
    | checkpoint_model                      | False                  | Save the best performing model during training. Models are   |
    |                                       |                        | stored to the location specified by `--out`. Only the one    |
    |                                       |                        | best model will be saved.                                    |
    |                                       |                        | Requires `evaluate_on_number_of_examples > 0` and            |
    |                                       |                        | `evaluate_every_number_of_epochs > 0`                        |
    +---------------------------------------+------------------------+--------------------------------------------------------------+
    | featurizers                           | []                     | List of featurizer names (alias names). Only features        |
    |                                       |                        | coming from the listed names are used. If list is empty      |
    |                                       |                        | all available features are used.                             |
    +---------------------------------------+------------------------+--------------------------------------------------------------+
    | ignore_intents_list                   | []                     | This parameter lets you configure `UnexpecTEDIntentPolicy` to ignore|
    |                                       |                        | the prediction of `action_unlikely_intent` for a subset of   |
    |                                       |                        | intents. You might want to do this if you come across a      |
    |                                       |                        | certain list of intents for which there are too many false   |
    |                                       |                        | warnings generated.                                          |
    +---------------------------------------+------------------------+--------------------------------------------------------------+
    | tolerance                             | 0.0                    | The `tolerance` parameter is a number that ranges from `0.0` |
    |                                       |                        | to `1.0` (inclusive). It helps to adjust the threshold score |
    |                                       |                        | used during prediction of `action_unlikely_intent` at        |
    |                                       |                        | inference time. Here, `0.0` means that the score threshold   |
    |                                       |                        | is the one that `UnexpecTEDIntentPolicy` had determined at training |
    |                                       |                        | time. A tolerance of `1.0` means that the threshold score    |
    |                                       |                        | is so low that `IntentTED` would not trigger                 |
    |                                       |                        | `action_unlikely_intent` for any of the "negative examples"  |
    |                                       |                        | that it has encountered during training. These negative      |
    |                                       |                        | examples are combinations of dialogue contexts and user      |
    |                                       |                        | intents that are _incorrect_. `UnexpecTEDIntentPolicy` generates    |
    |                                       |                        | these negative examples from your training data by picking a |
    |                                       |                        | random story part and pairing it with a random intent that   |
    |                                       |                        | doesn't occur at this point.                                 |
    +---------------------------------------+------------------------+--------------------------------------------------------------+
    ```

tolerance 参数调优

在查看真实对话时，我们鼓励在 `UnexpecTEDIntentPolicy` 配置中调整容差参数，以减少错误警告的数量（实际上可能在对话上下文中给出的意图）。当以 `0.05` 的步长将容差值从 `0` 增加到 `1` 时，错误警告的数量应该会减少。然而，增加容差也将导致触发 `action_unlikely_intent` 减少，因此更多训练故事中不存在的对话路径将在标记的对话集中丢失。如果改变 `max_history` 值并重新训练模型，你可能必须重新调整容差值。

!!! note "注意"

    `UnexpecTEDIntentPolicy` 仅针对[故事](/stories)进行训练，而不针对训练数据中的[规则](/rules)进行训练。

### Memoization Policy {#memoization-policy}

`MemoizationPolicy` 会记住训练数据中的故事。它检查当前对话是否与你的 `stories.yml` 文件中的故事匹配。如果是这样，它将根据训练数据的匹配故事预测下一个动作，置信度为 `1.0`。如果没有找到匹配的对话，则策略以 `0.0` 置信度预测 `None`。

在训练数据中寻找匹配项时，该策略将考虑对话的最后 `max_history` 轮数。一个“轮次”包括用户发送的消息以及对话机器人在等待下一条消息之前执行的任何动作。

你可以配置 `MemoizationPolicy` 应该在配置中使用的轮数：

```yaml title="config.yml"
policies:
  - name: "MemoizationPolicy"
    max_history: 3
```

### Augmented Memoization Policy {#augmented-memoization-policy}

像 `MemoizationPolicy` 一样，`AugmentedMemoizationPolicy` 可以记住最多 `max_history` 轮次的训练故事样本。此外，它还有一个遗忘机制，可以忘记对话历史中的某些步骤，并尝试在故事中找到与减少的历史相匹配的内容。如果找到匹配项，它以 `1.0` 置信度预测下一个动作，否则以 `0.0` 置信度预测 `None`。

!!! note "槽和预测"

    如果你的对话中在预测时设置的某些槽可能未在训练故事中设置（例如，在训练故事中以一个 [reminder](/reaching-out-to-user#reminders) 开始，并非所有先前的槽都已设置），请确保将没有槽的相关故事也添加到训练数据中。

## 基于规则的策略 {#rule-based-policies}

### Rule Policy {#rule-policy}

`RulePolicy` 是一种遵循固定行为（例如：业务逻辑）来处理对话的策略。它根据训练数据中的规则进行预测。有关如何定义规则的更多信息，请参见[规则文档](/rules)。

RulePolicy 具有如下配置选项：

```yaml title="config.yml"
policies:
  - name: "RulePolicy"
    core_fallback_threshold: 0.3
    core_fallback_action_name: action_default_fallback
    enable_fallback_prediction: true
    restrict_rules: true
    check_for_contradictions: true
```

- `core_fallback_threshold`（默认值：`3`）：请参见[回退文档](/fallback-handoff#handling-low-action-confidence)以获得更多信息。
- `core_fallback_action_name`（默认值：`action_default_fallback`）：请参见[回退文档](/fallback-handoff#handling-low-action-confidence)以获得更多信息。
- `enable_fallback_prediction`（默认值：`true`）：请参见[回退文档](/fallback-handoff#handling-low-action-confidence)以获得更多信息。
- `check_for_contradictions`（默认值：`true`）：在训练之前，RulePolicy 将执行检查以确保为所有规则定义一致的动作设置的槽和动作循环。以下代码包含一个不完整规则的示例：

    ```yaml
    rules:
    - rule: complete rule
      steps:
      - intent: search_venues
      - action: action_search_venues
      - slot_was_set:
        - venues: [{"name": "Big Arena", "reviews": 4.5}]

    - rule: incomplete rule
      steps:
      - intent: search_venues
      - action: action_search_venues
    ```

    在第二个不完整规则中，`action_search_venues` 应该设置 `venues` 槽，因为它是在完整规则中设置的，但是缺少此事件。有几种可能的方法可以修复此规则。

    在 `action_search_venues` 找不到一个 `venues` 且 `venues` 槽不应该被设置时，应该显式地将槽的值设置为 `null`。在如下的故事中，`RulePolicy` 将仅在未设置 `venues` 槽的情况下预测 `utter_venues_not_found`：

    ```yaml
    rules:
    - rule: fixes incomplete rule
      steps:
      - intent: search_venues
      - action: action_search_venues
      - slot_was_set:
        - venues: null
      - action: utter_venues_not_found
    ```

    如果你希望槽设置由不同的规则或故事处理，则应将 `wait_for_user_input: false` 添加到规则片段末尾：

    ```yaml
    rules:
    - rule: incomplete rule
      steps:
      - intent: search_venues
      - action: action_search_venues
      wait_for_user_input: false
    ```

    训练后，RulePolicy 将检查规则或故事是否相互矛盾。如下代码段是两个相互矛盾规则的示例：

    ```yaml
    rules:
    - rule: Chitchat
      steps:
      - intent: chitchat
      - action: utter_chitchat

    - rule: Greet instead of chitchat
      steps:
      - intent: chitchat
      - action: utter_greet  # `utter_greet` contradicts `utter_chitchat` from the rule above
    ```

- `restrict_rules`（默认值：`true`）：规则仅限于一个用户轮次，但可以有多个对话机器人事件，包括例如正在填写的表格及其之后的提交。将此参数改为 `false` 可能会导致意外行为。

    !!! warning "过度使用规则"

        随着复杂性的增加，除了[推荐用例](/rules)之外过度使用规则将使对话机器人变得非常难以维护。

## 配置策略 {#configuring-policies}

### 最大历史 {#max-history}

开源 Rasa 策略的一个重要超参数是 `max_history`。这控制了模型查看多少对话历史来决定下一步要采取的动作。

可以将 `max_history` 通过 `config.yml` 中的策略配置传递给你的策略来设置它。默认值为 `None`，这意味着会考虑在会话重启之后完整的对话历史。

```yaml title="config.yml" hl_lines="3"
policies:
  - name: TEDPolicy
    max_history: 5
    epochs: 200
    batch_size: 50
    max_training_samples: 300
```

!!! note "注意"

    RulePolicy 没有 `max_history` 参数，它总是考虑提供的规则的全部。更多信息请参见[规则](/rules)。

例如，假设你有一个描述离题用户消息的 `out_of_scope` 意图。如果对话机器人连续多次看到这个意图，你可能想告诉用户你可以帮助他们做些什么。所以你的故事可能是这样的：

```yaml
stories:
  - story: utter help after 2 fallbacks
    steps:
    - intent: out_of_scope
    - action: utter_default
    - intent: out_of_scope
    - action: utter_default
    - intent: out_of_scope
    - action: utter_help_message
```

为了让模型可以学习这种模式，`max_history` 必须至少为 4。

如果你增加 `max_history`，你的模型会变得更大，训练需要更长的时间。如果你有一些消息会在很长一段时间影响对话，则应该将其存储为槽。槽信息始终可用于每个特征化器。

### 数据增强 {#data-augmentation}

当训练模型时，开源 Rasa 将通过随机组合故事文件中的故事来创建更长的故事。以如下故事为例：

```yaml
stories:
  - story: thank
    steps:
    - intent: thankyou
    - action: utter_youarewelcome
  - story: say goodbye
    steps:
    - intent: goodbye
    - action: utter_goodbye
```

你实际上想教策略在对话历史不相关时忽略它，并且无论之前发生什么都以相同的动作做出响应。为了实现这一点，单个故事被连接成为更长的故事。从上面的示例中，数据增强可能会通过将 `thank` 和 `say goodbye` 再和 `thank` 拼接来产生一个故事，相当于：

```yaml
stories:
  - story: thank -> say goodbye -> thank
    steps:
    - intent: thankyou
    - action: utter_youarewelcome
    - intent: goodbye
    - action: utter_goodbye
    - intent: thankyou
    - action: utter_youarewelcome
```

你可以使用 `--augmentation` 标识来更改此行为，该标识允许你设置 `augmentation_factor`。`augmentation_factor` 确定在训练期间对多少个增强故事进行二次采样。增强的故事会在训练之前进行二次采样，由于它们的数据会很快变得非常大，因此需要对其进行限制。采样的故事数量为 `augmentation_factor` 的 10 倍。默认情况下 `augmentation_factor` 设置为 50，也就是最多生成 500 个增强故事。

`--augmentation 0` 会禁用所有增强行为。TEDPolicy 是唯一受增强影响的策略。其他策略，例如 MemoizationPolicy 或 RulePolicy 会自动忽略所有增强故事（不管 `augmentation_factor` 如何）。

`--augmentation` 是尝试减少 TEDPolicy 训练时间的一个重要参数。减少 `augmentation_factor` 会减少训练数据的大小，从而减少训练策略的时间。但是，减少数据增强也会降低 TEDPolicy 的性能。我们建议在减少数据增强时采用基于记忆的策略和 TEDPolicy 一起以进行补偿。

### 特征化器 {#featurizers}

为了将机器学习算法应用于对话式 AI，你需要建立对话的向量表示。

每个故事都对应一个追踪器，该追踪器由在采取每个动作之前的对话状态组成。

#### 状态特征化器 {#state-featurizers}

追踪器历史记录中的每个事件都会创建一个新状态（例如：运行一个对话机器人动作，接收一个用户消息，设置槽）。将追踪器的单个状态特征化有两个步骤：

1. 追踪器提供了一系列活动特征：

    - 表示意图和实体的特征，如果这是一个回合中的第一个状态，例如：是解析用户消息后采取的第一个动作（例如：`[intent_restaurant_search, entity_cuisine]`）
    - 表示当前定义了那些槽的特征，例如：`slot_location`，如果用户之前提到了他们正在寻找餐厅的区域。
    - 表示存储在槽中的任何 API 调用的结果的特征，例如 `slot_matches`。
    - 表示最后一个对话机器人动作或消息的特征，例如：`prev_action_listen`。
    - 表示是否有处于活动状态的循环以及是哪个的特征。

2. 将所有特征转换为数值向量：

    `SingleStateFeaturizer` 使用 Rasa NLU 管道将意图和对话机器人动作名称或消息转换为数值向量。有关如何配置 Rasa NLU 管道，请参见 [NLU 模型配置](/model-configuration)文档。

    实体、槽和活动状态的循环均采用独热编码特征化来表示它们。

!!! note "注意"

    如果领域定义了可能的 `action`，`[ActionGreet, ActionGoodbye]`，则添加 4 个额外的默认动作：`[ActionListen(), ActionRestart(), ActionDefaultFallback(), ActionDeactivateForm()]`。因此，标签 `0` 表示默认 action listen，标签 `1` 默认 restart，标签 `2` 表示 greeting，标签 `3` 表示 goodbye。

#### 追踪器特征化器 {#tracker-featurizers}

可以训练策略来学习两种标签

- 由对话机器人触发的下一个最合适的动作。例如，[`TEDPolicy`](/policies#ted-policy) 被训练用作这一任务。
- 用户可以表达的下一个最可能的意图。例如，[`UnexpecTEDIntentPolicy`](/policies#unexpected-intent-policy) 被训练用作这一任务。

因此，可以对追踪器进行特征化以学习上述标签。根据策略，目标标签对应对话机器人的动作或消息，表示为所有可能动作列表中的索引，或表示为所有可能意图列表中的意图集合。

追踪器特征化器有三种不同的风格：

1. Full Dialogue

    `FullDialogueTrackerFeaturizer` 创建故事的数值表示以传递至循环神经网络，其中整个对话被传递到网络中，梯度从所有步反向传播。目标标签是在对话上下文中触发的最合适的机器人动作或消息。`TrackerFeaturizer` 迭代追踪器状态并为每个状态调用 `SingleStateFeaturizer` 为策略创建数值输入特征。

2. Max History

    `MaxHistoryTrackerFeaturizer` 的操作与 `FullDialogueTrackerFeaturizer` 非常相似，因为它为每个对话机器人动作或消息创建了一组先前的追踪器状态，但参数 `max_history` 定义了进入输入特征每行的状态数。如果未指定 `max_history`，则算法会考虑整个对话长度。执行重复数据删除以根据其先前状态过滤重复的轮次（对话机器人动作或消息）。

    对于某些算法，需要单维特征，因此应将输入特征变换为 `(num_unique_turns, max_history * num_input_features)`。

3. Intent Max History

    `IntentMaxHistoryTrackerFeaturizer` 继承自 `MaxHistoryTrackerFeaturizer`。由于被 [`UnexpecTEDIntentPolicy`](/policies#unexpected-intent-policy) 使用，其创建的目标标签是用户可以在对话追踪器的上下文中表达的意图。与其他追踪器特征化器不同，其可以有多个目标标签。因此，它在右侧用一个常数值（`-1`）填充目标标签列表，以便为每个输入对话跟踪器返回一个大小相同的目标标签列表。

    就像 `MaxHistoryTrackerFeaturizer` 一样，它可执行重述数据删除来过滤重复的轮次。但是，它会为相应的追踪器的每个正确意图生成一个特征化追踪器。例如，如果输入对话追踪器的正确标签具有以下索引 `[0, 2, 4]`，则特征化器将产生三对特征化追踪器和目标标签。特征化追踪器将是相同的，但每对中的目标标签为 `[0, 2, 4]`，`[4, 0, 2]`，`[2, 4, 0]`。

## 自定义策略 {#custom-policies}

!!! info "3.0 版本新功能"

    开源 Rasa 3.0 版本统一了 NLU 组件和策略的实现。这需要更改为早期版本的开源 Rasa 编写的自定义组件。有关逐步的迁移指南请参见[迁移指南](/migration-guide#custom-policies-and-custom-components)。

你还可以编写自定义策略并在配置中引用它们。在如下的示例中，最后两行显示了如何使用自定义策略并将参数传递给它。有关自定义策略的完整指南，请参见[自定义图组件指南](/custom-graph-components)。

```yaml
policies:
  - name: "TEDPolicy"
    max_history: 5
    epochs: 200
  - name: "RulePolicy"
  - name: "path.to.your.policy.class"
    arg1: "..."
```
