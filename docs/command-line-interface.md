# 命令行界面

命令行界面（CLI）为你提供了易于记忆的常用任务命令。本页面描述了命令的行为以及需要传递的参数。

## Cheat Sheet {#cheat-sheet}

| 命令                    | 效果                                                         |
| ----------------------- | ------------------------------------------------------------ |
| `rasa init`             | 使用示例训练数据、动作和配置文件创建一个新项目。             |
| `rasa train`            | 根据你的 NLU 数据和故事训练一个模型，并保存至 `./models`。   |
| `rasa interactive`      | 通过与你的对话机器人聊天来启动交互式学习会话以创建新的训练数据。 |
| `rasa shell`            | 加载训练好的模型并让你可以通过命令行与对话机器人交谈。       |
| `rasa run`              | 使用训练好的模型启动服务。                                   |
| `rasa run actions`      | 使用 Rasa SDK 启动一个动作服务。                             |
| `rasa visualize`        | 生成故事的可视化表示。                                       |
| `rasa test`             | 在任意以 `test_` 开头的文件上测试训练好的 Rasa 模型。        |
| `rasa data split nlu`   | 对 NLU 训练数据进行 80/20 切分。                             |
| `rasa data convert`     | 转换不同测试的训练数据。                                     |
| `rasa data migrate`     | 将 2.0 的领域迁移到 3.0 格式。                               |
| `rasa data validate`    | 检查领域，NLU 和对话数据是否存在不一致。                     |
| `rasa export`           | 将对话从一个追踪器存储导出至一个事件代理。                   |
| `rasa evaluate markers` | 从一个现有的追踪器存储中提取标记。                           |
| `rasa -h`               | 显示所有可用的命令。                                         |

## 日志级别 {#log-level}

Rasa 可以生成不同级别的日志信息（例如：警告、信息、错误等）。使用 `--verbose`（与 `-v` 相同）或 `--debug`（与 `-vv` 相同）作为可选命令行参数来控制期望的日志级别。有关这些参数含义的更多解释，可以参见如下每个命令。

除了命令行参数外，还有若干环境变量可以用于更精细化地控制日志输出。通过这些环境变量，可以设置例如 Matplotlib、Pika 和 Kafka 外部库产生的日志。这些变量遵循 [Python 标准日志级别](https://docs.python.org/3/library/logging.html#logging-levels)。目前支持如下环境变量：

1. `LOG_LEVEL_LIBRARIES`：这是一个用于配置开源 Rasa 所用主要库日志级别的通用环境变量。包括 Tensorflow，`asyncio`，APScheduler，SocketIO，Matplotlib，RabbitMQ，Kafka。
2. `LOG_LEVEL_MATPLOTLIB`：这是一个专门用于配置 Matplotlib 日志级别的环境变量。
3. `LOG_LEVEL_RABBITMQ`：这是一个专门用于配置 AMQP 库日志级别的环境变量，目前可以处理 `aio_pika` 和 `aiormq` 的日志级别。
4. `LOG_LEVEL_KAFKA`：这是一个专门用于配置 kafka 日志级别的环境变量。

通用配置（`LOG_LEVEL_LIBRARIES`）的优先级相比于专用配置（`LOG_LEVEL_MATPLOTLIB`，`LOG_LEVEL_RABBITMQ` 等）更低。命令行参数设置了最低级别的日志。这意味着可以与这些变量一同使用，例如：

```shell
LOG_LEVEL_LIBRARIES=ERROR LOG_LEVEL_MATPLOTLIB=WARNING LOG_LEVEL_KAFKA=DEBUG rasa shell --debug
```

上述命令会产生如下效果：

- 默认处理比 `DEBUG` 级别更高的消息（由于 `--debug`）
- 对于 Matplotlib 库，处理比 `WARNING` 级别更高的消息
- 对于 kafka，处理比 `DEBUG` 级别更高的消息
- 对于其他未配置的库，处理比 `ERROR` 级别更高的消息

注意，命令行设置需要处理的最低级别日志消息，如下命令会将日志级别设置为 `INFO`（由于 `--verbose`），因此不会有任何调试信息展示（对库级别配置不会产生任何影响）：

```shell
LOG_LEVEL_LIBRARIES=DEBUG LOG_LEVEL_MATPLOTLIB=DEBUG rasa shell --verbose
```

命令行日志级别设置了根日志的级别（它包含了重要的 `coloredlogs` 处理器）。这意味着即使环境变量设置一个库日志较低的日志级别，根日志也会拒绝来自该库的消息。如果没有指定，命令行日志级别将被设置为 `INFO`。

## rasa init

该命令将会利用一些示例训练数据来配置一个完整的对话机器人：

```shell
rasa init
```

这将创建如下文件：

```
.
├── actions
│   ├── __init__.py
│   └── actions.py
├── config.yml
├── credentials.yml
├── data
│   ├── nlu.yml
│   └── stories.yml
├── domain.yml
├── endpoints.yml
├── models
│   └── <timestamp>.tar.gz
└── tests
   └── test_stories.yml
```

会询问是否使用此数据训练初始模型，如果回答为否，则 `models` 目录将为空。

任何模型的命令行参数均需要这个项目设置，因此这是开始的最佳方式。无需任何额外的配置即可运行 `rasa train`，`rasa shell` 和 `rasa test` 命令。

## rasa train

如下命令将会训练一个开源 Rasa 模型：

```shell
rasa train
```

如果在目录中已经存在模型（默认在 `models/` 下），只有修改过的模型才会被重新训练。例如，如果你只修改了 NLU 训练数据，则只有 NLU 部分会被训练。

如果希望单独训练 NLU 或对话模型，需要运行 `rasa train nlu` 或 `rasa train core`。如果只为其中之一提供了训练数据，则 `rasa train` 将会默认回退到这些命令之一。

`rasa train` 会将模型存储在 `--out` 定义的目录中，默认为 `models/`。模型的默认名称为 `<timestamp>.tar.gz`。如果想以不同的方式命名模型，可以通过 `--fixed-model-name` 参数来指定名称。

如下参数可用于设置训练过程：

```
usage: rasa train [-h] [-v] [-vv] [--quiet] [--data DATA [DATA ...]]
                  [-c CONFIG] [-d DOMAIN] [--out OUT] [--dry-run]
                  [--augmentation AUGMENTATION] [--debug-plots]
                  [--num-threads NUM_THREADS]
                  [--fixed-model-name FIXED_MODEL_NAME] [--persist-nlu-data]
                  [--force] [--finetune [FINETUNE]]
                  [--epoch-fraction EPOCH_FRACTION] [--endpoints ENDPOINTS]
                  {core,nlu} ...

positional arguments:
  {core,nlu}
    core                Trains a Rasa Core model using your stories.
    nlu                 Trains a Rasa NLU model using your NLU data.

optional arguments:
  -h, --help            show this help message and exit
  --data DATA [DATA ...]
                        Paths to the Core and NLU data files. (default:
                        ['data'])
  -c CONFIG, --config CONFIG
                        The policy and NLU pipeline configuration of your bot.
                        (default: config.yml)
  -d DOMAIN, --domain DOMAIN
                        Domain specification. This can be a single YAML file,
                        or a directory that contains several files with domain
                        specifications in it. The content of these files will
                        be read and merged together. (default: domain.yml)
  --out OUT             Directory where your models should be stored.
                        (default: models)
  --dry-run             If enabled, no actual training will be performed.
                        Instead, it will be determined whether a model should
                        be re-trained and this information will be printed as
                        the output. The return code is a 4-bit bitmask that
                        can also be used to determine what exactly needs to be
                        retrained: - 0 means that no extensive training is
                        required (note that the responses still might require
                        updating by running 'rasa train'). - 1 means the model
                        needs to be retrained - 8 means the training was
                        forced (--force argument is specified) (default:
                        False)
  --augmentation AUGMENTATION
                        How much data augmentation to use during training.
                        (default: 50)
  --debug-plots         If enabled, will create plots showing checkpoints and
                        their connections between story blocks in a file
                        called `story_blocks_connections.html`. (default:
                        False)
  --num-threads NUM_THREADS
                        Maximum amount of threads to use when training.
                        (default: None)
  --fixed-model-name FIXED_MODEL_NAME
                        If set, the name of the model file/directory will be
                        set to the given name. (default: None)
  --persist-nlu-data    Persist the NLU training data in the saved model.
                        (default: False)
  --force               Force a model training even if the data has not
                        changed. (default: False)
  --finetune [FINETUNE]
                        Fine-tune a previously trained model. If no model path
                        is provided, Rasa Open Source will try to finetune the
                        latest trained model from the model directory
                        specified via '--out'. (default: None)
  --epoch-fraction EPOCH_FRACTION
                        Fraction of epochs which are currently specified in
                        the model configuration which should be used when
                        finetuning a model. (default: None)
  --endpoints ENDPOINTS
                        Configuration file for the connectors as a yml file.
                        (default: endpoints.yml)

Python Logging Options:
  You can control level of log messages printed. In addition to these
  arguments, a more fine grained configuration can be achieved with
  environment variables. See online documentation for more info.

  -v, --verbose         Be verbose. Sets logging level to INFO. (default:
                        None)
  -vv, --debug          Print lots of debugging statements. Sets logging level
                        to DEBUG. (default: None)
  --quiet               Be quiet! Sets logging level to WARNING. (default:
                        None)
```

有关数据增强如何工作以及如果为参数设置值可以参阅[数据增强](/policies#data-augmentation)部分。注意 `TEDPilicy` 是唯一受到数据增强影响的策略。

有关 `--epoch-fraction` 参数的更多信息，请参见如下有关[增量训练](#incremental-training)的部分。

### 增量训练 {#incremental-training}

!!! info "2.2 版本新增内容"

    此功能是实验性的。我们通过社区反馈引入了这一实验性功能，因此鼓励大家进行尝试。但是，相关功在未来可能会发生变更或删除。如果有任何正面或负面反馈，可以在[Rasa 论坛](https://forum.rasa.com/)上进行分享。

为了提高对话机器人的性能，践行 [CDD](/conversation-driven-development) 和根据用户与机器人交谈的方式增加新的训练样本会很有帮助。通过 `rasa train --finetune` 可以使用已训练的模型来初始化整个流程，之后通过包含额外训练样本的新训练数据进行微调。这有助于减少训练新模型的时间。

默认情况下，命令会选择 `models/` 目录中的最新模型。如果你希望改进特定的模型，则需要通过执行 `rasa train --finetune <path to model to finetune>` 来指定路径。微调模型通常在训练机器学习组件例如：`DIETClassifier`，`ResponseSelector` 和 `TEDPolicy` 时相比从头开始需要更少的时间。要么使用定义更少批次的模型微调配置，要么使用 `--epoch-fraction` 参数。在模型配置文件中，`--epoch-fraction` 会为每个机器学习组件指定部分批次。例如，训练 `DIETClassifier` 配置使用 100 个批次，指定 `--epoch-fraction 0.5` 将仅使用 50 个批次进行微调。

通过 `rasa train nlu --finetune` 和 `rasa train core --finetune` 可以分别仅对 NLU 或对话管理模型进行微调。

为了能够对一个模型进行微调，必须满足如下条件：

1. 用于训练模型的配置应该与用于微调模型的配置完全相同。唯一可以修改的参数就是用于各个机器学习组件和策略的 `epochs`。
2. 用于基础模型训练的标签集（意图、动作、实体和槽）应该与用于模型微调训练数据中的标签集完全相同。这意味着在进行增量训练的过程中不能添加新的意图、动作、实体或槽标签。可以为每个现有标签添加新的训练样本。如果在训练数据中添加或删除标签，则需要从头开始训练。
3. 要微调的模型会使用当前安装版本 Rasa 的 `MINIMUM_COMPATIBLE_VERSION` 版本进行训练。

## rasa interactive

通过如下命令可以开启一个交互式学习会话：

```shell
rasa interactive
```

这将首先训练一个模型，然后启动一个交互式 Shell 会话。之后随着同对话机器人的交谈可以对其预测进行不断修正。如果 [UnexpecTEDIntentPolicy](/policies#unexpected-intent-policy) 包含在流程中，则可以在任意对话抡次中触发 [`action_unlikely_intent`](/default-actions#action_unlikely_intent)，之后会显示：

```
The bot wants to run 'action_unlikely_intent' to indicate that the last user message was unexpected
at this point in the conversation. Check out UnexpecTEDIntentPolicy docs to learn more.
```

正如消息所述，这表示你尝试了一个不在当前的训练故事集预期的一个对话路径，因此建议将该路径添加到训练故事中。同其他对话机器人响应一样，你可以选择接受或拒绝此操作。

如果使用 `--model` 参数提供一个训练好的模型，则会跳过训练过程直接加载这个模型。

在交互式学习过程中，Rasa 将会可视化当前对话和训练数据中的一些相似对话，来帮助你跟踪当前的进度。会话开始后，可以在 http://localhost:5005/visualization.html 中查看相关可视化。图表生成需要一些时间。运行 `rasa interactive --skip-visualization` 可以跳过可视化。

如下参数可用于配置交互式学习会话：

```
usage: rasa interactive [-h] [-v] [-vv] [--quiet] [--e2e] [-p PORT] [-m MODEL]
                        [--data DATA [DATA ...]] [--skip-visualization]
                        [--conversation-id CONVERSATION_ID]
                        [--endpoints ENDPOINTS] [-c CONFIG] [-d DOMAIN]
                        [--out OUT] [--augmentation AUGMENTATION]
                        [--debug-plots] [--finetune [FINETUNE]]
                        [--epoch-fraction EPOCH_FRACTION] [--force]
                        [--persist-nlu-data]
                        {core} ... [model-as-positional-argument]

positional arguments:
  {core}
    core                Starts an interactive learning session model to create
                        new training data for a Rasa Core model by chatting.
                        Uses the 'RegexMessageHandler', i.e. `/<intent>` input
                        format.
  model-as-positional-argument
                        Path to a trained Rasa model. If a directory is
                        specified, it will use the latest model in this
                        directory. (default: None)

optional arguments:
  -h, --help            show this help message and exit
  --e2e                 Save story files in e2e format. In this format user
                        messages will be included in the stories. (default:
                        False)
  -p PORT, --port PORT  Port to run the server at. (default: 5005)
  -m MODEL, --model MODEL
                        Path to a trained Rasa model. If a directory is
                        specified, it will use the latest model in this
                        directory. (default: None)
  --data DATA [DATA ...]
                        Paths to the Core and NLU data files. (default:
                        ['data'])
  --skip-visualization  Disable plotting the visualization during interactive
                        learning. (default: False)
  --conversation-id CONVERSATION_ID
                        Specify the id of the conversation the messages are
                        in. Defaults to a UUID that will be randomly
                        generated. (default: ddeafee8cf6749edb0a75cd7c36b39ac)
  --endpoints ENDPOINTS
                        Configuration file for the model server and the
                        connectors as a yml file. (default: endpoints.yml)

Python Logging Options:
  You can control level of log messages printed. In addition to these
  arguments, a more fine grained configuration can be achieved with
  environment variables. See online documentation for more info.

  -v, --verbose         Be verbose. Sets logging level to INFO. (default:
                        None)
  -vv, --debug          Print lots of debugging statements. Sets logging level
                        to DEBUG. (default: None)
  --quiet               Be quiet! Sets logging level to WARNING. (default:
                        None)

Train Arguments:
  -c CONFIG, --config CONFIG
                        The policy and NLU pipeline configuration of your bot.
                        (default: config.yml)
  -d DOMAIN, --domain DOMAIN
                        Domain specification. This can be a single YAML file,
                        or a directory that contains several files with domain
                        specifications in it. The content of these files will
                        be read and merged together. (default: domain.yml)
  --out OUT             Directory where your models should be stored.
                        (default: models)
  --augmentation AUGMENTATION
                        How much data augmentation to use during training.
                        (default: 50)
  --debug-plots         If enabled, will create plots showing checkpoints and
                        their connections between story blocks in a file
                        called `story_blocks_connections.html`. (default:
                        False)
  --finetune [FINETUNE]
                        Fine-tune a previously trained model. If no model path
                        is provided, Rasa Open Source will try to finetune the
                        latest trained model from the model directory
                        specified via '--out'. (default: None)
  --epoch-fraction EPOCH_FRACTION
                        Fraction of epochs which are currently specified in
                        the model configuration which should be used when
                        finetuning a model. (default: None)
  --force               Force a model training even if the data has not
                        changed. (default: False)
  --persist-nlu-data    Persist the NLU training data in the saved model.
                        (default: False)
```

## rasa shell

运行如下命令可以启动一个聊天会话：

```shell
rasa shell
```

默认情况下会加载最新的训练模型。你也可以通过 `--model` 参数指定要加载的其他模型。

如果仅使用 NLU 模型启动 shell，`rasa shell` 将输出输入消息的预测意图和实体。

如果已经训练了一个组合 Rasa 模型，但你只想看模型从文本中提取的意图和实体，可以使用 `rasa shell nlu`。

要提高调试的日志记录级别，可以运行：

```shell
rasa shell --debug
```

!!! note "注意"

    为了在外部频道中查看问候语和会话起始行为，你可以通过显式发送 `/session_start` 作为第一条消息。否则，会话起始行为将按照[会话配置](/domain#session-configuration)中的描述开始。

如下参数可以用于配置此命令。大部分参数同 `rasa run` 相同。有关这些参数的更多信息，请参见[如下部分](#rasa-run)。

注意，在运行 `rasa shell` 时，`--connector` 参数将会始终被设置为 `cmdline`。这意味着 credentials 文件中的所有 credentials 都会被忽略，如果你为 `--connector` 参数指定了值，也将被忽略。

```
usage: rasa shell [-h] [-v] [-vv] [--quiet]
                  [--conversation-id CONVERSATION_ID] [-m MODEL]
                  [--log-file LOG_FILE] [--use-syslog]
                  [--syslog-address SYSLOG_ADDRESS]
                  [--syslog-port SYSLOG_PORT]
                  [--syslog-protocol SYSLOG_PROTOCOL] [--endpoints ENDPOINTS]
                  [-i INTERFACE] [-p PORT] [-t AUTH_TOKEN] [--cors [CORS ...]]
                  [--enable-api] [--response-timeout RESPONSE_TIMEOUT]
                  [--request-timeout REQUEST_TIMEOUT]
                  [--remote-storage REMOTE_STORAGE]
                  [--ssl-certificate SSL_CERTIFICATE]
                  [--ssl-keyfile SSL_KEYFILE] [--ssl-ca-file SSL_CA_FILE]
                  [--ssl-password SSL_PASSWORD] [--credentials CREDENTIALS]
                  [--connector CONNECTOR] [--jwt-secret JWT_SECRET]
                  [--jwt-method JWT_METHOD]
                  {nlu} ... [model-as-positional-argument]

positional arguments:
  {nlu}
    nlu                 Interprets messages on the command line using your NLU
                        model.
  model-as-positional-argument
                        Path to a trained Rasa model. If a directory is
                        specified, it will use the latest model in this
                        directory. (default: None)

optional arguments:
  -h, --help            show this help message and exit
  --conversation-id CONVERSATION_ID
                        Set the conversation ID. (default:
                        78ccd7a0e719424ca7e910cdc99d1918)
  -m MODEL, --model MODEL
                        Path to a trained Rasa model. If a directory is
                        specified, it will use the latest model in this
                        directory. (default: models)
  --log-file LOG_FILE   Store logs in specified file. (default: None)
  --use-syslog          Add syslog as a log handler (default: False)
  --syslog-address SYSLOG_ADDRESS
                        Address of the syslog server. --use-sylog flag is
                        required (default: localhost)
  --syslog-port SYSLOG_PORT
                        Port of the syslog server. --use-sylog flag is
                        required (default: 514)
  --syslog-protocol SYSLOG_PROTOCOL
                        Protocol used with the syslog server. Can be UDP
                        (default) or TCP (default: UDP)
  --endpoints ENDPOINTS
                        Configuration file for the model server and the
                        connectors as a yml file. (default: endpoints.yml)

Python Logging Options:
  You can control level of log messages printed. In addition to these
  arguments, a more fine grained configuration can be achieved with
  environment variables. See online documentation for more info.

  -v, --verbose         Be verbose. Sets logging level to INFO. (default:
                        None)
  -vv, --debug          Print lots of debugging statements. Sets logging level
                        to DEBUG. (default: None)
  --quiet               Be quiet! Sets logging level to WARNING. (default:
                        None)

Server Settings:
  -i INTERFACE, --interface INTERFACE
                        Network interface to run the server on. (default:
                        0.0.0.0)
  -p PORT, --port PORT  Port to run the server at. (default: 5005)
  -t AUTH_TOKEN, --auth-token AUTH_TOKEN
                        Enable token based authentication. Requests need to
                        provide the token to be accepted. (default: None)
  --cors [CORS ...]     Enable CORS for the passed origin. Use * to whitelist
                        all origins. (default: None)
  --enable-api          Start the web server API in addition to the input
                        channel. (default: False)
  --response-timeout RESPONSE_TIMEOUT
                        Maximum time a response can take to process (sec).
                        (default: 3600)
  --request-timeout REQUEST_TIMEOUT
                        Maximum time a request can take to process (sec).
                        (default: 300)
  --remote-storage REMOTE_STORAGE
                        Set the remote location where your Rasa model is
                        stored, e.g. on AWS. (default: None)
  --ssl-certificate SSL_CERTIFICATE
                        Set the SSL Certificate to create a TLS secured
                        server. (default: None)
  --ssl-keyfile SSL_KEYFILE
                        Set the SSL Keyfile to create a TLS secured server.
                        (default: None)
  --ssl-ca-file SSL_CA_FILE
                        If your SSL certificate needs to be verified, you can
                        specify the CA file using this parameter. (default:
                        None)
  --ssl-password SSL_PASSWORD
                        If your ssl-keyfile is protected by a password, you
                        can specify it using this paramer. (default: None)

Channels:
  --credentials CREDENTIALS
                        Authentication credentials for the connector as a yml
                        file. (default: None)
  --connector CONNECTOR
                        Service to connect to. (default: None)

JWT Authentication:
  --jwt-secret JWT_SECRET
                        Public key for asymmetric JWT methods or shared
                        secretfor symmetric methods. Please also make sure to
                        use --jwt-method to select the method of the
                        signature, otherwise this argument will be
                        ignored.Note that this key is meant for securing the
                        HTTP API. (default: None)
  --jwt-method JWT_METHOD
                        Method used for the signature of the JWT
                        authentication payload. (default: HS256)
```

## rasa run

运行如下命令可以为训练好的模型启动一个服务：

```shell
rasa run
```

默认情况下，Rasa 服务使用 HTTP 进行通信。要使用 SSL 加密通讯并在 HTTPS 上运行服务，你需要提供一个有效的证书和对应的私钥文件。可以在 `rasa run` 命令中指定这些文件。如果在创建密钥文件过程中使用密码进行了加密，则还需要添加 `--ssl-passowrd` 参数。

```shell
rasa run --ssl-certificate myssl.crt --ssl-keyfile myssl.key --ssl-password mypassword
```

Rasa 模型监听每个可用的网络接口。通过 `-i` 命令行参数可以将其限制为特定的网络接口。

```shell
rasa run -i 192.168.69.150
```

Rasa 默认会连接到 credentials 文件中指定的所有频道。在 `--connector` 参数中指定频道的名称，可以仅连接一个频道并忽略 credentials 文件中的所有频道。

```shell
rasa run --connector rest
```

频道的名称需要同 credentials 文件中指定的名称相匹配。关于支持的频道请参见[消息和语音频道](/messaging-and-voice-channels)。

如下参数可用于配置 Rasa 服务：

```
usage: rasa run [-h] [-v] [-vv] [--quiet] [-m MODEL] [--log-file LOG_FILE]
                [--use-syslog] [--syslog-address SYSLOG_ADDRESS]
                [--syslog-port SYSLOG_PORT]
                [--syslog-protocol SYSLOG_PROTOCOL] [--endpoints ENDPOINTS]
                [-i INTERFACE] [-p PORT] [-t AUTH_TOKEN] [--cors [CORS ...]]
                [--enable-api] [--response-timeout RESPONSE_TIMEOUT]
                [--request-timeout REQUEST_TIMEOUT]
                [--remote-storage REMOTE_STORAGE]
                [--ssl-certificate SSL_CERTIFICATE]
                [--ssl-keyfile SSL_KEYFILE] [--ssl-ca-file SSL_CA_FILE]
                [--ssl-password SSL_PASSWORD] [--credentials CREDENTIALS]
                [--connector CONNECTOR] [--jwt-secret JWT_SECRET]
                [--jwt-method JWT_METHOD]
                {actions} ... [model-as-positional-argument]

positional arguments:
  {actions}
    actions             Runs the action server.
  model-as-positional-argument
                        Path to a trained Rasa model. If a directory is
                        specified, it will use the latest model in this
                        directory. (default: None)

optional arguments:
  -h, --help            show this help message and exit
  -m MODEL, --model MODEL
                        Path to a trained Rasa model. If a directory is
                        specified, it will use the latest model in this
                        directory. (default: models)
  --log-file LOG_FILE   Store logs in specified file. (default: None)
  --use-syslog          Add syslog as a log handler (default: False)
  --syslog-address SYSLOG_ADDRESS
                        Address of the syslog server. --use-sylog flag is
                        required (default: localhost)
  --syslog-port SYSLOG_PORT
                        Port of the syslog server. --use-sylog flag is
                        required (default: 514)
  --syslog-protocol SYSLOG_PROTOCOL
                        Protocol used with the syslog server. Can be UDP
                        (default) or TCP (default: UDP)
  --endpoints ENDPOINTS
                        Configuration file for the model server and the
                        connectors as a yml file. (default: endpoints.yml)

Python Logging Options:
  You can control level of log messages printed. In addition to these
  arguments, a more fine grained configuration can be achieved with
  environment variables. See online documentation for more info.

  -v, --verbose         Be verbose. Sets logging level to INFO. (default:
                        None)
  -vv, --debug          Print lots of debugging statements. Sets logging level
                        to DEBUG. (default: None)
  --quiet               Be quiet! Sets logging level to WARNING. (default:
                        None)

Server Settings:
  -i INTERFACE, --interface INTERFACE
                        Network interface to run the server on. (default:
                        0.0.0.0)
  -p PORT, --port PORT  Port to run the server at. (default: 5005)
  -t AUTH_TOKEN, --auth-token AUTH_TOKEN
                        Enable token based authentication. Requests need to
                        provide the token to be accepted. (default: None)
  --cors [CORS ...]     Enable CORS for the passed origin. Use * to whitelist
                        all origins. (default: None)
  --enable-api          Start the web server API in addition to the input
                        channel. (default: False)
  --response-timeout RESPONSE_TIMEOUT
                        Maximum time a response can take to process (sec).
                        (default: 3600)
  --request-timeout REQUEST_TIMEOUT
                        Maximum time a request can take to process (sec).
                        (default: 300)
  --remote-storage REMOTE_STORAGE
                        Set the remote location where your Rasa model is
                        stored, e.g. on AWS. (default: None)
  --ssl-certificate SSL_CERTIFICATE
                        Set the SSL Certificate to create a TLS secured
                        server. (default: None)
  --ssl-keyfile SSL_KEYFILE
                        Set the SSL Keyfile to create a TLS secured server.
                        (default: None)
  --ssl-ca-file SSL_CA_FILE
                        If your SSL certificate needs to be verified, you can
                        specify the CA file using this parameter. (default:
                        None)
  --ssl-password SSL_PASSWORD
                        If your ssl-keyfile is protected by a password, you
                        can specify it using this paramer. (default: None)

Channels:
  --credentials CREDENTIALS
                        Authentication credentials for the connector as a yml
                        file. (default: None)
  --connector CONNECTOR
                        Service to connect to. (default: None)

JWT Authentication:
  --jwt-secret JWT_SECRET
                        Public key for asymmetric JWT methods or shared
                        secretfor symmetric methods. Please also make sure to
                        use --jwt-method to select the method of the
                        signature, otherwise this argument will be
                        ignored.Note that this key is meant for securing the
                        HTTP API. (default: None)
  --jwt-method JWT_METHOD
                        Method used for the signature of the JWT
                        authentication payload. (default: HS256)
```

有关其他参数的更多信息，请参见[模型存储](/model-storage)。有关所有服务 API 的详细文档，请参见 Rasa [HTTP API](/http-api) 页面。

## rasa run actions

运行如下命令可以使用 Rasa SDK 启动动作服务：

```shell
rasa run actions
```

如下参数可用于调整服务设置：

```
usage: rasa run actions [-h] [-v] [-vv] [--quiet] [-p PORT]
                        [--cors [CORS ...]] [--actions ACTIONS]
                        [--ssl-keyfile SSL_KEYFILE]
                        [--ssl-certificate SSL_CERTIFICATE]
                        [--ssl-password SSL_PASSWORD] [--auto-reload]

optional arguments:
  -h, --help            show this help message and exit
  -p PORT, --port PORT  port to run the server at (default: 5055)
  --cors [CORS ...]     enable CORS for the passed origin. Use * to whitelist
                        all origins (default: None)
  --actions ACTIONS     name of action package to be loaded (default: None)
  --ssl-keyfile SSL_KEYFILE
                        Set the SSL certificate to create a TLS secured
                        server. (default: None)
  --ssl-certificate SSL_CERTIFICATE
                        Set the SSL certificate to create a TLS secured
                        server. (default: None)
  --ssl-password SSL_PASSWORD
                        If your ssl-keyfile is protected by a password, you
                        can specify it using this paramer. (default: None)
  --auto-reload         Enable auto-reloading of modules containing Action
                        subclasses. (default: False)

Python Logging Options:
  You can control level of log messages printed. In addition to these
  arguments, a more fine grained configuration can be achieved with
  environment variables. See online documentation for more info.

  -v, --verbose         Be verbose. Sets logging level to INFO. (default:
                        None)
  -vv, --debug          Print lots of debugging statements. Sets logging level
                        to DEBUG. (default: None)
  --quiet               Be quiet! Sets logging level to WARNING. (default:
                        None)
```

## rasa visualize

运行如下命令可以在浏览器中可视化故事：

```shell
rasa visualize
```

如果故事存储于默认位置 `data/` 以外的位置，可以通过 `--stories` 指定它们的位置。

如下参数可以用于配置此命令：

```
usage: rasa visualize [-h] [-v] [-vv] [--quiet] [-d DOMAIN] [-s STORIES]
                      [--out OUT] [--max-history MAX_HISTORY] [-u NLU]

optional arguments:
  -h, --help            show this help message and exit
  -d DOMAIN, --domain DOMAIN
                        Domain specification. This can be a single YAML file,
                        or a directory that contains several files with domain
                        specifications in it. The content of these files will
                        be read and merged together. (default: domain.yml)
  -s STORIES, --stories STORIES
                        File or folder containing your training stories.
                        (default: data)
  --out OUT             Filename of the output path, e.g. 'graph.html'.
                        (default: graph.html)
  --max-history MAX_HISTORY
                        Max history to consider when merging paths in the
                        output graph. (default: 2)
  -u NLU, --nlu NLU     File or folder containing your NLU data, used to
                        insert example messages into the graph. (default:
                        None)

Python Logging Options:
  You can control level of log messages printed. In addition to these
  arguments, a more fine grained configuration can be achieved with
  environment variables. See online documentation for more info.

  -v, --verbose         Be verbose. Sets logging level to INFO. (default:
                        None)
  -vv, --debug          Print lots of debugging statements. Sets logging level
                        to DEBUG. (default: None)
  --quiet               Be quiet! Sets logging level to WARNING. (default:
                        None)
```

## rasa test

运行如下命令可以利用测试数据评估模型：

```shell
rasa test
```

这会在以 `text_` 开头的文件中定义的故事上端到端的测试最新训练的模型。通过指定 `--model` 参数可以选择一个不同的模型。

如果想要分开评估对话和 NLU 模型，可以运行如下命令：

```shell
rasa test core
```

和

```shell
rasa rest nlu
```

你可以在[评估 NLU 模型](/testing-your-assistant#evaluating-an-nlu-model)和[评估对话管理模型](/testing-your-assistant#evaluating-a-dialogue-model)中找到有关每种测试类型参数的更多信息。

如下参数可以用于 `rasa test`：

```
usage: rasa test [-h] [-v] [-vv] [--quiet] [-m MODEL] [-s STORIES]
                 [--max-stories MAX_STORIES] [--endpoints ENDPOINTS]
                 [--fail-on-prediction-errors] [--url URL]
                 [--evaluate-model-directory] [-u NLU]
                 [-c CONFIG [CONFIG ...]] [-d DOMAIN] [--cross-validation]
                 [-f FOLDS] [-r RUNS] [-p PERCENTAGES [PERCENTAGES ...]]
                 [--no-plot] [--successes] [--no-errors] [--no-warnings]
                 [--out OUT]
                 {core,nlu} ...

positional arguments:
  {core,nlu}
    core                Tests Rasa Core models using your test stories.
    nlu                 Tests Rasa NLU models using your test NLU data.

optional arguments:
  -h, --help            show this help message and exit
  -m MODEL, --model MODEL
                        Path to a trained Rasa model. If a directory is
                        specified, it will use the latest model in this
                        directory. (default: models)
  --no-plot             Don't render evaluation plots. (default: False)
  --successes           If set successful predictions will be written to a
                        file. (default: False)
  --no-errors           If set incorrect predictions will NOT be written to a
                        file. (default: False)
  --no-warnings         If set prediction warnings will NOT be written to a
                        file. (default: False)
  --out OUT             Output path for any files created during the
                        evaluation. (default: results)

Python Logging Options:
  You can control level of log messages printed. In addition to these
  arguments, a more fine grained configuration can be achieved with
  environment variables. See online documentation for more info.

  -v, --verbose         Be verbose. Sets logging level to INFO. (default:
                        None)
  -vv, --debug          Print lots of debugging statements. Sets logging level
                        to DEBUG. (default: None)
  --quiet               Be quiet! Sets logging level to WARNING. (default:
                        None)

Core Test Arguments:
  -s STORIES, --stories STORIES
                        File or folder containing your test stories. (default:
                        .)
  --max-stories MAX_STORIES
                        Maximum number of stories to test on. (default: None)
  --endpoints ENDPOINTS
                        Configuration file for the connectors as a yml file.
                        (default: endpoints.yml)
  --fail-on-prediction-errors
                        If a prediction error is encountered, an exception is
                        thrown. This can be used to validate stories during
                        tests, e.g. on travis. (default: False)
  --url URL             If supplied, downloads a story file from a URL and
                        trains on it. Fetches the data by sending a GET
                        request to the supplied URL. (default: None)
  --evaluate-model-directory
                        Should be set to evaluate models trained via 'rasa
                        train core --config <config-1> <config-2>'. All models
                        in the provided directory are evaluated and compared
                        against each other. (default: False)

NLU Test Arguments:
  -u NLU, --nlu NLU     File or folder containing your NLU data. (default:
                        data)
  -c CONFIG [CONFIG ...], --config CONFIG [CONFIG ...]
                        Model configuration file. If a single file is passed
                        and cross validation mode is chosen, cross-validation
                        is performed, if multiple configs or a folder of
                        configs are passed, models will be trained and
                        compared directly. (default: None)
  -d DOMAIN, --domain DOMAIN
                        Domain specification. This can be a single YAML file,
                        or a directory that contains several files with domain
                        specifications in it. The content of these files will
                        be read and merged together. (default: domain.yml)
```

## rasa data split

运行如下命令可以对 NLU 训练数据进行拆分：

```shell
rasa data split
```

默认情况下，这将对数据进行 80/20 拆分为训练集和测试集。通过如下参数可以指定训练数据、比例和输出目录：

```
usage: rasa data split nlu [-h] [-v] [-vv] [--quiet] [-u NLU]
                           [--training-fraction TRAINING_FRACTION]
                           [--random-seed RANDOM_SEED] [--out OUT]

optional arguments:
  -h, --help            show this help message and exit
  -u NLU, --nlu NLU     File or folder containing your NLU data. (default:
                        data)
  --training-fraction TRAINING_FRACTION
                        Percentage of the data which should be in the training
                        data. (default: 0.8)
  --random-seed RANDOM_SEED
                        Seed to generate the same train/test split. (default:
                        None)
  --out OUT             Directory where the split files should be stored.
                        (default: train_test_split)

Python Logging Options:
  You can control level of log messages printed. In addition to these
  arguments, a more fine grained configuration can be achieved with
  environment variables. See online documentation for more info.

  -v, --verbose         Be verbose. Sets logging level to INFO. (default:
                        None)
  -vv, --debug          Print lots of debugging statements. Sets logging level
                        to DEBUG. (default: None)
  --quiet               Be quiet! Sets logging level to WARNING. (default:
                        None)
```

如果有用于检索动作的 NLG 数据，则会将其保存到单独的文件中：

```shell
ls train_test_split

      nlg_test_data.yml     test_data.yml
      nlg_training_data.yml training_data.yml
```

## rasa data convert nlu

你可以将 NLU 数据从：

- LUIS 数据格式
- WIT 数据格式
- Dialogflow 数据格式
- JSON

转换为：

- YAML
- JSON

通过如下命令可以执行转换：

```shell
rasa data convert nlu
```

如下参数可以指定输入文件或目录、输出文件或目录和输出格式：

```shell
usage: rasa data convert nlu [-h] [-v] [-vv] [--quiet] [-f {json,yaml}]
                             [--data DATA [DATA ...]] [--out OUT]
                             [-l LANGUAGE]

optional arguments:
  -h, --help            show this help message and exit
  -f {json,yaml}, --format {json,yaml}
                        Output format the training data should be converted
                        into. (default: yaml)
  --data DATA [DATA ...]
                        Paths to the files or directories containing Rasa NLU
                        data. (default: data)
  --out OUT             File (for `json`) or existing path (for `yaml`) where
                        to save training data in Rasa format. (default:
                        converted_data)
  -l LANGUAGE, --language LANGUAGE
                        Language of data. (default: en)

Python Logging Options:
  You can control level of log messages printed. In addition to these
  arguments, a more fine grained configuration can be achieved with
  environment variables. See online documentation for more info.

  -v, --verbose         Be verbose. Sets logging level to INFO. (default:
                        None)
  -vv, --debug          Print lots of debugging statements. Sets logging level
                        to DEBUG. (default: None)
  --quiet               Be quiet! Sets logging level to WARNING. (default:
                        None)
```

## rasa data migrate

领域是唯一在 2.0 和 3.0 版本之间发生格式改变的数据。你可以自动将 2.0 版本的领域迁移到 3.0 版本格式。

运行如下命令可以执行迁移：

```shell
rasa data migrate
```

如下参数可以指定输入文件或目录、输出文件或目录：

```
rasa data migrate -d DOMAIN --out OUT_PATH
```

如果没有指定参数，则默认领域路径（`domain.yml`）将用于输入和输出文件。

此命令还会将 2.0 版本的领域文件备份到不同的 `original_domain.yml` 文件或标有 `original_domain` 的目录中。

请注意，如果这些槽是表单 `required_slots` 的一部分，则迁移领域中的槽将包含[映射条件](/domain#mapping-conditions)。

!!! caution "警告"

    当领域文件无效或已经为 3.0 版本格式时，当槽或表单分布在多个领域文件中且原始文件中缺少槽或表单时，迁移过程会终止并触发异常。这样做是为了避免在领域文件中出现重复的迁移部分。请确保所有槽或表单的定义都被整合到一个文件中。

运行如下命令可以了解该命令的更多信息：

```shell
rasa data migrate --help
```

## rasa data validate

你可以检查领域、NLU 和故事数据是否存在错误或不一致。运行如下命令可以验证数据：

```shell
rasa data validate
```

校验器会在数据中搜寻错误，例如：两个意图具有一些相同的训练样本。校验器也会检查是否存不同的对话响应来自相同的对话历史的故事。故事之间的冲突会阻碍模型学习正确的对话模式。

如果在 `config.yml` 文件中对一个或多个策略提供了 `max_history` 的值，可以使用 `--max-history <max_history>` 参数为验证命令的提供对应的最小值。

运行如下命令可以仅验证故事的结构：

```shell
rasa data validate stories
```

!!! note "注意"

    运行 `asa data validate` 不会测试[规则](/rules)是否和故事一致。但是在训练期间，`RulePolicy` 会检查规则和故事之间的冲突。任何此类冲突都会终止训练。

    此外，如果你使用端到端的故事，那么可能无法捕获所有冲突。例如，如果两个用户的输入有不同的分词但有相同特征，那么这些输入之后可能存在冲突动作且不会被工具报告。

使用 `--fail-on-warnings` 参数可以在诸如未使用的意图或动作之类的小问题时也中断验证。

!!! warning "检查故事名称"

    `rasa data validate stories` 命令假定所有故事的名称都是唯一的。

你可以使用如下附加参数运行 `rasa data validate`，例如：指定数据和领域文件的位置：

```
usage: rasa data validate [-h] [-v] [-vv] [--quiet]
                          [--max-history MAX_HISTORY] [-c CONFIG]
                          [--fail-on-warnings] [-d DOMAIN]
                          [--data DATA [DATA ...]]
                          {stories} ...

positional arguments:
  {stories}
    stories             Checks for inconsistencies in the story files.

optional arguments:
  -h, --help            show this help message and exit
  --max-history MAX_HISTORY
                        Number of turns taken into account for story structure
                        validation. (default: None)
  -c CONFIG, --config CONFIG
                        The policy and NLU pipeline configuration of your bot.
                        (default: config.yml)
  --fail-on-warnings    Fail validation on warnings and errors. If omitted
                        only errors will result in a non zero exit code.
                        (default: False)
  -d DOMAIN, --domain DOMAIN
                        Domain specification. This can be a single YAML file,
                        or a directory that contains several files with domain
                        specifications in it. The content of these files will
                        be read and merged together. (default: domain.yml)
  --data DATA [DATA ...]
                        Paths to the files or directories containing Rasa
                        data. (default: data)

Python Logging Options:
  You can control level of log messages printed. In addition to these
  arguments, a more fine grained configuration can be achieved with
  environment variables. See online documentation for more info.

  -v, --verbose         Be verbose. Sets logging level to INFO. (default:
                        None)
  -vv, --debug          Print lots of debugging statements. Sets logging level
                        to DEBUG. (default: None)
  --quiet               Be quiet! Sets logging level to WARNING. (default:
                        None)
```

## rasa export

运行如下命令可以使用事件代理从追踪存储中导出事件：

```shell
rasa export
```

你可以指定环境文件的位置、发布时间时间戳的最小值和最大值、以及发布的对话 ID：

```
usage: rasa export [-h] [-v] [-vv] [--quiet] [--endpoints ENDPOINTS]
                   [--minimum-timestamp MINIMUM_TIMESTAMP]
                   [--maximum-timestamp MAXIMUM_TIMESTAMP]
                   [--conversation-ids CONVERSATION_IDS]

optional arguments:
  -h, --help            show this help message and exit
  --endpoints ENDPOINTS
                        Endpoint configuration file specifying the tracker
                        store and event broker. (default: endpoints.yml)
  --minimum-timestamp MINIMUM_TIMESTAMP
                        Minimum timestamp of events to be exported. The
                        constraint is applied in a 'greater than or equal'
                        comparison. (default: None)
  --maximum-timestamp MAXIMUM_TIMESTAMP
                        Maximum timestamp of events to be exported. The
                        constraint is applied in a 'less than' comparison.
                        (default: None)
  --conversation-ids CONVERSATION_IDS
                        Comma-separated list of conversation IDs to migrate.
                        If unset, all available conversation IDs will be
                        exported. (default: None)

Python Logging Options:
  You can control level of log messages printed. In addition to these
  arguments, a more fine grained configuration can be achieved with
  environment variables. See online documentation for more info.

  -v, --verbose         Be verbose. Sets logging level to INFO. (default:
                        None)
  -vv, --debug          Print lots of debugging statements. Sets logging level
                        to DEBUG. (default: None)
  --quiet               Be quiet! Sets logging level to WARNING. (default:
                        None)
```

!!! tips "将对话导入企业版 Rasa"

    此命令常用于将旧的对话导入企业版 Rasa 中并对其进行标注。可以在[导入对话至企业版 Rasa](https://rasa.com/docs/rasa-enterprise/installation-and-setup/deploy#1-import-existing-conversations-from-rasa-open-source)中获取更多信息。

## rasa evaluate markers

!!! caution "警告"

    此功能目前是实验性的，未来可能发生更改或删除。你可以在论坛中进行反馈，来帮助其变为生产可用。

如下命令将你在标记配置文件中定义的[标记](/markers)应用于存储在[追踪存储](/tracker-stores)中预先存在的对话，同时生成包含提取的标记和统计概要信息的 `.csv` 文件：

```shell
rasa evaluate markers all extracted_markers.csv
```

使用如下参数可以配置标记提取过程：

```
usage: rasa evaluate markers [-h] [-v] [-vv] [--quiet] [--config CONFIG] [--no-stats | --stats-file-prefix [STATS_FILE_PREFIX]] [--endpoints ENDPOINTS] [-d DOMAIN] output_filename {first_n,sample,all} ...

positional arguments:
  output_filename       The filename to write the extracted markers to (CSV format).
  {first_n,sample,all}
    first_n             Select trackers sequentially until N are taken.
    sample              Select trackers by sampling N.
    all                 Select all trackers.

optional arguments:
  -h, --help            show this help message and exit
  --config CONFIG       The config file(s) containing marker definitions. This can be a single YAML file, or a directory that contains several files with marker definitions in it. The content of these files will be read and
                        merged together. (default: markers.yml)
  --no-stats            Do not compute summary statistics. (default: True)
  --stats-file-prefix [STATS_FILE_PREFIX]
                        The common file prefix of the files where we write out the compute statistics. More precisely, the file prefix must consist of a common path plus a common file prefix, to which suffixes `-overall.csv` and
                        `-per-session.csv` will be added automatically. (default: stats)
  --endpoints ENDPOINTS
                        Configuration file for the tracker store as a yml file. (default: endpoints.yml)
  -d DOMAIN, --domain DOMAIN
                        Domain specification. This can be a single YAML file, or a directory that contains several files with domain specifications in it. The content of these files will be read and merged together. (default:
                        domain.yml)

Python Logging Options:
  -v, --verbose         Be verbose. Sets logging level to INFO. (default: None)
  -vv, --debug          Print lots of debugging statements. Sets logging level to DEBUG. (default: None)
  --quiet               Be quiet! Sets logging level to WARNING. (default: None)
```
