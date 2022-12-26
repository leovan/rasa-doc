# Cisco Webex Teams

你必须首先创建一个 cisco webex 应用来获取凭据。之后可以将它们添加到 `credentials.yml` 中。

## 获取凭据 {#getting-credentials}

如何获取 Cisco Webex Teams 凭据：

你需要设置一个机器人。查看 [Cisco Webex for Developers 文档](https://developer.webex.com/docs/bots)，了解有关如何创建机器人的信息。

通过 Cisco Webex Teams 创建对话机器人后，需要在 Cisco Webex Teams 中创建一个房间。然后在房间中添加话机机器人，就像在房间中添加人一样。

你需要记下创建房间的 ID，此房间 ID 将用于 `credentials.yml` 文件中的 `room` 变量。

请按照如下链接查找房间 ID：https://developer.webex.com/endpoint-rooms-get.html。

在 OAuth & Permissions 部分中，添加用于 Webex 转发消息的 Rasa Endpoint 的 URL。用于接收 Cisco Webex Teams 消息的 Endpoint 应类似于 `http://<host>:<port>/webhooks/botframework/webhook`，将主机和端口替换为你正在运行的 Rasa 服务器的适当值。

## 在 Cisco Webex Teams 上运行 {#running-on-cisco-webex-teams}

将 Webex Teams 凭据添加到 `credentials.yml` 文件中：

```yaml
webexteams:
  access_token: "YOUR-BOT-ACCESS-TOKEN"
  room: "YOUR-CISCOWEBEXTEAMS-ROOM-ID"
```

重启你的 Rasa 服务器，使新的频道 Endpoint 可供 Cisco Webex Teams 发送消息。
