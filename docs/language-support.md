# 语言支持

你可以使用 Rasa 来构建任何你需要语言的对话机器人。

你的 Rasa 对话机器人可使用任何语言的训练数据。如果你的语言没有词嵌入，可以使用提供的数据从头开始训练特征化器。

此外，我们还支持预训练的词嵌入，例如 spaCy。有关哪种管道最适合你的用例，请查看[选择管道](/tuning-your-model/#how-to-choose-a-pipeline)。

## 用任何语言训练模型 {#training-a-model-in-any-languages}

如下管道可用于以空白字符分词语言训练模型：

```yaml
language: "fr"  # your two-letter language code

pipeline:
  - name: WhitespaceTokenizer
  - name: RegexFeaturizer
  - name: LexicalSyntacticFeaturizer
  - name: CountVectorsFeaturizer
  - name: CountVectorsFeaturizer
    analyzer: "char_wb"
    min_ngram: 1
    max_ngram: 4
  - name: DIETClassifier
    epochs: 100
  - name: EntitySynonymMapper
  - name: ResponseSelector
    epochs: 100
```

要以首选语言训练 Rasa 模型，需要在 `config.yml` 中定义管道。在你定义管道并以选择的语言生成一些 [NLU 训练数据](/training-data-format/)后，通过运行如下命令来训练模型：

```shell
rasa train nlu
```

训练完成后，可以测试模型的语言性能。通过运行查看模型如何处理不同的输入消息：

```shell
rasa shell nlu
```

!!! note "注意"

    从头开始训练词嵌入更需如此，更多的训练数据可以获得更好的模型。如果发现模型无法识别你的输入，请尝试使用更多样本进行训练。

## 使用预训练语言模型 {#using-pre-trained-language-models}

如果可以在你的语言中找到，那么具有预训练词向量的语言模型是在开始具有较少数据时的好方法，因为词向量通常是在大量数据（例如 Wikipedia）上训练的。

### spaCy {#spacy}

通过[预训练的 Spacy 嵌入](/components/#spacynlp)，可以使用 spaCy 的[预训练语言模型](https://spacy.io/usage/models#languages)或加载适用于[数百种语言](https://github.com/facebookresearch/fastText/blob/master/docs/crawl-vectors.md)的 fastText 向量。如果想要将发现的自定义模型合并到 spaCy 中，请查看它们关于[添加语言](https://spacy.io/docs/usage/adding-languages)的页面。如文档所述，需要注册你的语言模型并将其链接到语言识别器，这将允许 Rasa 通过将你的语言识别器作为 `language` 选项传递来加载和使用新语言。

### MITIE {#mitie}

还可以使用 [MITIE](/components/#mitienlp) 从一个语言预料库中训练自己的词向量。为此：

- 获取一个干净的语言语料库（维基百科转储）作为一组文本文件。
- 在语料库上构建并运行 `MITIE Wordrep Tool`。这可能需要几小时/天，具体取决于你的数据集和工作站。你需要 128GB 的内存来运行 wordrep。是的，需要很多内存，可以尝试扩展 swap。
- 将新的 `total_word_feature_extractor.dat` 的路径设置为[配置](/components/#mitienlp)中的 `model` 参数。

有关如何训练 MITIE 词向量的完整示例，请查看这篇从中文维基百科转储中创建 MITIE 模型的[博文](http://www.crownpku.com/2017/07/27/%E7%94%A8Rasa_NLU%E6%9E%84%E5%BB%BA%E8%87%AA%E5%B7%B1%E7%9A%84%E4%B8%AD%E6%96%87NLU%E7%B3%BB%E7%BB%9F.html)。
