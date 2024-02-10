# 连接至消息和语音频道

开源 Rasa 提供了许多内置连接器来连接到常见的消息和语音频道。你还可以使用预配置的 REST 频道连接到你的网站或应用程序，或构建你自己的自定义连接器。

## 连接到频道 {#connecting-to-a-channel}

了解如何让对话机器人在如下频道中可用：

- [个人网站](connectors/your-own-website.md)
- [Facebook Messenger](connectors/facebook-messenger.md)
- [Slack](connectors/slack.md)
- [Telegram](connectors/telegram.md)
- [Twilio](connectors/twilio.md)
- [Microsoft Bot Framework](connectors/microsoft-bot-framework.md)
- [Cisco Webex Teams](connectors/cisco-webex-teams.md)
- [RocketChat](connectors/rocketchat.md)
- [Mattermost](connectors/mattermost.md)
- [Google Hangouts Chat](connectors/hangouts.md)
- [自定义连接器](connectors/custom-connectors.md)

## 在本地测试频道 {#testing-channels-on-your-local-machine}

如果你在 `localhost` 上运行开源 Rasa，由于 `localhost` 并未对互联网开放，大部分外部频道无法找到服务的 URL。

要使本地计算机上的端口在互联网上公开可用，你可以使用 [ngrok](https://ngrok.com/)。或者，可以参阅[此列表](https://github.com/anderspitman/awesome-tunneling)来追踪和比较其他频道解决方案。

安装 ngrok 后，运行：

```shell
ngrok http 5005; rasa run
```

当你按照说明使对话机器人在频道上可用时，可以使用 ngrok URL。具体来说，当说明使用 `https://<host>:<port>/webhooks/<CHANNEL>/webhook`，则使用 `<ngrok_url>/webhooks/<CHANNEL>/webhook`，将 `<ngrok_url>` 替换为 ngrok 终端窗口中显示的随机生成的 URL。例如，如果将你的机器人连接到 Slack，则 URL 应类似于 `https://26e7e7744191.ngrok.io/webhooks/slack/webhook`。

!!! warning "警告"

    使用 ngrok 的免费版，你可能会遇到每分钟建立链接数的限制。在撰写本文时，限制为 40 次连接每分钟。

或者，你可以使用 `-i` 命令行选项让对话机器人监听特定地址：

```shell
rasa run -p 5005 -i 192.168.69.150
```

当你面向互联网的机器使用 VPN 接口连接到后端服务器时特别有用。
