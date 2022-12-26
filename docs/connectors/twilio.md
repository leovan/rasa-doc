# Twilio

你可以使用 Twilio 连接器部署可通过短信获取消息的对话机器人。

## 获取凭据 {#getting-credentials}

你必须首先创建一个 Twilio 应用来获取凭据。之后可以将它们添加到 `credentials.yml` 中。

如何获取 Twilio 凭据，你需要设置 Twilio 机器人。

1. 创建 Twilio 账户后，你需要创建一个新项目。这里要选择的最重要的产品是 `Programmable SMS`。
2. 创建项目后，导航到 `Programmable SMS` 的仪表盘，然后单击 `Get Started`。按照步骤将电话号码连接到项目。
3. 现在可以在 `credentials.yml` 中使用 `Account SID`，`Auth Token` 和购买的电话号码了。
4. 导航到 Twilio 仪表盘中的 [`Phone Numbers`](https://www.twilio.com/console/phone-numbers/incoming) 并选择你的电话号码来配置 webhook URL。找到 `Messaging` 部分并将你的 webhook URL（例如：`https://<host>:<port>/webhooks/twilio/webhook`，将主机和端口替换为你正在运行的 Rasa 服务器的适当值）添加到 `A MESSAGE COMES IN` 设置中。

更多信息请参见 [Twilio REST API](https://www.twilio.com/docs/iam/api)。

### 连接到 WhatsApp {#connecting-to-whatsapp}

你可以通过 Twilio 将 Rasa 对话机器人部署到 WhatsApp。为此，你必须拥有 [WhatsApp Business](https://www.whatsapp.com/business/) 资料。将 WhatsApp Business 资料与你通过 Twilio 购买的电话号码相关联，从而访问[适用于 WhatsApp 的 Twilio API](https://www.twilio.com/docs/whatsapp/api)。

根据 [Twilio API 文档](https://www.twilio.com/docs/whatsapp/api#using-phone-numbers-with-whatsapp)，你使用的电话号码应以 `whatsapp:` 为前缀，如下面 `credentials.yml` 中的描述：

```yaml
twilio:
  account_sid: "ACbc2dxxxxxxxxxxxx19d54bdcd6e41186"
  auth_token: "e231c197493a7122d475b4xxxxxxxxxx"
  twilio_number: "+440123456789"  # if using WhatsApp: "whatsapp:+440123456789"
```

重启你的 Rasa 服务器，使新的频道 Endpoint 可供 Twilio 发送消息。
