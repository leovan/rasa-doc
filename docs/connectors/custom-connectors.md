# 自定义连接器

你可以通过 Python 类实现自定义频道连接器。可以使用 `rasa.core.channels.rest.RestInput` 类作为模板。

自定义连接器必须继承 `rasa.core.channels.channel.InputChannel` 并至少实现 `blueprint` 和 `name` 方法。

## `name` 方法 {#the-name-method}

`name` 方法定义连接器的 webhook 的 URL 前缀。它还定义了你应该在任何[频道特定的响应变体](../responses.md#channel-specific-response-variations)中使用的频道名称，以及你应该传递给[触发器意图 endpoint](https://www.rasa.com/docs/rasa/pages/http-api#operation/triggerConversationIntent) 上的 `output_channel` 查询参数的名称。

例如，如果自定义频道名为 `myio`，则 `name` 方法应定义为：

```python
from rasa.core.channels.channel import InputChannel

class MyIO(InputChannel):
    def name() -> Text:
        """Name of your custom channel."""
        return "myio"
```

你可以将特定于 `myio` 频道的响应变体写为：

```yaml title="domain.yml"
responses:
  utter_greet:
    - text: Hi! I'm the default greeting.
    - text: Hi! I'm the custom channel greeting
      channel: myio
```

提供给自定义频道来调用的 webhook 将为 `http://<host>:<port>/webhooks/myio/webhook`，将主机和端口替换为你正在运行的 Rasa 服务器的适当值。

## `blueprint` 方法 {#the-blueprint-method}

`blueprint` 方法需要创建一个可以附加 sanic 服务的 [sanic blueprint](https://sanicframework.org/en/guide/best-practices/blueprints.html#overview)。你的 `blueprint` 应该至少有两条路由，用于健康度检查的 `/` 和 `receive` 的 `/webhook`（参见如下自定义频道示例）。

作为 `receive` 端点的一部分，你需要告诉 Rasa 处理用用户消息。通过调用如下可以做到这一点：

```python
    on_new_message(
      rasa.core.channels.channel.UserMessage(
        text,
        output_channel,
        sender_id
      )
    )
```

调用 `on_new_message` 会将用户消息发送到 `handle_message` 方法。

更多详细信息请参见 [`UserMessage` 对象](https://www.rasa.com/docs/rasa/reference/rasa/core/channels/channel#usermessage-objects)。

`output_channel` 参数是指实现 [`OutputChannel`](https://www.rasa.com/docs/rasa/reference/rasa/core/channels/channel#outputchannel-objects) 类的输出频道。你可以使用特定频道的方法（例如发送文本和图像的方法）实现自己的输出频道类，也可以使用 [`CollectingOutputChannel`](https://www.rasa.com/docs/rasa/reference/rasa/core/channels/channel#collectingoutputchannel-objects) 收集 Rasa 在对话机器人处理消息时创建的对话机器人响应并将他们作为端点响应的一部分返回。这就是 `RestInput` 频道的实现方式。有关如何创建和使用自己的输出频道的示例，请查看其他输出频道的实现，例如 `rasa.core.channels.slack` 中的 `SlackBot`。

如下是一个使用 `CollectingOutputChannel` 的自定义频道连接器的简化示例：

```python title="custom_channel.py"
import asyncio
import inspect
from sanic import Sanic, Blueprint, response
from sanic.request import Request
from sanic.response import HTTPResponse
from typing import Text, Dict, Any, Optional, Callable, Awaitable, NoReturn

import rasa.utils.endpoints
from rasa.core.channels.channel import (
    InputChannel,
    CollectingOutputChannel,
    UserMessage,
)

class MyIO(InputChannel):
    def name() -> Text:
        """Name of your custom channel."""
        return "myio"

    def blueprint(
        self, on_new_message: Callable[[UserMessage], Awaitable[None]]
    ) -> Blueprint:

        custom_webhook = Blueprint(
            "custom_webhook_{}".format(type(self).__name__),
            inspect.getmodule(self).__name__,
        )

        @custom_webhook.route("/", methods=["GET"])
        async def health(request: Request) -> HTTPResponse:
            return response.json({"status": "ok"})

        @custom_webhook.route("/webhook", methods=["POST"])
        async def receive(request: Request) -> HTTPResponse:
            sender_id = request.json.get("sender") # method to get sender_id 
            text = request.json.get("text") # method to fetch text
            input_channel = self.name() # method to fetch input channel
            metadata = self.get_metadata(request) # method to get metadata

            collector = CollectingOutputChannel()

            # include exception handling

            await on_new_message(
                UserMessage(
                    text,
                    collector,
                    sender_id,
                    input_channel=input_channel,
                    metadata=metadata,
                )
            )

            return response.json(collector.messages)

        return custom_webhook
```

## 消息元数据 {#metadata-on-messages}

如果你需要在自定义操作中使用来自前端的额外信息，可以使用用户消息的 `metadata` 键传递此信息。如果适用，此信息将通过 Rasa 服务器随用户消息一起进入动作服务器，你可以在存储在 `tracker` 中找到。消息元数据不会直接影响 NLU 分类或动作预测。

`InputChannel` 类的 `get_metadata` 的默认实现忽略所有元数据。要在自定义连接器中提取元数据，请实现 `get_metadata` 方法。`SlackInput` 频道提供了一个 `get_metadata` 方法示例，该方法根据频道的响应格式提取元数据。

## 自定义频道的凭据 {#credentials-for-custom-channels}

要使用自定义频道，你需要在名为 `credentials.yml` 的凭据配置文件中为其提供凭据。此凭据文件必须包含自定义频道的模块路径（而不是频道名称）和任何必需的配置参数。

例如，对于保存在文件 `addons/custom_channel.py` 中名为 `MyIO` 的自定义连接器，模块路径将为 `addons.custom_channel.MyIO`，凭据可能如下所示：

```yaml title="credentials.yml"
addons.custom_channel.MyIO:
  username: "user_name"
  another_parameter: "some value"
```

要让 Rasa 服务器知道你的自定义频道，请在启动时使用命令行参数 `--credentials` 指定到 Rasa 服务器的 `credentials.yml` 的路径。

## 测试自定义连接器 Webhook {#testing-the-custom-connector-webhook}

要测试你的自定义连接器，可以使用具有如下格式的 JSON 正文将消息发布到 webhook：

```json
{
  "sender": "test_user",  // sender ID of the user sending the message
  "message": "Hi there!",
  "metadata": {}  // optional, any extra info you want to add for processing in NLU or custom actions
}
```

对于本地运行的 Rasa 服务器，CURL 请求如下所示：

```shell
curl --request POST \
     --url http://localhost:5005/webhooks/myio/webhook \
     --header 'Content-Type: application/json' \
     --data '{
            "sender": "test_user",
            "message": "Hi there!",
            "metadata": {}
          }'
```
