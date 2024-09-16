# 仅 NLU 服务器

你可以运行仅 NLU 的服务器并使用 HTTP API 连接到它。

## 连接到 NLU 服务器 {#connecting-to-an-nlu-server}

可以通过将连接详细信息添加到对话管理服务器的端点配置文件，将[仅 Rasa NLU 服务器](nlu-only.md#running-an-nlu-server)连接到单独运行的仅 Rasa 对话管理服务器：

```yaml title="endpoints.yml"
nlu:
    url: "http://<your nlu host>:<your nlu port>"
    token: <token>  # [optional]
    token_name: <name of the token> # [optional] (default: token)
```

`token` 和 `token_name` 指的是可选的[身份验证参数](http-api.md#token-based-auth)。

对话管理服务器应该为不包含 NLU 模型的模型提供服务。要获得仅对话管理的模型，请使用 `rasa train core` 或使用 `rasa train` 但排除所有 NLU 数据。

对话管理服务器收到消息后，会向 `http://<your nlu host>:<your nlu port>/model/parse` [发送请求](https://rasa.com/docs/rasa/pages/http-api#operation/parseModelMessage){:target="_blank"}，并使用和解析返回的信息。

!!! info "端点配置"

    对话管理服务器的端点配置将包括一个指向 NLU 唯一服务器的 `nlu` 端点。因此，你应该为 NLU 服务器使用单独的端点配置文件，不包括 `nlu` 端点。

如果你正在实现自定义 NLU 服务器（例如：非 Rasa NLU），服务器应该提供一个 `/model/parse` 端点，其应与 Rasa NLU 服务器以相同的格式响应请求。
