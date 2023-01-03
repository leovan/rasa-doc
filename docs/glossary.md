# Rasa 词汇表

## [动作 Action](/actions/) {#action}

对话机器人在对话中执行的单个步骤（例如：调用 API 或发送响应至用户）。

## [动作服务器 Action Server](/action-server/) {#action-server}

运行自定义动作代码的服务器，独立于开源 Rasa。Rasa 使用 Python 维护 Rasa SDK 来实现自定义动作，尽管也可以使用其他语言自定义动作。

## 标注 Annotation {#annotation}

为消息和对话添加标签，以便可以用于训练模型。

## [业务逻辑 Business Logic](/business-logic/) {#business-logic}

因业务需求而需要满足的条件。例如：需要名字和姓氏、地址和密码才能够创建账号。在 Rasa 对话机器人中，业务逻辑是使用基于规则的动作（例如[表单](/forms/)）实现。

## [闲聊 Chitchat](/chitchat-faqs/) {#chitchat}

一种对话模式，其中用户说的内容与他们的目标没有直接关系。包括问候、询问近况等。阅读有关处理[闲聊和 FAQ](/chitchat-faqs/) 来了解如何使用开源 Rasa 实现这一点。

## 内容管理系统 CMS {#cms}

一种在外部存储机器人响应而不是将他们直接包含在领域中的方法。内容管理系统将影响文本与训练数据分离。更多详细信息请参见 [NLG 服务器](/nlg/)。

## [对话驱动的开发 Conversation-Driven Development (CDD)](/conversation-driven-development/) {#conversation-driven-development-cdd}

结合工程最佳实践，使用用户消息和对话数据影响对话机器人设计和训练模型的过程。CDD 有 6 个步骤：共享、审查、标注、修复、跟踪和测试。

## [对话测试 Conversation Tests](/testing-your-assistant/) {#conversation-tests}

修改后的故事格式，除了意图标签外，还包括用户消息的全文。测试对话被存在测试文件（conversation_tests.md）用，以用于评估在整个对话中的模型预测。

## [组件 Component](/components/) {#component}

[模型配置](/model-configuration/)中对话机器人 [NLU 管道](/tuning-your-model/#how-to-choose-a-pipeline)中的一个元素。

传入消息由称为管道的一系列组件处理。组件可以执行从实体提取，到意图分类，再到预处理等任务。

## [条件响应变体 Conditional Response Variation](/responses/#conditional-response-variations) {#conditional-response-variation}

仅当当前对话状态满足领域或响应文件中定义的某些约束时才能使用的响应变体。如果约束和对话状态匹配，则 Rasa 可以使用这种变体。

## [自定义动作 Custom Action](/actions/#custom-actions) {#custom-action}

由对话机器人开发人员编写的可以运行任意代码的动作，主要用于与外部系统和 API 交互。

## [默认动作 Default Action](/actions/#default-actions) {#default-action}

带有预定义功能的内置动作。

## [双重意图和实体转换器 DIET](/components/#dietclassifier) {#diet}

双重意图和实体转换器。开源 Rasa 使用的默认 NLU 架构，其用于执行意图分类和实体提取。

## [领域 Domain](/domain/) {#domain}

定义对话机器人的输入和输出。

它包括对话机器人已知的所有意图、实体、槽、动作和表单列表。

## [实体 Entity](/training-data-format/#entities) {#entity}

从用户消息中提取的关键词。例如：电话号码、人名、地点、产品名称。

## [事件 Event](/action-server/events/) {#event}

对话中发生的事情。例如，`UserUttered` 事件表示用户输入消息，`ActionExecuted` 事件表示对话机器人执行动作。Rasa 中的所有对话都被表示为一系列事件。

## 常见问题 FAQs {#faqs}

常见问题（FAQs）是用户提出的常见问题。在构建对话机器人的上下文中，这通常意味着用户发送消息，对话机器人发送响应，而无需考虑对话的上下文。阅读有关处理[闲聊和 FAQ](/chitchat-faqs/) 的信息来了解如何使用开源 Rasa 实现这一点。

## [表单 Form](/forms/) {#form}

一种要求用户提供多条消息的自定义动作。

例如，如果你需要城市、菜系和价格范围来推荐餐厅，你可以创建餐厅表单来收集信息。你可以在表单中描述[业务逻辑](/glossary/#business-logic)，例如在客户提到食物过敏时为他们提供一组不同的菜单选项。

## 预期和非预期路径 Happy / Unhappy Paths {#happy--unhappy-paths}

用于描述用户的输入是符合预期还是非预期的术语。如果对话机器人向用户询问一些信息，并且用户提供了信息，我们称其为预期路径。非预期路径则为可能的边缘情况。例如，用户拒绝提供请求的输入，更改对话主题或纠正他们之前说的话。

## [意图 Intent](/nlu-training-data/) {#intent}

在给定的用户信息中，用户试图传达或完成的事情（例如，问候、指定位置）。

## [交互学习 Interactive Learning](/writing-stories/#using-interactive-learning) {#interactive-learning}

在 Rasa CLI 中，这是一种训练模式，开发人员在对话的每一步都可以纠正和验证对话机器人的预测。对话可以保存为故事并添加到对话机器人的训练数据中。

## [知识库和知识图谱 Knowledge Base / Knowledge Graph](/action-server/knowledge-bases/) {#knowledge-base--knowledge-graph}

表示对象之间复杂关系和层次结构的可查询数据库。知识库动作允许开源 Rasa 从知识库中获取信息并将其用于响应。

## [3 级对话机器人 Level 3 Assistant](https://blog.rasa.com/5-levels-of-conversational-ai-2020-update/) {#level-3-assistant}

可以处理比简单的来回交流更复杂对话的对话机器人。3 级对话机器人能够使用先前的对话轮次的上下文来选择适当的下一步动作。

## [消息频道 Messaging Channels](/messaging-and-voice-channels/) {#messaging-channels}

将开源 Rasa 与外部消息平台集成的连接器，最终用户可以在其中发送和接收消息。开源 Rasa 包括内置的消息频道，例如 Slack、Facebook Messenger 和网络聊天，以及创建自定义连接器的能力。

## [最小可行对话机器人 Minimum Viable Assistant](/conversation-driven-development/#cdd-in-early-stages-of-development) {#minimum-viable-assistant}

一个基本的对话机器人，可以处理大部分重要的预期路径故事。

## [自然语言生成 NLG](/nlg/) {#nlg}

自然语言生成（NLG）是生成自然语言信息发送给用户的过程。

Rasa 使用简单的基于模板的 NLG 方法。数据驱动的方法（例如神经网络 NLG）可以通过创建自定义 NLG 组件来实现。

## [自然语言理解 NLU](/nlu-training-data/) {#nlu}

自然语言理解（NLU）处理将人类语言解析和理解为结构化格式。

## [NLU 组件 NLU Component](/components/) {#nlu-component}

Rasa NLU 管道中（参见[管道](/glossary/#pipeline)）中处理传入消息的元素。组件执行从实体提取，到意图分类，再到预处理等任务。

## [管道 Pipeline](/tuning-your-model/) {#pipeline}

在 Rasa 对话机器人 NLU 系统中定义的 NLU 组件（参见 [NLU 组件](/glossary/#nlu-component)）列表。在返回最终的结构化输出之前，每个组件都会对用户消息进行处理。

## [策略 Policy](/policies/) {#policy}

用于预测对话系统下一个动作的开源 Rasa 组件。策略决定对话流应该如何进行。一个典型的配置包含多个策略，置信度最高的策略决定会话中要采取的下一步动作。

## Rasa Core {#rasa-core}

过时的。Rasa Core 和 Rasa NLU 在 1.x 中合并为一个包。Core 的功能现在称为对话管理。

对话引擎根据上下文决定对话中下一步要做什么。是开源 Rasa 开源库的一部分。

## Rasa NLU {#rasa-nlu}

过时的，Rasa Core 和 Rasa NLU 在 1.x 中合并为一个包。Rasa NLU 的功能现在成为 NLU。

Rasa NLU 是开源 Rasa 的一部分，它执行自然语言理解（[NLU](#nlu)），包括意图分类和实体提取。

## [Rasa X/Enterprise](https://rasa.com/docs/rasa-enterprise/) {#rasa-xenterprise}

一个用于对话驱动开发的工具。Rasa X/Enterprise 可以帮助团队共享和测试使用开源 Rasa 创建的对话机器人、标注用户消息和查看对话。

## [检索意图 Retrieval Intent](/chitchat-faqs/) {#retrieval-intent}

一种特殊类型的意图，可以分为更小的子意图。例如，FAQ 检索意图包含对话机器人如何回答每个单独问题的子意图。

## [REST 频道 REST Channel](/connectors/your-own-website/) {#rest-channel}

用于构建自定义连接器的消息传递频道。包括一个输入频道，可以将用户消息发送给开源 Rasa，并可以指定回调 URL，用于发送对话机器人的响应动作。

## [响应/模板/话语 Response / Template / Utterance](/responses/) {#response--template--utterance}

对话机器人发送给用户的消息。可以包含文本、按钮、图像和其他内容。

## [规则 Rules](/rules/) {#rules}

用于指定类似规则行为的特殊训练数据，其中一个特定条件会始终预测一个特定的下一步动作。示例包括回答 FAQs、填写[表单](/forms/)或处理[回退](/fallback-handoff/#fallbacks)。

## [槽 Slot](/domain/#slots) {#slot}

Rasa 用于在对话过程中跟踪信息的健值存储。

## [故事 Story](/stories/) {#story}

对话模型的训练数据格式，由用户和对话机器人之间的对话组成。用户的消息表示为带注释的意图和实体，对话机器人的响应表示为一系列动作。

## [TED 策略 TED Policy](/policies/#ted-policy) {#ted-policy}

Transformer 嵌入对话策略（Transformer Embedding Dialogue Policy）。TED 是开源 Rasa 使用的基于机器学习的默认对话策略。当没有规则用于预测下一步动作时，TED 通过处理未遇见过的情况来补充基于规则的策略。

## [追踪器 Tracker](/tracker-stores/) {#tracker}

用于维护对话状态的开源 Rasa 组件，表示为列出当前会话中数据的 JSON 对象。

## 用户目标 User Goal {#user-goal}

用户想要实现的总体目标，例如：查找问题的答案、预约或购买保险。

一些工具将用户目标称为“意图”，但在 Rasa 术语中，意图与每个单独的用户消息相关联。

## 词嵌入/词向量 Word embedding / Word vector {#word-embedding--word-vector}

一个表示单词含义的浮点数向量。具有相似含义的词往往具有相似的向量。词嵌入通常作为机器学习算法的输入。
