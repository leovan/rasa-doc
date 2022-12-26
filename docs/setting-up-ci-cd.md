# 设置 CI/CD

即使开发上下文对话机器人与开发传统软件不同，你仍应遵循软件开发最佳实践。设置持续集成（CI）和持续部署（CD）管道可以确保对机器人的增量更新是在改进它，而非损害它。

## 概述 {#overview}

持续集成（CI）是频繁合并代码更改并在提交更改时自动测试更改的做法。持续部署（CD）意味着将集成更改自动部署到预发或生产环境。它们一起使你可以更加频繁地改进你的对话机器人，并有效地测试和部署这些更改。

本指南将介绍特定于 Rasa 项目的 CI/CD 管道中应该包含的内容。如何实现这个管道取决于你。有很多 CI/CD 工具，例如 [Github Actions](https://github.com/features/actions)、[Gitlab CI/CD](https://docs.gitlab.com/ee/ci/)、[Jenkins](https://www.jenkins.io/doc/) 和 [CircleCI](https://circleci.com/docs/2.0/)。我们建议选择与你使用的 Git 仓库集成的工具。

## 持续集成（CI） {#continuous-integration-ci}

改进对话机器人的最佳方法是频繁地进行增量更新。无论更改有多小，你都希望确保它不会引入新问题或对对话机器人的性能产生负面影响。

通常最好在合并/拉取请求或提交代码时运行 CI 检查。大多数测试在每次更改时运行的足够快。同时，你可以选择仅在某些文件已更改或存在其他标识符时才运行资源密集型测试。例如，如果你的代码托管在 Github 上，则只有在拉取请求具有特定标签（例如“NLU testing required”）时才进行测试。

### CI 管道概述 {#ci-pipeline-overview}

你的 CI 管道应该包括模型训练和测试，来简化部署过程。保存新训练数据后的第一步是启动管道。这可以手动启动，也可以在创建或更新拉取请求时启动。

接下来，你需要于运行多组测试来查看更改的影响。这包括运行数据验证测试、NLU 交叉验证和故事测试。有关测试的更多信息，请参见[测试对话机器人](/testing-your-assistant)。

最后一步是查看测试结果并在测试成功时推送变更。一旦新模型经过训练和测试，就可以使用持续部署管道自动部署了。

### GitHub Actions CI 管道 {#github-actions-ci-pipeline}

你可以在 CI 管道中使用 [Rasa Train-Test Github Action](https://github.com/RasaHQ/rasa-train-test-gha) 来自动执行数据验证、训练和测试。

一个使用 Github Action 的示例 CI 管道如下：

```yaml
jobs:
  training-testing:
    name: Training and Testing
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v1
      - name: Rasa Train and Test GitHub Action
        uses: RasaHQ/rasa-train-test-gha@main
        with:
          requirements_file: requirements.txt
          data_validate: true
          rasa_train: true
          cross_validation: true
          rasa_test: true
          test_type: all
          publish_summary: true
          github_token: ${{ secrets.GITHUB_TOKEN }}
      - name: Upload model
        if: github.ref == 'refs/heads/main'
        uses: actions/upload-artifact@master
        with:
          name: model
          path: models
```

在这个管道中，Rasa Train-Test Github Action 在第一步中执行数据验证、模型训练和故事测试，在第二步中将模型作为制品上传。

Rasa Train-Test Github Action 的可配置参数的完整列表可在仓库的 [README](https://github.com/RasaHQ/rasa-train-test-gha#input-arguments) 中找到。

当 `publish_summary` 设置为 `true` 时，此操作会自动将模型的测试结果作为评论发布到关联的拉取请求中：

<figure markdown>
  ![](/images/setting-up-ci-cd/train-test-github-action.png){ width="600" }
  <figcaption>Rasa Train-Test Github Action</figcaption>
</figure>

根据评估结果可以批准或拒绝拉取请求，并在很多情况下，如果所有 CI 检查都通过，则希望自动部署模型。你可以继续阅读下一部分来了解有关持续部署的更多信息。

## 持续部署（CD） {#continuous-deployment-cd}

为了可以频繁的向用户提供改进，你希望可以尽可能多的自动化进行部署。

一旦 CI 检查成功，CD 通常会在推送或合并到某个分支时运行。

### 部署 Rasa 模型 {#deploying-your-rasa-model}

如果在 CI 管道中运行[测试故事](/testing-your-assistant)，你应该已经有一个训练好的模型。如果 CI 的结果令人满意，可以设置 CD 管道将经过训练的模型上传到 Rasa 服务器。例如，将模型上传到 Rasa X/Enterprise：

```shell
curl -k -F "model=@models/my_model.tar.gz" "https://example.rasa.com/api/projects/default/models?api_token={your_api_token}"
```

如果你使用 Rasa X/Enterprise，你还可以将上传的模型标记为生产（或者如果使用多个部署环境，可以标记为任何部署环境）：

```shell
curl -X PUT "https://example.rasa.com/api/projects/default/models/my_model/tags/production"
```

!!! caution "动作代码更新"

    如果你的更新包括对模型和动作代码的更改，并且这些更改相互依赖，则不应自动将模型标记为 `production`。你需要首先构建和部署更新的动作服务，以便新的模型不会调用更新前动作服务中不存在的动作。

### 部署动作服务 {#deploying-your-action-server}

你可以[为动作服务自动构建和上传一个新的镜像](https://rasa.com/docs/action-server/deploy-action-server#building-an-action-server-image)至镜像仓库来更新动作代码。如上所述，如果动作服务与当前生产模型不兼容，请小心自动将新的镜像部署到生产中。

## 示例 CI/CD 管道 {#example-cicd-pipelines}

作为示例，可以参见用于 [Rasa](https://github.com/RasaHQ/rasa-demo/blob/main/.github/workflows/continuous_integration.yml) 的 CI/CD 管道，Rasa 文档中的 Rasa 对话机器人，以及 [Carbon Bot](https://github.com/RasaHQ/carbon-bot/blob/master/.github/workflows/model_ci.yml)。这些均使用 [Github Actions](https://github.com/features/actions) 作为 CI/CD 工具。

这些示例仅为众多示例中的个别。如果你有自己喜欢的 CI/CD 配置，可以在 Rasa 社区[论坛](https://forum.rasa.com/) 上进行分享。
