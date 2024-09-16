# 标记

标记是用于描述和标记对话中兴趣点的条件。

!!! warning "警告"

    此功能目前是实验性的，未来可能发生更改或删除。你可以在论坛中进行反馈，来帮助其变为生产可用。

!!! danger "废弃"

    在即将发布的 Rasa Open Source 3.7 版本中，我们将删除此实验性功能。有关 Rasa Pro 中标记功能的文档，请点击[此处](monitoring/analytics/realtime-markers.md)。

## 概述 {#overview}

标记是允许你在对话中描述和标记兴趣点来评估对话机器人的条件。

在 Rasa 中，对话表示为一系列事件，其中包括执行的机器人动作、检测到的意图和设置的槽。标记允许你描述此类事件的条件。当条件满足时，对相关事件进行标记，以供进一步分析或检查。

标记有多种下游应用。例如，它们可用于定义和衡量对话机器人的关键绩效指标（KPI），例如对话完成或任务成功。以 [Carbon Bot](https://rasa.com/blog/using-conversation-tags-to-measure-carbon-bots-success-rate) 为例，它可以帮助用户抵消飞行中的碳排放。对于 Carbon Bot，你可以将对话完成定位为“所有强制槽都已填充”，将任务成功定义为“所有强制槽都已填充且碳估算已成功计算”。标记这些重要时间的发生时间可以让你衡量 Carbon Bot 的成功率。

标记还允许你通过显示重要事件来进行进一步检查并诊断对话。例如，你可能会观察到 Carbon Bot 倾向于成功设置 `travel_departure` 和 `travel_destination` 槽，但未能设置 `travel_flight_class` 槽。你可以定义一个标记来量化这种行为发生的频率，并将相关对话作为[对话驱动开发（CDD）](conversation-driven-development.md)的一部分进行审查。

标记以 YAML 编写定义在标记配置文件中。例如，如下是为 Carbon Bot 定义对话完成和任务成功的标记：

```yaml
marker_dialogue_completion:
  and:
    - slot_was_set: travel_departure
    - slot_was_set: travel_destination
    - slot_was_set: travel_flight_class

marker_task_success:
  and:
    - slot_was_set: travel_departure
    - slot_was_set: travel_destination
    - slot_was_set: travel_flight_class
    - action: provide_carbon_estimate
```

如下是除了 `travel_flight_class` 之外所有强制槽都被设置的对话标记：

```yaml
marker_dialogue_mandatory_slot_failure:
  and:
    - slot_was_set: travel_departure
    - slot_was_set: travel_destination
    - not:
      - slot_was_set: travel_flight_class
```

接下来的部分解释了如何编写标记定义，如何将他们应用到现在的对话中，以及输入格式是什么样的。

## 定义标记 {#defining-markers}

标记应在以 YAML 编写的标记配置文件中定义。每个标记都应该有一个唯一的标识符，并且至少包含一个事件条件。标记还可以包含运算符，允许你表示更细微的行为或组合事件条件。

考虑如下标记定义：

```yaml
marker_mood_expressed:
  or:
    - intent: mood_unhappy
    - intent: mood_great
```

唯一的标记符号是 `marker_mood_expressed`。此标记定义包含一个运算符和两个事件条件：`intent: mood_unhappy` 和 `intent: mood_great`。

在对话中用户表达 `mood_unhappy` 或 `mood_great` 的每个点上，此标记都是正确的。更准确地说，对于 `intent` 等于 `mood_unhappy` 或 `mood_great` 的 `UserUttered()` 的每个事件，该标记都为真。

### 事件条件 {#event-conditions}

支持如下事件条件标签：

- `action`：执行了指定的对话机器人动作。
- `intent`：检测到指定的用户意图。
- `slot_was_set`：指定的槽被设置。

还支持标签的否定形式：

- `not_action`：事件不是指定的对话机器人动作。
- `not_intent`：事件不是指定的用户意图。
- `slot_was_not_set`：指定的槽未被设置。

### 运算符 {#operators}

支持如下运算符：

- `and`：适用所有列出的条件。
- `or`：适用任意列出的条件。
- `not`：条件不适用。此运算符仅接受一个条件。
- `seq`：按指定顺序应用的条件列表，中间可以发生任意数量的事件。
- `at_least_once`：列出的标记定义至少出现一次。只有第一次出现会被标记。
- `never`：列出的标记定义从未发生过。

### 标记配置 {#marker-configuration}

如下是一个包含多个标记定义的标记配置文件示例。该示例是为 mood bot 创建的，使用新的 `name` 槽来说明标签 `slot_was_set` 的使用：

```yaml
marker_name_provided:
  slot_was_set: name

marker_mood_expressed:
  or:
    - intent: mood_unhappy
    - intent: mood_great

marker_cheer_up_failed:
  seq:
    - intent: mood_unhappy
    - action: utter_cheer_up
    - action: utter_did_that_help
    - intent: deny

marker_bot_not_challenged:
  never:
    - intent: bot_challenge

marker_cheer_up_attempted:
  at_least_once:
    - action: utter_cheer_up

marker_mood_expressed_and_name_not_provided:
  and:
    - or:
      - intent: mood_unhappy
      - intent: mood_great
    - not:
      - slot_was_set: name
```

请注意以下事项：

- 每个标记都有一个唯一标识符（或名称），例如：`marker_name_provided`。
- 标记定义可以包含单个条件，如 `marker_name_provided` 中所示。
- 标记定义可以包含带有条件列表的单个运算符，如 `marker_mood_expressed`、`marker_cheer_up_failed`、`marker_bot_not_challenged` 和 `marker_cheer_up_attempted` 所示。
- 标记定义可以包含嵌套运算符，如 `marker_mood_expressed_and_name_not_provided` 所示。
- 根据对话机器人的 `domain.yml` 文件，分配给事件条件的值必须有效。例如，在 `marker_mood_expressed` 中，意图 `mood_unhappy` 和 `mood_unhappy` 都是 mood bot 的 `domain.yml` 文件中列出的意图。

!!! info "注意"

    你不能在另一个标记的定义中重复使用现有标记的名称。

## 提取标记 {#extracting-markers}

!!! info "信息"

    Rasa Pro 支持实时标记处理，了解有关[实时标记](monitoring/analytics/realtime-markers.md)的更多信息。

标记是从已经存在在追踪器存储中的对话中提取的。要了解如何在追踪器存储中存储与对话机器人的交互，请参见[追踪器存储](tracker-stores.md)页面。

在标记配置文件中创建标记定义并在追踪器存储中存储一些对话后，你可以通过运行如下命令将标记应用于追踪器：

```shell
rasa evaluate markers all --config markers.yml extracted_markers.csv
```

此脚本将处理你在标记配置文件中提供的标记定义：`markers.yml`。该脚本将在指定的输出文件中输出提取的标记：`extracted_markers.csv`。它还将生成两个汇总统计文件。输出文件的格式将在下一节中描述。

默认情况下，该脚本将根据对话机器人的 `domain.yml` 文件验证标记定义。要指定不同的领域文件，请使用可选的 `--domain` 参数。

默认情况下，该脚本将处理对话机器人的 `endpoint.yml` 中的追踪器存储。但是，你可以使用可选的 `--endpoint` 参数指定不同的端点文件。

支持三种不同的追踪器加载策略：`all`，`sample_n` 和 `first_n`。选项 `all` 将处理追踪器存储中的所有追踪器。其他两种策略处理 N 个追踪器的子集，或者顺序地（通过使用 `first_n`），或者通过无放回的均匀抽样（通过使用 `sample_n`）。采样策略还允许设置随机数种子。有关每个策略用法的更多信息，请输入如下命令，将 `<strategy>` 替换为以下之一：`all`、`first_n` 和 `sample_n`：

```shell
rasa evaluate markers <strategy> --help
```

!!! info "注意"

    追踪器存储中的每个追踪器可以包含多个会话。该脚本分别处理每个会话，并通过 `session_idx` 对其进行索引。

接下来的两节描述了提取标记的格式和计算的统计数据。

### 提取的标记 {#extracted-markers}

对于标记配置文件中定义的每个标记，将提取如下信息：

1. 应用标记的事件的索引。
2. 在应用标记的事件之前的用户轮数。每个 `UserUttered` 事件都被视为一个用户轮次。

时间的索引和前面的用户轮次都可以只是到达重要事件（例如任务成功）所花费的时间。事件的索引将计算所有事件，包括不属于对话的事件，例如开始新会话或执行自定义操作。另一方面，前面的用户轮次可以更直观地指示对话长度，特别是从最终用户的角度来看。

之前的用户轮次可用于评估和改进对话机器人。例如，假设用户不得不多次改写他们的话语，这导致他们的对话变得更长。对话最终可能会成功完成任务，但是将其显示出来可以让你识别对话机器人无法理解的话语。然后，你可以将这些有挑战性的话语用作额外的训练数据，作为[对话驱动开发（CDD）](conversation-driven-development.md)的一部分来进一步改进对话机器人。

!!! info "注意"

    对于使用 `at_least_once` 运算符定义的标记，上述信息只会在第一次出现时被提取。

提取的标记以表格格式存储在你在脚本中指定的 `.csv` 文件中，例如：`extracted_markers.csv`。提取的标记输出文件包含如下列：

- `sender_id`：取自追踪器。
- `session_idx`：一个索引会话的整数，从 0 开始。
- `marker`：唯一的标记标识符。
- `event_idx`：一个索引事件的整数，从 0 开始。
- `num_preceding_user_turns`：一个整数，表示在应用标记的事件之前的用户轮次。

如下是提取的标记输出文件的示例（对于包含两个标记的标记配置文件：`marker_mood_expressed` 和 `marker_cheer_up_failed`）：

```csv
sender_id,session_idx,marker,event_idx,num_preceding_user_turns
3c1afa1ed72c4116ba6670a1668f1b4a,0,marker_mood_expressed,2,0
4d55093e9696452c8d1157fa33fd54b2,0,marker_mood_expressed,7,1
4d55093e9696452c8d1157fa33fd54b2,0,marker_cheer_up_failed,14,2
c00b3de97713427d85524c4374125db1,0,marker_mood_expressed,2,0
```

对于每个 `sender_id` 和 `session_idx`，每一行表示在标记列下指定的标记的出现。

### 计算的统计信息 {#computed-statistics}

默认情况下，该命令会计算有关收集的信息的摘要统计信息。要禁用统计计算，请使用可选标识 `--no-stats`。

该脚本计算如下统计信息：

- 对于每个会话和每个标记：每个会话统计信息包括在应用标记的事件之前的算数平均值、中值、最小和最大用户轮次。
- 对于所有会话和每个标记：
    - 总体统计数据包括在任何会话中应用标记的事件之前的算数平均值、中值、最小和最大用户轮次。
    - 每个标记至少应用一次的会话数和会话百分比。

结果以表格格式存储在 `stats-overall.csv` 和 `stats-per-session.csv` 中。你可以使用可选参数 `--stats-file-prefix` 更改文件名中的前缀统计信息。例如，如下脚本将生成文件 `my-statistics-overall.csv` 和 `my-statistics-per-session.csv`：

```shell
rasa evaluate markers all --stats-file-prefix "my-statistics" extracted_markers.csv
```

这两个统计文件包含如下列：

- `sender_id`：取自追踪器。如果统计是针对所有会话计算的，则这将为 `all`。
- `session_idx`：一个索引会话的整数，从 0 开始。如果统计是针对所有会话计算的，则这将为 `nan`（不是数字）。
- `marker`：唯一的标记标识符。
- `statistic`：计算的统计量描述。
- `value`：计算统计量的整数或浮点数值。如果统计信息不可用，则这将为 `nan`（不是数字）。

如下是一个示例 `stats-per-session.csv` 的输出：

```csv
sender_id,session_idx,marker,statistic,value
3c1afa1ed72c4116ba6670a1668f1b4a,0,marker_cheer_up_failed,count(number of preceding user turns),0
4d55093e9696452c8d1157fa33fd54b2,0,marker_cheer_up_failed,count(number of preceding user turns),1
c00b3de97713427d85524c4374125db1,0,marker_cheer_up_failed,count(number of preceding user turns),0
3c1afa1ed72c4116ba6670a1668f1b4a,0,marker_cheer_up_failed,max(number of preceding user turns),nan
4d55093e9696452c8d1157fa33fd54b2,0,marker_cheer_up_failed,max(number of preceding user turns),2
c00b3de97713427d85524c4374125db1,0,marker_cheer_up_failed,max(number of preceding user turns),nan
3c1afa1ed72c4116ba6670a1668f1b4a,0,marker_cheer_up_failed,mean(number of preceding user turns),nan
4d55093e9696452c8d1157fa33fd54b2,0,marker_cheer_up_failed,mean(number of preceding user turns),2.0
c00b3de97713427d85524c4374125db1,0,marker_cheer_up_failed,mean(number of preceding user turns),nan
3c1afa1ed72c4116ba6670a1668f1b4a,0,marker_cheer_up_failed,median(number of preceding user turns),nan
4d55093e9696452c8d1157fa33fd54b2,0,marker_cheer_up_failed,median(number of preceding user turns),2.0
c00b3de97713427d85524c4374125db1,0,marker_cheer_up_failed,median(number of preceding user turns),nan
3c1afa1ed72c4116ba6670a1668f1b4a,0,marker_cheer_up_failed,min(number of preceding user turns),nan
4d55093e9696452c8d1157fa33fd54b2,0,marker_cheer_up_failed,min(number of preceding user turns),2
c00b3de97713427d85524c4374125db1,0,marker_cheer_up_failed,min(number of preceding user turns),nan
3c1afa1ed72c4116ba6670a1668f1b4a,0,marker_mood_expressed,count(number of preceding user turns),1
4d55093e9696452c8d1157fa33fd54b2,0,marker_mood_expressed,count(number of preceding user turns),1
c00b3de97713427d85524c4374125db1,0,marker_mood_expressed,count(number of preceding user turns),1
3c1afa1ed72c4116ba6670a1668f1b4a,0,marker_mood_expressed,max(number of preceding user turns),0
4d55093e9696452c8d1157fa33fd54b2,0,marker_mood_expressed,max(number of preceding user turns),1
c00b3de97713427d85524c4374125db1,0,marker_mood_expressed,max(number of preceding user turns),0
3c1afa1ed72c4116ba6670a1668f1b4a,0,marker_mood_expressed,mean(number of preceding user turns),0.0
4d55093e9696452c8d1157fa33fd54b2,0,marker_mood_expressed,mean(number of preceding user turns),1.0
c00b3de97713427d85524c4374125db1,0,marker_mood_expressed,mean(number of preceding user turns),0.0
3c1afa1ed72c4116ba6670a1668f1b4a,0,marker_mood_expressed,median(number of preceding user turns),0.0
4d55093e9696452c8d1157fa33fd54b2,0,marker_mood_expressed,median(number of preceding user turns),1.0
c00b3de97713427d85524c4374125db1,0,marker_mood_expressed,median(number of preceding user turns),0.0
3c1afa1ed72c4116ba6670a1668f1b4a,0,marker_mood_expressed,min(number of preceding user turns),0
4d55093e9696452c8d1157fa33fd54b2,0,marker_mood_expressed,min(number of preceding user turns),1
c00b3de97713427d85524c4374125db1,0,marker_mood_expressed,min(number of preceding user turns),0
```

请注意，不可用统计信息的值为 `nan`。例如，因为 `marker_cheer_up_failed` 从未发生在追踪器 `3c1afa1ed72c4116ba6670a1668f1b4a` 会话 `0` 中，所以之前用户轮次的最小、最大、中值和平均数均为 `nan`。

如下是一个示例 `stats-overall.csv` 的输出：

```csv
sender_id,session_idx,marker,statistic,value
all,nan,-,total_number_of_sessions,3
all,nan,marker_cheer_up_failed,number_of_sessions_where_marker_applied_at_least_once,1
all,nan,marker_cheer_up_failed,percentage_of_sessions_where_marker_applied_at_least_once,33.333
all,nan,marker_mood_expressed,number_of_sessions_where_marker_applied_at_least_once,3
all,nan,marker_mood_expressed,percentage_of_sessions_where_marker_applied_at_least_once,100.0
all,nan,marker_cheer_up_failed,count(number of preceding user turns),1
all,nan,marker_cheer_up_failed,mean(number of preceding user turns),2.0
all,nan,marker_cheer_up_failed,median(number of preceding user turns),2.0
all,nan,marker_cheer_up_failed,min(number of preceding user turns),2
all,nan,marker_cheer_up_failed,max(number of preceding user turns),2
all,nan,marker_mood_expressed,count(number of preceding user turns),3
all,nan,marker_mood_expressed,mean(number of preceding user turns),0.333
all,nan,marker_mood_expressed,median(number of preceding user turns),0.0
all,nan,marker_mood_expressed,min(number of preceding user turns),0
all,nan,marker_mood_expressed,max(number of preceding user turns),1
```

请注意，因为每一行计算所有会话的统计信息，所以 `sender_id` 为 `all`，`session_idx` 为 `nan`。

## 配置 CLI 命令 {#configuring-the-cli-command}

访问 [CLI 页面](command-line-interface.md#rasa-evaluate-markers)，了解有关配置标记提取和统计计算过程的更多信息。
