# 部署 Rasa 对话机器人

本节说明何时以及如何部署一个由 Rasa 构建的对话机器人。这会向用户展示提供你的对话机器人，并为此设置一个生产就绪的环境。

!!! note "注意"

    你是否对 Docker、Kubernetes 和 Helm 不熟悉？在 [YouTube 频道](https://www.youtube.com/channel/UCJ0V6493mLvqdiVwOKWBODQ)上查看 [Understanding Rasa Deployments](https://www.youtube.com/watch?v=aAs_RS0ueEw&list=PL75e0qA87dlHmfmu7oPPYA22fmc6GJ2aW)。

## 何时部署对话机器人 {#when-to-deploy-your-assistant}

部署对话机器人并使其可用于测试用户的最佳时间是当它可以处理最重要的预期路径，或者我们称之为[最小可行对话机器人](/glossary/)时。然后，你可以使用传入的对话来进一步开发对话机器人。

## 推荐的部署方式 {#recommended-deployment-method}

[Rasa Helm Chart](https://github.com/RasaHQ/helm-charts/tree/main/charts/rasa) 是在 Kubernetes 或 Openshift 集群上进行生产部署的方法。更多详细信息，请参见[部署说明](/deploy/deploy-rasa/)。

### 集群要求 {#cluster-requirements}

要安装 Rasa Helm Chart，你需要一个 Kubernetes 或 OpenShift 集群。如果没有，你可以从云服务提供商处获取一个托管集群，例如：

- [Google Cloud](https://cloud.google.com/kubernetes-engine)
- [DigitalOcean](https://www.digitalocean.com/products/kubernetes/)
- [Microsoft Azure](https://azure.microsoft.com/en-us/services/kubernetes-service/)
- [Amazon EKS](https://aws.amazon.com/eks/)

## 替代部署方法 {#alternative-deployment-methods}

以下部署方法不适合生产环境部署，但可以用于开发和测试：

- [在命令行中本地运行对话机器人](/command-line-interface/#rasa-run)
- [在 Docker 容器中开发对话机器人](/docker/building-in-docker/)
- [使用 Docker Compose 部署对话机器人](/docker/deploying-in-docker-compose/)
