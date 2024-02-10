# 回退和移交人工

这是有关如何处理对话机器人各种错误的指南。

即使你完美地设计了你的对话机器人，用户也不可避免地会对你的对话机器人说一些没有预料到的话。在这些情况下，你的对话机器人会失效，重要的是你需要确保如何优雅的执行。

## 处理超出范围的消息 {#handling-out-of-scope-messages}

为了避免用户不满意，你可以处理你知道用户会问但尚未实现用户目标的问题。

### 创建超出范围的意图 {#creating-an-out-of-scope-intent}

你需要在 NLU 训练数据中定义一个 `out_of_scope` 意图，并将任意已知的超出范围的请求添加为训练样本，例如：

```yaml title='nlu.yml'
nlu:
- intent: out_of_scope
  examples: |
    - I want to order food
    - What is 2 + 2?
    - Who's the US President?
```

同其他意图一样，你应该[从真实对话](conversation-driven-development.md)中获取大部分样本。

### 定义响应消息 {#defining-the-response-message}

你需要在领域文件中定义一个超出范围的响应。使用 `utter_out_of_scope` 作为默认响应，如下所示：

```yaml title='domain.yml'
responses:
  utter_out_of_scope:
  - text: Sorry, I can't handle that request.
```

### 创建超出范围的规则 {#creating-an-out-of-scope-rule}

最后，你需要为超出范围的请求编写一个规则：

```yaml title='rules.yml'
rules:
- rule: out-of-scope
  steps:
  - intent: out_of_scope
  - action: utter_out_of_scope
```

### 处理特定的超出范围的消息 {#handling-specific-out-of-scope-messages}

如果你观察到你的用户询问将来你希望将其转化为用户目标的某些事情，你可以将这些作为单独的意图来处理，从而让用户知道你已经理解了他们的信息，但目前还没有解决方案。例如：如果用户询问“I want to apply for a job at Rasa”，我们可以回复“I understand you're looking for a job, but I'm afraid I can't handle that skill yet.”。

类似 `out_of_scope` 意图的样本，你需要使用训练样本创建一个新的意图，定义响应消息并创建规则。

## 回退 {#fallbacks}

尽管 Rasa 会泛化未见过的信息，但某些信息仍可能会收到较低的置信度。使用回退将有助于确保这些低置信度的消息得到妥善处理，让你的对话机器人可以选择使用默认消息进行响应或尝试消除用户输入的歧义。

### NLU 回退 {#nlu-fallback}

要处理 NLU 置信度较低的信息，可以使用 [FallbackClassifier](components.md#fallbackclassifier)。使用此配置，当所有其他意图预测低于配置的置信度阈值时，则将预测为 `nlu_fallback` 意图。然后你可以为预测 `nlu_fallback` 时机器人应该做什么编写规则。

#### 更新配置 {#updating-the-configuration}

要使用 FallbackClassifier，需要将其添加到 NLU 流水线中：

```yaml title='config.yml'
pipeline:
# other components
- name: FallbackClassifier
  threshold: 0.7
```

#### 定义响应消息 {#defining-the-response-message-1}

通过添加响应来定义当消息被分类为低置信度时对话机器人应发送的消息：

```yaml title='domain.yml'
responses:
  utter_please_rephrase:
  - text: I'm sorry, I didn't quite understand that. Could you rephrase?
```

#### 创建一个 NLU 回退规则 {#creating-an-nlu-fallback-rule}

以下[规则](rules.md)将要求用户在发送分类为低置信度消息时重新对话：

```yaml title='rules.yml'
rules:
- rule: Ask the user to rephrase whenever they send a message with low NLU confidence
  steps:
  - intent: nlu_fallback
  - action: utter_please_rephrase
```

### 处理低置信度动作 {#handling-low-action-confidence}

由于用户可能发送非预期的消息，因此他们的行为可能会导致走上未知的对话路径。Rasa 的机器学习策略（例如 [TED 策略](policies.md#ted-policy)）经过优化可以用来处理这些未知路径。

要处理当机器学习策略无法以高置信度预测下一个动作时的情况，你可以配置[规则策略](policies.md#rule-policy)在没有[策略](policies.md)能给出下一个预测置信度高于配置的阈值的情况下预测一个默认的动作。

你可以使用以下步骤配置在动作置信度低的情况下运行的动作以及相应的置信度阈值：

#### 更新配置 {#updating-the-configuration-1}

你需要将 RulePolicy 添加到 `config.yml` 的 `policies` 中。默认情况下，规则策略带有以下设置：

```yaml title='config.yml'
policies:
- name: RulePolicy
  # Confidence threshold for the `core_fallback_action_name` to apply.
  # The action will apply if no other action was predicted with
  # a confidence >= core_fallback_threshold
  core_fallback_threshold: 0.4
  core_fallback_action_name: "action_default_fallback"
  enable_fallback_prediction: True
```

#### 定义默认响应消息 {#defining-the-default-response-message}

要定义当动作置信度低于阈值时对话机器人说些什么，需要定义一个 `utter_default` 响应：

```yaml title='domain.yml'
responses:
  utter_default:
  - text: Sorry I didn't get that. Can you rephrase?
```

当动作置信度低于阈值时，Rasa 将运行 `action_default_fallback` 动作。这将发送 `utter_default` 响应并恢复到导致回退的用户消息之前的对话状态，因此这不会影响未来动作的预测。

#### 自定义默认动作（可选） {#customizing-the-default-action-optional}

`action_default_fallback` 是开源 Rasa 中的默认动作，它将发送 `utter_default` 响应给用户。你可以创建自己的自定义动作用作回退（更多信息请参见[自定义动作](actions.md#custom-actions)）。以下代码片段是一个自定义动作的实现，它与 `action_default_fallback` 相同但使用一个不同的模板 `utter_fallback_template`：

```python title='actions.py'
from typing import Any, Text, Dict, List

from rasa_sdk import Action, Tracker
from rasa_sdk.events import UserUtteranceReverted
from rasa_sdk.executor import CollectingDispatcher

class ActionDefaultFallback(Action):
    """Executes the fallback action and goes back to the previous state
    of the dialogue"""

    def name(self) -> Text:
        return ACTION_DEFAULT_FALLBACK_NAME

    async def run(
        self,
        dispatcher: CollectingDispatcher,
        tracker: Tracker,
        domain: Dict[Text, Any],
    ) -> List[Dict[Text, Any]]:
        dispatcher.utter_message(template="my_custom_fallback_template")

        # Revert user message which led to fallback.
        return [UserUtteranceReverted()]
```

### 两阶段回退 {#two-stage-fallback}

为了让对话机器人有机会弄清楚用户想要什么，通常希望它通过提出澄清问题来尝试消除用户信息的歧义。两阶段回退使用如下序列处理多个阶段中的低 NLU 置信度：

1. 用户消息被分类为低置信度
    - 要求用户确认意图
2. 用户确认或否认意图
    - 如果确认，对话会继续进行，就像从一开始就以高置信度被分类为该意图一样。不会采取进一步的回退步骤。
    - 如果否认，则要求用户重新表述他们的消息。
3. 用户重新表述他们的意图
    - 如果消息以高置信度分类，则对话继续进行，就像从一开始就以高置信度被分类为该意图一样。
    - 如果重新表述的消息置信度仍然较低，则要求用户确认意图。
4. 用户确认或否认重新表述的意图
    - 如果确认，对话会继续进行，就像从一开始就以高置信度被分类为该意图一样。
    - 如果否认，则会触发最终的回退动作（例如：移交人工）。默认的最终回退动作是调用 `action_default_fallback`。此动作会导致对话机器人发出 `utter_default` 响应并重置对话状态，就好像在两阶段回归期间的对话没有发生一样。

使用如下步骤可以启用两阶段回退：

#### 更新配置 {#updating-the-configuration-2}

将 FallbackClassifier 添加到流水线中，并将 [RulePolicy](policies.md#rule-policy) 添加到策略配置中：

```yaml title='config.yml'
recipe: default.v1
pipeline:
# other components
- name: FallbackClassifier
  threshold: 0.7

policies:
# other policies
- RulePolicy
```

#### 定义回退响应 {#defining-the-fallback-responses}

要定义对话机器人如何要求用户重新表述他们的消息，请定义 `utter_ask_rephrase` 响应：

```yaml title='domain.yml'
responses:
  utter_ask_rephrase:
  - text: I'm sorry, I didn't quite understand that. Could you rephrase?
```

Rasa 提供了默认实现来询问用户的意图以及要求用户重新表述。要自定义这些动作的行为，请参见[默认动作](default-actions.md)的文档。

#### 定义两阶段回退规则 {#defining-a-two-stage-fallback-rule}

将以下[规则](rules.md)添加到训练数据中。这个规则将确保在收到分类置信度较低的消息时激活两阶段回退：

```yaml title='rules.yml'
rules:
- rule: Implementation of the Two-Stage-Fallback
  steps:
  - intent: nlu_fallback
  - action: action_two_stage_fallback
  - active_loop: action_two_stage_fallback
```

#### 定义最终回退动作 {#defining-an-ultimate-fallback-action}

要定义在用户否认重新表述的意图时对话机器人的响应，请定义 `utter_default` 响应：

```yaml title='domain.yml'
responses:
  utter_default:
  - text: I'm sorry, I can't help you.
```

或者你可以通过编写[自定义动作](actions.md#custom-actions)来自定义 `action_default_fallback` 以获得更复杂的行为。例如，如果你希望机器人呼叫人类并停止与用户的交互：

```python title='actions.py'
from typing import Any, Dict, List, Text

from rasa_sdk import Action, Tracker
from rasa_sdk.events import UserUtteranceReverted
from rasa_sdk.executor import CollectingDispatcher

class ActionDefaultFallback(Action):
    def name(self) -> Text:
        return "action_default_fallback"

    def run(
        self,
        dispatcher: CollectingDispatcher,
        tracker: Tracker,
        domain: Dict[Text, Any],
    ) -> List[Dict[Text, Any]]:

        # tell the user they are being passed to a customer service agent
        dispatcher.utter_message(text="I am passing you to a human...")

        # assume there's a function to call customer service
        # pass the tracker so that the agent has a record of the conversation between the user
        # and the bot for context
        call_customer_service(tracker)

        # pause the tracker so that the bot stops responding to user input
        return [ConversationPaused(), UserUtteranceReverted()]
```

!!! warning "自定义最终回退动作返回的事件"

    你应该将 `UserUtteranceReverted()` 作为自定义 `action_default_fallback` 返回的事件之一。不包含此事件将导致追踪器包括在两阶段回退过程中发生的所有事件，这可能会干扰来自对话机器人策略流水线后续动作预测。最好将在两阶段回退过程中发生的事情视为没有发生，以便对话机器人可以应用其规则或记忆的故事来正确预测下一个动作。

## 移交人工 {#human-handoff}

作为回退动作的一部分，你可能希望对话机器人将其移交给人工。例如作为两阶段回退中的最终动作，或者当用户明确要求人工时。实现人工切换的一种直接方法是配置[消息或语音频道](messaging-and-voice-channels.md)，来根据特定的机器人或用户消息切换它监听的主机。

例如作为两阶段回退的最后一个动作，对话机器人可以问用户：“Would you like to be transferred to a human assistant?”，如果回答是，对话机器人会发送一条带有特定有效负载的消息，例如“handoff_to_human”到频道中。当频道看到此消息时，它将停止监听 Rasa 服务器，并向人工频道发送一条消息，其中包含截至该点的聊天对话记录。

从前端传递给人类的实现取决于你使用的频道。你可以在 [Financial 示例](https://github.com/RasaHQ/financial-demo) 和 [Helpdesk-Assistant](https://github.com/RasaHQ/helpdesk-assistant) 中看到使用[聊天室](https://github.com/scalableminds/chatroom)频道改编的示例实现。

## 总结 {#summary}

为了让你的对话机器人优雅地处理错误，你应该处理已知的超出范围的消息并添加一种回退行为。如果你想增加移交人工，可以添加它或者作为回退设置中的最后一步。以下是你需要对每种方法进行更改的摘要：

对于超出范围的意图：

- 对每个超出范围的意图在 NLU 数据中添加训练样本
- 定义超出范围的响应或动作
- 对每个超出范围的意图定义规则
- 将 RulePolicy 添加到 config.yml 中

对于单阶段 NLU 回退：

- 在 config.yml 中将 FallbackClassifier 添加到流水线中
- 定义回退响应或动作
- 定义用于 `nlu_fallback` 意图的规则
- 将 RulePolicy 添加到 config.yml 中

对于处理低置信度：

- 在 config.yml 中为核心回退配置 RulePolicy
- 可选地自定义配置的回退动作
- 定义一个 `utter_default` 响应

对于两阶段回退：

- 在 config.yml 中将 FallbackClassifier 添加到流水线中
- 为触发 `action_two_stage_fallback` 动作的 `nlu_fallback` 意图定义规则
- 在领域中定义超出范围的意图
- 将 RulePolicy 添加到 config.yml 中

对于移交人工：

- 配置前端以切换主机
- 编写自定义动作（可能为回退动作）来发送移交负载
- 添加用于触发移交的规则（如果不是回退的一部分）
- 将 RulePolicy 添加到 config.yml 中
