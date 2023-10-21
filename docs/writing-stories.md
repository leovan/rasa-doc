# 编写对话数据

对话数据包括用于 Rasa 对话机器人对话管理模型训练数据的故事和规则。精心编写的对话数据可以使得对话机器人能够可靠地遵循设置的对话路径并泛化到预期外的路径。

## 设计故事 {writing-stories}

在设计故事时，需要考虑两种对话交互：预期的和非预期的路径。预期的路径描述了当用户遵循预期的对话流程，并在收到提示时始终提供必要的信息。但是，用户经常会因为问题、闲聊或询问其他信息而偏离预期的路径，我们称这些为非预期的路径。

让对话机器人优雅地处理非预期的路径很重要，但同时无法预测一个用户可能采取的路径。通常，开发者在设计非预期的路径时会尝试考虑所有可能的分歧路径。为状态机中的每个可能状态（其中很多永远无法达到）进行规划需要大量额外的工作并会显著增加训练时间。

相反，我们建议在设计非预期的路径时采用[对话驱动的开发](conversation-driven-development.md)。对话驱动开发建议尽早与测试用户分享对话机器人，并收集真实的对话数据，这些数据可以准确地告诉你用户是如何偏离预期路径的。从这些数据中，你可以创建故事来完成用户的请求，并思考如何引导他们回到预期的路径上来。

## 何时编写故事或规则 {#when-to-write-stories-vs-rules}

[规则](rules.md)是对话管理用于处理应始终遵循相同路径的对话片段的一种训练数据。

规则在实现如下场景中很有用：

- [单轮交互](chitchat-faqs.md)：有些消息不需要上下文就可以进行回答。规则是一种将意图映射到响应的简单方法，可以为这些消息指定固定的答案。
- [回退行为](fallback-handoff.md)：结合 [`FallbackClassifier`](components.md#fallbackclassifier)，可以编写规则来响应具有一定回退行为的低置信度用户消息。
- [表单](forms.md)：激活和提交表单通常都会遵循固定的路径。你可以编写规则来处理表单期间的[非预期输入](unexpected-input.md)。

因为规则无法泛化到未遇见过的对话，因此你应该将他们保留为单轮对话片段，并使用故事来训练多轮对话。

一个对话机器人将意图为 `greet` 的用户消息返回固定响应 `utter_greet` 的规则示例如下：

```yaml
rules:
- rule: Greeting Rule
  steps:
  - intent: greet
  - action: utter_greet
```

对于多轮交互，你需要定义一个故事，例如：

```yaml
stories:
 - story: Greeting and ask user how they're doing
   steps:
   - intent: greet
   - action: utter_greet
   - action: utter_ask_how_doing
   - intent: doing_great
   - action: utter_happy
```

## 管理对话流 {#managing-the-conversation-flow}

以下是管理故事中对话流的一些提示：

### 何时使用槽来影响对话 {#when-to-use-slots-to-influence-conversations}

槽充当的是对话机器人的内存。当定义一个槽时，你可以定义一个[槽](domain.md#slots)是否应该影响对话。`influence_conversation` 属性设置为 `false` 的槽将仅用于存储信息。`influence_conversation` 属性设置为 `true` 的槽可以基于存储的信息来影响对话流。

可以基于[槽映射](domain.md#slot-mappings)在每条用户信息之后设置槽。它们也可以通过响应用于消息的[自定义动作](actions.md)来进行设置。所有影响对话的槽都需要添加到故事或规则中。例如：你可以使用自定义动作设置的布尔值槽使用如下故事并根据其值控制对话流：

```yaml
stories:
- story: Welcome message, premium user
  steps:
   - intent: greet
   - action: action_check_profile
   - slot_was_set:
     - premium_account: true
   - action: utter_welcome_premium

- story: Welcome message, basic user
  steps:
   - intent: greet
   - action: action_check_profile
   - slot_was_set:
     - premium_account: false
   - action: utter_welcome_basic
   - action: utter_ask_upgrade
```

如果你不希望槽影响对话流，则应将槽的 `influence_conversation` 属性设置为 `false`。不需要在故事中对不影响对话的槽包含 `slot_was_set` 事件。

### 实现分支逻辑 {#implementing-branching-logic}

在编写故事时，有时候下一个动作将取决于一个自定义动作的返回值。在这些情况下，重要的是要在返回槽和直接使用自定义动作代码来影响对话机器人下一步做什么之间找到平衡。

当某个值仅用于确定对话机器人的响应的情况时，可以考虑将决策逻辑嵌入自定义动作中，而不是在故事中使用一个特征化的槽。这将有助于降低整体复杂性并使得故事更易于管理。

例如，你可以将这些故事：

```yaml
stories:
- story: It's raining now
  steps:
  - intent: check_for_rain
  - action: action_check_for_rain
  - slot_was_set:
    - raining: true
  - action: utter_is_raining
  - action: utter_bring_umbrella

- story: It isn't raining now
  steps:
  - intent: check_for_rain
  - action: action_check_for_rain
  - slot_was_set:
    - raining: false
  - action: utter_not_raining
  - action: utter_no_umbrella_needed
```

通过如下自定义动作代码：

```python
def run(self, dispatcher, tracker, domain):
    is_raining = check_rain()
    if is_raining:
        dispatcher.utter_message(template="utter_is_raining")
        dispatcher.utter_message(template="utter_bring_umbrella")
    else:
        dispatcher.utter_message(template="utter_not_raining")
        dispatcher.utter_message(template="utter_no_umbrella_needed")
    return []
```

转换为一个单独的故事：

```yaml
stories:
- story: check for rain
  steps:
  - intent: check_for_rain
  - action: action_check_for_rain
```

在某个值用于影响未来的动作流的情况下，则返回一个特征化槽来确定故事。例如，如果你想收集有关新用户的信息，而不是返回用户的信息，则故事可能如下所示：

```yaml
stories:
- story: greet new user
  steps:
  - intent: greet
  - action: check_user_status
  - slot_was_set:
    - new_user: true
  - action: utter_greet
  - action: new_user_form
  - active_loop: new_user_form
  - active_loop: null

- story: greet returning user
  steps:
  - intent: greet
  - action: check_user_status
  - slot_was_set:
    - new_user: false
  - action: utter_greet
  - action: utter_how_can_help
```

### 使用或语句和检查点 {#using-or-statements-and-checkpoints}

[或语句](stories.md#or-statements)和[检查点](stories.md#checkpoints)可用于减少必须编写的故事数量。但你应该谨慎使用它们。过度使用或语句和检查点会减慢训练速度，创建过多的检查点会让你的故事难以理解。

#### 或语句 {#or-statements}

在对话机器人以相同方式处理不同意图或槽事件的故事中，可以使用或语句作为创建新故事的替代方法。

例如，你可以将如下两个故事：

```yaml
stories:
- story: newsletter signup
  steps:
  - intent: signup_newsletter
  - action: utter_ask_confirm_signup
  - intent: affirm
  - action: action_signup_newsletter

- story: newsletter signup, confirm via thanks
  steps:
  - intent: signup_newsletter
  - action: utter_ask_confirm_signup
  - intent: thanks
  - action: action_signup_newsletter
```

用或语句合并为一个故事：

```yaml hl_lines="6 7 8"
stories:
- story: newsletter signup with OR
  steps:
  - intent: signup_newsletter
  - action: utter_ask_confirm_signup
  - or:
    - intent: affirm
    - intent: thanks
  - action: action_signup_newsletter
```

在训练阶段，这个故事将被划分为两个原始故事。

!!! warning "考虑重组数据"

    如果注意到在故事中经常使用或语句，请考虑重构意图来降低粒度并更加广泛地捕获用户信息。

#### 检查点 {#checkpoints}

检查点对于将故事模块化成经常重复的单独部分时很有用。例如，如果你希望对话机器人在每个对话流结束时询问用户反馈，可以使用检查点来避免在每个故事结束是包含反馈交互：

```yaml
stories:
- story: beginning of conversation
  steps:
  - intent: greet
  - action: utter_greet
  - intent: goodbye
  - action: utter_goodbye
  - checkpoint: ask_feedback

- story: user provides feedback
  steps:
  - checkpoint: ask_feedback
  - action: utter_ask_feedback
  - intent: inform
  - action: utter_thank_you
  - action: utter_anything_else

- story: user doesn't have feedback
  steps:
  - checkpoint: ask_feedback
  - action: utter_ask_feedback
  - intent: deny
  - action: utter_no_problem
  - action: utter_anything_else
```

!!! danger "避免过度使用"

    检查点旨在使在不同故事中重复使用某些对话部分变得更加容易。强烈反对在现有检查点内部使用检查点，这会显著增加训练时间并使故事难以理解。

### 在故事中创建逻辑中断 {#creating-logical-breaks-in-stories}

在设计对话流时，经常会创建长的故事来从头到尾捕获完整的对话交互。在很多情况下，由于需要考虑分支路径，这会增加需要训练故事的数量。相反，可以考虑将较长的故事分成较小的对话块来处理子任务。

一个用户处理信用卡丢失的预期路径可能如下所示：

```yaml
stories:
- story: Customer loses a credit card, reviews transactions, and gets a new card
  steps:
  - intent: card_lost
  - action: check_transactions
  - slot_was_set:
    - reviewed_transactions: ["starbucks"]
  - action: utter_ask_fraudulent_transactions
  - intent: inform
  - action: action_update_transactions
  - intent: affirm
  - action: utter_confirm_transaction_dispute
  - action: utter_replace_card
  - action: mailing_address_form
  - active_loop: mailing_address
  - active_loop: null
  - action: utter_sent_replacement
  - action: utter_anything_else
  - intent: affirm
  - action: utter_help
```

处理一个信用卡丢失涉及一系列子任务，包括用于欺诈交易的消费历史检查、确认替换卡的邮寄地址、然后跟进用户的其他请求。在对话中，对话机器人会在多个地方提示用户输入，创建需要用到的分支路径。

例如：当提示 `utter_ask_fraudulent_transactions` 时，如果并不适用，用户可能会以 `deny` 意图进行响应。当被问及对话机器人是否可以帮助他们时，用户可以以 `deny` 意图进行响应。

我们可以将这个长故事分成几个小故事：

```yaml
stories:
- story: Customer loses a credit card
  steps:
  - intent: card_lost
  - action: utter_card_locked
  - action: spending_history_form
  - active_loop: spending_history_form
  - active_loop: null
  - slot_was_set:
    - reviewed_transactions: ["starbucks"]
  - action: utter_ask_fraudulent_transactions

- story: Customer reviews transactions and gets a new card
  steps:
  - action: utter_ask_fraudulent_transactions
  - intent: inform
  - action: action_update_transactions
  - slot_was_set:
    - reviewed_transactions: ["target", "starbucks"]
  - intent: affirm
  - action: utter_confirm_transaction_dispute
  - action: utter_replace_card
  - action: mailing_address_form
  - active_loop: mailing_address
  - active_loop: null
  - action: utter_sent_replacement
  - action: utter_anything_else

- story: Customer has something else they need help with
  steps:
  - action: utter_anything_else
  - intent: affirm
  - action: utter_help
```

## 处理上下文切换 {#handling-context-switching}

通常，用户不会用你向他们询问的消息做出回应，而是会用不相关的问题来偏离预期的路径。使用 [CDD](conversation-driven-development.md) 来了解用户采用的非预期的路径，通过创建故事来处理上下文切换。

### 使用规则进行上下文切换 {#using-rules-for-context-switching}

在某些情况下，例如单轮插入语，你可以使用[规则](rules.md)而非[故事](stories.md)来处理上下文切换。

考虑如下对话场景：

<figure markdown>
  ![](images/writing-conversation-data/single-turn-interjection.png){ width="600" }
  <figcaption>单轮插入语</figcaption>
</figure>

在这个示例中，用户在支付信用卡账单的过程中询问账户余额，之后被引导回信用卡支付表单。因为无论上下文如何询问用户余额都应该始终得到相同的响应，因此你可以在现有流程中创建一个可以自动触发的规则：

```yaml
rules:
- rule: Check my account balance
  steps:
  - intent: check_account_balance
  - action: action_get_account_balance
```

默认情况下，表单将持续保持活动状态并重新提示必要的信息，而无需创建额外的训练故事。

### 使用故事进行上下文切换 {#using-stories-for-context-switching}

当用户的插入语需要多轮对话时，你需要编写额外的故事来处理上下文切换。如果你有两个不同对话流并希望用户能在流之间切换，则需要创建故事来指定切换将如何发生以及如何维护上下文。

例如，如果你想在用户询问时切换上下文，然后在询问完成后返回原始流程：

<figure markdown>
  ![](images/writing-conversation-data/contextual-interjection.png){ width="600" }
  <figcaption>上下文插入语</figcaption>
</figure>

你需要创建一个故事来描述这种上下文切换交互：

```yaml
stories:
- story: Context switch from credit card payment to money transfer
  steps:
  - intent: pay_credit_card
  - action: credit_card_payment_form
  - active_loop: credit_card_payment_form
  - intent: transfer_money                         # - user requests a money transfer
  - active_loop: null                              # - deactivate the credit card form
  - action: transfer_money_form                    # - switch to the money transfer form
  - active_loop: transfer_money_form
  - active_loop: null
  - action: utter_continue_credit_card_payment     # - once the money transfer is completed,
                                                   #   ask the user to return to the
                                                   #   credit card payment form
```

## 管理对话数据文件 {#managing-conversation-data-files}

你可以将训练数据作为单个文件或包含多个文件的目录提供给开源 Rasa。在编写故事和规则时，通常最好根据所表示的对话类型创建单独的文件。

例如，你可以创建一个 `chitchat.yml` 文件来处理闲聊，创建一个 `faqs.yml` 来处理 FAQ。有关在一个复杂对话机器人中的故事文件管理，可以参见 [rasa-demo 机器人](https://github.com/RasaHQ/rasa-demo)。

## 使用交互式学习 {#using-interactive-learning}

通过与对话机器人交流并提供反馈，交互式学习可以轻松编写故事。这是探索对话机器人可以做什么的一个强大方法，也是修复其所犯错误最简单的方法。基于机器学习的对话的一个优点是，当你的机器人还不知道如何做某事时，你可以直接教它！

在开源 Rasa 中，你可以使用 [`rasa interactive`](command-line-interface.md#rasa-interactive) 在命令行中运行交互式学习。

### 命令行交互式学习 {#command-line-interactive-learning}

`rasa interactive` 命令将在命令行中启动交互式学习。如果对话机器人有自定义动作，请确保也要在单独的终端窗口中[运行你的动作服务](action-server/running-action-server.md)。

在交互模式下，你将会被要求在机器人继续之前确认每个意图和动作的预测，如下为一个示例：

```
? Next user input:  hello

? Is the NLU classification for 'hello' with intent 'hello' correct?  Yes

------
Chat History

 #    Bot                        You
────────────────────────────────────────────
 1    action_listen
────────────────────────────────────────────
 2                                    hello
                         intent: hello 1.00
------

? The bot wants to run 'utter_greet', correct?  (Y/n)
```

你可以在对话的每个步骤中查看对话历史记录和槽值。

如果输入 ++y++ 来确定预测，机器人将继续，如果输入 ++n++，你将有机会在继续之前更正预测：

```
? What is the next action of the bot?  (Use arrow keys)
 » <create new action>
   1.00 utter_greet
   0.00 ...
   0.00 action_back
   0.00 action_deactivate_loop
   0.00 action_default_ask_affirmation
   0.00 action_default_ask_rephrase
   0.00 action_default_fallback
   0.00 action_listen
   0.00 action_restart
   0.00 action_session_start
   0.00 action_two_stage_fallback
   0.00 utter_cheer_up
   0.00 utter_did_that_help
   0.00 utter_goodbye
   0.00 utter_happy
   0.00 utter_iamabot
```

在任何时候都可以使用 ++ctrl+c++ 访问菜单，这允许你创建更多故事并从故事中导出数据。

```
? Do you want to stop?  (Use arrow keys)
 » Continue
   Undo Last
   Fork
   Start Fresh
   Export & Quit
```
