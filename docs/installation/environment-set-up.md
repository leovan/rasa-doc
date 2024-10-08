# 环境配置

## Python 环境配置 {#python-environment-setup}

检查 Python 环境是否已经配置：

```shell
python3 --version
pip2 --version
```

!!! info "Python 3 版本支持"

    目前 Rasa 支持 Python 3.7、3.8、3.9 和 3.10。请注意，仅 3.4.x 或更高版本支持 Python 3.10。此外，苹果芯片机器的 Python 3.10 环境无法安装 3.4.x，需要 3.5.x 或更高版本。

如果这些已经安装，则命令应该显示相关版本号，可以直接跳到下一步。

否则，请按照如下说明进行安装。

=== "Ubuntu"

    使用 `apt` 获取相关包，并使用 `pip` 安装 virtualenv。

    ```shell
    sudo apt update
    sudo apt install python3-dev python3-pip
    ```

=== "macOS"

    如果未安装 [Homebrew](https://brew.sh/) 包管理器请先安装。

    完成后即可安装 Python 3。

    ```shell
    brew update
    brew install python
    ```

=== "Windows"

    确保安装了 Microsoft VC++ 编译器，以便 Python 可以编译相关依赖项。你可以从 [Visual Studio](https://visualstudio.microsoft.com/visual-cpp-build-tools/) 获取编译器。下载安装程序并在列表中选择 VC++ 构建工具。安装适用于 Windows 的 64 位版本 [Python 3](https://www.python.org/downloads/windows/https://www.python.org/downloads/windows/)。

    ```powershell
    C:\> pip3 install -U pip
    ```

## 虚拟环境配置 {#virtual-environment-setup}

本步骤是可选的，但我们强烈建议使用虚拟环境隔离 Python 项目。[virtualenv](https://virtualenv.pypa.io/en/latest/) 和 [virtualenvwrapper](https://virtualenvwrapper.readthedocs.io/en/latest/) 等工具提供了隔离的 Python 环境，这比在系统范围内安装包更干净（因为这可以防止依赖冲突）。它们还允许你在没有 root 权限的情况下安装扩展包。

=== "Ubuntu"

    选择 Python 解释器并创建一个 `./venv` 目录来保存并创建一个新的虚拟环境：

    ```shell
    python3 -m venv ./venv
    ```

    激活虚拟环境：

    ```shell
    source ./venv/bin/activate
    ```

=== "macOS"

    选择 Python 解释器并创建一个 `./venv` 目录来保存并创建一个新的虚拟环境：

    ```shell
    python3 -m venv ./venv
    ```

    激活虚拟环境：

    ```shell
    source ./venv/bin/activate
    ```

=== "Windows"

    选择 Python 解释器并创建一个 `.\\venv` 目录来保存并创建一个新的虚拟环境：

    ```powershell
    C:\> python3 -m venv ./venv
    ```

    激活虚拟环境：

    ```powershell
    C:\> .\venv\Scripts\activate
    ```

## M1/M2（苹果芯片）限制 {#m1--m2-apple-silicon-limitations}

默认情况下，苹果芯片上的 Rasa 安装不使用 [Apple Metal](https://developer.apple.com/metal/)。我们发现在苹果芯片上使用 GPU 会显著增加 [DIETClassifier](../components.md#dietclassifier) 和 [TEDPolicy](../policies.md#ted-policy) 的训练时间。你可以安装可选依赖来自行测试或使用其他组件进行尝试：`pip3 install rasa[metal]`。

目前，并非所有 Rasa 的依赖性都原生支持 Apple Silicon。这会导致：

- 你不能在 Apple Silicon 上将 Duckling 作为 Docker 容器运行。如果使用 [duckling 实体提取器](../components.md#ducklingentityextractor)，建议在云端部署 duckling。相关进展请参见 [Duckling 项目](https://github.com/facebook/duckling/issues/695)。
- Apple Silicon 上的 Rasa 不支持 [ConveRTFeaturizer 组件](../components.md#convertfeaturizer)或包含它的管道。该组件依赖 Apple Silicon 目前还不可用的 `tensorflow-text`。相关进展请参见 [Tensorflow Text 项目](https://github.com/tensorflow/text/issues/823)。
