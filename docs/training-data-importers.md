# 训练数据导入器

开源 Rasa 具有用于收集和加载以 Rasa 格式编写的训练数据的内置逻辑，但你也可以使用自定义训练数据导入器自定义训练数据的导入方式。

使用 [`--data` 命令行参数](command-line-interface.md)，你可以指定 Rasa 应该在磁盘上查找训练数据的位置。然后 Rasa 会加载任何潜在的训练文件并使用它们来训练对话机器人。

如果需要，你还可以自定义 Rasa 导入训练数据的方式。潜在的用例可能有：

- 使用自定义解析器以其他格式加载训练数据。
- 使用不同的方式来收集训练数据（例如从不同的资源加载它们）。

你可以[编写自定义导入器](training-data-importers.md#writing-a-custom-importer)并通过将 `importers` 部分添加到配置文件并指定导入器及完整类路径来指示 Rasa 使用它：

```yaml title="config.yml" hl_lines="2 3 4"
importers:
- name: "module.CustomImporter"
  parameter1: "value"
  parameter2: "value2"
- name: "RasaFileImporter"
```

`name` 键用于确定应该加载哪个导入器。任何额外的参数都作为构造函数参数传递给加载的导入器。

!!! info "注意"

    `TrainingDataImporter` 及其子类从 Rasa 3.0 开始不包含异步方法。为了迁移自定义导入器并是他们与 Rasa 3.0 兼容，你还需要将异步方法替换为同步方法。请参见[迁移指南](migration-guide.md)了解更多信息。

!!! tip "提示"

    你可以指定多个导入器。Rasa 会自动合并他们的结果。

## RasaFileImporter（默认） {#rasafileimporter-default}

默认情况下，Rasa 使用 `RasaFileImporter` 导入器。如果你想单独使用它，则不必在配置文件中指定任何内容。如果你想与其他导入器一起使用，请将其添加到配置文件中：

```yaml title="config.yml" hl_lines="5"
importers:
- name: "module.CustomImporter"
  parameter1: "value"
  parameter2: "value2"
- name: "RasaFileImporter"
```

## MultiProjectImporter（实验性） {#multiprojectimporter-experimental}

!!! info "1.3 版本新特性"

    此功能目前是实验性的，未来可能发生更改或删除。你可以在论坛中进行反馈，来帮助其变为生产可用。

使用此导入器，可以通过组合多个可复用的 Rasa 项目来训练模型。例如，你可能会在一个项目中处理闲聊，在另一个项目中处理问候用户。这些项目可以单独开发，然后在训练对话机器人时组合起来。

例如，考虑如下目录结构：

```
.
├── config.yml
└── projects
    ├── GreetBot
    │   ├── data
    │   │   ├── nlu.yml
    │   │   └── stories.yml
    │   └── domain.yml
    └── ChitchatBot
        ├── config.yml
        ├── data
        │   ├── nlu.yml
        │   └── stories.yml
        └── domain.yml
```

这里，上下文 AI 对话机器人导入 `ChitchatBot` 项目，然后导入 `GreetBot` 项目。项目导入在每个项目的配置文件中定义。

要指示 Rasa 使用 `MultiProjectImporter` 模块，你需要将其添加到根 `config.yml` 的 `importers` 列表中。

```yaml title="./config.yml"
importers:
- name: MultiProjectImporter
```

然后，在同一个文件中，通过将它们添加到 `imports` 列表来指定要导入的项目。

```yaml title="./config.yml"
imports:
- projects/ChitchatBot
```

`ChitchatBot` 的配置文件需要参考 `GreetBot`：

```yaml title="./ChitchatBot/config.yml"
imports:
- ../GreetBot
```

由于 `GreetBot` 项目没有指定要导入的其他项目，因此不需要 `config.yml`。

Rasa 使用配置文件中的相对路径来导入项目。这些可以是文件系统上允许文件访问的任何地方。

在训练过程中，Rasa 会导入所有需要的训练文件，将它们组合起来，训练出一个统一的 AI 对话机器人。训练数据在运行时合并，因此不会创建额外的训练数据文件。

!!! warning "策略和 NLU 流水线"

    Rasa 在训练时会使用项目根目录的策略和 NLU 流水线配置。导入项目的策略和 NLU 配置将被忽略。

!!! warning "注意合并"

    相同的意图、实体、槽、响应、动作和表单将被合并。例如：如果两个项目有 `greet` 意图的训练数据，它们的训练数据将被合并。

## 编写自定义导入器 {#writing-a-custom-importer}

如果你正在编写自定义导入器，则此导入器必须实现 `TrainingDataImporter` 接口：

```python
from typing import Optional, Text, Dict, List, Union

import rasa
from rasa.shared.core.domain import Domain
from rasa.shared.nlu.interpreter import RegexInterpreter
from rasa.shared.core.training_data.structures import StoryGraph
from rasa.shared.importers.importer import TrainingDataImporter
from rasa.shared.nlu.training_data.training_data import TrainingData


class MyImporter(TrainingDataImporter):
    """Example implementation of a custom importer component."""

    def __init__(
        self,
        config_file: Optional[Text] = None,
        domain_path: Optional[Text] = None,
        training_data_paths: Optional[Union[List[Text], Text]] = None,
        **kwargs: Dict
    ):
        """Constructor of your custom file importer.

        Args:
            config_file: Path to configuration file from command line arguments.
            domain_path: Path to domain file from command line arguments.
            training_data_paths: Path to training files from command line arguments.
            **kwargs: Extra parameters passed through configuration in configuration file.
        """

        pass

    def get_domain(self) -> Domain:
        path_to_domain_file = self._custom_get_domain_file()
        return Domain.load(path_to_domain_file)

    def _custom_get_domain_file(self) -> Text:
        pass

    def get_stories(
        self,
        interpreter: "NaturalLanguageInterpreter" = RegexInterpreter(),
        exclusion_percentage: Optional[int] = None,
    ) -> StoryGraph:
        from rasa.shared.core.training_data.story_reader.yaml_story_reader import (
            YAMLStoryReader,
        )

        path_to_stories = self._custom_get_story_file()
        return YAMLStoryReader.read_from_file(path_to_stories, self.get_domain())

    def _custom_get_story_file(self) -> Text:
        pass

    def get_config(self) -> Dict:
        path_to_config = self._custom_get_config_file()
        return rasa.utils.io.read_config_file(path_to_config)

    def _custom_get_config_file(self) -> Text:
        pass

    def get_nlu_data(self, language: Optional[Text] = "en") -> TrainingData:
        from rasa.shared.nlu.training_data import loading

        path_to_nlu_file = self._custom_get_nlu_file()
        return loading.load_data(path_to_nlu_file)

    def _custom_get_nlu_file(self) -> Text:
        pass
```
