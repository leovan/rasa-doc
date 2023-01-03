# Facebook Messenger

## Facebook 设置 {#facebook-setup}

你首选需要设置一个 Facebook 页面和应用来获取连接到 Facebook Messenger 的凭据。之后可以将它们添加到 `credentials.yml` 中。

### 获取凭据 {#getting-credentials}

如何获取 Facebook 凭据：你需要设置一个 Facebook 应用和一个页面。

1. 要创建应用，请前往 [Facebook for Developers](https://developers.facebook.com/)，然后单击 `My Apps -> Add New App`。
2. 进入应用的仪表盘，在 `Products` 下找到 `Messenger`，然后单击 `Set Up`。向下滚动到 `Token Generation`，然后单击链接为你的应用创建一个新页面。
3. 创建你的页面并在 `Token Generation` 生成的下拉菜单中选择它。显示的 `Page Access Token` 是稍后需要的 `page-access-token`。
4. 在 `Settings -> Basic` 下的应用仪表盘中找到 `App Secret`。这将是你的 `secret`。
5. 在 `credentials.yml` 使用收集的 `secret` 和 `page-access-token`，并添加一个名为 `verify` 的字段，其中包含你选择的字符串。使用 `--credentials credentials.yml` 选项启动 `rasa run`。
6. 设置一个 `Webhook` 并至少选择 `messaging` 和 `messaging_postback` 订阅。插入你的回调 URL，类似 `https://<host>:<port>/webhooks/facebook/webhook`，用你正在运行的 Rasa 服务器中适当值替换主机和端口。

插入必须与 `credentials.yml` 中的 `verify` 项相匹配的 `Verify Token`。

!!! note "配置 HTTPS"

    Facebook Messenger 仅通过 https 将消息转到到 endpoint，因此请采取适当的措施将其添加到你的设置中。有关对话机器人的本地测试，请参见[在本地机器上测试频道](/messaging-and-voice-channels/#testing-channels-on-your-local-machine)。

有关更详细的步骤，请访问 [Messenger 文档](https://developers.facebook.com/docs/graph-api/webhooks)。

### 在 Facebook Messenger 上运行 {#running-on-facebook-messenger}

将 Facebook 凭据添加到 `credentials.yml` 中：

```yaml
facebook:
  verify: "rasa-bot"
  secret: "3e34709d01ea89032asdebfe5a74518"
  page-access-token: "EAAbHPa7H9rEBAAuFk4Q3gPKbDedQnx4djJJ1JmQ7CAqO4iJKrQcNT0wtD"
```

重启你的 Rasa 服务器，使新的频道 Endpoint 可供 Facebook Messenger 发送消息。

## 支持的回复附件 {#supported-response-attachments}

除了典型的文本、图像和自定义响应之外，Facebook Messenger 频道还支持如下附加回复附件：

- [按钮](https://developers.facebook.com/docs/messenger-platform/send-messages/buttons)的结构与其他 Rasa 按钮相同。Facebook API 将在消息中发送按钮的数量限制为 3 个。如果消息中提供的按钮超过 3 个，Rasa 将忽略所有提供的按钮。
- [快速回复](https://developers.facebook.com/docs/messenger-platform/send-messages/quick-replies)提供了一种在对话中呈现最多 13 个按钮的方法，其中包含标题和可选图像，并显著显示在组合器上方。你还可以使用快速回复来请求某人的电子邮件地址或电话号码。

    ```yaml
    utter_fb_quick_reply_example:
      - text: Hello World!
        quick_replies:
          - title: Text quick reply
            payload: /example_intent
          - title: Image quick reply
            payload: /example_intent
            image_url: http://example.com/img/red.png
          # below are Facebook provided quick replies
          # the title and payload will be filled
          # with the user's information from their profile
          - content_type: user_email
            title:
            payload:
          - content_type: user_phone_number
            title:
            payload:
    ```

    !!! note "注意"

        Facebook Messenger 中的快速回复和按钮标题均限制为 20 个字符。超过 20 个字符的标题将被截断。

- [元素](https://developers.facebook.com/docs/messenger-platform/send-messages/template/generic)提供了一种创建水平滚动列表的方法，该列表最多可包含 10 个内容元素，这些元素将按钮、图像等与文本一起集成到单个消息中。

    ```yaml
    utter_fb_element_example:
      - text: Hello World!
        elements:
          - title: Element Title 1
            subtitle: Subtitles are supported
            buttons: # note the button limit still applies here
              - title: Example button A
                payload: /example_intent
              - title: Example button B
                payload: /example_intent
              - title: Example button C
                payload: /example_intent
          - title: Element Title 2
            image_url: http://example.com/img/red.png
            buttons:
              - title: Example button D
                payload: /example_intent
              - title: Example button E
                payload: /example_intent
              - title: Example button F
                payload: /example_intent
    ```
