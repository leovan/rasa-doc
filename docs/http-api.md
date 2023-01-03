# HTTP API

你可以使用 HTTP API 与正在运行的 Rasa 服务器进行交互。可以使用 API 进行模型训练、发送消息、运行测试等。

!!! tip "寻找 API Endpoint？"

    查看 [API 规范](https://rasa.com/docs/rasa/pages/http-api/)来获取所有可用的 Endpoint 以及他们的请求和响应格式。

## 启用 HTTP API {#enabling-the-http-api}

默认情况下，运行 Rasa 服务器不会启动 API Endpoint。与对话机器人的交互可以通过暴露的 `webhooks/<channel>/webhook` Endpoint 实现。

要启用 API 来同对话追踪器和其他对话机器人 Endpoint 直接交互，请将 `--enable-api` 参数添加到命令中运行：

```shell
rasa run --enable-api
```

请主要，使用仅 NLU 模型启动服务器，并非所有可用 Endpoint 都可以调用。一些 Endpoint 将返回 409 状态码，应为需要经过训练的对话模型来处理请求。

!!! caution "注意"

    确保通过限制对服务器的访问（例如使用防火墙）或启用身份验证方法来保护你的服务器。请参阅[安全注意事项](/http-api/#security-considerations)。

默认情况下，HTTP 服务器作为单个进程运行。可以使用 `SANIC_WORKERS` 环境变量更改工作进程的数量。建议将工作进程的数量设置为可用的 CPU 核数（更多信息请参见 [Sanic 文档](https://sanicframework.org/en/guide/deployment/running.html#workers)）。这只能与 `RedisLockStore` 结合使用（请参见[锁存储](/lock-stores/)）。

!!! caution "注意"

    [SocketIO 频道](/connectors/your-own-website/#websocket-channel)不支持多个工作进程。

## 安全注意事项 {#security-considerations}

建议不要将 Rasa 服务器直接暴露给外部，而是通过例如 Nginx 来连接它。

其内置了两种身份验证方法：

### 基于令牌的身份验证 {#token-based-auth}

要使用纯文本令牌来保护服务器，请在启动服务器时在 `--auth-token thisismysecret` 参数中指定令牌。

```shell
rasa run \
    --enable-api \
    --auth-token thisismysecret
```

任何向服务器发送请求的客户端都必须将令牌作为查询参数进行传递，否则将拒绝请求。例如，要从服务器获取追踪器：

```shell
curl -XGET localhost:5005/conversations/default/tracker?token=thisismysecret
```

### 基于 JWT 的身份验证 {#jwt-based-auth}

要使用基于 JWT 的身份验证，请在服务器启动时在 `--jwt-secret thisismysecret` 参数中指定 JWT 密码：

```shell
rasa run \
    --enable-api \
    --jwt-secret thisismysecret
```

客户端对服务器的请求需要在 `Authorization` 头中包含一个有效的 JWT 令牌，该令牌使用此密钥和 `HS256` 算法进行签名，例如：

```
"Authorization": "Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJ"
                 "zdWIiOiIxMjM0NTY3ODkwIiwibmFtZSI6IkpvaG4gRG9lIi"
                 "wiaWF0IjoxNTE2MjM5MDIyfQ.qdrr2_a7Sd80gmCWjnDomO"
                 "Gl8eZFVfKXA6jhncgRn-I"
```

令牌的有效负载必须包含 `user` 键下的对象，该对象又必须包含 `username` 和 `role` 属性。如下是 JWT 令牌的示例负载：

```json
{
    "user": {
        "username": "<sender_id>",
        "role": "user"
    }
}
```

如果 `role` 为 `admin`，则所有 Endpoint 都可以访问。如果 `role` 是 `user`，则只有当 `sender_id` 与有效负载的 `username` 属性匹配时，才能访问带有 `sender_id` 参数的 Endpoint。

```shell
rasa run \
    -m models \
    --enable-api \
    --jwt-secret thisismysecret
```

要创建和编码令牌，可以使用 [JWT 调试器](https://jwt.io/)等工具或 [PyJWT](https://pyjwt.readthedocs.io/en/latest/) 等 Python 模块。
