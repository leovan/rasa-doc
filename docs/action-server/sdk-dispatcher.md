# 分发器

分发器是 `CollectingDispatcher` 类的一个实例，用于生成响应来发送回用户。

## CollectingDispatcher {#collectingdispatcher}

`CollectingDispatcher` 有一个 `utter_message` 方法和一个 `messages` 属性。它用于动作的 `run` 方法中，用来向返回到 Rasa 服务器的有效负载添加响应。Rasa 服务器依次将 `BotUttered` 事件添加到每个响应的追踪器。因此，使用调度程序添加的响应不应作为[事件](/action-server/events)显式返回。例如，如下自定义动作不显式返回任何事件，但将返回响应“Hi, User!”给用户：

```python
class ActionGreetUser(Action):
    def name(self) -> Text:
        return "action_greet_user"

    async def run(
        self,
        dispatcher: CollectingDispatcher,
        tracker: Tracker,
        domain: Dict[Text, Any],
    ) -> List[EventType]:

        dispatcher.utter_message(text = "Hi, User!")

        return []
```

### `CollectingDispatcher.utter_message` {#collectingdispatcherutter_message}

`utter_message` 方法可用于向用户返回任意类型的响应。

参数：

`utter_message` 方法采用如下可选参数。不传递任何参数将导致向用户返回一条空消息。传递多个参数将导致向用户返回丰富的响应（例如文本和按钮）。

- `text`：返回给用户的文本。

    ```python
    dispatcher.utter_message(text = "Hey there")
    ```

- `image`：用于向用户显示图像的 URL 或文件路径。

    ```python
    dispatcher.utter_message(image = "https://i.imgur.com/nGF1K8f.jpg")
    ```

- `json_message`：字典格式的自定义 JSON 有效负载。它可用于发送[特定于频道的响应](/responses)。如下示例将在 Slack 中返回日期选择器：

    ```python
    date_picker = {
      "blocks":[
        {
          "type": "section",
          "text":{
            "text": "Make a bet on when the world will end:",
            "type": "mrkdwn"
          },
          "accessory":
          {
            "type": "datepicker",
            "initial_date": "2019-05-21",
            "placeholder":
            {
              "type": "plain_text",
              "text": "Select a date"
            }
          }
        }
      ]
    }
    dispatcher.utter_message(json_message = date_picker)
    ```

- `response`：返回给用户的响应的名称。此响应应在对话机器人领域中指定。

    ```python
    dispatcher.utter_message(response = "utter_greet")
    ```

- `attachment`：要返回给用户的附件的 URL 或文件路径。

    ```python
    dispatcher.utter_message(attachment = "")
    ```

- `buttons`：返回给用户的按钮列表。每个按钮都是一个字典，应该有一个 `text` 和一个 `payload` 键。一个按钮可以包含其他键，但只有在特定频道查找它们时才会使用这些键。如果用户单击按钮，按钮的 `payload` 将作为用户消息发送。

    ```python
    dispatcher.utter_message(buttons = [
                    {"payload": "/affirm", "title": "Yes"},
                    {"payload": "/deny", "title": "No"},
                ])
    ```

- `elements`：特定于使用 Facebook 作为消息传递频道。有关预期格式的详细信息，请参阅 [Facebook 的文档](https://developers.facebook.com/docs/messenger-platform/send-messages/template/generic/)。
- `**kwargs`：任意关键字参数，可用于指定[响应变化中变量](/responses)的值。例如。给定如下响应：

    ```yaml
    responses:
      utter_greet_name:
      - text: Hi {name}!
    ```

    你可以使用如下命令指定名称：

    ```python
    dispatcher.utter_message(response = "utter_greet_name", name = "Aimee")
    ```

返回类型：

`None`
