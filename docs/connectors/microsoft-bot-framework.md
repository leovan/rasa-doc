# Microsoft Bot Framework

你必须首先创建一个 Miscrosoft 应用来获取凭据。之后可以将它们添加到 `credentials.yml` 中。

Microsoft Bot Framework 用于发送的消息的 URL 应类似于 `http://<host>:<port>/webhooks/botframework/webhook`，将主机和端口替换为你正在运行的 Rasa 服务器的适当值。

## 在 Microsoft Bot Framework 上运行 {#running-on-microsoft-bot-framework}

将 Botframework 凭据添加到 `credentials.yml` 文件中：

```yaml
telegram:
  access_token: "490161424:AAGlRxinBRtKGb21_rlOEMtDFZMXBl6EC0o"
  verify: "your_bot"
  webhook_url: "https://your_url.com/webhooks/telegram/webhook"
```

重启你的 Rasa 服务器，使新的频道端点可供 Telegram 发送消息。
