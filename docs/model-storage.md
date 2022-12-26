# 模型存储

训练对话机器人后，模型可以存储在不同的位置。本页面解释了如何配置 Rasa 来加载你的模型。

你可以通过三种不同的方式加载经过训练的模型：

1. 从本地磁盘加载模型（请参见[从磁盘加载模型](/model-storage#load-model-from-disk)）
2. 从 HTTP 服务器获取模型（请参见[从服务器加载模型](/model-storage#load-model-from-server)）
3. 从 S3 等云存储中获取模型（请参见[从云处加载模型](/model-storage#load-model-from-cloud)）

默认情况下，Rasa CLI 的所有命令都会从本地磁盘加载模型。

## 从磁盘加载模型 {#load-model-from-disk}

默认情况下，模型将从本地磁盘加载。可以使用 `--model` 参数指定模型的路径：

```shell
rasa run --model models/20190506-100418.tar.gz
```

如果要在目录中加载最新模型，可以指定目录而不是文件：

```shell
rasa run --model models/
```

Rasa 将检查该目录中的所有模型并加载最近训练的模型。

如果不指定 `--model` 参数，Rasa 将在 `models/` 目录中查找模型。以下两个调用将加载相同的模型：

```shell
# this command will load the same model
rasa run --model models/
# ... as this command (using defaults)
rasa run
```

## 从服务器加载模型 {#load-model-from-server}

你可以将 Rasa 服务器配置为定期从服务器获取模型并进行部署。

### 如何配置 Rasa {#how-to-configure-rasa}

可以将 HTTP 服务器配置为从另一个 URL 获取模型，方式是将其添加到 `endpoints.yml` 中：

```yaml title="endpoints.yml"
models:
  url: http://my-server.com/models/default
  wait_time_between_pulls: 10   # In seconds, optional, default: 100
```

服务器将每隔 `wait_time_between_pulls` 秒查询压缩模型的 `url`。

如果只想在启动服务器时拉取模型，可以将拉取之间的时间设置为 `null`：

```yaml title="endpoints.yml"
models:
  url: http://my-server.com/models/default
  wait_time_between_pulls: null  # fetches model only once
```

### 如何配置服务器 {#how-to-configure-your-server}

Rasa 将向在 `endpoints.yml` 中指定的 URL 发送 `GET` 请求，例如上例中的 `http://my-server.com/models/default`。你可以使用任何 URL。`GET` 请求将包含一个 `If-None-Match` 的头，其中包含它下载的最后一个模型的哈希值。从开源 Rasa 到你的服务器的示例请求如下所示：

```shell
curl --header "If-None-Match: d41d8cd98f00b204e9800998ecf8427e" http://my-server.com/models/default
```

服务器对此 `GET` 请求的响应应该是如下之一：

- `200` 状态码和一个压缩的 Rasa 模型，并在模型哈希的响应中设置 `ETag` 头。
- `304` 状态码和空响应，如果请求的 `If-None-Match` 头与你希望模型返回的模型匹配。

Rasa 使用 `If-None-Match` 和 `ETag` 头进行缓存。设置头将避免一遍又一遍地重复下载相同的模型，从而节省带宽和计算资源。

## 从云处加载模型 {#load-model-from-cloud}

还可以配置 Rasa 服务器从远程存储中获取模型：

```shell
rasa run --model 20190506-100418.tar.gz --remote-storage aws
```

压缩后的模型将从云存储中下载、解压缩并部署。Rasa 支持从如下位置加载模型：

- 亚马逊 S3
- 谷歌云存储
- Azure 存储
- 其他自定义实现的远程存储

模型需要存储在存储服务的根文件夹中。目前无法手动指定云存储上的路径。

### 亚马逊 S3 存储 {#amazon-s3-storage}

使用 `boto3` 包来支持亚马逊 S3，需要使用 `pip3` 来安装附加依赖：

```shell
pip3 install boto3
```

为了让 Rasa 能够验证和下载模型，需要在运行任何需要存储的命令之前设置如下环境变量：

- `AWS_SECRET_ACCESS_KEY`：AWS S3 访问密钥。
- `AWS_ACCESS_KEY_ID`：AWS S3 访问密钥 ID。
- `AWS_DEFAULT_REGION`：AWS S3 存储桶区域。
- `BUCKET_NAME`：AWS S3 存储桶名称。
- `AWS_ENDPOINT_URL`：AWS S3 请求的完整 URL。需要指定一个完整的 URL（包括 http/https 协议）。

设置好所有环境变量后，可以将 `remote-storage` 选项设置为 `aws` 来启动 Rasa 服务器：

```shell
rasa run --model 20190506-100418.tar.gz --remote-storage aws
```

### 谷歌云存储 {#google-cloud-storage}

使用 `google-cloud-storage` 包支持谷歌云存储（GCS），需要使用 `pip3` 来安装附加依赖：

```shell
pip3 install google-cloud-storage
```

如果你在 Google App Engine 或 Compute Engine 上运行 Rasa，则已设置身份验证凭据（针对同一项目中的 GCS）。在这种情况下，可以跳过设置任何其他环境变量。

如果你在本地或 GAE 或 GCE 之外的其他机器上运行，只需要手动向 Rasa 提供身份验证详细信息：

1. 查看 [GCS 文档](https://cloud.google.com/docs/authentication/getting-started#auth-cloud-implicit-python)并按照“Creating a service account”和“Setting the environment variable”的说明进行操作。
2. 按照 GCS 说明完成后，应该将名为 `GOOGLE_APPLICATION_CREDENTIALS` 的环境变量设置为有权访问 GCS 的服务账号密钥文件的路径。

设置好所有环境变量后，可以将 `remote-storage` 选项设置为 `gcs` 来启动 Rasa 服务器：

```shell
rasa run --model 20190506-100418.tar.gz --remote-storage gcs
```

### Azure 存储 {#azure-storage}

使用 `azure-storage-blob` 包支持 Azure 存储，需要使用 `pip3` 来安装附加依赖：

```shell
pip3 install azure-storage-blob
```

为了让 Rasa 能够验证和下载模型，需要在运行任何需要存储的命令之前设置如下环境变量：

- `AZURE_CONTAINER`：Azure 容器名称。
- `AZURE_ACCOUNT_NAME`：Azure 帐户名称。
- `AZURE_ACCOUNT_KEY`：Azure 账户密钥。

设置好所有环境变量后，可以将 `remote-storage` 选项设置为 `azure` 来启动 Rasa 服务器：

```shell
rasa run --model 20190506-100418.tar.gz --remote-storage azure
```

### 其他远程存储 {#other-remote-storages}

如果你想使用任何其他云存储，可以提供自己的 [`rasa.nlu.persistor.Persistor`](https://rasa.com/docs/rasa/reference/rasa/nlu/persistor/) 类的 Python 实现。

将 `remote-storage` 设置为持久器实现的模块路径来启动 Rasa 服务器：

```shell
rasa run --remote-storage <your module>.<class name>
```
