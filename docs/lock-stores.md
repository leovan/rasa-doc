# 锁存储

Rasa 使用一种票据锁机制来确保以正确的顺序处理给定对话 ID 的传入消息，并在主动处理消息时锁定对话。这意味着多个 Rasa 服务器可以作为复制服务并行运行，并且客户端在发送给定对话 ID 的消息时不一定需要寻址同一个节点。

## InMemoryLockStore（默认） {#inmemorylockstore-default}

### 描述 {#description}

`InMemoryLockStore` 是默认的锁存储。它在单个进程中维护对话锁。

!!! note "注意"

    当多个 Rasa 服务器并行运行时，不应该使用此锁存储。

### 配置 {#configuration}

要使用 `InMemoryLockStore` 不需要任何配置。

## ConcurrentRedisLockStore {#concurrentredislockstore}

<button data-md-color-primary="amber">仅 Rasa Pro</button>

!!! tip "Rasa Pro 许可证"

    你需要一个许可证才能使用 Rasa Pro。[请与 Rasa 专家联系](https://rasa.com/connect-with-rasa/)。

    如果不确定是否需要 Rasa Pro？[可以免费试用](https://info.rasa.com/rasa-platform-trial-request/)。

`ConcurrentRedisLockStore` 是一个新的锁存储，它使用 Redis 作为持久层，可以安全地与多个 Rasa 服务器副本一起使用。参阅[迁移部分](/lock-stores/#migration-guide)来了解如何切换到此锁存储。

### 描述 {#description-1}

`ConcurrentRedisLockStore` 使用 Redis 作为已发放[票据](https://rasa.com/docs/rasa/reference/rasa/core/lock/#ticket-objects)实例和最后发放票据号的持久层。

票据号从 1 开始，而 `RedisLockStore` 的初始化从 0 开始，如果票据过期，票据号将不会重新分配给未来的票据。因此，票据号对于票据实例是唯一的。票据号使用 Redis 原子事务 INCR 在持久的最后发放票据号上递增。

`ConcurrentRedisLockStore` 确保只有一个 Rasa 实例可以在任何时间点处理对话。因此，`LockStore` 的这个 Redis 实现可以处理不同 Rasa 服务器作为同一会话并行接收的消息。这是运行一组 Rasa 服务器副本的推荐锁存储。

### 配置 {#configuration-1}

要使用 Redis 设置 Rasa，需要执行如下步骤：

1. 启动 Redis 实例。
2. 将所需的配置添加到 `endpoints.yml`：

    ```yaml
    lock_store:
        type: rasa_plus.components.concurrent_lock_store.ConcurrentRedisLockStore
        host: <host of the redis instance, e.g. localhost>
        port: <port of your redis instance, usually 6379>
        password: <password used for authentication>
        db: <number of your database within redis, e.g. 0>
        key_prefix: <alphanumeric value to prepend to lock store keys>
    ```

3. 要使用 Redis 后端启动 Rasa Core 服务器，请添加 `--endpoints` 标识，例如：

    ```shell
    rasa run -m models --endpoints endpoints.yml
    ```

### 参数 {#parameters}

- `url`（默认：`localhost`）：Redis 实例 URL。
- `port`（默认：`6379`）：Redis 运行的端口。
- `db`（默认：`1`）：Redis 数据库的数量。
- `key_prefix`（默认：`None`）：锁存储键的前缀。必须是字母数字。
- `password`（默认：`None`）：用于认证的密码（`None` 表示不认证）。
- `use_ssl`（默认：`False`）：通信是否加密。
- `socket_timeout`（默认：`10`）：超过此时间秒数 Redis 没有响应则引发错误。

### 迁移指南 {#migration-guide}

要从 `RedisLockStore` 切换到 `ConcurrentRedisLockStore`，请在 `endpoints.yml` 中指定 `type` 为 `ConcurrentRedisLockStore` 类的完整模块路径：

```yaml
lock_store:
    type: rasa_plus.components.concurrent_lock_store.ConcurrentRedisLockStore
    host: <host of the redis instance, e.g. localhost>
    port: <port of your redis instance, usually 6379>
    password: <password used for authentication>
    db: <number of your database within redis, e.g. 0>
    key_prefix: <alphanumeric value to prepend to lock store keys>
```

你必须将 Redis 锁存储配置中的 `url` 字段替换为包含 Redis 实例的主机名的 `host` 字段。

切换到 `ConcurrentRedisLockStore` 时无需进行数据库迁移。可以使用与之前使用 `RedisLockStore` 时相同的 Redis 实例和数据库编号。如果使用相同的 Redis 数据库编号，你可能希望删除所有预先存在的键。`ConcurrentRedisLockStore` 不再需要这些以前的键值项，可以清除数据库。

使用 `RedisLockStore` 和 `ConcurrentRedisLockStore` 时存储的键值项没有重叠，因为 `RedisLockStore` 保留序列化的 `TicketLock` 实例，而 `ConcurrentRedisLockStore` 存储单个 `Ticket` 实例以及最后发出的票据号。`ConcurrentRedisLockStore` 从持久的 `Ticket` 实例重新创建 `TicketLock`，这允许它处理相同会话 ID 的并发消息。

## RedisLockStore {#redislockstore}

### 描述 {#description-2}

`RedisLockStore` 使用 Redis 作为持久层来维护对话锁。这是运行一组 Rasa 服务器副本的推荐锁存储。

### 配置 {#configuration-2}

要使用 Redis 设置 Rasa，需要执行如下步骤：

1. 启动 Redis 实例。
2. 将所需的配置添加到 `endpoints.yml` 中：

    ```yaml
    lock_store:
        type: "redis"
        url: <url of the redis instance, e.g. localhost>
        port: <port of your redis instance, usually 6379>
        password: <password used for authentication>
        db: <number of your database within redis, e.g. 0>
        key_prefix: <alphanumeric value to prepend to lock store keys>
    ```

3. 要使用 Redis 后端启动 Rasa Core 服务器，请添加 `--endpoints` 标识，例如：

    ```shell
    rasa run -m models --endpoints endpoints.yml
    ```

### 参数 {#parameters-1}

- `url`（默认：`localhost`）：Redis 实例 URL。
- `port`（默认：`6379`）：Redis 运行的端口。
- `db`（默认：`1`）：Redis 数据库的数量。
- `key_prefix`（默认：`None`）：锁存储键的前缀。必须是字母数字。
- `password`（默认：`None`）：用于认证的密码（`None` 表示不认证）。
- `use_ssl`（默认：`False`）：通信是否加密。
- `ssl_keyfile`（默认：`None`）：SSL 私钥路径。
- `ssl_certfile`（默认：`None`）：SSL 证书路径。
- `ssl_ca_certs`（默认：`None`）：PEM 格式的串联 CA 证书文件的路径。
- `socket_timeout`（默认：`10`）：超过此时间秒数 Redis 没有响应则引发错误。

## 自定义锁存储 {#custom-lock-store}

如果你需要一个无法开箱即用的锁存储，可以实现一个自定义的。通过扩展基类 `LockStore` 可以完成。

自定义锁存储类必须实现如下基类方法：

- `get_lock`：从存储中获取 `conversation_id` 的锁，需要 `conversation_id` 文本参数并返回 `TicketLock` 实例。（[源代码](https://github.com/RasaHQ/rasa/blob/main/rasa/core/lock_store.py#L59)）
- `save_lock`：提交锁对象到存储，需要类型为 `TicketLock` 的 `lock` 参数并返回 `None`。（[源代码](https://github.com/RasaHQ/rasa/blob/main/rasa/core/lock_store.py#L67)）
- `delete_lock`：从存储中删除 `conversation_id` 的锁，需要 `conversation_id` 文本参数并返回 `None`。（[源代码](https://github.com/RasaHQ/rasa/blob/main/rasa/core/lock_store.py#L63)）

### 配置 {#configuration-3}

将自定义锁存储的模块路径和所需的参数写入 `endpoints.yml` 中：

```yaml title="endpoints.yml"
lock_store:
  type: path.to.your.module.Class
  url: localhost
  a_parameter: a value
  another_parameter: another value
```
