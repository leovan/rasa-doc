# 个人网站

如果你已经有一个网站并想为其添加 Rasa 对话机器人，可以使用 [Rasa Chat Widget](your-own-website.md#chat-widget) 小组件，通过添加 HTML 片段将其合并到现有的网页中。

或者，你也可以构建自己的聊天小组件。

## REST 频道 {#rest-channels}

`RestInput` 和 `CallbackInput` 频道可用于自定义集成。它们提供了一个 URL，你可以在其中发送消息并直接接收响应消息，或者通过 webhook 异步接收。

### `RestInput` {#restinput}

REST 频道将提供一个 REST endpoint，你可以在其中发送用户消息并接收对话机器人的响应消息。

将 REST 频道添加到 `credentials.yml` 中：

```yaml
rest:
  # you don't need to provide anything here - this channel doesn't
  # require any credentials
```

重启 Rasa 服务器，以使 REST 频道可用于接收消息。然后，你可以将消息发送到 `http://<host>:<port>/webhooks/rest/webhook`，用你正在运行的 Rasa 服务器中适当值替换主机和端口。

#### 请求和响应各式 {#request-and-response-format}

`rest` 输入频道可用后，你可以 `POST` 消息到 `http://<host>:<port>/webhooks/rest/webhook`，格式如下：

```json
{
  "sender": "test_user",  // sender ID of the user sending the message
  "message": "Hi there!"
}
```

Rasa 的响应是对话机器人响应的 JSON 体，例如：

```json
[
  {"text": "Hey Rasa!"}, {"image": "http://example.com/image.jpg"}
]
```

### `CallbackInput` {#callbackinput}

Callback 频道的行为与 REST 频道非常相似，但它不是直接将机器人消息返回给发送消息的 HTTP 请求，而是调用可以指定的 URL 来发送机器人消息。

要使用 Callback 频道，请将凭据添加到 `credentials.yml` 中：

```yaml
callback:
  # URL to which Core will send the bot responses
  url: "http://localhost:5034/bot"
```

重启 Rasa 服务器，以使 Callback 频道可用于接收消息。然后，你可以将消息发送到 `http://<host>:<port>/webhooks/rest/webhook`，用你正在运行的 Rasa 服务器中适当值替换主机和端口。

#### 请求和响应各式 {#request-and-response-format-1}

`callback` 输入频道可用后，你可以 `POST` 消息到 `http://<host>:<port>/webhooks/rest/webhook`，格式如下：

```json
{
  "sender": "test_user",  // sender ID of the user sending the message
  "message": "Hi there!"
}
```

如果成功，则响应将为 `success`。一旦 Rasa 准备好向用户发送消息，它将使用包含对话机器人响应的 JSON 体的 `POST` 请求调用 `credentials.yml` 中指定的 `url`：

```json
[
  {"text": "Hey Rasa!"}, {"image": "http://example.com/image.jpg"}
]
```

## Websocket 频道 {#websocket-channel}

SocketIO 频道使用 websockets 并且是实时的。要使用 SocketIO 频道，请将凭据添加到 `credentials.yml` 中：

```yaml
socketio:
  user_message_evt: user_uttered
  bot_message_evt: bot_uttered
  session_persistence: true/false
```

前两个配置值定义了 Rasa 在通过 `socket.io` 发送或接受消息时使用的事件名称。

重启 Rasa 服务器，以使 SocketIO 频道可用于接收消息。然后，你可以将消息发送到 `http://<host>:<port>/socket.io`，用你正在运行的 Rasa 服务器中适当值替换主机和端口。

!!! info "会话持久性"

    默认情况下，SocketIO 频道使用套接字 ID 作为 `sender_id`，这会导致会话在每次页面重新加载时重新启动。`session_persistence` 设置为 `true` 可以避免这种情况。在这种情况下，前端负责生成会话 ID 并将其发送到 Rasa Core 服务器，方法是在连接事件之后立即发出带有 `{session_id: [session_id]}` 事件的 `session_request`。

    示例 [Webchat](https://github.com/botfront/rasa-webchat) 实现了这种会话创建机制（版本 >= 0.5.0）。

!!! info "ScoketIO 客户端 / 服务器兼容性"

    连接 Rasa 的 SocketIO 客户端版本必须与 Rasa 使用的 [`python-socketio`](https://github.com/miguelgrinberg/python-socketio) 或 [`python-engineio`](https://github.com/miguelgrinberg/python-engineio) 包的版本兼容。请参考你的 Rasa 版本相关的 [`pyproject.toml`](https://github.com/RasaHQ/rasa/blob/main/pyproject.toml) 文件和官方的 `python-socketio` 兼容性表。

### JWT 身份验证 {#jwt-authentication}

通过在 `credentials.yml` 文件中定义 `jwt_key` 和可选的 `jwt_method`，可以选择将 SocketIO 频道配置为在连接时执行 JWT 身份验证。

```yaml
socketio:
  user_message_evt: user_uttered
  bot_message_evt: bot_uttered
  session_persistence: true
  jwt_key: my_public_key
  jwt_method: HS256
```

最初请求连接时，客户端应将编码的有效载荷作为密钥令牌下的 JSON 对象传递：

```json
{
  "token": "jwt_encoded_payload"
}
```

### Chat 小组件 {#chat-widget}

一旦设置了 SocketIO 频道，你就可以在任何网页上使用官方的 Rasa Chat 小组件。只需要将如下内容粘贴到你站点的 HTML 中，并将 Rasa 实例的 URL 粘贴到 `data-websocket-url` 属性中：

```html
<div id="rasa-chat-widget" data-websocket-url="https://your-rasa-url-here/"></div>
<script src="https://unpkg.com/@rasahq/rasa-chat" type="application/javascript"></script>
```

有关更多信息，包括如何为网站完全自定义小组件，可以查看[完整文档](https://chat-widget-docs.rasa.com/)。

或者，如果你想将小组件嵌入到 React 应用中，可以使用 [NPM 包存储库中的一个库](https://www.npmjs.com/package/@rasahq/rasa-chat)。
