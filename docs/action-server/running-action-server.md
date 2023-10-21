# 运行一个 Rasa SDK 动作服务器

有两种方法可以运行动作服务器，具体取决于你是否使用安装了 `rasa` 环境。

如果安装了 `rasa`，可以使用 `rasa` 命令运行动作服务器：

```shell
rasa run actions
```

或者，可以使用 `SANIC_HOST` 环境变量让你的对话机器人监听特定地址：

```shell
SANIC_HOST=192.168.69.150 rasa run actions
```

如果未安装 `rasa`，可以直接将动作服务器作为 Python 模块运行：

```shell
python -m rasa_sdk --actions actions
```

将动作服务器直接作为 Python 模块运行也可以使用 `SANIC_HOST`：

```shell
SANIC_HOST=192.168.69.150 python -m rasa_sdk --actions actions
```

使用上述命令，`rasa_sdk` 将在名为 `actions.py` 的文件或名为 `actions` 的包目录内寻找你的动作。可以使用 `--actions` 标识指定不同的动作模块或包。

使用命令运行动作服务器的完整选项列表如下：

```
usage: __main__.py [-h] [-p PORT] [--cors [CORS ...]] [--actions ACTIONS]
                   [--ssl-keyfile SSL_KEYFILE]
                   [--ssl-certificate SSL_CERTIFICATE]
                   [--ssl-password SSL_PASSWORD] [--auto-reload] [-v] [-vv]
                   [--quiet] [--log-file LOG_FILE]
                   [--logging-config_file LOGGING_CONFIG_FILE]

starts the action endpoint

options:
  -h, --help            show this help message and exit
  -p PORT, --port PORT  port to run the server at
  --cors [CORS ...]     enable CORS for the passed origin. Use * to whitelist
                        all origins
  --actions ACTIONS     name of action package to be loaded
  --ssl-keyfile SSL_KEYFILE
                        Set the SSL certificate to create a TLS secured
                        server.
  --ssl-certificate SSL_CERTIFICATE
                        Set the SSL certificate to create a TLS secured
                        server.
  --ssl-password SSL_PASSWORD
                        If your ssl-keyfile is protected by a password, you
                        can specify it using this paramer.
  --auto-reload         Enable auto-reloading of modules containing Action
                        subclasses.
  -v, --verbose         Be verbose. Sets logging level to INFO
  -vv, --debug          Print lots of debugging statements. Sets logging level
                        to DEBUG
  --quiet               Be quiet! Sets logging level to WARNING
  --log-file LOG_FILE   Store logs in specified file.
  --logging-config_file LOGGING_CONFIG_FILE
                        If set, the name of the logging configuration file
                        will be set to the given name.
```
