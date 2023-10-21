# 组件

组件构成 NLU 流水线并按顺序工作以将用户输入处理为结构化输出。这些组件可用于实体提取、意图分类、响应选择、预处理等。

## 语言模型 {#language-models}

如果你想在流水线中使用预训练的词向量，以下组件会加载所需的预训练模型。

### MitieNLP {#mitienlp}

- 概要

    MITIE 初始化器

- 输出

    无

- 要求

    无

- 描述

    初始化 MITIE 结构。每个 MITIE 组件都依赖于此，因此应将其放在使用任何 MITIE 组件的流水线开头。

- 配置

    MITIE 库需要一个语言模型文件，必须在配置中指定：

    ```yaml
    pipeline:
    - name: "MitieNLP"
      # language model to load
      model: "data/total_word_feature_extractor.dat"
    ```

    有关从何处获取该文件的更多信息，请参见[安装 MITIE](installation/installing-rasa-open-source.md#dependencies-for-mitie)。

你还可以使用 MITIE 从一种语言语料训练自己的词向量。为此需要：

- 获取一个干净的语料（例如维基百科）作为一组文本文件。
- 在语料库上构建并运行 [MITIE Wordrep Tool](https://github.com/mit-nlp/MITIE/tree/master/tools/wordrep)。这可能需要几个小时或几天，具体取决于你的数据集和服务器。你需要 128G 的内训来运行 wordrep，可以尝试扩展 swap。
- 将新的 `total_word_feature_extractor.dat` 路径设置为[配置](model-configuration.md)文件中 `MitieNLP` 组件的模型参数。

    有关如何训练 MITIE 词向量的完整示例，请参见[用 Rasa NLU 构建自己的中文 NLU 系统](http://www.crownpku.com/2017/07/27/%E7%94%A8Rasa_NLU%E6%9E%84%E5%BB%BA%E8%87%AA%E5%B7%B1%E7%9A%84%E4%B8%AD%E6%96%87NLU%E7%B3%BB%E7%BB%9F.html)，这是一篇通过从中文维基百科构建 MITIE 模型的博客。

### SpacyNLP {#spacynlp}

- 概要

    spaCy 语言初始器

- 输出

    无

- 要求

    无

- 描述

    初始化 spaCy 结构。每个 spaCy 组件都依赖于此，因此应该将其放在使用任何 spaCy 组件的流水线开头。

- 配置

    你需要指定要使用的语言模型。该名称将传递给 `spacy.load(name)`。你可以在 [spaCy 文档](https://spacy.io/usage/models)中找到有关可用模型的更多信息。

    ```yaml
    pipeline:
    - name: "SpacyNLP"
      # language model to load
      model: "en_core_web_md"

      # when retrieving word vectors, this will decide if the casing
      # of the word is relevant. E.g. `hello` and `Hello` will
      # retrieve the same vector, if set to `False`. For some
      # applications and models it makes sense to differentiate
      # between these two words, therefore setting this to `True`.
      case_sensitive: False
    ```

    有关如何下载 spaCy 模型的更多信息，请参见[安装 spaCy](installation/installing-rasa-open-source.md#dependencies-for-spacy)。

    除了 spaCy 的预训练语言模型外，你还可以使用此主键附加你自己训练的 spaCy 模型。

## 分词器 {#tokenizers}

分词器将文本拆分为词条。如果你想将意图拆分为多个标签，例如，为了预测多个意图或对分层意图结构进行建模，请将以下标志与任何分词器一起使用：

- `intent_tokenization_flag` 指示是否标记意图标签。将其设置为 `True` 以便对意图标签进行标记。
- `intent_split_symbol` 设置分割符字符串来拆分意图标签，默认为下划线（`_`）。

### WhitespaceTokenizer {#whitespacetokenizer}

- 概要

    使用空白字符作为分割符的分词器

- 输出

    用户消息、响应（如果存在）和意图（如果指定）的词条

- 要求

    无

- 描述

    为每个空白分割的字符串创建一个标记。

    如果字符满足以下任何条件，则任何不在 `a-zA-Z0-9_#@&` 中的字符将在分割之前用空格替换：

    - 字符后跟一个空格：`"word! "` → `"word"`
    - 字符前有一个空格：`" !word"` → `"word"`
    - 字符在字符串开头：`"!word"` → `"word"`
    - 字符在字符串结尾：`"word!"` → `"word"`

    注意：

    - `"wo!rd"` → `"wo!rd"`

    此外，任何不在 `a-zA-Z0-9_#@&.~:\/?[]()!$*+,;=-` 中的字符都将被替换为空格，然后再按空格分割，如果该字符不是在数字之间：

    - `"twenty{one"` → `"twenty"`，`"one"`（`"{"` 没有在数字之间）
    - `"20{1"` → `"20{1"`（`"{"` 在数字之间）

    注意：

    - `"name@example.com"` → `"name@example.com"`
    - `"10,000.1"` → `"10,000.1"`
    - `"1 - 2"` → `"1"`，`"2"`

- 配置

    ```yaml
    pipeline:
    - name: "WhitespaceTokenizer"
      # Flag to check whether to split intents
      "intent_tokenization_flag": False
      # Symbol on which intent should be split
      "intent_split_symbol": "_"
      # Regular expression to detect tokens
      "token_pattern": None
    ```

### JiebaTokenizer {#jiebatokenizer}

- 概要

    用于中文的 Jieba 分词器

- 输出

    用户消息、响应（如果存在）和意图（如果指定）的词条

- 要求

    暂无

- 描述

    使用专门针对中文的 Jieba 分词器创建词条。仅适用于中文。

    !!! info "注意"

        要使用 `JiebaTokenizer`，你需要先通过 `pip3 install jieba` 来安装 Jieba。

- 配置

    用户的自定义字典文件可以通过 `dictionary_path` 指定文件的目录路径来自动加载。如果 `dictionary_path` 为 `None`（默认值），则不会使用自定义字典。

    ```yaml
    pipeline:
    - name: "JiebaTokenizer"
      dictionary_path: "path/to/custom/dictionary/dir"
      # Flag to check whether to split intents
      "intent_tokenization_flag": False
      # Symbol on which intent should be split
      "intent_split_symbol": "_"
      # Regular expression to detect tokens
      "token_pattern": None
    ```

### MitieTokenizer {#mitietokenizer}

- 概要

    使用 MITIE 的分词器

- 输出

    用户消息、响应（如果存在）和意图（如果指定）的词条

- 要求

    [MitieNLP](components.md#mitienlp)

- 描述

    使用 MITIE 分词器创建的词条

- 配置

    ```yaml
    pipeline:
    - name: "MitieTokenizer"
      # Flag to check whether to split intents
      "intent_tokenization_flag": False
      # Symbol on which intent should be split
      "intent_split_symbol": "_"
      # Regular expression to detect tokens
      "token_pattern": None
    ```

### SpacyTokenizer {#spacytokenizer}

- 概要

    使用 spaCy 的分词器

- 输出

    用户消息、响应（如果存在）和意图（如果指定）的词条

- 要求

    [SpacyNLP](components.md#spacynlp)

- 描述

    使用 spaCy 分词器创建的词条

- 配置

    ```yaml
    pipeline:
    - name: "SpacyTokenizer"
      # Flag to check whether to split intents
      "intent_tokenization_flag": False
      # Symbol on which intent should be split
      "intent_split_symbol": "_"
      # Regular expression to detect tokens
      "token_pattern": None
    ```

## 特征化器 {#featurizers}

文本特征化器分为两类：稀疏特征化器和稠密特征化器。稀疏特征化器是返回具有大量缺失值（例如：零）特征向量的特征化器。由于这些特征向量通常会占用大量内存，因此我们将它们存储为稀疏特征。稀疏的特征仅存储非零值及其在向量中的位置。因此，可以节省大量内存并能够在更大的数据集上进行训练。

所有特征化器可以返回两种不同的特征：序列特征和句子特征。序列特征是一个词条长度 $\times$ 特征维度大小的矩阵。该矩阵包含序列中每个词条的特征向量。这使得可以训练序列模型。句子特征是一个 $1 \times$ 特征维度大小的矩阵。它包含完整消息的特征向量。句子特征可以用于任何词袋模型。因此，相应的分类器可以决定使用哪种特征。注意：序列和句子特征的特征维度不一定相同。

### MitieFeaturizer {#mitiefeaturizer}

- 概要

    使用 MITIE 特征化器创建用户消息和响应（如果指定）的向量表示。

- 输出

    用于用户消息和响应的 `dense_features`

- 要求

    [MitieNLP](components.md#mitienlp)

- 类型

    稠密特征化器

- 描述

    使用 MITIE 特征化器为实体提取、意图分类和响应分类创建特征。

    !!! info "注意"

        不适用于 `MitieIntentClassifier` 组件。但可以被流水线中后续使用 `dense_features` 的任何组件使用。

- 配置

    句子向量，即完整消息的向量，可以通过两种不同方式计算，均值或最大池化。可以使用 `pooling` 选项在配置文件中指定池化方法。默认池化方法设置为 `mean`。

    ```yaml
    pipeline:
    - name: "MitieFeaturizer"
      # Specify what pooling operation should be used to calculate the vector of
      # the complete utterance. Available options: 'mean' and 'max'.
      "pooling": "mean"
    ```

### SpacyFeaturizer {#spacyfeaturizer}

- 概要

    使用 spaCy 特征化器创建用户消息和响应（如果指定）的向量表示。

- 输出

    用于用户消息和响应的 `dense_features`

- 要求

    [SpacyNLP](components.md#spacynlp)

- 类型

    稠密特征化器

- 描述

    使用 spaCy 特征化器为实体提取、意图分类和响应分类创建特征。

- 配置

    句子向量，即完整消息的向量，可以通过两种不同方式计算，均值或最大池化。可以使用 `pooling` 选项在配置文件中指定池化方法。默认池化方法设置为 `mean`。

    ```yaml
    pipeline:
    - name: "SpacyFeaturizer"
      # Specify what pooling operation should be used to calculate the vector of
      # the complete utterance. Available options: 'mean' and 'max'.
      "pooling": "mean"
    ```

### ConveRTFeaturizer {#convertfeaturizer}

- 缩写

    使用 [ConveRT](https://github.com/PolyAI-LDN/polyai-models) 模型创建用户消息和响应（如果指定）的向量表示。

- 输出

    用于用户消息和响应的 `dense_features`

- 类型

    稠密特征化器

- 描述

    为实体提取、意图分类和响应分类创建特征。其使用 [default signature](https://github.com/PolyAI-LDN/polyai-models#tfhub-signatures) 来计算输入文本的向量表示。

    !!! info "注意"

        由于 `ConveRT` 模型仅在英文语料库上进行训练，因此仅当你的训练数据为英语时才应使用此特征化器。

    !!! info "注意"

        要使用 `ConveRTFeaturizer`，请使用 `pip3 install rasa[convert]` 安装开源 Rasa。

        注意此组件目前无法运行在 M1/M2 架构的 macOS 上。有关此限制的更多信息，请参见[此处](installation/environment-set-up.md#m1--m2-apple-silicon-limitations)。

- 配置

    ```yaml
    pipeline:
    - name: "ConveRTFeaturizer"
      # Remote URL/Local directory of model files(Required)
      "model_url": None
    ```

    !!! warning "警告"

        由于 ConveRT 模型的公共 URL 已下线，现在必须将 `model_url` 参数设置为社区或自托管 URL 或包含模型文件的本地目录路径。

### LanguageModelFeaturizer {#languagemodelfeaturizer}

- 概要

    使用预训练语言模型创建用户消息和响应（如果指定）的向量表示。

- 输出

    用于用户消息和响应的 `dense_features`

- 类型

    稠密特征化器

- 描述

    为实体提取、意图分类和响应分类创建特征。其使用预训练的语言模型来计算输入文本的向量表示。

    !!! info "注意"

        请确保你使用的语言模型与你的训练数据在相同的语言预料上预训练。

- 配置

    在此组件之前包含一个[分词器](components.md#tokenizers)组件。

    你应该通过 `model_name` 参数指定要加载的语言模型。有关当前支持的语言模型，请参见下表。要加载的权重可以通过附加参数 `model_weights` 指定。如果留空，它将使用表中列出的默认模型权重。

    ```
    +----------------+--------------+-------------------------+
    | Language Model | Parameter    | Default value for       |
    |                | "model_name" | "model_weights"         |
    +----------------+--------------+-------------------------+
    | BERT           | bert         | rasa/LaBSE              |
    +----------------+--------------+-------------------------+
    | GPT            | gpt          | openai-gpt              |
    +----------------+--------------+-------------------------+
    | GPT-2          | gpt2         | gpt2                    |
    +----------------+--------------+-------------------------+
    | XLNet          | xlnet        | xlnet-base-cased        |
    +----------------+--------------+-------------------------+
    | DistilBERT     | distilbert   | distilbert-base-uncased |
    +----------------+--------------+-------------------------+
    | RoBERTa        | roberta      | roberta-base            |
    +----------------+--------------+-------------------------+
    ```

    除了默认的预训练模型权重之外，如果满足一下条件，可以从 [HuggingFace 模型](https://huggingface.co/models)中选用更多模型（上述文件可以在模型网站的“Files and versions”部分找到）。

    - 模型架构是支持的语言模型之一（检查 `config.json` 中的 `model_type` 是否在表的 `model_name` 列中）
    - 模型具有预训练的 Tensorflow 权重（检查 `tf_model.h5` 文件是否存在）
    - 模型使用默认的分词器（`config.json` 不应该包含自定义的 `tokenizer_class` 设置）

    !!! info "注意"

        为 bert 架构默认加载的 LaBSE 权重提供了在 112 中语言上训练的多语言模型（请参见[教程](https://www.youtube.com/watch?v=7tAWk_Coj-s)和[论文](https://arxiv.org/pdf/2007.01852.pdf)）。我们强烈建议在尝试使用其他权重和架构优化此组件之前将其用作基线并端到端地测试对话机器人。

    以下配置使用 rasa/LaBSE 权重加载 BERT 语言模型，可以参见[此处](https://huggingface.co/rasa/LaBSE/tree/main)：

    ```yaml
    pipeline:
    - name: LanguageModelFeaturizer
      # Name of the language model to use
      model_name: "bert"
      # Pre-Trained weights to be loaded
      model_weights: "rasa/LaBSE"

      # An optional path to a directory from which
      # to load pre-trained model weights.
      # If the requested model is not found in the
      # directory, it will be downloaded and
      # cached in this directory for future use.
      # The default value of `cache_dir` can be
      # set using the environment variable
      # `TRANSFORMERS_CACHE`, as per the
      # Transformers library.
      cache_dir: null
    ```

### RegexFeaturizer {#regexfeaturizer}

- 概要

    使用正则表达式创建用户消息和响应（如果指定）的向量表示。

- 输出

    用于用户消息的 `sparse_features` 和 `tokens.pattern`

- 要求

    `tokens`

- 类型

    稀疏特征化器

- 描述

    为实体提取和意图分类创建特征。在训练期间，`RegexFeaturizer` 创建以训练数据格式定义的正则表达式列表。对于每个正则表达式，将设置一个特征，标记该表达式是否在用户信息中找到。所有特征稍后将被送入意图分类器和实体提取器来简化分类（假设分类器在训练阶段已经学习，这该集合特征表示某个意图或实体）。目前仅 [CRFEntityExtractor](components.md#crfentityextractor) 和 [DIETClassifier](components.md#dietclassifier) 组件支持用于实体提取的正则表达式特征。

- 配置

    通过添加 `case_sensitive: False` 选项使特征化器不区分大小写，默认为 `case_sensitive: True`。

    要正确处理诸如中文等不使用空格进行分词的语言，用户需要添加 `use_word_boundaries: False` 选项，默认为 `use_word_boundaries: True`。

    ```yaml
    pipeline:
    - name: "RegexFeaturizer"
      # Text will be processed with case sensitive as default
      "case_sensitive": True
      # use match word boundaries for lookup table
      "use_word_boundaries": True
    ```

    配置增量训练

    为了确保 `sparse_features` 在增量训练期间具有固定大小，应配置组件来考虑将来可能添加到训练数据中的其他模式。为此，请在从头开始训练基本模型时配置 `number_additional_patterns` 参数：

    ```yaml hl_lines="3"
    pipeline:
    - name: RegexFeaturizer
      number_additional_patterns: 10
    ```

    如果未配置，该组件将使用训练数据中当前存在的模式数量的两倍（包括查找表和正则表达式模式）作为 `number_additional_patterns` 的默认值。这个数字至少保持在 10，以避免在增量训练期间过于频繁的用完新模式的额外槽。一旦组件用完额外的模式槽，新模式就会被丢弃，并且在特征化过程中不会被考虑。此时，建议从头开始训练新模型。

### CountVectorsFeaturizer {#countvectorsfeaturizer}

- 概要

    创建用户消息、意图和响应的词袋表示。

- 输出

    用于用户消息、意图和响应的 `sparse_features`

- 要求

    `tokens`

- 类型

    稀疏特征化器

- 描述

    为意图分类和响应分类创建特征。使用 [sklearn 的 CountVectorizer](https://scikit-learn.org/stable/modules/generated/sklearn.feature_extraction.text.CountVectorizer.html) 创建用户消息、意图和响应的词袋表示。所有仅由数字组成的词条（例如：123 和 99，但不包括 a123d）将被指定相同的特征。

- 配置

    有关配置参数的详细说明，请参见 [sklearn CountVectorizer 的文档](https://scikit-learn.org/stable/modules/generated/sklearn.feature_extraction.text.CountVectorizer.html)。

    使用 `analyzer` 配置参数可以将特征化器配置为使用单词或词组 n-grams。默认情况下，`analyzer` 设置为 `word`，因此单词标记计数作为特征使用。如果要使用字符 n-grams，请将 `analyzer` 设置为 `char` 或 `char_wb`。n-grams 的下界和上界可以通过参数 `min_ngram` 和 `max_ngram` 进行设置。默认情况下，他们的值都为 `1`。默认情况下，特征化器采用单词的词元，而不是直接使用单词。单词的词元目前仅由 [SpacyTokenizer](components.md#spacytokenizer) 设置。你可以通过将 `use_lemma` 设置为 `False` 来禁用此行为。

    !!! info "注意"

        `char_wb` 选项仅从单词边界内的文本创建字符 n-gram。单词边缘的 n-grams 用空格填充。此选项可用于创建 [Subword Semantic Hashing](https://arxiv.org/abs/1810.07150)。

    !!! info "注意"

        对于字符 n-grams，不要忘记添加 `min_ngram` 和 `max_ngram` 参数。否则，词汇表将只包含单个字母。

    处理词汇表外（OOV）单词：

    !!! info "注意"

        仅当 `analyzer` 为 `word` 时启用。

    由于训练是在有限的词汇数据上进行的，因此不能保证在预测过程中算法不会遇到未知单词（训练期间未见过的单词）。为了告诉算法如何处理未知词，训练数据中的一些词可以用通用词 `OOV_token` 代替。在这种情况下，在预测期间，所有未知词都将被视为 `OOV_token` 这个通用词。

    例如，可以在训练数据中创建单独的 `outofscope` 意图，其中包含不同数量的 `OOV_token` 的消息，并且可能还有一些额外的通用词。然后，算法可能会将带有未知词的消息分类为 `outofscope` 意图。

    可以设置 `OOV_token` 或 `OOV_words` 单词列表：

    - `OOV_token` 设置为用于未见过词的关键字。如果训练数据在某些消息中包含 `OOV_token` 作为单词，则在训练期间为见过的词在预测期间将替换为 `OOV_token`。如果 `OOV_token=None`（默认行为）在训练期间未见过的词在预测期间将被忽略。
    - `OOV_words` 设置在训练期间被视为 `OOV_token` 的单词列表。如果已知应被视为词汇表外的单词列表，则可以将其设置为 `OOV_words` 而不是在训练数据中手动更改它或使用自定义预处理器。

    !!! info "注意"

        该特征化器通过单词计数来创建词袋表示，因此句子中 `OOV_token` 的数量可能很重要。

    !!! info "注意"

        提供 `OOV_words` 是可选的，训练数据可以包含手动输入的 `OOV_token` 或自定义附加预处理器。仅当训练数据中存在此标记或提供了 `OOV_words` 列表时，才会用 `OOV_token` 替换为见过的单词。

    如果要在用户消息和意图之间共享词汇表，则需要将 `use_shared_vocab` 选项设置为 `True`。在这种情况下，在意图中的标记和用户消息之间会建立一个通用词汇表。

    ```yaml
    pipeline:
    - name: "CountVectorsFeaturizer"
      # Analyzer to use, either 'word', 'char', or 'char_wb'
      "analyzer": "word"
      # Set the lower and upper boundaries for the n-grams
      "min_ngram": 1
      "max_ngram": 1
      # Set the out-of-vocabulary token
      "OOV_token": "_oov_"
      # Whether to use a shared vocab
      "use_shared_vocab": False
    ```

    用于增量训练的配置

    为了确保 `sparse_features` 在[增量训练](command-line-interface.md#incremental-training)期间具有固定大小，应将组件配置为考虑将来额外词汇可能添加作为新训练样本的一部分。为此，请在从头开始训练基本模型是配置 `additional_vocabulary_size` 参数。

    ```yaml hl_lines="3-6"
    pipeline:
    - name: CountVectorsFeaturizer
      additional_vocabulary_size:
        text: 1000
        response: 1000
        action_text: 1000
    ```

    如上例所示，你可以为每个 `text`（用户消息）、`response`（
`ResponseSelector` 使用的对话机器人响应）和 `action_text`（`ResponseSelector` 未使用的对话机器人响应）定义额外的词汇量。如果你正在构建一个共享词汇表（`use_shared_vocab=True`），只需要为 `text` 属性定义一个值。如果用户未配置任何属性，则该组件将当前词汇大小的一半作为该属性的附加词汇大小的默认值。这个数字至少保持在 1000，以避免在增量训练期间过于频繁地用完额外的词汇槽。一旦组件用完额外的词汇槽，新的词汇词条就会被丢弃并且在特征化过程重中不会被考虑。此时，建议从头开始重新训练模型。

    上述配置参数是模型拟合数据所需要配置的参数。除此之外还有一些附加参数。

    ??? info "更多配置参数"

        ```
        +---------------------------+-------------------------+--------------------------------------------------------------+
        | Parameter                 | Default Value           | Description                                                  |
        +===========================+=========================+==============================================================+
        | use_shared_vocab          | False                   | If set to 'True' a common vocabulary is used for labels      |
        |                           |                         | and user message.                                            |
        +---------------------------+-------------------------+--------------------------------------------------------------+
        | analyzer                  | word                    | Whether the features should be made of word n-gram or        |
        |                           |                         | character n-grams. Option 'char_wb' creates character        |
        |                           |                         | n-grams only from text inside word boundaries;               |
        |                           |                         | n-grams at the edges of words are padded with space.         |
        |                           |                         | Valid values: 'word', 'char', 'char_wb'.                     |
        +---------------------------+-------------------------+--------------------------------------------------------------+
        | strip_accents             | None                    | Remove accents during the pre-processing step.               |
        |                           |                         | Valid values: 'ascii', 'unicode', 'None'.                    |
        +---------------------------+-------------------------+--------------------------------------------------------------+
        | stop_words                | None                    | A list of stop words to use.                                 |
        |                           |                         | Valid values: 'english' (uses an internal list of            |
        |                           |                         | English stop words), a list of custom stop words, or         |
        |                           |                         | 'None'.                                                      |
        +---------------------------+-------------------------+--------------------------------------------------------------+
        | min_df                    | 1                       | When building the vocabulary ignore terms that have a        |
        |                           |                         | document frequency strictly lower than the given threshold.  |
        +---------------------------+-------------------------+--------------------------------------------------------------+
        | max_df                    | 1                       | When building the vocabulary ignore terms that have a        |
        |                           |                         | document frequency strictly higher than the given threshold  |
        |                           |                         | (corpus-specific stop words).                                |
        +---------------------------+-------------------------+--------------------------------------------------------------+
        | min_ngram                 | 1                       | The lower boundary of the range of n-values for different    |
        |                           |                         | word n-grams or char n-grams to be extracted.                |
        +---------------------------+-------------------------+--------------------------------------------------------------+
        | max_ngram                 | 1                       | The upper boundary of the range of n-values for different    |
        |                           |                         | word n-grams or char n-grams to be extracted.                |
        +---------------------------+-------------------------+--------------------------------------------------------------+
        | max_features              | None                    | If not 'None', build a vocabulary that only consider the top |
        |                           |                         | max_features ordered by term frequency across the corpus.    |
        +---------------------------+-------------------------+--------------------------------------------------------------+
        | lowercase                 | True                    | Convert all characters to lowercase before tokenizing.       |
        +---------------------------+-------------------------+--------------------------------------------------------------+
        | OOV_token                 | None                    | Keyword for unseen words.                                    |
        +---------------------------+-------------------------+--------------------------------------------------------------+
        | OOV_words                 | []                      | List of words to be treated as 'OOV_token' during training.  |
        +---------------------------+-------------------------+--------------------------------------------------------------+
        | alias                     | CountVectorFeaturizer   | Alias name of featurizer.                                    |
        +---------------------------+-------------------------+--------------------------------------------------------------+
        | use_lemma                 | True                    | Use the lemma of words for featurization.                    |
        +---------------------------+-------------------------+--------------------------------------------------------------+
        | additional_vocabulary_size| text: 1000              | Size of additional vocabulary to account for incremental     |
        |                           | response: 1000          | training while training a model from scratch                 |
        |                           | action_text: 1000       |                                                              |
        +---------------------------+-------------------------+--------------------------------------------------------------+
        ```

### LexicalSyntacticFeaturizer {#lexicalsyntacticfeaturizer}

- 概要

    创建用户消息的词法和句法特征以支持实体提取。

- 输出

    用于用户消息的 `sparse_features`

- 要求

    `tokens`

- 类型

    稀疏特征化器

- 描述

    为实体提取创建特征。在用户消息中的每个词条上移动一个滑动窗口，并根据配置创建特征（见下文）。由于存在默认配置，因此你无需指定配置。

- 配置

    你可以配置特征化器应该提取何种词法和句法特征。可以使用如下特征：

    ```
    ==============  ==========================================================================================
    Feature Name    Description
    ==============  ==========================================================================================
    BOS             Checks if the token is at the beginning of the sentence.
    EOS             Checks if the token is at the end of the sentence.
    low             Checks if the token is lower case.
    upper           Checks if the token is upper case.
    title           Checks if the token starts with an uppercase character and all remaining characters are
                    lowercased.
    digit           Checks if the token contains just digits.
    prefix5         Take the first five characters of the token.
    prefix2         Take the first two characters of the token.
    suffix5         Take the last five characters of the token.
    suffix3         Take the last three characters of the token.
    suffix2         Take the last two characters of the token.
    suffix1         Take the last character of the token.
    pos             Take the Part-of-Speech tag of the token (``SpacyTokenizer`` required).
    pos2            Take the first two characters of the Part-of-Speech tag of the token
                    (``SpacyTokenizer`` required).
    ==============  ==========================================================================================
    ```

    当特征化器在带有滑动窗口的用户消息中的词条上移动时，你可以在滑动窗口中定义前一个词条、当前词条和下一个词条的特征。将特征定义为 \[之前，当前，之后\] 数组。如果想为前一个词条、当前词条和下一个词条定义特征，特征配置将如下所示：

    ```yaml
    pipeline:
    - name: LexicalSyntacticFeaturizer
      "features": [
        ["low", "title", "upper"],
        ["BOS", "EOS", "low", "upper", "title", "digit"],
        ["low", "title", "upper"],
      ]
    ```

    此配置也是默认配置。

    !!! info "注意"

        如果你想使用 `pos` 或 `pos2`，你需要将 `SpacyTokenizer` 添加到你的流水线中。

## 意图分类器 {#intent-classifiers}

意图分类器将领域文件中定义的意图之一指定给传入的用户消息。

### MitieIntentClassifier {#mitieintentclassifier}

- 概要

    MITIE 意图分类器（使用一个[文本分类器](https://github.com/mit-nlp/MITIE/blob/master/examples/python/text_categorizer_pure_model.py)）

- 输出

    `intent`

- 要求

    用户消息的 `tokens` 和 [MitieNLP](components.md#mitienlp)

- 输出示例

    ```json
    {
        "intent": {"name": "greet", "confidence": 0.98343}
    }
    ```

- 描述

    该分类器使用 MITIE 来执行意图分类。底层分类器使用具有稀疏线性核的多分类线性 SVM（请参见 [MITIE 训练器代码](https://github.com/mit-nlp/MITIE/blob/master/mitielib/src/text_categorizer_trainer.cpp)中的 `train_text_categorizer_classifier` 函数）。

    !!! info "注意"

        该分类器不依赖于任何特征化器，因为其自己可以提取特征。

- 配置

    ```yaml
    pipeline:
    - name: "MitieIntentClassifier"
    ```

### LogisticRegressionClassifier {#logisticregressionclassifier}

- 概要

    使用 [scikit-learn 实现](https://scikit-learn.org/stable/modules/generated/sklearn.linear_model.LogisticRegression.html)的逻辑回归意图分类。

- 输出

    `intent` 和 `intent_ranking`。

- 要求

    需要有 `sparse_features` 或 `dense_features`。

- 输出示例

    ```json
    {
        "intent": {"name": "greet", "confidence": 0.780},
        "intent_ranking": [
            {
                "confidence": 0.780,
                "name": "greet"
            },
            {
                "confidence": 0.140,
                "name": "goodbye"
            },
            {
                "confidence": 0.080,
                "name": "restaurant_search"
            }
        ]
    }
    ```

- 描述

    该分类器使用 scikit-learn 的逻辑回归实现来执行意图分类。它能够只使用稀疏特征，但也可以使用任何提供的稠密特征。一般来说，DIET 应该产生更准确的结果，但这个分类器训练速度更快，并且可以用作轻量的基准测试。我们的实现采用 scikit-learn 的基本设置，除了 `class_weight` 参数，因为我们假设采用 `balanced` 的设置。

- 配置

    包含默认值的一个示例配置如下：

    ```yaml
    pipeline:
    - name: LogisticRegressionClassifier
      max_iter: 100
      solver: lbfgs
      tol: 0.0001
      random_state: 42
    ```

    参数简要说明如下：

    - `max_iter`：求解器的最大迭代次数。
    - `solver`：使用的求解器。对于非常小的数据集，可以考虑使用 `liblinear`。
    - `tol`：优化器停止标准的容差。
    - `random_state`：用于训练前进行打乱数据。

    有关参数的更多信息，请参见 [scikit-learn 文档页面](https://scikit-learn.org/stable/modules/generated/sklearn.linear_model.LogisticRegression.html)。

### SklearnIntentClassifier {#sklearnintentclassifier}

- 概要

    Sklearn 意图分类器。

- 输出

    `intent` 和 `intent_ranking`。

- 要求

    用户消息的 `dense_features`。

- 输出示例

    ```json
    {
        "intent": {"name": "greet", "confidence": 0.7800},
        "intent_ranking": [
            {
                "confidence": 0.7800,
                "name": "greet"
            },
            {
                "confidence": 0.1400,
                "name": "goodbye"
            },
            {
                "confidence": 0.0800,
                "name": "restaurant_search"
            }
        ]
    }
    ```

- 描述

    Sklearn 意图分类器训练的一个通过网格搜索优化的线性 SVM。它还提供了未预测标签的排名。`SklearnIntentClassifier` 之前需要在流水线中有一个稠密特征化器。这个稠密特征化器创建用于分类的特征。有关算法的更多信息，请参见 [GridSearchCV](https://scikit-learn.org/stable/modules/generated/sklearn.model_selection.GridSearchCV.html) 文档。

- 配置

    在 SVM 训练期间，运行超参数搜索来寻找最优参数集。在配置中，可以指定要尝试的参数。

    ```yaml
    pipeline:
    - name: "SklearnIntentClassifier"
      # Specifies the list of regularization values to
      # cross-validate over for C-SVM.
      # This is used with the ``kernel`` hyperparameter in GridSearchCV.
      C: [1, 2, 5, 10, 20, 100]
      # Specifies the kernel to use with C-SVM.
      # This is used with the ``C`` hyperparameter in GridSearchCV.
      kernels: ["linear"]
      # Gamma parameter of the C-SVM.
      "gamma": [0.1]
      # We try to find a good number of cross folds to use during
      # intent training, this specifies the max number of folds.
      "max_cross_validation_folds": 5
      # Scoring function used for evaluating the hyper parameters.
      # This can be a name or a function.
      "scoring_function": "f1_weighted"
    ```

### KeywordIntentClassifier {#keywordintentclassifier}

- 概要

    简单的关键词匹配意图分类器，适用于小型短期项目。

- 输出

    `intent`。

- 要求

    无。

- 输出示例

    ```json
    {
        "intent": {"name": "greet", "confidence": 1.0}
    }
    ```

- 描述

    该分类器通过在消息中搜索关键字来工作。默认情况下，匹配区分大小写，仅搜索用户消息中关键字字符串的完全匹配。用于意图的关键字即为 NLU 训练数据中该意图的样本。这意味着整个样本为关键字，而不是样本中的单个单词。

    !!! info "注意"

        这个分类器适用于小型或入门项目。如果你的 NLU 训练数据很少，可以查看[模型调优](tuning-your-model.md)中建议的流水线。

- 配置

    ```yaml
    pipeline:
    - name: "KeywordIntentClassifier"
      case_sensitive: True
    ```

### DIETClassifier {#dietclassifier}

- 概要

    多意图实体 Transformer（Dual Intent Entity Transformer，DIET），用于意图分类和实体提取。

- 输出

    `entities`，`intent` 和 `intent_ranking`。

- 要求

    用于用户消息和可选意图的 `dense_features` 和/或 `sparse_features`。

- 输出示例

    ```json
    {
        "intent": {"name": "greet", "confidence": 0.7800},
        "intent_ranking": [
            {
                "confidence": 0.7800,
                "name": "greet"
            },
            {
                "confidence": 0.1400,
                "name": "goodbye"
            },
            {
                "confidence": 0.0800,
                "name": "restaurant_search"
            }
        ],
        "entities": [{
            "end": 53,
            "entity": "time",
            "start": 48,
            "value": "2017-04-10T00:00:00.000+02:00",
            "confidence": 1.0,
            "extractor": "DIETClassifier"
        }]
    }
    ```

- 描述

    DIET（Dual Intent Entity Transformer）是一种用于意图分类和实体识别的多任务架构。该架构基于一个共享两个任务的 Transformer。实体标签序列基于输入词条序列对应的 Transformer 输出序列利用条件随机场在标记层进行预测。对于意图标签，用于完整消息和输出和意图标签被嵌入到同一个语义向量空间中。我们使用点积损失来最大化与目标标签的相似度同时最小化与负样本的相似度。

    DIET 不提供预训练的词嵌入和语言模型，但是如果将其添加到流水线中，则可以使用这些特征。想了解有关该模型的更多信息，请查看 Youtube 上的 [Algorithm Whiteboard](https://www.youtube.com/playlist?list=PL75e0qA87dlG-za8eLI6t0_Pbxafk-cxb) 系列视频，其中详细解释了模型的架构。

    !!! info "注意"

        在预测期间，如果消息仅包含训练期间为见过的单词，并且没有使用 Out-Of-Vocabulary 预处理器，则会以置信度为 `0` 的 `None` 空意图作为预测结果。如果仅将 [CountVectorsFeaturizer](components.md#countvectorsfeaturizer) 与 `word` 分析器一起用作特征化器，则可能发生这种情况。如使用 `char_wb` 分析器，你可以始终获得置信度大于 `0.0` 的意图。

    如果你只想将 `DIETClassifier` 用于意图分类，请将 `entity_recognition` 设置为 `False`。如果你只想进行实体识别，请将 `intent_classification` 设置为 `False`。默认情况下，`DIETClassifier` 会同时进行两项任务，即 `entity_recognition` 和 `intent_classification` 均设置为 `True`。

    你可以定义多个超参数来调整模型。如果要调整模型，可以通过修改如下参数：

    - `epochs`：此参数设置算法使用训练数据的次数（默认值：`300`）。一个 `epoch` 等于所有训练样本的一次前向传导和一次反向传导。有时模型需要更多的轮次才能很好地学习。有时更多的轮次不会影响性能。轮次越少，模型训练得越快。
    - `hidden_layers_sizes`：此参数允许定义用户消息和意图的前馈层数及其输出维度（默认值：`text: [], label: []`）。列表中的每个值对应一个前馈层。比如如果设置 `text: [256, 128]`，则会在 Transformer 前添加两个前馈层。输入词条的向量（来自用户消息）将被传递到这些层。第一层的输出维度为 256，第二层的输出维度为 128。如果使用空列表（默认值），则不会添加前馈层。确保仅使用正整数。通常使用 2 的幂指数。此外，通常的做法是在列表中设置递减的值，即下一个值小于等于前一个值。
    - `embedding_dimension`：此参数定义模型内部使用的嵌入层的输出维度（默认值：`20`）。在模型架构中使用了多个嵌入层。例如，完整消息和意图的向量在计算损失和比较之前会经过一个嵌入层。
    - `number_of_transformer_layers`：此参数设置要使用的 Transformer 层数（默认值：`2`）。Transformer 层数对应模型使用的 Transformer 块。
    - `transformer_size`：此参数设置 Transformer 中的单元数量（默认值：`256`）。从 Transformer 输出的向量将具有给定的 `transformer_size`。
    - `connection_density`：此参数定义模型中所有前馈层设置为非零值的核权重的比例（默认值：`0.2`）。该值应介于 0 和 1 之间。如果将 `connection_density` 设置为 1，则不会将核权重设置为 0，该层则为标准的前馈层。不应该将 `connection_density` 设置为 0，因为这将导致所有核权重为 0，则模型无法进行学习。
    - `constrain_similarities`：当设置为 `True` 时，此参数对所有相似项应用 sigmoid 交叉熵损失。这有助于将输入标签和负标签之间的相似度保持为较小的值。这有助于将模型更好的推广到现实世界的测试集中。
    - `model_confidence`：此参数允许用户配置在推理过程中如何计算置信度。它只能接受一个值作为输入，即 `softmax` [^components-softmax]。在 `softmax` 中，置信度在 `[0, 1]` 之间。计算出的相似度使用 `softmax` 激活函数进行归一化。

    上述配置参数是应该进行配置的，从而使模型可以拟合数据。同时，还有更多参数可以进行调整。

    ??? info "更多配置参数"

        ```
        +---------------------------------+------------------+--------------------------------------------------------------+
        | Parameter                       | Default Value    | Description                                                  |
        +=================================+==================+==============================================================+
        | hidden_layers_sizes             | text: []         | Hidden layer sizes for layers before the embedding layers    |
        |                                 | label: []        | for user messages and labels. The number of hidden layers is |
        |                                 |                  | equal to the length of the corresponding list.               |
        +---------------------------------+------------------+--------------------------------------------------------------+
        | share_hidden_layers             | False            | Whether to share the hidden layer weights between user       |
        |                                 |                  | messages and labels.                                         |
        +---------------------------------+------------------+--------------------------------------------------------------+
        | transformer_size                | 256              | Number of units in transformer.                              |
        +---------------------------------+------------------+--------------------------------------------------------------+
        | number_of_transformer_layers    | 2                | Number of transformer layers.                                |
        +---------------------------------+------------------+--------------------------------------------------------------+
        | number_of_attention_heads       | 4                | Number of attention heads in transformer.                    |
        +---------------------------------+------------------+--------------------------------------------------------------+
        | use_key_relative_attention      | False            | If 'True' use key relative embeddings in attention.          |
        +---------------------------------+------------------+--------------------------------------------------------------+
        | use_value_relative_attention    | False            | If 'True' use value relative embeddings in attention.        |
        +---------------------------------+------------------+--------------------------------------------------------------+
        | max_relative_position           | None             | Maximum position for relative embeddings.                    |
        +---------------------------------+------------------+--------------------------------------------------------------+
        | unidirectional_encoder          | False            | Use a unidirectional or bidirectional encoder.               |
        +---------------------------------+------------------+--------------------------------------------------------------+
        | batch_size                      | [64, 256]        | Initial and final value for batch sizes.                     |
        |                                 |                  | Batch size will be linearly increased for each epoch.        |
        |                                 |                  | If constant `batch_size` is required, pass an int, e.g. `8`. |
        +---------------------------------+------------------+--------------------------------------------------------------+
        | batch_strategy                  | "balanced"       | Strategy used when creating batches.                         |
        |                                 |                  | Can be either 'sequence' or 'balanced'.                      |
        +---------------------------------+------------------+--------------------------------------------------------------+
        | epochs                          | 300              | Number of epochs to train.                                   |
        +---------------------------------+------------------+--------------------------------------------------------------+
        | random_seed                     | None             | Set random seed to any 'int' to get reproducible results.    |
        +---------------------------------+------------------+--------------------------------------------------------------+
        | learning_rate                   | 0.001            | Initial learning rate for the optimizer.                     |
        +---------------------------------+------------------+--------------------------------------------------------------+
        | embedding_dimension             | 20               | Dimension size of embedding vectors.                         |
        +---------------------------------+------------------+--------------------------------------------------------------+
        | dense_dimension                 | text: 128        | Dense dimension for sparse features to use.                  |
        |                                 | label: 20        |                                                              |
        +---------------------------------+------------------+--------------------------------------------------------------+
        | concat_dimension                | text: 128        | Concat dimension for sequence and sentence features.         |
        |                                 | label: 20        |                                                              |
        +---------------------------------+------------------+--------------------------------------------------------------+
        | number_of_negative_examples     | 20               | The number of incorrect labels. The algorithm will minimize  |
        |                                 |                  | their similarity to the user input during training.          |
        +---------------------------------+------------------+--------------------------------------------------------------+
        | similarity_type                 | "auto"           | Type of similarity measure to use, either 'auto' or 'cosine' |
        |                                 |                  | or 'inner'.                                                  |
        +---------------------------------+------------------+--------------------------------------------------------------+
        | loss_type                       | "cross_entropy"  | The type of the loss function, either 'cross_entropy'        |
        |                                 |                  | or 'margin'. If type 'margin' is specified,                  |
        |                                 |                  | "model_confidence=cosine" will be used which is deprecated   |
        |                                 |                  | as of 2.3.4. See footnote (1).                               |
        +---------------------------------+------------------+--------------------------------------------------------------+
        | ranking_length                  | 10               | Number of top intents to report. Set to 0 to report all      |
        |                                 |                  | intents.                                                     |
        +---------------------------------+------------------+--------------------------------------------------------------+
        | renormalize_confidences         | False            | Normalize the reported top intents. Applicable only with loss|
        |                                 |                  | type 'cross_entropy' and 'softmax' confidences.              |
        +---------------------------------+------------------+--------------------------------------------------------------+
        | maximum_positive_similarity     | 0.8              | Indicates how similar the algorithm should try to make       |
        |                                 |                  | embedding vectors for correct labels.                        |
        |                                 |                  | Should be 0.0 < ... < 1.0 for 'cosine' similarity type.      |
        +---------------------------------+------------------+--------------------------------------------------------------+
        | maximum_negative_similarity     | -0.4             | Maximum negative similarity for incorrect labels.            |
        |                                 |                  | Should be -1.0 < ... < 1.0 for 'cosine' similarity type.     |
        +---------------------------------+------------------+--------------------------------------------------------------+
        | use_maximum_negative_similarity | True             | If 'True' the algorithm only minimizes maximum similarity    |
        |                                 |                  | over incorrect intent labels, used only if 'loss_type' is    |
        |                                 |                  | set to 'margin'.                                             |
        +---------------------------------+------------------+--------------------------------------------------------------+
        | scale_loss                      | False            | Scale loss inverse proportionally to confidence of correct   |
        |                                 |                  | prediction.                                                  |
        +---------------------------------+------------------+--------------------------------------------------------------+
        | regularization_constant         | 0.002            | The scale of regularization.                                 |
        +---------------------------------+------------------+--------------------------------------------------------------+
        | negative_margin_scale           | 0.8              | The scale of how important it is to minimize the maximum     |
        |                                 |                  | similarity between embeddings of different labels.           |
        +---------------------------------+------------------+--------------------------------------------------------------+
        | connection_density              | 0.2              | Connection density of the weights in dense layers.           |
        |                                 |                  | Value should be between 0 and 1.                             |
        +---------------------------------+------------------+--------------------------------------------------------------+
        | drop_rate                       | 0.2              | Dropout rate for encoder. Value should be between 0 and 1.   |
        |                                 |                  | The higher the value the higher the regularization effect.   |
        +---------------------------------+------------------+--------------------------------------------------------------+
        | drop_rate_attention             | 0.0              | Dropout rate for attention. Value should be between 0 and 1. |
        |                                 |                  | The higher the value the higher the regularization effect.   |
        +---------------------------------+------------------+--------------------------------------------------------------+
        | use_sparse_input_dropout        | True             | If 'True' apply dropout to sparse input tensors.             |
        +---------------------------------+------------------+--------------------------------------------------------------+
        | use_dense_input_dropout         | True             | If 'True' apply dropout to dense input tensors.              |
        +---------------------------------+------------------+--------------------------------------------------------------+
        | evaluate_every_number_of_epochs | 20               | How often to calculate validation accuracy.                  |
        |                                 |                  | Set to '-1' to evaluate just once at the end of training.    |
        +---------------------------------+------------------+--------------------------------------------------------------+
        | evaluate_on_number_of_examples  | 0                | How many examples to use for hold out validation set.        |
        |                                 |                  | Large values may hurt performance, e.g. model accuracy.      |
        +---------------------------------+------------------+--------------------------------------------------------------+
        | intent_classification           | True             | If 'True' intent classification is trained and intents are   |
        |                                 |                  | predicted.                                                   |
        +---------------------------------+------------------+--------------------------------------------------------------+
        | entity_recognition              | True             | If 'True' entity recognition is trained and entities are     |
        |                                 |                  | extracted.                                                   |
        +---------------------------------+------------------+--------------------------------------------------------------+
        | use_masked_language_model       | False            | If 'True' random tokens of the input message will be masked  |
        |                                 |                  | and the model has to predict those tokens. It acts like a    |
        |                                 |                  | regularizer and should help to learn a better contextual     |
        |                                 |                  | representation of the input.                                 |
        +---------------------------------+------------------+--------------------------------------------------------------+
        | tensorboard_log_directory       | None             | If you want to use tensorboard to visualize training         |
        |                                 |                  | metrics, set this option to a valid output directory. You    |
        |                                 |                  | can view the training metrics after training in tensorboard  |
        |                                 |                  | via 'tensorboard --logdir <path-to-given-directory>'.        |
        +---------------------------------+------------------+--------------------------------------------------------------+
        | tensorboard_log_level           | "epoch"          | Define when training metrics for tensorboard should be       |
        |                                 |                  | logged. Either after every epoch ('epoch') or for every      |
        |                                 |                  | training step ('batch').                                 |
        +---------------------------------+------------------+--------------------------------------------------------------+
        | featurizers                     | []               | List of featurizer names (alias names). Only features        |
        |                                 |                  | coming from the listed names are used. If list is empty      |
        |                                 |                  | all available features are used.                             |
        +---------------------------------+------------------+--------------------------------------------------------------+
        | checkpoint_model                | False            | Save the best performing model during training. Models are   |
        |                                 |                  | stored to the location specified by `--out`. Only the one    |
        |                                 |                  | best model will be saved.                                    |
        |                                 |                  | Requires `evaluate_on_number_of_examples > 0` and            |
        |                                 |                  | `evaluate_every_number_of_epochs > 0`                        |
        +---------------------------------+------------------+--------------------------------------------------------------+
        | split_entities_by_comma         | True             | Splits a list of extracted entities by comma to treat each   |
        |                                 |                  | one of them as a single entity. Can either be `True`/`False` |
        |                                 |                  | globally, or set per entity type, such as:                   |
        |                                 |                  | ```                                                          |
        |                                 |                  | ...                                                          |
        |                                 |                  | - name: DIETClassifier                                       |
        |                                 |                  |   split_entities_by_comma:                                   |
        |                                 |                  |     address: True                                            |
        |                                 |                  |     ...                                                      |
        |                                 |                  | ...                                                          |
        |                                 |                  | ```                                                          |
        +---------------------------------+------------------+--------------------------------------------------------------+
        | constrain_similarities          | False            | If `True`, applies sigmoid on all similarity terms and adds  |
        |                                 |                  | it to the loss function to ensure that similarity values are |
        |                                 |                  | approximately bounded. Used only if `loss_type=cross_entropy`|
        +---------------------------------+------------------+--------------------------------------------------------------+
        | model_confidence                | "softmax"        | Affects how model's confidence for each intent               |
        |                                 |                  | is computed. Currently, only one value is supported:         |
        |                                 |                  | 1. `softmax` - Similarities between input and intent         |
        |                                 |                  | embeddings are post-processed with a softmax function,       |
        |                                 |                  | as a result of which confidence for all intents sum up to 1. |
        |                                 |                  | This parameter does not affect the confidence for entity     |
        |                                 |                  | prediction.                                                  |
        +---------------------------------+------------------+--------------------------------------------------------------+
        ```

        !!! info "注意"

            在 `maximum_negative_similarity = maximum_positive_similarity` 和 `use_maximum_negative_similarity = False` 的情况下，将参数 `maximum_negative_similarity` 设置为负数来模仿原始 starspace 算法。更多信息请参见 [starspace 论文](https://arxiv.org/abs/1709.03856)。

[^components-softmax]: 请注意，`model_confidence: cosineis` 在 2.3.4 版本中已弃用（请参见[变更日志](https://rasa.com/docs/rasa/changelog#234---2021-02-26){:target="_blank"}），并且无法在配置中设置，但如果指定了 `loss_type: margin`，则无论设置如何，都将使用 `model_confidence: cosine`。

### FallbackClassifier {#fallbackclassifier}

- 概要

    如果 NLU 意图分类分数不明确，则使用 `nlu_fallback` 作为消息的分类。置信度设置为与回退阈值相同。

- 输出

    `entities`，`intent` 和 `intent_ranking`。

- 要求

    之前意图分类器的 `intent` 和 `intent_ranking` 输出。

- 输出示例

    ```json
    {
        "intent": {"name": "nlu_fallback", "confidence": 0.7183846840434321},
        "intent_ranking": [
            {
                "confidence": 0.7183846840434321,
                "name": "nlu_fallback"
            },
            {
                "confidence": 0.28161531595656784,
                "name": "restaurant_search"
            }
        ],
        "entities": [{
            "end": 53,
            "entity": "time",
            "start": 48,
            "value": "2017-04-10T00:00:00.000+02:00",
            "confidence": 1.0,
            "extractor": "DIETClassifier"
        }]
    }
    ```

- 描述

    `FallbackClassifier` 使用 `nlu_fallback` 意图对用户消息进行分类，以防先前的意图分类器无法对置信度大于或等于 `FallbackClassifier` 的 `threshold` 的意图进行分类。它可以在排名靠前的两个意图的置信度分数比 `ambiguity_threshold` 更接近的情况下预测回退意图。

    你可以使用 `FallbackClassifier` 来实现一个[回退动作](fallback-handoff.md#fallbacks)，来处理带有不确定 NLU 预测的消息。

    ```yaml
    rules:
    - rule: Ask the user to rephrase in case of low NLU confidence
      steps:
      - intent: nlu_fallback
      - action: utter_please_rephrase
    ```

- 配置

    `FallbackClassifier` 只会在没有其他意图预测的置信度大于或等于阈值的情况下添加 `nlu_fallback` 意图的预测值。

    - `threshold`：此参数设置用于预测 `nlu_fallback` 意图的阈值。如果先前的意图分类器预测的意图没有大于或等于 `threshold` 的置信度，则 `FallbackClassifier` 将添加 `nlu_fallback` 意图的预测值，置信度为 `1.0`。
    - `ambiguity_threshold`：如果配置了 `ambiguity_threshold`，则 `FallbackClassifier` 还将预测 `nlu_fallback` 意图，以防两个排名最高的意图的置信度分数之差小于 `ambiguity_threshold`。

## 实体提取器 {#entity-extractors}

实体提取器从用户消息中提取实体，例如人名或位置。

!!! info "注意"

    如果你使用多个实体提取器，我们建议每个提取器都针对一组专有的实体类型。例如，使用 [Duckling](components.md#ducklingentityextractor) 提取日期和时间，使用 [DIETClassifier](components.md#dietclassifier-1) 提取人名。否则，如果多个提取器针对相同的实体类型，则很可能会多次提取实体。

    例如，如果你使用两个或更多通用提取器，如 [MitieEntityExtractor](components.md#mitieentityextractor)，[DIETClassifier](components.md#dietclassifier-1) 或 [CRFEntityExtractor](components.md#crfentityextractor)，则训练数据中的实体类型均将被找到和提取。如果填充的[槽](domain.md#slots)的值是 `text` 类型，则会采用流水线中最后一个提取器的结果。如果槽是 `list` 类型，则左右结果都将被添加到列表中，包括重复项。

    即使提取器专注于不同的实体类型，也可能发生另一种不太明显的重复/重叠的提取情况。假设一个送餐对话机器人和一条用户消息 `I would like to order the Monday special`。假设，如果时间提取器的性能不是很好，它可能会在这里提取 `Monday` 作为订单的时间，而其他提取器可能会提取 `Monday special` 作为餐点。如果遇到此类重复的实体，添加额外的训练数据来改进提取器可能是有意义的。如果这还不够，可以添加一个[自定义组件](components.md#custom-components)，根据你自己的逻辑解决实体提取中的冲突。

### MitieEntityExtractor {#mitieentityextractor}

- 概要

    MITIE 实体提取（使用一个 [MITIE NER 训练器](https://github.com/mit-nlp/MITIE/blob/master/mitielib/src/ner_trainer.cpp)）。

- 输出

    `entities`。

- 要求

    [MitieNLP](components.md#mitienlp) 和 `tokens`。

- 输出示例

    ```json
    {
        "entities": [{
            "value": "New York City",
            "start": 20,
            "end": 33,
            "confidence": null,
            "entity": "city",
            "extractor": "MitieEntityExtractor"
        }]
    }
    ```

- 描述

    `MitieEntityExtractor` 使用 MITIE 实体提取来查找消息中的实体。底层分类器使用具有稀疏线性内核和自定义特征的多分类线性 SVM。MITIE 组件不提供实体置信度。

    !!! info "注意"

        该实体提取器不依赖任何特征化器，因为它自己可以提取特征。

- 配置

    ```yaml
    pipeline:
    - name: "MitieEntityExtractor"
    ```

### SpacyEntityExtractor {#spacyentityextractor}

- 概要

    spaCy 实体提取。

- 输出

    `entities`。

- 要求

    [SpacyNLP](components.md#spacynlp)。

- 输出示例

    ```json
    {
        "entities": [{
            "value": "New York City",
            "start": 20,
            "end": 33,
            "confidence": null,
            "entity": "city",
            "extractor": "SpacyEntityExtractor"
        }]
    }
    ```

- 描述

    该组件使用 spaCy 预测消息中的实体。spaCy 使用统计 BILOU 转换模型。截至目前，该组件只能使用 spaCy 内置的实体提取模型，不能重新训练。此提取器不提供任何置信度分数。

    你可以在这个[交互式 Demo](https://explosion.ai/demos/displacy-ent) 中测试 spaCy 的实体提取模型。请注意，某些 spaCy 模型高度区分大小写。

    !!! info "注意"

        `SpacyEntityExtractor` 提取器不提供 `confidence` 等级并始终返回 `null`。

- 配置

    配置 spaCy 组件应该提取的维度，即实体类型。可以在 spaCy 文档中找到可用维度的完整列表。未指定维度选项将提取所有维度。

    ```yaml
    pipeline:
    - name: "SpacyEntityExtractor"
      # dimensions to extract
      dimensions: ["PERSON", "LOC", "ORG", "PRODUCT"]
    ```

### CRFEntityExtractor {#crfentityextractor}

- 概要

    条件随机场（CRF）实体提取。

- 输出

    `entities`。

- 要求

    `tokens` 和 `dense_features`（可选）。

- 输出示例

    ```json
    {
        "entities": [{
            "value": "New York City",
            "start": 20,
            "end": 33,
            "entity": "city",
            "confidence": 0.874,
            "extractor": "CRFEntityExtractor"
        }]
    }
    ```

- 描述

    该组件实现了一个条件随机场（CRF）来进行命名实体识别。CRF 可以看作一个无向马尔可夫链，其中单词作为时间，实体类型作为状态。单词的特征（大写，POS 标记等）给出了某些实体类型的概率，相邻实体标签之间的转换也是如此，然后计算并返回最可能的标签集合。

    如果你想将自定义特征（例如预训练的词嵌入）传递给 `CRFEntityExtractor`，你可以在 `CRFEntityExtractor` 之前将任何稠密特征化器添加到流水线中，然后将 `"text_dense_feature"` 添加到特征配置中来使用稠密特征。`CRFEntityExtractor` 自动查找额外的稠密特征并检查稠密特征是否为 `len(tokens)` 的可迭代项，其中每项都是一个向量。如果检查失败，将显示警告。但是，`CRFEntityExtractor` 将继续训练，而无需额外的自定义特征。如果存在稠密特征，`CRFEntityExtractor` 会将稠密特征传递给 `sklearn_crfsuite` 并使用它们进行训练。

- 配置

    `CRFEntityExtractor` 包含一个要使用的默认特征列表。但是，你可以覆盖默认配置。可用的特征如下：

    ```
    ===================  ==========================================================================================
    Feature Name         Description
    ===================  ==========================================================================================
    low                  Checks if the token is lower case.
    upper                Checks if the token is upper case.
    title                Checks if the token starts with an uppercase character and all remaining characters are
                        lowercased.
    digit                Checks if the token contains just digits.
    prefix5              Take the first five characters of the token.
    prefix2              Take the first two characters of the token.
    suffix5              Take the last five characters of the token.
    suffix3              Take the last three characters of the token.
    suffix2              Take the last two characters of the token.
    suffix1              Take the last character of the token.
    pos                  Take the Part-of-Speech tag of the token (``SpacyTokenizer`` required).
    pos2                 Take the first two characters of the Part-of-Speech tag of the token
                        (``SpacyTokenizer`` required).
    pattern              Take the patterns defined by ``RegexFeaturizer``.
    bias                 Add an additional "bias" feature to the list of features.
    text_dense_features  Adds additional features from a dense featurizer.
    ===================  ==========================================================================================
    ```

    当特征化器在带有滑动窗口的用户消息的词条上移动时，你可以在滑动窗口中定义前一个词条、当前词条和下一个词条的特征。特征定义为 [before, token, after] 数组。

    另外，你可以设置一个标志来确定是否使用 BILOU 标记模式。

    - `BILOU_flag` 确定是否使用 BILOU 标记，默认为 `True`。

    ```yaml
    pipeline:
      - name: "CRFEntityExtractor"
      # BILOU_flag determines whether to use BILOU tagging or not.
      "BILOU_flag": True
      # features to extract in the sliding window
      "features": [
          ["low", "title", "upper"],
          [
          "bias",
          "low",
          "prefix5",
          "prefix2",
          "suffix5",
          "suffix3",
          "suffix2",
          "upper",
          "title",
          "digit",
          "pattern",
          "text_dense_features"
          ],
          ["low", "title", "upper"],
      ]
      # The maximum number of iterations for optimization algorithms.
      "max_iterations": 50
      # weight of the L1 regularization
      "L1_c": 0.1
      # weight of the L2 regularization
      "L2_c": 0.1
      # Name of dense featurizers to use.
      # If list is empty all available dense features are used.
      "featurizers": []
      # Indicated whether a list of extracted entities should be split into individual entities for a given entity type
      "split_entities_by_comma":
          address: False
          email: True
    ```

    !!! info "注意"

        如果使用 POS 特征（`pos` 和 `pos2`），需要在流水线中包含 `SpacyTokenizer`。

    !!! info "注意"

        如果使用 `pattern` 特征，需要在流水线中包含 `RegexFeaturizer`。

    !!! info "注意"

        如果使用 `text_dense_features` 特征，需要在流水线中包含一个稠密特征化器（例如：`LanguageModelFeaturizer`）。

### DucklingEntityExtractor {#ducklingentityextractor}

- 概要

    Duckling 可以提过多种语言的常见实体，例如：日期、金额、距离等。

- 输出

    `entities`。

- 要求

    无。

- 输出示例

    ```json
    {
        "entities": [{
            "end": 53,
            "entity": "time",
            "start": 48,
            "value": "2017-04-10T00:00:00.000+02:00",
            "confidence": 1.0,
            "extractor": "DucklingEntityExtractor"
        }]
    }
    ```

- 描述

    要使用这个组件，你需要运行一个 Duckling 服务。最简单的选择是使用 `docker run -p 8000:8000 rasa/duckling` 来启动 Docker 容器。

    或者，你可以[直接在机器上安装 Duckling](https://github.com/facebook/duckling#quickstart) 并启动服务。

    Duckling 可以识别日期、数字、距离和其他结构化实体并将他们标准化。请注意，Duckling 会尝试在不提供排名的情况下提取尽可能多的实体类型。例如，如果你将 `number` 和 `time` 都指定为 Duckling 组件的维度，则该组件将提取两个实体：`I will be there in 10 minutes` 文本中的 `10` 作为一个 `number`，`in 10 minutes` 作为一个 `time`。在这种情况下，你的应用程序必须决定哪种实体类型是正确的。提取器将始终返回 1.0 作为置信度，因为它是基于规则的系统。

    支持的语言列表可以在 [Duckling Github 仓库](https://github.com/facebook/duckling/tree/master/Duckling/Dimensions)中找到。

- 配置

    配置 Duckling 组件应该提取哪些维度，即实体类型。可以在 [Duckling 文档](https://duckling.wit.ai)中找到可用维度的完整列表。未指定维度选项将提取所有可用维度。

    ```yaml
    pipeline:
    - name: "DucklingEntityExtractor"
      # url of the running duckling server
      url: "http://localhost:8000"
      # dimensions to extract
      dimensions: ["time", "number", "amount-of-money", "distance"]
      # allows you to configure the locale, by default the language is
      # used
      locale: "de_DE"
      # if not set the default timezone of Duckling is going to be used
      # needed to calculate dates from relative expressions like "tomorrow"
      timezone: "Europe/Berlin"
      # Timeout for receiving response from http url of the running duckling server
      # if not set the default timeout of duckling http url is set to 3 seconds.
      timeout : 3
    ```

### DIETClassifier {#dietclassifier-1}

- 概要

    多意图实体 Transformer（Dual Intent Entity Transformer，DIET）用于意图分类和实体提取。

- 描述

    可以在意图分类器部分找到 [DIETClassifier](components.md#dietclassifier) 的详细说明。

### RegexEntityExtractor {#regexentityextractor}

- 概要

    使用在训练数据中定义的查找表和/或正则表达式提取实体。

- 输出

    `entities`。

- 要求

    无。

- 描述

    该组件使用训练数据中定义的[查找表](nlu-training-data.md#lookup-tables)和[正则表达式](nlu-training-data.md#regular-expressions-for-entity-extraction)提取实体。该组件检查用户消息中是否包含查找表之一的条目或匹配的一个正则表达式。如果找到匹配项，则将该值提取为实体。

    该组件仅使用名称与训练数据中定义的一个实体相同的那些正则表达式特征。确保每个实体至少有一个样本。

    !!! info "注意"

        当将此提取器与 [MitieEntityExtractor](components.md#mitieentityextractor)，[CRFEntityExtractor](components.md#crfentityextractor) 或 [DIETClassifier](components.md#dietclassifier-1) 结合使用时，它可能会导致实体被多次提取。特别是如果很多训练句子具有正则表达式也定义了的实体类型标注。有关多重提取的更多信息，请参见[实体提取器部分](components.md#entity-extractors)开头的信息框。

        如果需要 RegexEntityExtractor 和另一个上述统计提取器，建议考虑如下两个选项之一。

        选项一是当对每种类型的提取器都具有专有实体类型时，建议使用本选项。为了确保提取器不会互相干扰，只为每个正则表达式/查找表实体类型配置一个句子样本，不要有多个。

        选项二是当想使用正则表达式匹配作为统计提取器的附加，但没有单独的实体类型是，本选项会很有用。在这种情况下，你需要 1) 在流水线中的提取器之前添加 [RegexFeaturizer](components.md#regexfeaturizer)，2) 在训练数据中标注所有实体样本，3) 从流水线中删除 RegexEntityExtractor。这样，你的统计提取器将收到有关存在正则表达式匹配的附加信息，并且能够统计确定何时依赖这些匹配以及何时不依赖这些匹配。

- 配置

    通过添加 `case_sensitive: True` 选项使实体提取器区分大小写，默认为 `case_sensitive: False`。

    要正确处理诸如中文等不使用空格进行分词的语言，用户需要添加 `use_word_boundaries: False` 选项，默认为 `use_word_boundaries: True`。

    ```yaml
    pipeline:
    - name: RegexEntityExtractor
      # text will be processed with case insensitive as default
      case_sensitive: False
      # use lookup tables to extract entities
      use_lookup_tables: True
      # use regexes to extract entities
      use_regexes: True
      # use match word boundaries for lookup table
      use_word_boundaries: True
    ```

### EntitySynonymMapper {#entitysynonymmapper}

- 概要

    将同义实体值映射到相同值。

- 输出

    修改先前实体提取组件找到的实体。

- 要求

    [实体提取器](components.md)中的一个提取器。

- 描述

    如果训练数据包含定义的同义词，该组件将确保检测到的实体值将映射到相同的值。例如，如果你的训练数据包含一下示例：

    ```json
    [
        {
          "text": "I moved to New York City",
          "intent": "inform_relocation",
          "entities": [{
            "value": "nyc",
            "start": 11,
            "end": 24,
            "entity": "city",
          }]
        },
        {
          "text": "I got a new flat in NYC.",
          "intent": "inform_relocation",
          "entities": [{
            "value": "nyc",
            "start": 20,
            "end": 23,
            "entity": "city",
          }]
        }
    ]
    ```

    该组件允许将实体 `New York City` 和 `NYC` 映射到 `nyc`。即使消息包含 `NYC`，实体提取也会返回 `nyc`。当此组件更改现有实体时，它会将自身附加到此实体的处理器列表中。

- 配置

    ```yaml
    pipeline:
    - name: "EntitySynonymMapper"
    ```

    !!! info "注意"

        当使用 `EntitySynonymMapper` 作为 NLU 流水线的一部分时，需要将其放置在配置文件中的任何实体提取器下方。

## 组合的意图分类器和实体提取器 {#combined-intent-classifiers-and-entity-extractors}

### DIETClassifier {#dietclassifier-2}

- 概要

    多意图实体 Transformer（Dual Intent Entity Transformer，DIET）用于意图分类和实体提取。

- 输出

    `entities`，`intent` 和 `intent_ranking`。

- 要求

    用户消息和可选意图的 `dense_features` 和/或 `sparse_features`。

- 输出示例

    ```json
    {
        "intent": {"name": "greet", "confidence": 0.7800},
        "intent_ranking": [
            {
                "confidence": 0.7800,
                "name": "greet"
            },
            {
                "confidence": 0.1400,
                "name": "goodbye"
            },
            {
                "confidence": 0.0800,
                "name": "restaurant_search"
            }
        ],
        "entities": [{
            "end": 53,
            "entity": "time",
            "start": 48,
            "value": "2017-04-10T00:00:00.000+02:00",
            "confidence": 1.0,
            "extractor": "DIETClassifier"
        }]
    }
    ```

- 描述

    参见[上文](#dietclassifier)。

- 配置

    参见[上文](#dietclassifier)。

## 选择器 {#selectors}

选择器从一组候选响应中预测一个对话机器人响应。

### ResponseSelector {#responseselector}

- 概要

    响应选择器。

- 输出

    一个字典，键作为响应选择器的检索意图，值包含预测的响应、置信度和检索意图下的响应键。

- 要求

    用户消息和响应的 `dense_features` 和/或 `sparse_features`。

- 输出示例

    NLU 的解析输出将具有一个名为 `response_selector` 的属性，其中包含每个响应选择器组件的输出。每个响应选择器由该响应选择器的检索意图参数表示并存储两个属性：

    - `response`：对应检索意图下的预测响应键、预测的置信度和相关响应。
    - `ranking`：前 10 个候选响应键的置信度排名。

    示例结果：

    ```json
    {
        "response_selector": {
          "faq": {
            "response": {
              "id": 1388783286124361986,
              "confidence": 0.7,
              "intent_response_key": "chitchat/ask_weather",
              "responses": [
                {
                  "text": "It's sunny in Berlin today",
                  "image": "https://i.imgur.com/nGF1K8f.jpg"
                },
                {
                  "text": "I think it's about to rain."
                }
              ],
              "utter_action": "utter_chitchat/ask_weather"
            },
            "ranking": [
              {
                "id": 1388783286124361986,
                "confidence": 0.7,
                "intent_response_key": "chitchat/ask_weather"
              },
              {
                "id": 1388783286124361986,
                "confidence": 0.3,
                "intent_response_key": "chitchat/ask_name"
              }
            ]
          }
        }
    }
    ```

    如果一个特定响应选择器的 `retrieval_intent` 参数保留其默认值，则相应的响应选择器将在返回的输出中被标识为 `default`。

    ```json hl_lines="3"
    {
        "response_selector": {
          "default": {
            "response": {
              "id": 1388783286124361986,
              "confidence": 0.7,
              "intent_response_key": "chitchat/ask_weather",
              "responses": [
                {
                  "text": "It's sunny in Berlin today",
                  "image": "https://i.imgur.com/nGF1K8f.jpg"
                },
                {
                  "text": "I think it's about to rain."
                }
              ],
              "utter_action": "utter_chitchat/ask_weather"
            },
            "ranking": [
              {
                "id": 1388783286124361986,
                "confidence": 0.7,
                "intent_response_key": "chitchat/ask_weather"
              },
              {
                "id": 1388783286124361986,
                "confidence": 0.3,
                "intent_response_key": "chitchat/ask_name"
              }
            ]
          }
        }
    }
    ```

- 描述

    响应选择器组件可用于构建响应检索模型，来直接从一组候选响应中预测对话机器人响应。对话管理器使用该模型的预测来发出预测的响应。它将用户输入和响应标签嵌入到相同的空间中，并遵循与 `DIETClassifier` 完全相同的神经网络架构和优化。

    要使用此组件，训练数据应包含[检索意图](glossary.md#retrieval-intent)。要定义这些，请查看有关 [NLU 训练样本的文档](training-data-format.md#training-examples)和有关[为检索意图定义响应消息的文档](responses.md#defining-responses)。

    !!! info "注意"

        在预测期间，如果消息仅包含训练期间为见过的单词，并且没有使用 Out-Of-Vocabulary 预处理器，则会以置信度为 `0` 的 `None` 空意图作为预测结果。如果仅将 [CountVectorsFeaturizer](components.md#countvectorsfeaturizer) 与 `word` 分析器一起用作特征化器，则可能发生这种情况。如使用 `char_wb` 分析器，你可以始终获得置信度大于 `0.0` 的意图。

- 配置

    该算法几乎包括了 `DIETClassifier` 使用的所有超参数，具体参见[上文](#dietclassifier)。

    该组件还可以配置为针对特定检索意图训练响应选择器。参数 `retrieval_intent` 设置了训练此响应选择器模型的检索意图的名称。默认为 `None`，即模型针对所有检索意图进行训练。

    在其默认配置中，该组件使用带有响应键（例如：`faq/ask_name`）的检索意图作为训练的标签。或者，也可以通过将 `use_text_as_label` 切换为 `True` 来将其配置为使用响应的文本作为训练标签。在这种模式下，组件将使用具有文本属性的第一个可用响应进行训练。如果没有找到，则回退到使用检索意图和响应键作为标签。

    !!! info "示例和教程"

        查看 [responseselectorbot](https://github.com/RasaHQ/rasa/tree/main/examples/responseselectorbot) 作为一个示例来了解如何在对话机器人中使用 `ResponseSelector` 组件。此外，你会发现此教程在使用 `ResponseSelector` 来[处理 FAQs](chitchat-faqs.md) 也很有用。

## 自定义组件 {#custom-components}

!!! info "3.0 版本新特性"

    开源 Rasa 3.0 版本统一了 NLU 组件和策略的实现。这需要更改为早期版本的开源 Rasa 编写的自定义组件。有关逐步的迁移指南请参见[迁移指南](migration-guide.md#custom-policies-and-custom-components)。

你可以创建一个自定义组件来执行 NLU 目前不提供的特定任务（例如：情感分析）。

你可以通过添加模块路径将自定义组件添加到流水线中。因此，如果你有一个包含名为 `SentimentAnalyzer` 的类的模块：

```yaml
pipeline:
- name: "sentiment.SentimentAnalyzer"
```

有关自定义组件的完整指南，请参见[自定义图组件指南](custom-graph-components.md)。还请务必阅读有关[组件生命周期](tuning-your-model.md#component-lifecycle)的部分。
