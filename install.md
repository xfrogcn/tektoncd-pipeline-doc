# 安装Tekton管道

本指南介绍如何安装Tekton管道，包含以下主题：

* [开始之前](#开始之前)
* [在Kubernetes集群中安装Tekton管道](#在Kubernetes集群中安装Tekton管道)
* [在OpenShift中安装Tekton管道](#在OpenShift中安装Tekton管道)
* [配置物料存储](#配置物料存储)
* [自定义基础执行参数](#自定义基础执行参数)
* [创建自定义Tekton管道版本](#创建自定义Tekton管道版本)
* [下一步](#下一步)

## 开始之前

1. 选择你想要安装的Tekton管道版本:

   * **[`正式版`](https://github.com/tektoncd/pipeline/releases)** - 在没有其他特定原因时可选择安装此版本.
   * **[`每夜构建`](../tekton/README.md#nightly-releases)** - 每日构建的非稳定版本，可能包含bug，镜像为`gcr.io/tekton-nightly`.
   * **[`HEAD`]**   这是最新的代码，它包含未发布的代码，可能产生无法预料的行为，请参考 [开发指南](https://github.com/tektoncd/pipeline/blob/master/DEVELOPMENT.md).

2. 如果你还没有现成的Kubernetes集群，请先安装，要求为1.15及以上版本:

   ```bash
   #在GKE上创建集群的命令示例
   gcloud container clusters create $CLUSTER_NAME \
     --zone=$CLUSTER_ZONE --cluster-version=1.15.11-gke.5
   ```

3. 为当前用户赋予 `cluster-admin` 权限:

   ```bash
   kubectl create clusterrolebinding cluster-admin-binding \
   --clusterrole=cluster-admin \
   --user=$(gcloud config get-value core/account)
   ```

   请参考 [基于角色的访问控制](https://cloud.google.com/kubernetes-engine/docs/how-to/role-based-access-control#prerequisites_for_using_role-based_access_control).

## 在Kubernetes集群中安装Tekton管道

要在Kubernetes集群中安装Tekton，需要:

1. 运行以下命令安装Tekton管道及其依赖:

   ```bash
   kubectl apply --filename https://storage.googleapis.com/tekton-releases/pipeline/latest/release.yaml
   ```
   你可以通过指定 `previous/$VERSION_NUMBER`来安装特定的版本，例如:

   ```bash
    kubectl apply --filename https://storage.googleapis.com/tekton-releases/pipeline/previous/v0.2.0/release.yaml
   ```

   如果你的容器运行时不支持 `image-reference:tag@digest`
   (例如, 像在OpenShift 4.x中使用的 `cri-o` ), 那么使用 `release.notags.yaml` 文件来代替:

   ```bash
   kubectl apply --filename https://storage.googleapis.com/tekton-releases/pipeline/latest/release.notags.yaml
   ```

1. 通过以下命令监控安装的进度，直到所有的Pod都变成运行状态:

   ```bash
   kubectl get pods --namespace tekton-pipelines --watch
   ```

   **注意:** 通过CTRL+C来停止监控.

恭喜! 你已经成功地在Kubernetes集群中安装了Tekton管道。接下来，你可查看以下主题:

* [配置物料存储](#配置物料存储) 来配置Tekton管道的存储.
* [自定义基础执行参数](#自定义基础执行参数) 如果你需要自定义你的服务账户，超时时间或Pod模板值的话，请参考此节内容。

### 在OpenShift中安装Tekton管道

要在OpenShift集群中安装Tekton，你首先需要应用`anyuid`安全上下文到`tekton-pipelines-controller`服务账户。这是运行webhook Pod的必须条件。
参考
[安全上下文约束](https://docs.openshift.com/container-platform/4.3/authentication/managing-security-context-constraints.html)
查看更多信息。

1. 以`cluster-admin`权限登录，以下使用默认的`system:admin` 用户:

   ```bash
   # For MiniShift: oc login -u admin:admin
   oc login -u system:admin
   ```

1. 设置命令空间（项目）以及配置其服务账户:

   ```bash
   oc new-project tekton-pipelines
   oc adm policy add-scc-to-user anyuid -z tekton-pipelines-controller
   ```
1. 安装Tekton管道:

   ```bash
   oc apply --filename https://storage.googleapis.com/tekton-releases/pipeline/latest/release.notags.yaml
   ```
   参考
   [OpenShift命令行文档](https://docs.openshift.com/container-platform/4.3/cli_reference/openshift_cli/getting-started-cli.html)
   查看有关 `oc` 命令的说明.

1. 使用以下命令查看安装进度，直到所有组件都显示为`运行`状态:

   ```bash
   oc get pods --namespace tekton-pipelines --watch
   ```

   **注意:** 通过 CTRL + C 停止监控.

恭喜! 你已经成功地在OpenShift集群中安装了Tekton管道。接下来，你可参考以下主题:

* [配置物料存储](#配置物料存储)通过此节了解如何为Tekton配置物料存储.
* [自定义基础执行参数](#自定义基础执行参数) 如果你需要自定义你的服务账户，超时时间或Pod模板值的话，请参考此节内容.

If you want to run OpenShift 4.x on your laptop (or desktop), you
should take a look at [Red Hat CodeReady Containers](https://github.com/code-ready/crc).

## 配置物料存储

Tekton管道中的`Tasks`需要从一个或多个位置中获取输入或存储输出。
 你可以使用以下方式来配置Tekton管道所需的存储资源:

  * [辞旧话卷](#configuring-a-persistent-volume)
  * [云存储](#configuring-a-cloud-storage-bucket)

**注意:** `Tasks`所需的输入与输出位置可通过 [`PipelineResources`](https://github.com/tektoncd/pipeline/blob/master/docs/resources.md)来定义。

对于Tekton管道来说，两种选项具有相同的功能，你可以按照你的实际业务需要进行选择。例如:

 - 在一些环境中，创建一个持久化卷可能比传输文件到云存储更慢。
 - 如果集群运行在多个可用区，访问持久化卷可能就不合适了。

**注意:** 要自定义持久化卷所用`ConfigMaps`的名称 (例如避免与其他服务冲突的情况), 重命名你的 `ConfigMap` 名称并更新[controller.yaml](https://github.com/tektoncd/pipeline/blob/e153c6f2436130e95f6e814b4a792fb2599c57ef/config/controller.yaml#L66-L75)中引用的值.

### 配置持久化卷

要配置 [持久化卷](https://kubernetes.io/docs/concepts/storage/persistent-volumes/), 通过使用一个名称为 `config-artifact-pvc`的`ConfigMap` 然后配置以下属性:

- `size`: 卷的大小. 默认为5GiB.
- `storageClassName`: 持久化卷所使用的[存储类](https://kubernetes.io/docs/concepts/storage/storage-classes/)名称。默认为默认存储类。

### 配置云存储

要配置 [S3 桶](https://aws.amazon.com/s3/)或者 [GCS 桶](https://cloud.google.com/storage/),
使用一个名称为`config-artifact-bucket`的`ConfigMap`然后配置以下属性:

- `location` - 桶的位置, 例如 `gs://mybucket` 或者 `s3://mybucket`.
- `bucket.service.account.secret.name` - 服务账号使用的包含可访问桶的凭证的密文名称。
- `bucket.service.account.secret.key` - 服务账号所需JSON文件的键名称。
- `bucket.service.account.field.name` - 当指定密文路径时，所使用的环境变量名称。默认为 `GOOGLE_APPLICATION_CREDENTIALS`. 如果使用S3，需设置为`BOTO_CONFIG`.

**重要:** 配置你的桶的回收策略为在运行`Tasks`后删除所有文件。

**注意:** 你只能使用 `us-east-1` 可用区的S3桶. 这是[`gsutil`](https://cloud.google.com/storage/docs/gsutil) 的限制.


#### S3 桶配置示例

以下为使用S3桶的示例:

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: tekton-storage
  namespace: tekton-pipelines
type: kubernetes.io/opaque
stringData:
  boto-config: |
    [Credentials]
    aws_access_key_id = AWS_ACCESS_KEY_ID
    aws_secret_access_key = AWS_SECRET_ACCESS_KEY
    [s3]
    host = s3.us-east-1.amazonaws.com
    [Boto]
    https_validate_certificates = True
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: config-artifact-bucket
  namespace: tekton-pipelines
data:
  location: s3://mybucket
  bucket.service.account.secret.name: tekton-storage
  bucket.service.account.secret.key: boto-config
  bucket.service.account.field.name: BOTO_CONFIG
```

#### GCS桶示例

以下为使用GCS桶的示例:

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: tekton-storage
  namespace: tekton-pipelines
type: kubernetes.io/opaque
stringData:
  gcs-config: |
    {
      "type": "service_account",
      "project_id": "gproject",
      "private_key_id": "some-key-id",
      "private_key": "-----BEGIN PRIVATE KEY-----\nME[...]dF=\n-----END PRIVATE KEY-----\n",
      "client_email": "tekton-storage@gproject.iam.gserviceaccount.com",
      "client_id": "1234567890",
      "auth_uri": "https://accounts.google.com/o/oauth2/auth",
      "token_uri": "https://oauth2.googleapis.com/token",
      "auth_provider_x509_cert_url": "https://www.googleapis.com/oauth2/v1/certs",
      "client_x509_cert_url": "https://www.googleapis.com/robot/v1/metadata/x509/tekton-storage%40gproject.iam.gserviceaccount.com"
    }
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: config-artifact-bucket
  namespace: tekton-pipelines
data:
  location: gs://mybucket
  bucket.service.account.secret.name: tekton-storage
  bucket.service.account.secret.key: gcs-config
  bucket.service.account.field.name: GOOGLE_APPLICATION_CREDENTIALS
```

## 自定义基础执行参数

你可以自定义Tekton管道运行`TaskRun` 及 `PipelineRun`时所用的服务账号 (`ServiceAccount`), 超时时间 (`Timeout`), 以及Pod模板 (`PodTemplate`) , 要自定义这些值，可直接修改配置文件 `config-defaults` 。

以下示例将自定义:

- 默认服务账号从`default` 修改为 `tekton`.
- 默认超时时间从60 分钟修改为 20 分钟.
- 为所有由执行`TaskRuns`的Pod附加统一的 `app.kubernetes.io/managed-by` 标签.
- 为默认的Pod模板添加一个节点选择器.
  更多信息, 请参考 [`TaskRuns`中的`PodTemplate`](./taskruns.md#pod-template) 或者 [`PipelineRuns`中的`PodTemplate`](./pipelineruns.md#pod-template).

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: config-defaults
data:
  default-service-account: "tekton"
  default-timeout-minutes: "20"
  default-pod-template: |
    nodeSelector:
      kops.k8s.io/instancegroup: build-instance-group
  default-managed-by-label-value: "my-tekton-installation"
```

**注意:**  由[config-defaults.yaml](./../config/config-defaults.yaml)提供的`_example`键
的内容示例都可由你来自定义。

### 自定义管道控制器行为

要自定义管道控制器的行为，可修改配置文件 `feature-flags` :

- `disable-home-env-overwrite` - 设置此标记为`true`，如果你想阻止Tekton执行步骤时覆盖你的$HOME环境变量, 默认为`false`, 更多信息请参考 [相关issue](https://github.com/tektoncd/pipeline/issues/2013).

- `disable-working-directory-overwrite` - 设置此标记为`true`，如果你想阻止Tekton执行步骤时覆盖你的工作区目录，默认为`false`，这将允许Tekton在执行每一个步骤时将工作目录设置为 `/workspace`.
更多信息请参考 [相关issue](https://github.com/tektoncd/pipeline/issues/1836).

例如:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: feature-flags
data:
  disable-home-env-overwrite: "true" # Tekton will not override the $HOME variable for individual Steps.
  disable-working-directory-overwrite: "true" # Tekton will not override the working directory for individual Steps.
```

## 创建自定义Tekton管道版本

你可以创建Tekton管道的自定义发布版本，参考以下步骤 [创建一个正式发布](https://github.com/tektoncd/pipeline/blob/master/tekton/README.md#create-an-official-release). 例如, 你可能需要自定义Tekton管道所使用的容器镜像。

## 下一步

开始使用Tekton管道, 参考[Tekton管道手册](./tutorial.md) 以及查看我们的 [示例](https://github.com/tektoncd/pipeline/tree/master/examples).

---

除非另有说明，本页内容采用[Creative Commons Attribution 4.0 License](https://creativecommons.org/licenses/by/4.0/)授权协议，示例代码采用[Apache 2.0 License](https://www.apache.org/licenses/LICENSE-2.0)授权协议
