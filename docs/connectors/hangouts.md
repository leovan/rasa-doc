# Google Hangouts Chat

## Hangouts Chat 设置 {#hangouts-chat-setup}

此频道的工作方式类似于标准的 Rasa REST 频道。对于来自通道的每个请求，对话机器人将发送一个响应。响应将以文本或所谓的卡片的形式显示给用户（更多信息请参见卡片部分）。

为了将你的 Rasa 对话机器人连接到 Google Hangouts Chat，你必须首先在 Google Developer Console 中创建一个包含 Hangouts API 的项目。你可以在那里指定对话机器人的 endpoint。endpoint 类似于 `http://<host>:<port>/webhooks/hangouts/webhook`，将主机和端口替换为你正在运行的 Rasa 服务器的适当值。

!!! note "配置 HTTPS"

    Hangouts Chat 仅通过 https 将消息转到到 endpoint，因此请采取适当的措施将其添加到你的设置中。有关对话机器人的本地测试，请参见[在本地机器上测试频道](/messaging-and-voice-channels#testing-channels-on-your-local-machine)。

在 Google Developer Console 中，获取你的项目 ID（也称为项目编号或应用 ID），其决定了 OAuth2 授权的范围，以防要使用 OAuth2。Hangouts Chat API 会随着每个请求发送一个 Bearer 令牌，但实际验证取决于对话机器人，因此该频道在没有此令牌的情况下也能正常工作。更多信息请参见 [Google Hangouts 文档](https://developers.google.com/hangouts/chat)。如果你希望完成验证，请务必在如下所示的 `credentials.yml` 文件中包含 `project_id`。

Hangouts Chat 和对话机器人之间实现异步通信是可能的，但由于 Rasa 对话机器人通常是同步的，因此该功能不包含在此频道中。

## 在 Hangouts Chat 上运行 {#running-on-hangouts-chat}

将 Hangouts 凭据添加到 `credentials.yml` 文件中：

```yaml
hangouts:
  # no credentials required here
```

如果要使用 OAuth2，请从 Google Developer Console 获取项目 ID：

```yaml
hangouts:
  project_id: "12345678901"
```

重启你的 Rasa 服务器，使新的频道 Endpoint 可供 Google Hangouts 发送消息。

### 卡片和互动卡片 {#cards-and-interactive-cards}

Hangouts Chat 显示对话机器人消息的方式有两种，一种是文本，另一种是卡片。对于每个收到的请求，对话机器人将在一个响应中发送所有消息。如果其中一条消息是卡片（例如图像），则所有其他消息也会转换为卡片格式。

交互式卡片触发 `CARD_CLICKED` 事件来进行用户交互，例如：单击按钮时。创建交互式卡片时，例如：通过 `actions.py` 中的 `dispatcher.utter_button_message()`，你可以为每个按钮指定一个有效载荷，该按钮与 `CARD_CLICKED` 事件一起返回并由 `HangoutsInput` 频道提取（例如：`buttons=[{"text":"Yes!", "payload":"/affirm"}, {"text":"Nope.", "payload":"/deny"}]`）。目前尚不支持更新卡片。

有关卡片的更多信息请参见 [Hangouts 文档](https://developers.google.com/hangouts/chat/reference)。

### 其他 Hangouts Chat 事件 {#other-hangouts-chat-events}

除了 `MESSAGE` 和 `CARD_CLICKED` 之外，Hangouts Chat 还有另外两种事件类型，`ADDED_TO_SPACE` 和 `REMOVED_FROM_SPACE`。它们会在对话机器人从消息直接或聊天室控件中被添加或删除时触发。可以在 `HangoutsInput` 构造方法中修改这些事件的默认意图名称。
