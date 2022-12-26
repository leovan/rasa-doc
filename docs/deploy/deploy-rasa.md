# 部署 Rasa

本页面介绍如何使用 Helm 部署开源 Rasa。

!!! note "注意"

    Rasa Helm Chart 是开源的，位于 [helm-charts 仓库](https://github.com/rasahq/helm-charts)中。如果你发现任何错误或有改进建议，可以在次仓库中[创建 issue](https://github.com/RasaHQ/helm-charts/issues/new)。

## 安装依赖 {#installation-requirements}

1. 检查你是否已安装 Kubernetes 或 OpenShift 命令行界面（CLI）。你可以使用如下命令进行检查：

    === "Kubernets"

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

    如果命令报错，请根据你使用的集群安装 [Kubernets CLI](https://kubernetes.io/docs/tasks/tools/install-kubectl/) 或 [OpenShift CLI](https://docs.openshift.com/container-platform/4.7/cli_reference/openshift_cli/getting-started-cli.html#installing-openshift-cli)。

2. 确保 Kubernetes / OpenShift CLI 可以正确连接到你的集群。可以使用如下命令执行此操作：

    === "Kubernets"

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

    如果执行命令报错，则说明你未连接到集群。要获取连接到集群的命令，请咨询你的集群管理员或参见云服务提供商的文档。

3. 确保你已安装 [Helm CLI](https://helm.sh/docs/intro/install/)。要检查这一点，请运行：

    ```shell
    helm version --short

    # The output should be similar to this
    # v3.6.0+g7f2df64
    ```

    如果命令报错，请安装 [Helm CLI](https://helm.sh/docs/intro/install/)。

    如果你使用的 Helm 版本 `< 3.5`，请更新到 `> 3.5` 版本的 Helm。

## 安装 {#installation}

### 创建命名空间 {#create-namespace}

我们建议将开源 Rasa 安装到单独的命名空间中，以避免干扰现有的集群部署。要创建新的[命名空间](https://kubernetes.io/docs/concepts/overview/working-with-objects/namespaces/)，请运行如下命令：

=== "Kubernets"

    ```shell
    kubectl create namespace <your namespace>
    ```

=== "OpenShift"

    ```shell
    oc create namespace <your namespace>
    ```

### 创建值文件 {#create-values-file}

准备一个名为 `rasa-values.yml` 的空文件，其中将包含使用 Helm 进行安装的所有自定义配置。

你可以在 [Rasa Helm Chart 仓库](https://github.com/RasaHQ/helm-charts/tree/main/charts/rasa#values)中找到所有可用的值。

!!! note "注意"

    Rasa chart 的默认配置会部署一个开源 Rasa 服务器，下载一个模型，并为下载的模型提供服务。访问 [Rasa Helm Chart 仓库](https://github.com/RasaHQ/helm-charts/tree/main/charts/rasa#quick-start)来获取更多配置示例。

### 加载初始模型 {#loading-an-initial-model}

第一次安装 Rasa 时，你可能还没有可用的模型服务器，或者你可能需要一个轻量级模型来测试部署。为此，你可以选择训练或下载初始模型，默认情况下，Rasa chart 从 Github 下载示例模型。使用这个配置不需要改变任何内容。

如果你想定义现有模型来从指定的 URL 下载，请根据以下配置使用使用 URL 更新 `rasa-values.yml`：

```yaml
applicationSettings:
  initialModel: "https://github.com/RasaHQ/rasa-x-demo/blob/master/models/model.tar.gz?raw=true"
```

!!! note "注意"

    初始模型下载的 URL 必须指向一个 tar.gz 文件，同时不能要求身份验证。

如果要训练初始模型，你可以通过将 `applicationSettings.trainInitialModel` 设置为 `true` 来实现。它将初始化容器并根据 `/app` 目录中的数据训练一个模型。如果 `/app` 目录为空，则会创建一个新项目。你可以在 Rasa Helm Charts 示例中找到如何从一个 git 仓库中下载数据文件并训练初始模型的[示例](https://github.com/RasaHQ/helm-charts/blob/main/examples/rasa/train-model-helmfile.yaml)。

### 部署开源 Rasa 对话机器人 {#deploy-rasa-assistant}

运行如下命令：

```shell
# Add the repository which contains the Rasa Helm Chart
helm repo add rasa https://helm.rasa.com

# Deploy Rasa Open Source
helm install \
    --namespace <your namespace> \
    --values rasa-values.yml \
    <release name> \
    rasa/rasa
```

!!! note "注意"

    仅适用于 OpenShift：如果部署失败并且 `oc get events` 返回 `1001 is not an allowed group spec.containers[0].securityContext.securityContext.runAsUser`，请使用如下值重新安装命令：

    ```yaml
    postgresql:
      volumePermissions:
        securityContext:
        runAsUser: "auto"
      securityContext:
        enabled: false
      shmVolume:
        chmod:
        enabled: false
    nginx:
      image:
        name: nginxinc/nginx-unprivileged
        port: 8080
    ```

    然后等待部署就绪。如果要检查状态，如下命令将阻塞直至 Rasa 部署就绪：

    === "Kubernetes"

        ```shell
        kubectl --namespace <your namespace> \
            wait \
            --for=condition=available \
            --timeout=20m \
            --selector app.kubernetes.io/instance=<release name> \
            deployment
        ```

    === "OpenShift"

        ```shell
        oc --namespace <your namespace> \
            wait \
            --for=condition=available \
            --timeout=20m \
            --selector app.kubernetes.io/instance=<release name> \
            deployment
        ```

### 访问开源 Rasa 对话机器人 {#access-rasa-assistant}

默认情况下，Rasa 部署通过 `rasa (<release name>)` 公开服务，并且只能在 Kebernetes 集群中访问。要使用 `kubectl port-forward` 访问开源 Rasa 对话机器人，请使用如下命令：

=== "Kubernetes"

    ```shell
    export SERVICE_PORT=$(oc get --namespace <your namespace> -o jsonpath="{.spec.ports[0].port}" services <release name>)
    oc port-forward --namespace <your namespace> svc/<release name> ${SERVICE_PORT}:${SERVICE_PORT} &
    ```

=== "OpenShift"

    ```shell
    export SERVICE_PORT=$(oc get --namespace <your namespace> -o jsonpath="{.spec.ports[0].port}" services <release name>)
    oc port-forward --namespace <your namespace> svc/<release name> ${SERVICE_PORT}:${SERVICE_PORT} &
    ```

然后可以访问 `http://127.0.0.1:${SERVICE_PORT}` 上的部署。

另一种选择是在 `NodePort` 上公开部署并直接访问它。

1. 准备将 Rasa 服务切换到 NodePort 的配置。

    ```yaml
    # rasa-values.yaml
    service:
      type: "NodePort"
    ```

2. 更新部署。

    ```shell
    helm upgrade --namespace <NAMESPACE> --reuse-values -f rasa-values.yaml <RELEASE NAME> rasa/rasa
    ```

3. 获取 Rasa 服务的节点端口和地址。

    ```shell
    export NODE_PORT=$(kubectl get --namespace <NAMESPACE> -o jsonpath="{.spec.ports[0].nodePort}" services <RELEASE NAME>)

    $ curl http://127.0.0.1:${NODE_PORT}
    Hello from Rasa: 2.8.7
    ```

访问 [Rasa Helm Chart README](https://github.com/RasaHQ/helm-charts/tree/main/charts/rasa#exposing-the-rasa-deployment-to-the-public) 来了解其他公开部署的方法。

## 下一步

访问 [Rasa Helm Chart 仓库](https://github.com/RasaHQ/helm-charts/tree/main/charts/rasa)，你可以在其中找到配置示例。
