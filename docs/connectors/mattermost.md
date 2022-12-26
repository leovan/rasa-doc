# Mattermost

你必须首先创建一个 mattermost 应用来获取凭据。之后可以将它们添加到 `credentials.yml` 中。

## 获取凭据 {#getting-credentials}

Mattermost 现在使用机器人账户来提高安全性。因此，你可以使用他们的指南来创建你的机器人，以获取 credentials.yml 文件所需的令牌。

有关创建机器人账户的更多信息，请参见 [Bot Creation](https://docs.mattermost.com/developer/bot-accounts.html#bot-account-creation)。

有关将现有账户转换为机器人账户的信息，请参见 [User Conversion](https://docs.mattermost.com/developer/bot-accounts.html#how-do-i-convert-an-existing-account-to-a-bot-account)。

如何设置传出 webhook：

1. 要创建Mattermost 传出 webhook，请登录到您的 Mattermost 团队网站，然后转到 `Main Menu > Integrations > Outgoing Webhooks`。
2. 单击 `Add outgoing webhook`。
3. 填写详细信息，包括希望机器人进入的频道。你需要确保使用 `@yourbotname` 设置 `trigger words` 部分，以便对话机器人不会在所有内容上触发。
4. `Content Type` 必须设置为 `application/json`。
5. 确保 `trigger when` 设置为 `first word matches a trigger word exactly`。
6. 添加回调 URL，其类似 `http://<host>:<port>/webhooks/mattermost/webhook`，将主机和端口替换为你正在运行的 Rasa 服务器的适当值。

更多详细步骤，请参见 [Mattermost 文档](https://docs.mattermost.com/guides/developer.html)。

## 在 Mattermost 上运行 {#running-on-mattermost}

将 Mattermost 凭据添加到 `credentials.yml` 文件中：

```yaml
mattermost:
  url: "https://chat.example.com/api/v4"
  token: "xxxxx" #  the token for the bot account from creating the bot step.
  webhook_url: "https://server.example.com/webhooks/mattermost/webhook"  # this should match the callback url from step 6
```

重启你的 Rasa 服务器，使新的频道 Endpoint 可供 Mattermost 发送消息。
