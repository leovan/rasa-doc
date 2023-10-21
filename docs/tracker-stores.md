# 追踪器存储

你的对话机器人的对话存储在追踪器存储中。Rasa 提供开箱即用的不同存储类型的实现，或者你可以创建自定义的存储。

## InMemoryTrackerStore（默认） {#inmemorytrackerstore-default}

`InMemoryTrackerStore` 是默认的追踪器存储。如果没有配置其他追踪器存储，则默认使用它。它将对话历史存储在内存中。

!!! info "注意"

    由于此存储将所有历史记录保存在内存中，因此如果你重新启动 Rasa 服务器，则整个历史记录都会丢失。

### 配置 {#configuration}

使用 `InMemoryTrackerStore` 无需任何配置。

## SQLTrackerStore {#sqltrackerstore}

你可以使用 `SQLTrackerStore` 将对话机器人的对话历史存储在 SQL 数据库中。

### 配置 {#configuration-1}

要使用 SQL 设置 Rasa，需要执行如下步骤：

1. 将所需的配置添加到 `endpoints.yml` 中：

    ```yaml title="endpoints.yml"
    tracker_store:
        type: SQL
        dialect: "postgresql"  # the dialect used to interact with the db
        url: ""  # (optional) host of the sql db, e.g. "localhost"
        db: "rasa"  # path to your db
        username:  # username used for authentication
        password:  # password used for authentication
        query: # optional dictionary to be added as a query string to the connection URL
        driver: my-driver
    ```

2. 要使 Rasa 服务器使用 SQL 后端，请添加 `--endpoints` 标识，例如：

    ```shell
    rasa run -m models --endpoints endpoints.yml
    ```

3. 如果在 Docker Compose 中部署模型，请将服务添加到 `docker-compose.yml` 中：

    ```yaml title="docker-compose.yml"
    postgres:
    image: postgres:latest
    ```

4. 要将请求路由到新服务，请确保 `endpoints.yml` 中的 URL 引用服务名称：

    ```yaml title="endpoints.yml" hl_lines="4"
    tracker_store:
        type: SQL
        dialect: "postgresql"  # the dialect used to interact with the db
        url: "postgres"
        db: "rasa"  # path to your db
        username:  # username used for authentication
        password:  # password used for authentication
        query: # optional dictionary to be added as a query string to the connection URL
            driver: my-driver
    ```

#### 配置参数 {#configuration-parameters}

- `domain`（默认：`None`）：与此追踪器存储关联的领域对象。
- `dialect`（默认：`sqlite`）：用于与 SQL 后端通信的方言。有关可用的方言，请参见 [SQLAlchemy 文档](https://docs.sqlalchemy.org/en/latest/core/engines.html#database-urls)。
- `url`（默认：`None`）：SQL 服务器的 URL。
- `port`（默认：`None`）：SQL 服务器的端口。
- `db`（默认：`rasa.db`）：数据库路径。
- `username`（默认：`None`）：用于身份验证的用户名。
- `password`（默认：`None`）：用于身份验证的密码。
- `event_broker`（默认：`None`）：用于发布事件的事件代理。
- `login_db`（默认：`None`）：初始连接的备用数据库名称，并创建 `db` 指定的数据库（仅限于 PostgreSQL）。
- `query`（默认：`None`）：连接时要传递给方言和/或 DBAPI 的选项字典。

#### 兼容的数据库 {#compatible-databases}

如下数据库与 `SQLTrackerStore` 官方兼容：

- PostgreSQL
- Oracel > 11.0
- SQLite

#### 配置 Oracle {#configuring-oracle}

要将 SQLTrackerStore 与 Oracle 一起使用，还需要一些额外步骤。首先，在 Oracle 数据库中创建一个数据库追踪器并创建一个可以访问它的用户。使用如下命令在数据库中创建一个序列，其中 `username` 是创建的用户（有关创建序列的更多信息请参见 [Oracle 文档](https://docs.oracle.com/cd/B28359_01/server.111/b28310/views002.htm#ADMIN11794)）：

```sql
CREATE SEQUENCE username.events_seq;
```

接下来，你必须扩展 Rasa 镜像来包含必要的驱动程序和客户端。首先下载 Oracle Instant Client，将其重命名为 `oracle.rpm` 并将其存储在要构建 docker 镜像的目录中。将如下内容复制到名为 `Dockerfile` 的文件中：

```
FROM rasa/rasa:3.2.5-full

# Switch to root user to install packages
USER root

RUN apt-get update -qq && apt-get install -y --no-install-recommends alien libaio1 && apt-get clean && rm -rf /var/lib/apt/lists/* /tmp/* /var/tmp/*

# Copy in oracle instaclient
# https://www.oracle.com/database/technologies/instant-client/linux-x86-64-downloads.html
COPY oracle.rpm oracle.rpm

# Install the Python wrapper library for the Oracle drivers
RUN pip install cx-Oracle

# Install Oracle client libraries
RUN alien -i oracle.rpm

USER 1001
```

然后构建 docker 镜像：

```shell
docker build . -t rasa-oracle:3.2.5-oracle-full
```

现在可以如上述在 `endpoints.yml` 中配置追踪器存储，并启动容器了。此设置的方言参数为 `oracle+cx_oracle`。更多信息请参见[部署 Rasa 对话机器人](deploy/introduction.md)。

## RedisTrackerStore {#redistrackerstore}

可以使用 `RedisTrackerStore` 将对话机器人的历史记录存储在 [Redis](https://redis.io) 中。Redis 是一种快速的内存键值存储，其可选的也可以持久化数据。

### 配置 {#configuration-2}

要使用 Redis 设置 Rasa，需要执行以下步骤：

1. 启动 Redis 实例。
2. 将所需的配置添加到 `endpoints.yml` 中：

    ```yaml title="endpoints.yml"
    tracker_store:
        type: redis
        url: <url of the redis instance, e.g. localhost>
        port: <port of your redis instance, usually 6379>
        key_prefix: <alphanumeric value to prepend to tracker store keys>
        db: <number of your database within redis, e.g. 0>
        password: <password used for authentication>
        use_ssl: <whether or not the communication is encrypted, default `false`>
    ```

    ```shell
    rasa run -m models --endpoints endpoints.yml
    ```

3. 如果在 Docker Compose 中部署模型，请将服务添加到 `docker-compose.yml` 中：

    ```yaml title="docker-compose.yml"
    redis:
    image: redis:latest
    ```

    要将请求路由到新服务，请确保 `endpoints.yml` 中的 URL 引用服务名称：

    ```yml title="endpoints.yml" hl_lines="3"
    tracker_store:
        type: redis
        url: <url of the redis instance, e.g. localhost>
        port: <port of your redis instance, usually 6379>
        db: <number of your database within redis, e.g. 0>
        key_prefix: <alphanumeric value to prepend to tracker store keys>
        password: <password used for authentication>
        use_ssl: <whether or not the communication is encrypted, default `false`>
    ```

#### 配置参数 {#configuration-parameters-1}

- `url`（默认：`localhost`）：Redis 实例的 URL。
- `port`（默认：`6379`）：Redis 运行的端口。
- `db`（默认：`0`）：Redis 数据库的数量。
- `key_prefix`（默认：`None`）：添加到追踪器存储键的前缀。必须是字母数字。
- `username`（默认：`None`）：用于认证的用户名（`None` 表示不认证）。
- `password`（默认：`None`）：用于认证的密码（`None` 表示不认证）。
- `record_exp`（默认：`None`）：过期秒数。
- `use_ssl`（默认：`False`）：是否使用 SSL 进行传输加密。

## MongoTrackerStore {#mongotrackerstore}

可以使用 `MongoTrackerStore` 将对话机器人的对话历史存储在 [MongoDB](https://www.mongodb.com) 中。MongoDB 是一个免费开源的跨平台面向文档的 NoSQL 数据库。

### 配置 {#configuration-3}

1. 启动 MongoDB 实例。
2. 将所需的配置添加到 `endpoints.yml` 中：

    ```yaml title="endpoints.yml"
    tracker_store:
        type: mongod
        url: <url to your mongo instance, e.g. mongodb://localhost:27017>
        db: <name of the db within your mongo instance, e.g. rasa>
        username: <username used for authentication>
        password: <password used for authentication>
        auth_source: <database name associated with the user's credentials>
    ```

    还可以通过附加到 URL 的参数来添加更高级的配置（例如启用 SSL），例如：`mongodb://localhost:27017/?ssl=true`。

3. 要使用配置的 MongoDB 实例来启动 Rasa 服务器，请添加 `--endpoints` 标识，例如：

    ```shell
    rasa run -m models --endpoints endpoints.yml
    ```

4. 如果在 Docker Compose 中部署模型，请将服务添加到 `docker-compose.yml` 中：

    ```yaml title="docker-compose.yml"
    mongo:
    image: mongo
    environment:
        MONGO_INITDB_ROOT_USERNAME: rasa
        MONGO_INITDB_ROOT_PASSWORD: example
    mongo-express:  # this service is a MongoDB UI, and is optional
    image: mongo-express
    ports:
        - 8081:8081
    environment:
        ME_CONFIG_MONGODB_ADMINUSERNAME: rasa
        ME_CONFIG_MONGODB_ADMINPASSWORD: example
    ```

    要将请求路由到此数据库，请确保将 `endpoints.yml` 中的 `url` 设置为服务名称，并指定用户名和密码：

    ```yaml title="endpoints.yml"
    tracker_store:
        type: mongod
        url: mongodb://mongo:27017
        db: <name of the db within your mongo instance, e.g. rasa>
        username: <username used for authentication>
        password: <password used for authentication>
        auth_source: <database name associated with the user's credentials>
    ```

#### 配置参数 {#configuration-parameters-2}

- `url`（默认：`mongodb://localhost:27017`）：MongoDB 的 URL。
- `db`（默认：`rasa`）：数据库名称。
- `username`（默认：`0`）：用于身份验证的用户名。
- `password`（默认：`None`）：用于身份验证的密码。
- `auth_source`（默认：`admin`）：与用户凭据关联的数据库名称。
- `collection`（默认：`conversations`）：用于存储对话的集合名称。

## DynamoTrackerStore {#dynamotrackerstore}

可以使用 `DynamoTrackerStore` 将对话机器人的对话历史记录存储在 [DynamoDB](https://aws.amazon.com/dynamodb) 中。DynamoDB 是由 Amazon Web Services (AWS) 提供的托管 NoSQL 数据库。

### 配置 {#configuration-4}

1. 启动 DynamoDB 实例。
2. 将所需的配置添加到 `endpoints.yml` 中：

    ```yaml title="endpoints.yml"
    tracker_store:
        type: dynamo
        table_name: <name of the table to create, e.g. rasa>
        region: <name of the region associated with the client>
    ```

3. 要使用配置的 DynamoDB 实例启动 Rasa 服务器，请添加 `--endpoints` 标识，例如：

    ```shell
    rasa run -m models --endpoints endpoints.yml
    ```

#### 配置参数 {#configuration-parameters-3}

- `table_name`（默认值：`states`）：DynamoDB 表的名称。
- `region`（默认值：`us-east-1`）：与客户端关联的区域名称。

## 自定义追踪器存储 {#custom-tracker-store}

如果你需要一个无法开箱即用的追踪器存储，可以实现一个自定义的。通过扩展基类 `TrackerStore` 并提供实现 `serialise_tracker` 方法的混入类来完成：`SerializedTrackerAsText` 或 `SerializedTrackerAsDict`。

要编写自定义追踪器存储，需扩展 `TrackerStore` 基类。构造函数必须提供 `host` 参数。构造函数还需要使用 `domain` 和 `event_broker` 参数对基类 `TrackerStore` 调用 `super`。

```python
super().__init__(domain, event_broker, **kwargs)
```

自定义追踪器存储类还必须实现如下三个方法：

- `save`：将对话保存到追踪器存储。（[源代码](https://github.com/RasaHQ/rasa/blob/main/rasa/core/tracker_store.py#L243)）
- `retrieve`：检索最新对话会话追踪器。（[源代码](https://github.com/RasaHQ/rasa/blob/main/rasa/core/tracker_store.py#L261)）
- `keys`：返回追踪器存储主键的值的集合。（[源代码](https://github.com/RasaHQ/rasa/blob/main/rasa/core/tracker_store.py#L319)）

### 配置 {#configuration-5}

将模块路径放到自定义追踪器存储中，并将需要的参数放在 `endpoints.yml` 中：

```yaml title="endpoints.yml"
tracker_store:
  type: path.to.your.module.Class
  url: localhost
  a_parameter: a value
  another_parameter: another value
```

如果在 Docker Compose 中进行部署，有两种选项可以将此存储添加到 Rasa 中：扩展 Rasa 镜像来包含模块，或将模块装载为卷。

确保也添加相应的服务。例如，将其安装为卷如下所示：

```yaml title="docker-compose.yml" hl_lines="4 5"
rasa:
  <existing rasa service configuration>
  volumes:
    - <existing volume mappings, if there are any>
    - ./path/to/your/module.py:/app/path/to/your/module.py
custom-tracker-store:
  image: custom-image:tag
```

```yaml title="endpoints.yml" hl_lines="3"
tracker_store:
  type: path.to.your.module.Class
  url: custom-tracker-store
  a_parameter: a value
  another_parameter: another value
```

## 回退追踪存储

如果 `endpoints.yml` 中配置的主追踪存储不可用时，rasa 代理将发出错误信息并使用 `InMemoryTrackerStore`。每次都会启动一个新的对话会话，该对话会话将单独保存在 `InMemoryTrackerStore` 回退存储中。

一旦主追踪存储回复，它将取代回退追踪存储并保存从此刻开始的对话。但是请注意保存在 `InMemoryTrackerStore` 回退存储中的任何先前状态都将丢失。

!!! danger "使用同一个 Redis 作为锁存储和追踪器存储"

    不应该使用同一个 Redis 实例作为锁存储和追踪器存储。如果 Redis 实例不可用，会话将被挂起，因为没有为锁存储实线回退机制。
