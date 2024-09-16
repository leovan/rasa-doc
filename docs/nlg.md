# NLG 服务器

重新训练对话机器人只是为了更改文本副本对于某些工作流程来说可能不是最理想的。这就是为什么 Rasa 还允许你将响应生成和对话学习分开。

对话机器人将学习预测动作并根据过去的对话对用户输入做出反应，但它发送会用户的响应将在 Rasa 之外生成。

当对话机器人想要向用户发送消息时，它将调用你定义的外部 HTTP 服务器。

## 响应请求 {#responding-to-requests}

### 请求格式 {#request-format}

当你的模型预测对话机器人应该向用户发送响应时，它会向你的服务器发送请求，为你提供选择或生成响应所需的信息。

发送到 NLG 端点的 POST 请求的主体结构如下：

!!! info "3.6 版本新特性"

    我们在请求体中添加 `id` 字段。该字段包含响应变体的 ID。你可以使用此信息在 NLG 服务器上组装/选择合适的响应变体。

```json
{
  "response":"utter_what_can_do",
  "arguments":{
    
  },
  "tracker":{
    "sender_id":"user_0",
    "slots":{
      
    },
    "latest_message":{
      "intent":{
        "id":3014457480322877053,
        "name":"greet",
        "confidence":0.9999994039535522
      },
      "entities":[
        
      ],
      "text":"Hello",
      "message_id":"94838d6f49ff4366b254b6f6d23a90cf",
      "metadata":{
        
      },
      "intent_ranking":[
        {
          "id":3014457480322877053,
          "name":"greet",
          "confidence":0.9999994039535522
        },
        {
          "id":8842445304628198686,
          "name":"ask_forget_reminders",
          "confidence":5.675940428773174e-07
        },
        {
          "id":-2566831912141022859,
          "name":"bye",
          "confidence":3.418941929567154e-08
        },
        {
          "id":8340513453672591403,
          "name":"ask_id",
          "confidence":2.5274500714544956e-08
        },
        {
          "id":5822154213939471096,
          "name":"ask_remind_call",
          "confidence":2.4177523982871207e-08
        }
      ]
    },
    "latest_event_time":1599476297.694504,
    "followup_action":null,
    "paused":false,
    "events":[
      {
        "event":"action",
        "timestamp":1599476297.68784,
        "name":"action_session_start",
        "policy":null,
        "confidence":null
      },
      {
        "event":"session_started",
        "timestamp":1599476297.6878452
      },
      {
        "event":"action",
        "timestamp":1599476297.6878562,
        "name":"action_listen",
        "policy":null,
        "confidence":null
      },
      {
        "event":"user",
        "timestamp":1599476297.694504,
        "text":"Hello",
        "parse_data":{
          "intent":{
            "id":3014457480322877053,
            "name":"greet",
            "confidence":0.9999994039535522
          },
          "entities":[
            
          ],
          "text":"Hello",
          "message_id":"94838d6f49ff4366b254b6f6d23a90cf",
          "metadata":{
            
          },
          "intent_ranking":[
            {
              "id":3014457480322877053,
              "name":"greet",
              "confidence":0.9999994039535522
            },
            {
              "id":8842445304628198686,
              "name":"ask_forget_reminders",
              "confidence":5.675940428773174e-07
            },
            {
              "id":-2566831912141022859,
              "name":"bye",
              "confidence":3.418941929567154e-08
            },
            {
              "id":8340513453672591403,
              "name":"ask_id",
              "confidence":2.5274500714544956e-08
            },
            {
              "id":5822154213939471096,
              "name":"ask_remind_call",
              "confidence":2.4177523982871207e-08
            }
          ]
        },
        "input_channel":"rest",
        "message_id":"94838d6f49ff4366b254b6f6d23a90cf",
        "metadata":{
          
        }
      }
    ],
    "latest_input_channel":"rest",
    "active_loop":{
      
    },
    "latest_action_name":"action_listen"
  },
  "channel":{
    "name":"collector"
  }
}
```

如下是发布请求中高级键的描述：

| 键          | 描述                                 |
| ----------- | ------------------------------------ |
| `response`  | Rasa 预测的响应的名称                |
| `arguments` | 可以由自定义动作提供的可选关键字参数 |
| `tracker`   | 包含整个对话历史的字典               |
| `channel`   | 此消息将发送到的频道                 |

你可以使用任何或所有这些信息来决定如何生成回复。

### 响应格式 {#response-format}

端点需要使用生成的响应进行响应。然后 Rasa 会将此响应发送回给用户。

如下是响应的可能键及其类型：

```json
{
    "text": "Some text",
    "buttons": [],
    "image": null,  # string of image URL
    "elements": [],
    "attachments": [],
    "custom": {}
}
```

你可以选择仅提供文本，也可以选择提供不同类型的丰富响应的组合。就像[领域文件中定义的响应](responses.md)一样，响应至少需要包含 `text` 或 `custom` 才能成为有效响应。

!!! warning "从故事中调用响应"

    如果你使用外部 NLG 服务，则无需在领域中的 `responses` 下指定响应。但是，如果你想直接从你的故事中调用他们，仍需要将响应名称添加到领域的 `actions` 列表中。

## 配置服务器 URL {#configuring-the-server-url}

要告诉 Rasa 在哪里可以找到你的 NLG 服务器，请将 URL 添加到 `endpoints.yml` 中：

```yaml title="endpoints.yml"
nlg:
  url: http://localhost:5055/nlg
```

如果你的 NLG 服务器受到保护并且 Rasa 需要身份验证才能访问它，可以在端点中配置身份验证：

```yaml title="endpoints.yml"
nlg:
  url: http://localhost:5055/nlg
  #
  # You can also specify additional parameters, if you need them:
  # headers:
  #   my-custom-header: value
  # token: "my_authentication_token"  # will be passed as a GET parameter
  # basic_auth:
  #   username: user
  #   password: pass
```
