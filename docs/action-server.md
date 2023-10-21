# Rasa 动作服务器简介

Rasa 动作服务器为开源 Rasa 对话机器人运行[自定义动作](custom-actions.md)。

## 如何运行 {#how-it-works}

当你的对话机器人预测自定义动作时，Rasa 服务器会向动作服务器发送一个 `POST` 请求，包含一个 json 有效负载，其中包括预测动作的名称、对话 ID、追踪器的内容和领域的内容。

当动作服务器完成运行自定义动作时，它会返回[响应](responses.md)和[事件](action-server/events.md)的 json 有效负载。有关请求和响应负载的详细信息，请参阅 [API 规范](https://rasa.com/docs/rasa/pages/action-server-api.md){:target="_blank"}。

Rasa 服务器然后将响应返回给用户并将事件添加到对话追踪器。

## 用于自定义动作的 SDK {#sdks-for-custom-actions}

可以使用以任何语言编写的动作服务器来运行自定义动作，只要它实现[所需的 API](https://rasa.com/docs/rasa/pages/action-server-api){:target="_blank"}。

### Rasa SDK（Python） {#rasa-sdk-python}

Rasa SDK 是一个用于运行自定义动作的 Python SDK。除了实现所需的 API 之外，它还提供了与对话追踪器交互以及编写事件和响应的方法。如果你还没有动作服务器并且不需要它使用 Python 以外的语言，那么使用 Rasa SDK 将是最简单的入门方法。

### 其他动作服务器 {#other-action-servers}

如果你有其他语言的遗留代码或现有业务逻辑，你可能不想使用 Rasa SDK。在这种情况下，你可以使用任何你想要的语言编写自己的动作服务器。动作服务器的唯一要求是它提供一个 `/webhook` Endpoint，该 Endpoint 接受来自 Rasa 服务器的 HTTP POST 请求并返回[事件](action-server/events.md)和响应的有效负载。有关所需 `/webhook` Endpoint 的详细信息，请参阅 [API 规范](https://rasa.com/docs/rasa/pages/action-server-api){:target="_blank"}。
