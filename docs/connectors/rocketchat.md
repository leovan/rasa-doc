# RocketChat

## 获取凭据 {#getting-credentials}

如何设置 Rocket.Chat：

1. 创建用于发送消息的用户，并在凭据文件中设置其凭据。
2. 以管理员身份登录 Rocket.Chat，转到 `Administration > Integrations > New Integration` 来创建 Rocket.Chat 的传出 webhook。
3. 选择 `Outgoing Webhook`。
4. 将 `Event Trigger` 部分设置为 `Message Sent`。
5. 填写详细信息，包括你希望对话机器人侦听的频道。或者可以使用 `@yourbotname` 设置 `Trigger Words` 部分，以便对话机器人不会在所有内容上触发。
6. 在 `URLs` 部分，将 URL 设置为 `http://<host>:<port>/webhooks/rocketchat/webhook`，将主机和端口替换为你正在运行的 Rasa 服务器的适当值。

有关 Rocket.Chat Webhook 的更多信息，请参阅 [Rocket.Chat 指南](https://docs.rocket.chat/guides/administrator-guides/integrations)。

## 在 RocketChat 上运行 {#running-on-rocketchat}

将 RocketChat 凭据添加到 `credentials.yml` 文件中：

```yaml
rocketchat:
  user: "yourbotname"
  password: "YOUR_PASSWORD"
  server_url: "https://demo.rocket.chat"
```

重启你的 Rasa 服务器，使新的频道端点可供 RocketChat 发送消息。
