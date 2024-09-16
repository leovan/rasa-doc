# Telegram

你必须首先创建一个 Telegram 机器人来获取凭据。之后可以将它们添加到 `credentials.yml` 中。

## 获取凭据 {#getting-credentials}

如何获取 Telegram 凭据，你需要设置 Telegram 机器人。

1. 要创建机器人，请转到 [Bot Father](https://web.telegram.org/#/im?p=@BotFather)，输入 `/newbot` 并按照说明进行操作。Telegram 用于发送的消息的 URL 应类似于 `http://<host>:<port>/webhooks/telegram/webhook`，将主机和端口替换为你正在运行的 Rasa 服务器的适当值。
2. 最后你应该得到 `access_token`，并且设置的用户名会进行验证。
3. 如果你想在群组设置中使用你的机器人，建议通过输入 `/setprivacy` 打开群组隐私模式。然后机器人只会在用户的消息以 `/bot` 开头时才会侦听。

更多信息请参见 [Telegram HTTP API](https://core.telegram.org/bots/api)。

## 在 Telegram 上运行 {#running-on-telegram}

将 Telegram 凭据添加到 `credentials.yml` 文件中：

```yaml
telegram:
  access_token: "490161424:AAGlRxinBRtKGb21_rlOEMtDFZMXBl6EC0o"
  verify: "your_bot"
  webhook_url: "https://your_url.com/webhooks/telegram/webhook"
```

重启你的 Rasa 服务器，使新的频道端点可供 Telegram 发送消息。

!!! info "处理 `/START` 消息"

    在对话开始时，用户将按 Telegram 中的 Start 按钮。这将触发发送一条内容为 `/start` 的消息。在 NLU 训练数据文件中设定特定意图以确保对话机器人可以这个消息。然后将此 `start` 意图与故事或规则一起添加到领域中来处理它。

## 支持的回复附件 {#supported-response-attachments}

除了典型的 `text` 响应之外，此频道还支持来自 [Telegram API](https://core.telegram.org/bots/api/#message) 的如下组件：

- `button` 参数：
    - button_type：inline | vertical | reply
- `custom` 参数：
    - photo
    - audio
    - document
    - sticker
    - video
    - video_note
    - animation
    - voice
    - media
    - latitude, longitude (location)
    - latitude, longitude, title, address (venue)
    - phone_number
    - game_short_name
    - action

示例：

```yaml
utter_ask_transfer_form_confirm:
- buttons:
  - payload: /affirm
    title: Yes
  - payload: /deny
    title: No, cancel the transaction
  button_type: vertical
  text: Would you like to transfer {currency}{amount_of_money} to {PERSON}?
  image: "https://i.imgur.com/nGF1K8f.jpg"
```

```yaml
utter_giraffe_sticker:
- text: Here's my giraffe sticker!
  custom:
    sticker: "https://github.com/TelegramBots/book/raw/master/src/docs/sticker-fred.webp"
```
