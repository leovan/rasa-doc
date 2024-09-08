# 安装开源 Rasa

## 安装开源 Rasa {#install-rasa-open-source}

首先确保你的 `pip` 已经更新到最新版本：

```shell
pip3 install -U pip
```

安装开源 Rasa：

```shell
pip3 install rasa
```

!!! info "遥测报告"

    当第一次运行开源 Rasa 时，你会看到一条消息通知你关于匿名使用数据的收集。你可以在[遥测文档](../telemetry/telemetry.md)中阅读更多关于如何收集数据及其用途的信息。

恭喜，你已经成功安装了开源 Rasa！

现在可以使用如下命令创建一个新项目：

```shell
rasa init
```

你可以在[命令行界面](../command-line-interface.md)中了解重要的 Rasa 命令。

## 从源代码构建 {#building-from-source}

如果需要使用开发版本的开源 Rasa，可以从 GitHub 上获取：

```shell
curl -sSL https://raw.githubusercontent.com/python-poetry/poetry/master/get-poetry.py | python
git clone https://github.com/RasaHQ/rasa.git
cd rasa
poetry install
```

## 额外依赖 {#additional-dependencies}

对于某些机器学习算法，你需要安装额外的 Python 包。为了占用较小的空间默认并不会安装它们。

[模型调优](../tuning-your-model.md)页面可以帮助你为对话机器人选择正确的配置并提醒你相关的其他依赖项。

!!! tip "安装全部"

    如果你不介意其他依赖项，可以：

    ```shell
    pip3 install rasa[full]
    ```

    来安装所有配置所需的依赖项。

### Python 3.10 依赖 {#python-310-requirements}

如果你使用的是 Linux 系统，安装 `rasa[full]` 可能会因为 `tokenizers` 和 `cryptography` 安装包导致失败。

为了解决这个问题，你必须按照如下步骤安装 Rust 编译器：

```shell
apt install rustc && apt install cargo
```

初始化 Rust 编译器后，你应该重新启动控制台并校验安装：

```shell
rustc --version
```

如果 `PATH` 变量未自动设置，请执行：

```shell
export PATH="$HOME/.cargo/bin:$PATH"
```

如果你使用的是 macOS 系统，安装 `rasa[full]`（无论是通过 `pip` 还是源码安装）可能会因为 `tokenizers` 安装包导致失败（详见 [issue](https://github.com/huggingface/tokenizers/issues/1050)）。

为了解决这个问题，你必须按照如下步骤安装 Rust 编译器：

```shell
brew install rustup
rustup-init
```

初始化 Rust 编译器后，你应该重新启动控制台并校验安装：

```shell
rustc --version
```

如果 `PATH` 变量未自动设置，请执行：

```shell
export PATH="$HOME/.cargo/bin:$PATH"
```

### spaCy 依赖项 {#dependencies-for-spacy}

有关 spaCy 模型的更多信息，请参见 [spaCy 文档](https://spacy.io/usage/models)。

可以使用如下命令进行安装：

```shell
pip3 install rasa[spacy]
python3 -m spacy download en_core_web_md
```

!!! tip "使用 zsh"

    在 zsh 中，方括号被解释为命令行中的模式。要运行带有方括号的命令，你可以将带有方括号的参数放在引号中，例如 `pip3 install 'rasa[spacy]'`，或使用反斜杠进行转义，例如 `pip3 install rasa\[spacy\]`。我们建议使用第一种方法（`pip3 install 'rasa[spacy]'`），其可以在任意 shell 中正常运行。

这将安装开源 Rasa、spaCy 及其英语语言模型，其他语言模型也可用。建议至少使用中等大小的模型（`_md`）而不是 spaCy 默认的 `en_core_web_sm` 小模型。小模型占用更少的内存，但可能会降低意图分类的性能。

### MITIE 依赖项 {#dependencies-for-mitie}

首先，运行：

```shelll
pip3 install git+https://github.com/mit-nlp/MITIE.git
pip3 install rasa[mitie]
```

然后下载 [MITIE 模型](https://github.com/mit-nlp/MITIE/releases/download/v0.4/MITIE-models-v0.2.tar.bz2)。所需的文件是 `total_word_feature_extractor.dat`。保存到任意路径，如果使用 MITIE，则需要告诉它在哪里可以找到这个文件（在本例中，其保存在项目目录的 `data` 文件夹中）。

## 升级版本 {#upgrading-versions}

将已安装的 Rasa 升级至 PyPI 的最新版本：

```shell
pip3 install --upgrade rasa
```

要下载指定版本，需指定版本号：

```shell
pip3 install rasa==3.0
```
