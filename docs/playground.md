# Rasa Playground

通过此交互式指南可以了解如何利用开源 Rasa 构建一个对话机器人的基础知识。你可以自定义对话机器人，与其对话并下载整个工程，从而继续进一步构建。

## 构建你的对话机器人 {#build-your-assistant}

在本指南中，我们将创建一个对话机器人来帮助用户订阅时事通讯。通过如下每个步骤来了解如何构建一个简答的对话机器人。

=== "NLU 数据"

    在订阅时事通讯时，人们通常会和对话机器人说些什么呢？

    无论用户如何表达他们的信息，为了让对话机器人能够识别用户在说什么，我们需要提供示例信息用于学习。根据信息所表达的想法或目的（也称为意图）对这些示例进行分组。例如代码块中，我们添加了一个名为 `greet` 的意图，其中包含诸如“Hi”，“Hey”和“good morning”之类的示例消息。

    意图和示例可以用作对话机器人的自然语言理解（NLU）模型训练的数据。

    [了解有关 NLU 数据及其格式的更多信息](/training-data-format/)

    ```yaml
    nlu:
    - intent: greet
      examples: |
        - Hi
        - Hey!
        - Hallo
        - Good day
        - Good morning

    - intent: subscribe
      examples: |
        - I want to get the newsletter
        - Can you send me the newsletter?
        - Can you sign me up for the newsletter?

    - intent: inform
      examples: |
        - My email is example@example.com
        - random@example.com
        - Please send it to anything@example.com
        - Email is something@example.com
    ```

=== "响应"

    既然对话机器人已经理解了用户的一些信息，它需要一些可以返回给用户的响应。

    “Hello, how can I help you?”和“what's your email address?”是我们的对话机器人将使用的一些响应。你将在如下的步骤中了解如何连接用户的消息和响应。

    在代码块中，我们列出了一些响应，并为每个响应添加了一个或多个文本选项。如果一个响应有多个文本选项，则预测该响应时会随机选择其中一个。

    [了解有关响应的更多信息](/responses/)

    ```yaml
    responses:
      utter_greet:
        - text: |
            Hello! How can I help you?
        - text: |
            Hi!
      utter_ask_email:
        - text: |
            What is your email address?
      utter_subscribed:
        - text: |
            Check your inbox at {email} in order to finish subscribing to the newsletter!
        - text: |
            You're all set! Check your inbox at {email} to confirm your subscription.
    ```

=== "故事"

    [故事](/stories/)是用于训练对话机器人根据用户之前的对话内容做出正确响应的示例对话。故事的格式展示了用户消息的意图，然后是对话机器人的动作和响应。

    你的第一个故事应该为一个对话流，其中对话机器人以简单直接的方式来帮助用户实现他们的目标。之后，可以为用户不想提供信息或切换到其他主题的情况添加故事。

    在代码块中，我们添加了一个故事，用户和对话机器人相互问候，用户请求订阅时事通讯，对话机器人开始收集表单所需要的信息。你可以在下一步中了解表单。

    [了解有关故事的更多信息](/writing-stories/)

    ```yaml
    stories:
      - story: greet and subscribe
        steps:
        - intent: greet
        - action: utter_greet
        - intent: subscribe
        - action: newsletter_form
        - active_loop: newsletter_form
    ```

=== "表单"

    在很多情况下，对话机器人需要从用户收集信息。例如：当一个用户想要订阅时事通讯时，对话机器人必须询问其电子邮件地址。

    在 Rasa 中可以使用表单实现。在代码块中，我们添加了 `newsletter_form` 并使用它来收集用户的电子邮件地址。

    [了解有关表单的更多信息](/forms/)

    ```yaml
    slots:
      email:
        type: text
        mappings:
        - type: from_text
          conditions:
          - active_loop: newsletter_form
            requested_slot: email
    forms:
      newsletter_form:
        required_slots:
        - email
    ```

=== "规则"

    规则描述了对话中应该始终遵循的相同路径，如论在之前的对话中说过什么。

    我们希望对话机器人能够始终以特定动作来响应一个特定意图，因此我们需要使用规则将动作映射到意图。

    在代码块中，我们添加了一条规则，该规则在用户表达“订阅”意图时触发 `newsletter_form`。我们还添加了一条规则用于一旦提供了所有必需的信息就触发 `utter_subscribed` 动作。第二条规则仅在 `newsletter_form` 开始激活时适用，一单它不再处于激活状态（`active_loop: null`）后，表单就完成了。

    [了解有关规则和如何写规则的更多信息](/rules/)

    现在你已经完成了所有的步骤，向下滚动开始与你的对话机器人交谈吧。

    ```yaml
    rules:
      - rule: activate subscribe form
        steps:
        - intent: subscribe
        - action: newsletter_form
        - active_loop: newsletter_form

      - rule: submit form
        condition:
        - active_loop: newsletter_form
        steps:
        - action: newsletter_form
        - active_loop: null
        - action: utter_subscribed
    ```

## 训练并与对话机器人交谈 {#train-and-talk-to-your-assistant}

上述步骤完成后，你就可以开始训练对话机器人了。训练过程会根据提供的训练数据生成一个新的机器学习模型。

单击 `Train` 按钮开始基于上述 NUL 数据、故事、表单、规则和响应训练对话机器人。

!!! info "提示"

    此处为交互操作界面，具体内容详见[原始文档](/playground/#train-and-talk-to-your-assistant)。

## 寻求挑战？自定义对话机器人 {#looking-for-a-challenge-customize-your-assistant}

你可以使用本页面创建一个对话机器人来帮助用户订阅时事通讯。

尝试为对话机器人选择一项简单的任务，例如：点披萨或进行预约。在每一步中调整相关代码来适应新场景，然后再次训练对话机器人来查看实际效果。

## 已经创建了对话机器人！下一步？ {#you-have-built-your-assistant-whats-next}

你可以下载此项目并继续构建一个进阶的对话机器人。

!!! info "提示"

    此处为交互操作界面，具体内容详见[原始文档](/playground/#you-have-built-your-assistant-whats-next)。

[安装 Rasa 来继续构建](/installation/installing-rasa-open-source/)

在训练一个模型时，你会想检查你的对话机器人是否能够按照你的预期行事。可以通过与对话机器人交谈来观测其是否正常工作。但是，随着对话机器人变得越来越复杂，你需要使用测试故事来保证模型做出正确的预测。

尝试运行 `rasa test` 来确保对话机器人通过测试：

```yaml
stories:
- story: test for greet and subscribe
  steps:
  - user: |
     Hello there
    intent: greet
  - action: utter_greet
  - user: |
     I want to subscribe to the newsletter. My email is example@email.com
    intent: subscribe
  - action:  utter_subscribed
```

查看其他文档页面来了解有关 [Rasa CLI](/command-line-interface/)，[领域](/domain/)，[动作](/actions/)，以及配置中的[管道](/tuning-your-model/)和[策略](/policies/)。
