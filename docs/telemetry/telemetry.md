# Rasa 遥测

Rasa 使用遥测技术报告匿名使用信息。此信息对于帮助为所有用户改进 Rasa 至关重要。

对于在 Rasa 工作的团队来说，了解产品的使用方式很重要。它使我们能够正确地优先考虑我们的研究工作和功能开发。

首次运行 Rasa 时，你将收到有关遥测报告的通知。

## 如保退出 {#how-to-opt-out}

可以通过运行如下命令随时选择退出遥测报告：

```shell
rasa telemetry disable
```

或通过设置环境变量 `RASA_TELEMETRY_ENABLED=false`。如果要再次启动报告，可以运行：

```shell
rasa telemetry enable
```

## 为什么使用遥测报告 {#why-do-we-use-telemetry-reporting}

匿名遥测数据是我们能够根据使用情况优先考虑我们的研究工作和功能开发。我们希望收集有关使用情况和可靠性的汇总信息，以确保提供高产品的质量。

那么我们将如何使用报告的遥测数据呢？如下是我们使用数据的一些示例：

- 我们将能够知道使用了哪些语言、管道和策略。这将是我们能够将我们的研究工作导向对我们的用户产生最大影响的文本和对话处理项目。
- 我们将能够知道数据集的大小和一般结构（例如意图的数量）。这使我们能够更好地在不同类型的数据集上测试我们的软件并优化框架性能。
- 我们能够更详细地了解你在构建对话机器人时遇到的错误类型（例如初始化、训练等）。这将使我们能够提高框架的质量，并更好地将时间集中在解决更常见、令人沮丧的问题上。

## 敏感数据呢？ {#what-about-sensitive-data}

你的敏感数据永远只会留在本地。

- 不会报告任何个人身份信息。
- 不会报告训练数据。
- 不会报告对话机器人收到或发送的任何消息。

!!! note "检查报告内容"

    你可以通过设置环境变量 `RASA_TELEMETRY_DEBUG=true` 来查看报告的所有遥测信息，例如在运行 `train` 命令时：

    ```shell
    RASA_TELEMETRY_DEBUG=true rasa train
    ```

    当设置 `RASA_TELEMETRY_DEBUG` 时，不会将任何信息发送到任何服务器，而是将其作为 JSON 转储记录到命令行以供检查。

## 我们报告什么？ {#what-do-we-report}

Rasa 报告汇总的使用细节、命令调用、性能测量和错误。我们使用遥测数据来更好地了解使用模式。报告的数据将直接使我们能够更好地决定如何设计未来的功能并优先考虑当前的工作。

具体来说，我们为所有遥测事件收集如下信息：

- 报告事件的类型（例如：Training Started）
- Rasa 机器 ID：使用 UUID 生成，并存储在 `~/.config/rasa/global.yml` 的全局 Rasa 配置中，并作为 `metrics_id` 发送
- 当前工作目录的单向哈希或 git 远程的哈希
- 一般操作系统级别信息（操作系统，CPU 数量，GPU 数量以及命令是否在 CI 中运行）
- 当前的 Rasa 和 Python 版本

如下是一个示例报告，显示了运行 `rasa train` 后报告给 Rasa 的数据：

```json
{
  "userId": "38d23c36c9be443281196080fcdd707d",
  "event": "Training Started",
  "properties": {
    "language": "en",
    "num_intent_examples": 68,
    "num_entity_examples": 0,
    "num_actions": 17,
    "num_templates": 6,
    "num_conditional_response_variations": 5,
    "num_slot_mappings": 10,
    "num_custom_slot_mappings": 2,
    "num_conditional_slot_mappings": 3,
    "num_slots": 0,
    "num_forms": 0,
    "num_intents": 6,
    "num_entities": 0,
    "num_story_steps": 5,
    "num_lookup_tables": 0,
    "num_synonyms": 0,
    "num_regexes": 0,
    "metrics_id": "38d23c36c9be443281196080fcdd707d"
  },
  "context": {
    "os": {
      "name": "Darwin",
      "version": "19.4.0"
    },
    "ci": false,
    "project": "a0a7178e6e5f9e6484c5cfa3ea4497ffc0c96d0ad3f3ad8e9399a1edd88e3cf4",
    "python": "3.7.5",
    "rasa_open_source": "2.0.0",
    "cpu": 16
  }
}
```

我们无法从数据集中识别单个用户。它是匿名的，无法追溯到用户。
