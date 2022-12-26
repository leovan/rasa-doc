# 部署 Rasa 动作服务器

本页介绍如何使用 Helm 构建 Action Server 镜像和部署 Rasa Action Server。部署 Rasa 对话机器人的三个步骤中的第二个。

!!! note "注意"

    [Rasa Action Server chart](https://github.com/RasaHQ/helm-charts/tree/main/charts/rasa-action-server) 为开源项目，可以在 [helm-charts 代码库](https://github.com/rasahq/helm-charts)中找到。如果发现任何问题或有改进建议，请在此代码库中[创建 issue](https://github.com/RasaHQ/helm-charts/issues/new)。

在如下章节中，你可以了解如何使用 helm 部署 Rasa Action Server，以及如何将 Action Server 部署连接到 Rasa 部署。

## 安装依赖 {#installation-requirements}

1. 检查是否已安装 Kubernetes 或 Openshift 命令行界面（CLI）。你可以使用如下命令进行检查：

    === "Kubernetes"

        ```shell
        kubectl version --short --client

        # The output should be similar to this
        # Client Version: v1.19.11
        ```

    === "OpenShift"

        ```shell
        oc version --client

        # The output should be similar to this
        # Client Version: 4.7.13
        ```

    如果此命令导致错误，请根据使用的集群安装 [Kubernetes CLI](https://kubernetes.io/docs/tasks/tools/install-kubectl/) 或 [OpenShift CLI](https://docs.openshift.com/container-platform/4.7/cli_reference/openshift_cli/getting-started-cli.html#installing-openshift-cli)。

2. 确保 Kubernetes / OpenShift CLI 正确连接到你的集群。可以使用如下命令执行此操作：

    === "Kubernetes"

        ```shell
        kubectl version --short

        # The output should be similar to this
        # Client Version: v1.19.11
        # Server Version: v1.19.10
        ```

    === "OpenShift"

        ```shell
        oc version

        # The output should be similar to this
        # Client Version: 4.7.13
        # Kubernetes Version: v1.20.0+df9c838
        ```

    如果执行命令时出现错误，则说明你未连接到集群。要获取连接到集群的命令，请咨询集群管理员或云提供商的文档。

3. 确保你已安装 [Helm CLI](https://helm.sh/docs/intro/install/)。要检查这一点，请运行：

    ```shell
    helm version --short

    # The output should be similar to this
    # v3.6.0+g7f2df64
    ```

    如果此命令导致错误，请安装 [Helm CLI](https://helm.sh/docs/intro/install/)。

    如果你使用的 Helm 版本 < 3.5，请更新到 Helm 版本 >= 3.5。

## 安装 {#installation}

### 创建空间 {#create-namespace}

我们建议将 Rasa Action Server 安装在单独的命名空间中，以避免干扰现有的集群部署。要创建新的命名空间，请运行如下命令：

=== "Kubernetes"

    ```shell
    kubectl create namespace <your namespace>
    ```

=== "OpenShift"

    ```shell
    oc create namespace <your namespace>
    ```

### 部署 Rasa Action Server {#deploy-rasa-action-server}

运行如下命令：

```shell
# Add the repository which contains the Rasa Action Server Helm chart
helm repo add rasa https://helm.rasa.com

# Deploy Rasa Action Server
helm install \
    --namespace <your namespace> \
    <release name> \
    rasa/rasa-action-server
```

### 访问 Rasa Action Server {#access-rasa-action-server}

默认情况下，Rasa Action Server 部署通过 `rasa-action-server (<release name>)` 服务公开，并且只能在 Kubernetes 集群中访问。你可以使用如下命令获取 IP 地址：

=== "Kubernetes"

    ```shell
    export SERVICE_PORT=$(kubectl get --namespace <your namespace> -o jsonpath="{.spec.ports[0].port}" services <release name>)
    kubectl port-forward --namespace <your namespace> svc/<release name> ${SERVICE_PORT}:${SERVICE_PORT} &
    ```

=== "OpenShift"

    ```shell
    export SERVICE_PORT=$(oc get --namespace <your namespace> -o jsonpath="{.spec.ports[0].port}" services <release name>)
    oc port-forward --namespace <your namespace> svc/<release name> ${SERVICE_PORT}:${SERVICE_PORT} &
    ```

然后可以通过 `http://127.0.0.1:${SERVICE_PORT}` 访问部署。

!!! note "注意"

    Rasa Action Server Helm chart 默认使用 rasa-x-demo Docker 镜像。在构建 Action Server 镜像中，你可以了解如何构建和使用自定义 Action Server 镜像。

## 构建 Action Server 镜像 {#building-an-action-server-image}

如果你构建包含动作代码的镜像并将其存储在容器注册表中，可以将其作为部署的一部分运行，无需在服务器之间移动代码。此外，可以添加系统或 Python 库的任何其他依赖，这些依赖是动作代码的一部分，但未包含在基础 `rasa/rasa-sdk` 镜像中。

### 自动化动作服务器镜像构建 {#automating-your-action-server-image-builds}

除了手动创建新的 Action Server 镜像之外，还可以使用 [Rasa Action Server Github Action](https://github.com/RasaHQ/action-server-gha) 来自动构建镜像。如果你不熟悉 Github Actions，可以通过 [Github Actions 文档](https://docs.github.com/en/actions)来获取帮助。

如下步骤假设你已经创建了一个 Github 代码库并且你有一个 DockerHub 账户。

要创建用于构建 Docker 镜像并将其推送到 DockerHub 注册表的工作流：

1. 使用你的 DockerHub 用户名和密码添加 Github Secrets。可以在 [Github 文档](https://docs.github.com/en/actions/configuring-and-managing-workflows/creating-and-storing-encrypted-secrets#creating-encrypted-secrets-for-a-repository)中找到有关如何为代码库创建加密 secrets 的详细信息。

    该示例使用如下 secrets：

    `DOCKER_HUB_LOGIN`：DockerHub 用户名
    `DOCKER_HUB_PASSWORD`：DockerHub 密码

2. 在你的 Github 存储库中创建一个文件 `.github/workflows/action_server.yml`。

每当 `actions/` 目录中的文件发生更改并将更改推送到 `main` 分支时，下面的 GitHub Action 工作流程都会创建一个新的 Docker 镜像。

```yaml
on:
  push:
    branches:
      - main
    paths:
    - 'actions/**'

jobs:
  build_and_deploy:
    runs-on: ubuntu-latest
    name: Build Action Server image and upgrade Rasa X/Enterprise deployment
    steps:
    - name: Checkout repository
      uses: actions/checkout@v2

    - id: action_server
      name: Build an action server with custom actions
      uses: RasaHQ/action-server-gha@main
      # Full list of parameters: https://github.com/RasaHQ/action-server-gha/tree/master#input-arguments
      with:
        docker_image_name: 'account_username/repository_name'
        docker_registry_login: ${{ secrets.DOCKER_HUB_LOGIN }}
        docker_registry_password: ${{ secrets.DOCKER_HUB_PASSWORD }}
        # More details about github context:
        # https://docs.github.com/en/actions/reference/context-and-expression-syntax-for-github-actions#github-context
        #
        # github.sha - The commit SHA that triggered the workflow run
        docker_image_tag: ${{ github.sha }}
```

1. 将更改推送到 `main` 分支。推送更改后，工作流将构建新镜像并将其推送到 DockerHub 注册表中。
2. 现在，可以使用新的 Docker 镜像。
3. 还可以扩展工作流，这样就不必手动更新你的 Rasa X/Enterprise 部署。如下示例显示了如何通过更新 Rasa X/Enterprise Helm Chart 部署的附加步骤来扩展工作流。

```yaml
on:
  push:
    branches:
      - main

jobs:
  build_and_deploy:
    runs-on: ubuntu-latest
    name: Build Action Server image and upgrade Rasa X/Enterprise deployment
    steps:
    [..]

    # This step shows only the example of output parameter usage
    # and it's not focused on deployment itself.
    - name: "Upgrade a Rasa Action Server deployment"
      run: |
        helm upgrade --install --reuse-values \
          --set image.name=${{ steps.action_server.outputs.docker_image_name }} \
          --set image.tag=${{ steps.action_server.outputs.docker_image_tag }} rasa-action-server rasa/rasa-action-server
```

如你所见，可以使用 `action_server` 步骤中的输出变量。`steps.action_server.outputs.docker_image_name` 变量返回一个 Docker 镜像名称，而 `steps.action_server.outputs.docker_image_tag` 变量返回一个 Docker 镜像标签。

有关如何使用和自定义 [Rasa GitHub Actions](https://github.com/RasaHQ/action-server-gha) 的更多示例，可以在 [Rasa GitHub Actions](https://github.com/RasaHQ/action-server-gha) 代码库中找到。

### 手动构建 Action Server {#manually-building-an-action-server}

要创建你的镜像：

1. 确保动作在 `actions/actions.py` 中定义。`rasa/rasa-sdk` 镜像将自动查找此文件中的动作。
2. 如果动作有任何额外的依赖，请在 `actions/requirements-actions.txt` 中创建相关列表。
3. 在项目目录中创建一个名为 Dockerfile 的文件，在其中扩展官方 SDK 镜像、复制代码并添加任何自定义依赖项（如有必要）。例如：

```yaml
# Extend the official Rasa SDK image
FROM rasa/rasa-sdk:3.2.1

# Use subdirectory as working directory
WORKDIR /app

# Copy any additional custom requirements, if necessary (uncomment next line)
# COPY actions/requirements-actions.txt ./

# Change back to root user to install dependencies
USER root

# Install extra requirements for actions code, if necessary (uncomment next line)
# RUN pip install -r requirements-actions.txt

# Copy actions folder to working directory
COPY ./actions /app/actions

# By best practices, don't run the code with root user
USER 1001
```

然后，可以通过如下命令构建镜像：

```shell
docker build . -t <account_username>/<repository_name>:<custom_image_tag>
```

`<custom_image_tag>` 应该引用此镜像与其他镜像的不同之处。例如，可以对标签进行版本化或日期化，以及为生产和开发服务器创建具有不同代码的不同标签。每当更新代码并想要重新部署它时，都应该创建一个新标签。

### 使用自定义动作服务器镜像 {#using-your-custom-action-server-image}

如果你正在构建此镜像来使其可以从另一台服务器使用，则应将镜像推送到云存储库。

本文档假设你将镜像推送到 [DockerHub](https://hub.docker.com/)。DockerHub 允许你免费托管多个公共存储库和一个私有存储库。请务必先[创建一个账户](https://hub.docker.com/signup/)并[创建一个存储库](https://hub.docker.com/signup/)来存储你的镜像。还可以将镜像推送到不同的 Docker 注册表，例如 [Google Container Registry](https://cloud.google.com/container-registry)、[Amazon Elastic Container Registry](https://aws.amazon.com/ecr/) 或 [Azure Container Registry](https://azure.microsoft.com/en-us/services/container-registry/)。

你可以通过如下方法将镜像推送到 DockerHub：

```shell
docker login --username <account_username> --password <account_password>
docker push <account_username>/<repository_name>:<custom_image_tag>
```

要对镜像进行身份验证并将其推送到不同的容器注册表，请参阅你选择的容器注册表文档。

## 设置自定义动作服务器镜像 {#setting-a-custom-action-server-image}

为了将自定义动作服务器镜像与 Rasa 动作服务器部署一起使用，你必须为部署使用如下值。

```yaml
# values.yaml
image:
  name: "image_name"
  tag: "image_tag"
```

然后通过执行如下命令升级部署：

```shell
helm upgrade --namespace <namespace> --reuse-values \
  -f values.yaml <release name> rasa/rasa-action-server
```

## 将 Rasa Action Server 连接到 Rasa 部署 {#connect-rasa-action-server-to-a-rasa-deployment}

如果你已经使用 Rasa Helm chart 部署了对话机器人，并且还部署了 Rasa Action Server。现在是时候将它们连接在一起了。你可以按照如下步骤轻松完成此操作：

1. 创建一个 `rasa-values.yaml` 文件，其中包含 Rasa 部署的配置。

    ```yaml
    # rasa-values.yaml
    rasa-action-server:
      external:
        # -- Determine if external URL is used
        enabled: true
        # -- URL to Rasa Action Server
        url: "http://rasa-action-server/webhook"
    ```

    上述配置告诉 Rasa 使用哪个 Rasa Action Server。在示例中使用了 `http://rasa-action-server/webhook` URL，该 URL 表示 Rasa Action Server 以 `rasa-action-server` 名称进行部署，并且与 Rasa 服务器部署位于相同的命名空间中。

2. 升级 Rasa 部署。

    ```shell
    helm upgrade -n <namespace> --reuse-values -f rasa-values.yaml \
      <release name> rasa/rasa
    ```
