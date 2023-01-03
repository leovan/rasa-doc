# 触达用户

有时候你希望对话机器人在没有用户消息的情况下触达用户。例如：你可能希望对话机器人在用户打开聊天窗口时就发送消息，或者你可能希望在用户有一段时间没有发送消息时提示用户。本页面是帮助对话机器人能够主动与用户联系的指南。

## 率先触达 {#reaching-out-first}

在大多数用例中，当用户打开同对话机器人的聊天窗口时，你会希望对话机器人发送第一条消息。这样可以让用户了解机器人能做什么或不能做什么，并让他们可以有一个更成功的对话。某些[消息或语音频道](/messaging-and-voice-channels/)具有现成的配置选项，可以在用户首次开始对话时向对话机器人发送有效负载，但你也可以将此选项添加到你自己的[自定义频道](/connectors/your-own-website/)中。

配置为频道发送有效负载后，你需要指定对话机器人应该如何回应和问候用户。你可以重复使用现有意图行为，也可以为此指定新的意图和规则。以下是有关如何指定一个欢迎规则的指南。

### 更新配置 {#update-the-configuration}

由于对该行为使用规则，因此需要将 [RulePolicy](/policies/#rule-policy) 添加到配置文件中：

```yaml title='config.yml'
policies:
  # other policies
  - name: RulePolicy
```

### 添加规则 {#add-a-rule}

要让对话机器人在对话开始时使用欢迎消息响应 `greet` 意图，需要添加如下规则：

```yaml title='rules.yml'
rules:
  - rule: welcome user
    conversation_start: true  # this rule only applies at the beginning of a conversation
    steps:
      - intent: greet
      - action: utter_welcome
```

### 添加响应 {#add-a-response}

最后，在领域中添加用于 `utter_welcome` 动作的响应：

```yaml title='domain.yml'
responses:
  utter_welcome:
  - text: Hi there! What can I help you with today?
```

## 外部事件 {#external-events}

有时候，你希望使用外部设备来改变正在进行的对话过程。例如，如果你在树莓派上连接了一个湿度传感器，你可以使用它来通知你的对话机器人在植物需要浇水时通知你。

以下示例来自[提醒对话机器人](https://github.com/RasaHQ/rasa/blob/main/examples/reminderbot)，其中包括提醒和外部事件。

### 触发意图 {#trigger-an-intent}

要让来自外部设备的事件改变正在进行的对话过程，你可以让设备 POST 请求对话的 [`trigger_intent` 接口](/pages/http-api/#operation/triggerConversationIntent)。`trigger_intent` 接口将用户意图（可能带有实体）注入到你的对话中。对于 Rasa，就好像你输入了一条按特定意图和实体分类的消息。然后，对话机器人像往常一样预测并执行下一个动作。

例如，如下 POST 请求会将 `EXTERNAL_dry_plant` 意图和 `plant` 实体注入到同 ID 为 `user123` 的对话中：

```shell
curl -H "Content-Type: application/json" -X POST \
  -d '{"name": "EXTERNAL_dry_plant", "entities": {"plant": "Orchid"}}' \
  "http://localhost:5005/conversations/user123/trigger_intent?output_channel=latest"
```

### 获取对话 ID {#get-the-conversation-id}

在现实生活中，你的外部设备会从 API 或数据库汇中获取对话 ID。在植物提醒示例中，你可以有一个植物数据库、给植物浇水的用户以及用户的对话 ID。树莓派将直接从数据库中获取对话 ID。要在本地测试提醒对话机器人示例，你需要手动获取对话 ID。更多有关信息请参见提醒对话机器人的 [README](https://github.com/RasaHQ/rasa/blob/main/examples/reminderbot) 文件。

### 添加 NLU 训练数据 {#add-nlu-training-data}

在植物提醒对话机器人示例中，树莓派将带有 `EXTERNAL_dry_plant` 意图的消息发送到 `trigger_intent` 接口。这个意图将保留给树莓派使用，因此不会有任何 NLU 训练样本。

```yaml title='domain.yml'
intents:
  - EXTERNAL_dry_plant
```

!!! note "注意"

    你应该使用 `EXTERNAL_` 前缀来命名来自其他设备的意图，因为这样在处理训练数据时可以更加轻松地查看那些意图来自外部设备。

### 更新领域 {#update-the-domain}

要告诉对话机器人哪种植物需要浇水，你可以定义一个实体，将其与意图一并附带在请求中。为了能够直接在响应中使用实体值，可以为 `plant` 槽定义一个 `from_entity` 槽映射：

```yaml title='domain.yml'
entities:
  - plant

slots:
  plant:
    type: text
    influence_conversation: false
    mappings:
    - type: from_entity
      entity: plant
```

### 添加规则 {#add-a-rule}

你需要一个规则来告诉你的对话机器人在收到来自树莓派的消息时如何响应。

```yaml title='rules.yml'
rules:
  - rule: warn about dry plant
    steps:
    - intent: EXTERNAL_dry_plant
    - action: utter_warn_dry
```

### 添加响应 {#add-a-response}

你需要为 `utter_warn_dry` 定义响应文本：

```yaml title='domain.yml'
responses:
  utter_warn_dry:
  - text: "Your {plant} needs some water!"
```

响应将使用来自 `plant` 槽的值来警告需要浇水的特定植物。

### 试用 {#try-it-out}

要试用植物提醒对话机器人，你需要启动一个 [CallbackChannel](/connectors/your-own-website/#callbackinput)。

!!! caution "注意"

    外部事件和提醒在例如 `rest` 频道或 `rasa shell` 的 request-response 频道中不起作用。实现提醒或外部事件的对话机器人的自定义连接器应该建立在 CallbackInput 频道而非 RestInput 频道之上。

    更多有关如何在本地测试的说明请参见[提醒对话机器人的 README](https://github.com/RasaHQ/rasa/blob/main/examples/reminderbot/README.md) 文件。

使用你的对话 ID 运行如下 POST 请求来模拟外部事件：

```shell
curl -H "Content-Type: application/json" -X POST -d \
'{"name": "EXTERNAL_dry_plant", "entities": {"plant": "Orchid"}}' \
"http://localhost:5005/conversations/user1234/trigger_intent?output_channel=latest"
```

你将会在你的频道中看到对话机器人的响应：

<figure markdown>
  ![](/images/reaching-out-to-user/reminder.png){ width="600" }
  <figcaption>提醒</figcaption>
</figure>

## 提醒 {#reminders}

你可以使用[提醒](/action-server/events/#ReminderScheduled)让对话机器人在设定的时间后与用户联系。如下示例来自[提醒示例对话机器人](https://github.com/RasaHQ/rasa/blob/main/examples/reminderbot)。你可以克隆并按照 `README` 中的说明试用完整版。

### 设置提醒 {#scheduling-reminders}

#### 定义提醒 {#define-a-reminder}

要设置一个提醒，你需要定义一个返回 ReminderScheduled 事件的自定义动作。例如，如下自定义动作会在 5 分钟后进行提醒：

```python title='actions.py'
import datetime
from rasa_sdk.events import ReminderScheduled
from rasa_sdk import Action

class ActionSetReminder(Action):
    """Schedules a reminder, supplied with the last message's entities."""

    def name(self) -> Text:
        return "action_set_reminder"

    async def run(
        self,
        dispatcher: CollectingDispatcher,
        tracker: Tracker,
        domain: Dict[Text, Any],
    ) -> List[Dict[Text, Any]]:

        dispatcher.utter_message("I will remind you in 5 minutes.")

        date = datetime.datetime.now() + datetime.timedelta(minutes=5)
        entities = tracker.latest_message.get("entities")

        reminder = ReminderScheduled(
            "EXTERNAL_reminder",
            trigger_date_time=date,
            entities=entities,
            name="my_reminder",
            kill_on_user_message=False,
        )

        return [reminder]
```

`ReminderScheduled` 事件的第一个参数是提醒的名称，在本例中为 `EXTERNAL_reminder`。提醒的名称稍后将用作触发提醒反馈的一个意图。使用 `EXTERNAL_` 前缀命名提醒名称，以便更方便地查看训练数据中发生的事情。

你可以看到最后一条消息的 `entities` 也传递给了提醒，这允许用于对提醒做出反馈的动作使用来自用户设置信息中的实体。

例如，如果你想让对话机器人提醒你给朋友打电话，你可以发送一条消息，比如“Remind me to call Paul”。如果“Paul”被提取为 `PERSON` 实体，则对提醒做出的反馈动作可以让它说“Remember to call Paul!”。

#### 添加规则 {#add-a-rule-1}

要设置一个提醒，你需要添加一个规则：

```yaml title='rules.yml'
rules:
- rule: Schedule a reminder
  steps:
  - intent: ask_remind_call
    entities:
    - PERSON
  - action: action_set_reminder
```

#### 添加训练数据 {#add-training-data}

你需要为设置提醒添加 NLU 训练样本：

```yaml title='nlu.yml'
nlu:
- intent: ask_remind_call
  examples: |
    - remind me to call John
    - later I have to call Alan
    - Please, remind me to call Vova
    - please remind me to call Tanja
    - I must not forget to call Juste
```

同时需要将其添加到领域中：

```yaml title='domain.yml'
intents:
  - ask_remind_call
```

#### 更新管道 {#add-training-data}

在 `config.yml` 文件中将 SpacyNLP 和 SpacyEntityExtractor 添加到管道中，同时不需要在训练数据中进行标注，因为 Spacy 具有 `PERSON` 实体类型：

```yaml title='config.yml'
pipeline:
# other components
- name: SpacyNLP
  model: "en_core_web_md"
- name: SpacyEntityExtractor
  dimensions: ["PERSON"]
```

### 提醒反馈 {#reacting-to-reminders}

#### 定义反馈 {#define-a-reaction}

在 `trigger_intent` 接口收到 POST 请求后，对话机器人则会触达用户。但是，提醒会在一定时间后使用 `ReminderScheduled` 事件中定义的名称自动将请求发送到正确的对话 ID 上。

要定义提醒的反馈，你需要编写一个[规则](/rules/)，告诉对话机器人在收到提醒意图时采取什么行动。

在呼叫提醒示例中，你希望使用提醒附带的实体来提醒你呼叫特定的人，因此你需要编写一个自定义动作来执行此操作：

```python title='actions.py'
class ActionReactToReminder(Action):
    """Reminds the user to call someone."""

    def name(self) -> Text:
        return "action_react_to_reminder"

    async def run(
        self,
        dispatcher: CollectingDispatcher,
        tracker: Tracker,
        domain: Dict[Text, Any],
    ) -> List[Dict[Text, Any]]:

        name = next(tracker.get_slot("PERSON"), "someone")
        dispatcher.utter_message(f"Remember to call {name}!")

        return []
```

#### 添加规则 {#add-a-rule-2}

要告诉对话机器人在触发提醒时要运行什么动作，需要添加一条规则。

```yaml title='rules.yml'
rules:
- rule: Trigger `action_react_to_reminder` for `EXTERNAL_reminder`
  steps:
  - intent: EXTERNAL_reminder
  - action: action_react_to_reminder
```

#### 添加训练数据 {#add-training-data-1}

你需要定义触发对提醒做出反馈的意图。你不需要添加任何训练数据，因为该意图是为提醒保留的。

```yaml title='domain.yml'
intents:
- intent: EXTERNAL_reminder
```

### 取消提醒 {#cancelling-reminders}

#### 定义取消提醒的动作 {#define-an-action-that-cancels-a-reminder}

要取消已经设置的提醒，你需要一个返回 `ReminderCancelled()` 事件的自定义动作。

返回 `ReminderCancelled()` 会取消当前设置的所有提醒。如果你只想取消某些提醒，你可以指定一些参数来缩小提醒的范围：

- `ReminderCancelled(intent="EXTERNAL_greet")` 会取消所有带有 `EXTERNAL_greet` 意图的提醒
- `ReminderCancelled(entities={})` 会取消给定实体的所有提醒
- `ReminderCancelled("...")` 会取消在创建过程中提供的具有给定名称 `...` 的唯一提醒

对于呼叫提醒示例，你可以定义取消所有提醒的自定义动作 `action_forget_reminders`：

```python title='actions.py'
class ForgetReminders(Action):
    """Cancels all reminders."""

    def name(self) -> Text:
        return "action_forget_reminders"

    async def run(
        self, dispatcher, tracker: Tracker, domain: Dict[Text, Any]
    ) -> List[Dict[Text, Any]]:

        dispatcher.utter_message(f"Okay, I'll cancel all your reminders.")

        # Cancel all reminders
        return [ReminderCancelled()]
```

!!! caution "注意"

    当关闭 Rasa 服务时，所有提醒都会取消。

#### 添加规则 {#add-a-rule-3}

你需要为取消提醒添加一个规则：

```yaml title='rules.yml'
rules:
- rule: Cancel a reminder
  steps:
  - intent: ask_forget_reminders
  - action: action_forget_reminders
```

#### 添加训练数据 {#add-training-data-2}

你需要定一个意图来触发取消提醒：

```yaml title='nlu.yml'
nlu:
- intent: ask_forget_reminders
  examples: |
    - Forget about the reminder
    - do not remind me
    - cancel the reminder
    - cancel all reminders please
```

并将其添加到 `domain.yml` 中：

```yaml title='domain.yml'
intents:
- intent: ask_forget_reminders
```

### 测试 {#try-it-out-1}

要测试提醒，你需要启动一个 [CallbackChannel](/connectors/your-own-website/#callbackinput)。你还需要启动动作服务来设置、反馈和取消提醒。更多信息请参见提醒对话机器人的 [README](https://github.com/RasaHQ/rasa/blob/main/examples/reminderbot) 文件。

然后，如果你向机器人发送类似 `Remind me to call Paul Pots` 的消息。你将在 5 分钟后收到一条提醒 `Remember to call Paul Pots!`。
