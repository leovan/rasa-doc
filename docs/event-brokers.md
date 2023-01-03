# 事件代理

事件代理允许将正在运行的对话机器人连接到处理来自对话的数据的其他服务。事件代理将消息发送到消息流服务（也称为消息代理），来将 Rasa [事件](https://rasa.com/docs/action-server/events)从 Rasa 服务器转发到其他服务。

## 格式 {#format}

每次追踪器更新其状态时，所有事件都会作为序列化字典流式传输到代理。`default` 追踪器发出的示例事件如下所示：

```json
{
    "sender_id": "default",
    "timestamp": 1528402837.617099,
    "event": "bot",
    "text": "what your bot said",
    "data": "some data about e.g. attachments"
    "metadata" {
          "a key": "a value",
     }
}
```

`event` 字段采用事件的 `type_name`（有关事件类型的更多信息，请参见[事件](/action-server/events/)文档）。

## Pika 事件代理 {#pika-event-broker}

如下展示的示例实现了使用 [RabbitMQ](https://www.rabbitmq.com/) 的 Python 客户端库 [Pika](https://pika.readthedocs.io/)。

### 使用 Endpoint 配置添加 Pika 事件代理 {#adding-a-pika-event-broker-using-the-endpoint-configuration}

可以通过将 `event_broker` 部分添加到 `endpoints.yml` 来指示 Rasa 将所有事件流式传输到 Pika 事件代理：

```yaml
event_broker:
  type: pika
  url: localhost
  username: username
  password: password
  queues:
    - queue-1
#   you may supply more than one queue to publish to
#   - queue-2
#   - queue-3
  exchange_name: exchange
```

可以在[参考文档](https://rasa.com/docs/rasa/reference/rasa/core/brokers/pika/#__init__)中找到在 `endpoints.yml` 文件中自定义的所有参数的完整列表。当重新启动 Rasa 服务时，Rasa 将自动启动流式传输事件。

### 向 Pika 事件代理添加 SSL 选项 {#adding-ssl-options-to-the-pika-event-broker}

可以通过设置如下必需的环境变量来创建 RabbitMQ SSL 选项：

- `RABBITMQ_SSL_CLIENT_CERTIFICATE`：SSL 客户端证书的路径。
- `RABBITMQ_SSL_CLIENT_KEY`：SSL 客户端密钥的路径。

请注意，不再支持通过环境变量指定 `RABBITMQ_SSL_CA_FILE`，以及指定 `RABBITMQ_SSL_KEY_PASSWORD`，请改用未加密的密钥文件。

### 在 Python 中添加 Pika 事件代理 {#adding-a-pika-event-broker-in-python}

如下是使用 Python 代码添加它的方法：

```python
import asyncio

from rasa.core.brokers.pika import PikaEventBroker
from rasa.core.tracker_store import InMemoryTrackerStore

pika_broker = PikaEventBroker('localhost',
                              'username',
                              'password',
                              queues=['rasa_events'],
                              event_loop=event_loop
                              )
asyncio.run(pika_broker.connect())

tracker_store = InMemoryTrackerStore(domain=domain, event_broker=pika_broker)
```

### 实现 Pika 事件消费者 {#implementing-a-pika-event-consumer}

需要运行一个 RabbitMQ 服务器，以及另一个使用事件的应用。消费者需要使用 `callback` 操作来实现 Pika 的 `start_consuming()` 方法。示例如下：

```python
import json
import pika


def _callback(ch, method, properties, body):
        # Do something useful with your incoming message body here, e.g.
        # saving it to a database
        print("Received event {}".format(json.loads(body)))

if __name__ == "__main__":

    # RabbitMQ credentials with username and password
    credentials = pika.PlainCredentials("username", "password")

    # Pika connection to the RabbitMQ host - typically 'rabbit' in a
    # docker environment, or 'localhost' in a local environment
    connection = pika.BlockingConnection(
        pika.ConnectionParameters("rabbit", credentials=credentials)
    )

    # start consumption of channel
    channel = connection.channel()
    channel.basic_consume(queue="rasa_events", on_message_callback=_callback, auto_ack=True)
    channel.start_consuming()
```

## Kafka 事件代理 {#kafka-event-broker}

虽然 RabbitMQ 是默认的事件代理，但可以使用 [Kafka](https://kafka.apache.org/) 作为事件的主要代理。Rasa 使用 [kafka-python](https://kafka-python.readthedocs.io/en/master/usage.html) 库，这是一个用 Python 编写的 Kafka 客户端。你需要一个正在运行的 Kafka 服务器。

### 分区键 {#partition-key}

Rasa 的 Kafka 生产者可以选择配置为按对话 ID 对消息进行分区。可以通过将 `endpoints.yml` 文件中的 `partition_by_sender` 设置为 `True` 来配置。默认情况下，该参数设置为 `False`，生产者会为每条消息随机分配一个分区。

```python title="endpoints.yml"
event_broker:
  type: kafka
  partition_by_sender: True
  security_protocol: PLAINTEXT
  topic: topic
  url: localhost
  client_id: kafka-python-rasa
```

### 身份验证和授权 {#authentication-and-authorization}

Rasa 的 Kafka 生产者接受以下类型的安全协议：`SASL_PLAINTEXT`、`SSL`、`PLAINTEXT` 和 `SASL_SSL`。

对于开发环境，或者如果代理服务器和客户端位于同一台机器中，可以使用 `SASL_PLAINTEXT` 或 `PLAINTEXT` 的简单身份验证。通过使用此协议，客户端和服务器之间交换的凭据和信息将以明文形式发送。因此，这不是最安全的方法，但由于它易于配置，对于简单的集群配置很有用。`SASL_PLAINTEXT` 协议需要设置先前在代理服务器中配置的用户名和密码。

如果 Kafka 集群中的客户端或代理位于不同的机器上，则使用 `SSL` 或 `SASL_SSL` 协议来确保数据加密和客户端身份验证则非常重要。在为代理和客户端生成有效证书后，必须提供证书路径和为生产者生成的密钥，以及 CA 的根证书。

使用 `SASL_PLAINTEXT` 和 `SASL_SSL` 协议时，可选配置 `sasl_mechanism`，默认设置为 `PLAIN`。`sasl_mechanism` 的有效值为：`PLAIN`、`GSSAPI`、`OAUTHBEARER`、`SCRAM-SHA-256` 和 `SCRAM-SHA-512`。

如果 `GSSAPI` 用于 `sasl_mechanism`，则需要额外安装 [python-gssapi](https://pypi.org/project/python-gssapi/) 和必要的 C 库 Kerberos 依赖。

如果启用了 `ssl_check_hostname` 参数，客户端将验证代理的主机名是否与证书匹配。它用于客户端连接和代理间的连接，以防止中间人攻击。

### 使用 Endpoint 配置添加 Kafka 事件代理 {#adding-a-kafka-event-broker-using-the-endpoint-configuration}

可以通过将 `event_broker` 部分添加到 `endpoints.yml` 来指示 Rasa 将所有事件流式传输到 Kafka 事件代理。

使用 `SASL_PLAINTEXT` 协议，Endpoint 文件必须具有如下项：

```yaml
event_broker:
  type: kafka
  security_protocol: SASL_PLAINTEXT
  topic: topic
  url: localhost
  partition_by_sender: True
  sasl_username: username
  sasl_password: password
  sasl_mechanism: PLAIN
```

使用 `PLAINTEXT` 协议，Endpoint 文件必须具有如下项：

```yaml
event_broker:
  type: kafka
  security_protocol: PLAINTEXT
  topic: topic
  url: localhost
  client_id: kafka-python-rasa
```

如果使用 `SSL` 协议，Endpoint 文件应如下所示：

```yaml
event_broker:
  type: kafka
  security_protocol: SSL
  topic: topic
  url: localhost
  ssl_cafile: CARoot.pem
  ssl_certfile: certificate.pem
  ssl_keyfile: key.pem
  ssl_check_hostname: True
```

如果使用 `SASL_SSL` 协议，Endpoint 文件应如下所示：

```yaml
event_broker:
  type: kafka
  security_protocol: SASL_SSL
  topic: topic
  url: localhost
  sasl_username: username
  sasl_password: password
  sasl_mechanism: PLAIN
  ssl_cafile: CARoot.pem
  ssl_certfile: certificate.pem
  ssl_keyfile: key.pem
  ssl_check_hostname: True
```

## SQL 事件代理 {#sql-event-broker}

可以将 SQL 数据库用作事件代理。使用 [SQLAlchemy](https://www.sqlalchemy.org/) 建立与数据库的连接，SQLAlchemy 是一个可以与多种不同类型的 SQL 数据库（例如：[SQLite](https://sqlite.org/index.html)、[PostgreSQL](https://www.postgresql.org/) 等）进行交互的 Python 库。默认的 Rasa 安装允许连接到 SQLite 和 PostgreSQL 数据库。其他选项请参见 [SQLAlchemy 文档的 SQL 方言](https://docs.sqlalchemy.org/en/13/dialects/index.html)。

### 使用 Endpoint 配置添加 SQL 事件代理 {#adding-a-sql-event-broker-using-the-endpoint-configuration}

要指示 Rasa 将所有事件保存到 SQL 事件代理，请将 `event_broker` 部分添加到 `endpoints.yml` 中。例如，一个有效的 SQLite 配置如下所示：

```yaml title="endpoints.yml"
event_broker:
  type: SQL
  dialect: sqlite
  db: events.db
```

也可以使用 PostgreSQL 数据库：

```yaml title="endpoints.yml"
event_broker:
  type: SQL
  url: 127.0.0.1
  port: 5432
  dialect: postgresql
  username: myuser
  password: mypassword
  db: mydatabase
```

应用此配置后，Rasa 将在数据上创建一个名为 `events` 的表，将在其中添加所有事件。

## 文件事件代理 {#fileeventbroker}

可以将 `FileEventBroker` 用作事件代理。此实现会将事件记录到 JSON 格式文件中。如果希望覆盖默认文件名 `rasa_event.log`，可以在 `endpoints.yml` 文件中提供路径键。

## 自定义事件代理 {#custom-event-broker}

如果你需要一个无法开箱即用的事件代理，可以实现一个自定义的。通过扩展基类 `EventBroker` 可以完成。

自定义事件代理类必须实现如下基类方法：

- `from_endpoint_config`：从 Endpoint 配置创建一个 `EventBroker` 对象。（[源代码](https://github.com/RasaHQ/rasa/blob/main/rasa/core/brokers/broker.py#L45)）
- `publish`：将 JSON 格式的 [Rasa 事件](https://rasa.com/docs/rasa/reference/rasa/shared/core/events/)发布到事件队列中。（[源代码](https://github.com/RasaHQ/rasa/blob/main/rasa/core/brokers/broker.py#L63)）
- `is_ready`：判断事件代理是否准备好。（[源代码](https://github.com/RasaHQ/rasa/blob/main/rasa/core/brokers/broker.py#L67)）
- `close`：关闭与事件代理的连接。（[源代码](https://github.com/RasaHQ/rasa/blob/main/rasa/core/brokers/broker.py#L75)）

### 配置 {#configuration}

将自定义事件代理的模块路径和所需的参数写入 `endpoints.yml` 中：

```yaml title="endpoints.yml"
event_broker:
  type: path.to.your.module.Class
  url: localhost
  a_parameter: a value
  another_parameter: another value
```
