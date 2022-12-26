# 安装开源 Rasa

!!! hint "想先探索？"

    你可以在安装开源 Rasa 之前使用 [Rasa Playground](/playground) 进行在线探索。在教程结束后，你可以下载生成的对话机器人，在机器上安装 Rasa 后继续本地开发。

## 安装开源 Rasa

首先确保你的 `pip` 已经更新到最新版本：

```shell
pip3 install -U pip
```

安装开源 Rasa：

```shell
pip3 install rasa
```

!!! info "遥测报告"

    当第一次运行开源 Rasa 时，你会看到一条消息通知你关于匿名使用数据的收集。你可以在[遥测文档](/telemetry/telemetry)中阅读更多关于如何收集数据及其用途的信息。

恭喜，你已经成功安装了开源 Rasa！

现在可以使用如下命令创建一个新项目：

```shell
rasa init
```

你可以在[命令行界面](/command-line-interface)中了解重要的 Rasa 命令。

## 从源代码构建

如果需要使用开发版本的开源 Rasa，可以从 GitHub 上获取：

```shell
curl -sSL https://raw.githubusercontent.com/python-poetry/poetry/master/get-poetry.py | python
git clone https://github.com/RasaHQ/rasa.git
cd rasa
poetry install
```

!!! note "注意"

    由于缺乏对 Apple M1 的 Tensorflow 官方支持，开源 Rasa 目前无法使用 M1 训练模型。

## 额外依赖

对于某些机器学习算法，你需要安装额外的 Python 包。为了占用较小的空间默认并不会安装它们。

[模型调优](/tuning-your-model)页面可以帮助你为对话机器人选择正确的配置并提醒你相关的其他依赖项。

!!! hint "全部都来！"

    如果你不介意其他依赖项，可以：

    ```shell
    pip3 install rasa[full]
    ```

    来安装所有配置所需的依赖项。

### spaCy 依赖项

有关 spaCy 模型的更多信息，请参见 [spaCy 文档](https://spacy.io/usage/models)。

可以使用如下命令进行安装：

```shell
pip3 install rasa[spacy]
python3 -m spacy download en_core_web_md
```

这将安装开源 Rasa、spaCy 及其英语语言模型，其他语言模型也可用。建议至少使用中等大小的模型（`_md`）而不是 spaCy 默认的 `en_core_web_sm` 小模型。小模型占用更少的内存，但可能会降低意图分类的性能。

### MITIE 依赖项

首先，运行：

```shelll
pip3 install git+https://github.com/mit-nlp/MITIE.git
pip3 install rasa[mitie]
```

然后下载 [MITIE 模型](https://github.com/mit-nlp/MITIE/releases/download/v0.4/MITIE-models-v0.2.tar.bz2)。所需的文件是 `total_word_feature_extractor.dat`。保存到任意路径，如果使用 MITIE，则需要告诉它在哪里可以找到这个文件（在本例中，其保存在项目目录的 `data` 文件夹中）。

## 升级版本

将已安装的 Rasa 升级至 PyPI 的最新版本：

```shell
pip3 install --upgrade rasa
```

要下载指定版本，需指定版本号：

```shell
pip3 install rasa==3.0
```
