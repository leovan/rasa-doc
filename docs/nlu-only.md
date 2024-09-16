# 仅使用 NLU

了解如何将 Rasa NLU 用作对话机器人或虚拟助手的独立 NLU 服务。

可以只将 Rasa 用作 NLU 组件。

## 仅训练 NLU 模型 {#training-nlu-only-models}

要仅训练 NLU 模型，请运行：

```shell
rasa train nlu
```

这将在 `data/` 目录中查找 NLU 训练数据文件，并将经过训练的模型保存在 `models/` 目录中。模型的名称将以 `nlu-` 开头。

## 在命令行上测试 NLU 模型 {#testing-your-nlu-model-on-the-command-line}

要在命令行上使用 NLU 模型，请运行如下命令：

```shell
rasa shell nlu
```

这将启动 rasa shell 并要求你输入要测试的消息。你可以继续输入任意数量的消息。

或者，你可以省略 `nlu` 参数并直接仅传入 `nlu` 模型：

```shell
rasa shell -m models/nlu-20190515-144445.tar.gz
```

## 运行 NLU 服务器 {#running-an-nlu-server}

要使用 NLU 模型启动服务器，请在运行时传入模型名称：

```shell
rasa run --enable-api -m models/nlu-20190515-144445.tar.gz
```

然后，可以使用 `/model/parse` 端点从你的模型中请求预测。为此，请运行：

```shell
curl localhost:5005/model/parse -d '{"text":"hello"}'
```
