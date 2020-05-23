<!--
---
linkTitle: "PipelineRuns"
weight: 4
---
-->
# PipelineRuns

本文档说明`PipelineRuns`对象及其功能.

[`Pipeline`](pipelines.md)定义需要运行那些[`Tasks`](tasks.md)以及[它们的运行顺序](pipelines.md#ordering).要执行`Pipeline`中的`Tasks`，你必须创建一个`PipelineRun`.

创建一个`PipelineRun`将会触发为管道中的每一个`Task`创建其[`TaskRuns`](taskruns.md).

---

- [PipelineRuns](#pipelineruns)
  - [语法](#语法)
    - [指定管道](#指定管道)
    - [资源](#资源)
    - [参数](#参数)
    - [服务账户](#服务账户)
    - [多个服务账户](#多个服务账户)
    - [Pod模版](#Pod模版)
  - [持久化卷声明](#持久化卷声明)
  - [工作区](#工作区)
  - [取消PipelineRun](#取消PipelineRun)
  - [LimitRanges](#limitranges)

## 语法

定义`PipelineRun`资源的配置文件，你可以指定以下字段:

- 必须:
  - [`apiVersion`][kubernetes-overview] - API版本, 例如
    `tekton.dev/v1beta1`
  - [`kind`][kubernetes-overview] - k8s资源类型，固定为`PipelineRun`.
  - [`metadata`][kubernetes-overview] - `PipelineRun`资源对象的元数据，如`name`.
  - [`spec`][kubernetes-overview] - `PipelineRun`资源对象的配置信息.
    - [`pipelineRef` 或者 `pipelineSpec`](#指定管道) - 指定你需要运行的[`Pipeline`](pipelines.md).
- 可选:
  - [`resources`](#资源) - 指定此`PipelineRun`所需要使用的
    [`PipelineResources`](resources.md) .
  - [`params`](#参数) - 指定传递给管道的参数.
  - [`serviceAccountName`](#服务账户) - 指定一个`ServiceAccount`资源对象允许你的构建在定义的认证信息下运行，当没有指定`ServiceAccount`时，将使用配置映射中的`default-service-account`配置 - 默认配置将会被使用.
  - [`serviceAccountNames`](#多个服务账户) - 指定一个`serviceAccountName`列表，以及`PipelineTask`对允许你为具体的`PipelineTask`设置对应的`ServiceAccount`.
  - `timeout` - 设置`PipelineRun`运行超时时间，如果`timeout`为空，将使用默认的超时时间，如果值设置为0，表示没有超时时间，`PipelineRun`为`TaskRun`共享相同的默认超时时间，你可以通过[here](taskruns.md#配置默认超时时间)配置默认超时时间，方式与`TaskRun`一致.
  - [`podTemplate`](#Pod模版) - 指定[pod模版](./podtemplates.md)，这将作为`Task`运行时对应Pod的基础模版.

[kubernetes-overview]:
  https://kubernetes.io/docs/concepts/overview/working-with-objects/kubernetes-objects/#required-fields

### 指定管道

由于`PipelineRun`是[`Pipeline`](pipelines.md)的执行者，所以你必须指定那个`Pipeline`要被执行.

你可以直接引用一个以及存在的`Pipeline`:

```yaml
spec:
  pipelineRef:
    name: mypipeline

```

或者直接在`PipelineRun`里面内嵌定义`Pipeline`:

```yaml
spec:
  pipelineSpec:
    tasks:
    - name: task1
      taskRef:
        name: mytask
```

[此处](../examples/v1beta1/pipelineruns/pipelinerun-with-pipelinespec.yaml)是一个直接在`PipelineRun`中定义`Pipeline`配置的示例.

在创建示例`PipelineRun`之后，与此相关的pod将会显示“早上好”的消息:

```bash
kubectl logs $(kubectl get pods -o name | grep pipelinerun-echo-greetings-echo-good-morning)
Good Morning, Bob!
```

以及这个Pod的日志将会显示“晚上好”:

```bash
kubectl logs $(kubectl get pods -o name | grep pipelinerun-echo-greetings-echo-good-night)
Good Night, Bob!
```

甚至于，你可以直接在`Pipeline`中内嵌`Task`的定义:

```yaml
spec:
  pipelineSpec:
    tasks:
    - name: task1
      taskSpec:
        steps:
          ...
```

[此处](../examples/v1beta1/pipelineruns/pipelinerun-with-pipelinespec-and-taskspec.yaml)是一个直接通过`PipelineRun`内嵌`Pipeline`定义及`Task`定义的示例.

### 资源

当运行[`Pipeline`](pipelines.md), 你需要指定它所需要的[`PipelineResources`](resources.md)，一个`Pipeline`在不同的场景可能需要不同的`PipelineResources`，例如:

- 当触发器运行一个`Pipeline`来处理一个pull request，触发器系统就需要指定一个具有commit-ish的git`管道资源`
- 当根据自己的配置来手动调用一个`Pipeline`时，我们需要确保有自己的GitHub fork（通过git`管道资源`），镜像仓库（通过image`管道资源`）以及Kubernetes集群（通过cluster`管道资源`）.

要指定`PipelineRun`中的`PipelineResources`，需要通过`resources`节点来配置，例如:

```yaml
spec:
  resources:
    - name: source-repo
      resourceRef:
        name: skaffold-git
    - name: web-image
      resourceRef:
        name: skaffold-image-leeroy-web
    - name: app-image
      resourceRef:
        name: skaffold-image-leeroy-app
```

或者直接在`PipelineRun`中定义`Resource`:

```yaml
spec:
  resources:
    - name: source-repo
      resourceSpec:
        type: git
        params:
          - name: revision
            value: v0.32.0
          - name: url
            value: https://github.com/GoogleContainerTools/skaffold
    - name: web-image
      resourceSpec:
        type: image
        params:
          - name: url
            value: gcr.io/christiewilson-catfactory/leeroy-web
    - name: app-image
      resourceSpec:
        type: image
        params:
          - name: url
            value: gcr.io/christiewilson-catfactory/leeroy-app
```

### 参数

当在编写一个PipelineRun时，我们可以指定需要绑定到对应Pipeline的输入参数.

这意味着管道可以在不同的输入参数下运行，通过编写PipelineRun来绑定不同的输入值的方式来实现.

```yaml
spec:
  params:
  - name: pl-param-x
    value: "100"
  - name: pl-param-y
    value: "500"
```

### 服务账户

指定一个`ServiceAccount`资源对象的`name`，通过`serviceAccountName`字段可使你的`Pipeline`在指定的服务账户权限下运行，如果未指定`serviceAccountName`字段，`TaskRuns`将会使用`configmap-defaults`默认配置映射中的设置，如果没有，将使用`TaskRun`资源对象所在[namespace](https://kubernetes.io/docs/concepts/overview/working-with-objects/namespaces/)的 [`default` 服务账户](https://kubernetes.io/docs/tasks/configure-pod-container/configure-service-account/#use-the-default-service-account-to-access-the-api-server).

有关指定服务账户的示例及更多详情，请参考
[`ServiceAccount`](./auth.md).

### 多个服务账户

指定多个`serviceAccountName` 和 `PipelineTask` 对，特定的`PipelineTask`将会运行于所配置对应的`ServiceAccount`, 最终覆盖[`serviceAccountName`](#服务账户)配置, 例如:

```yaml
spec:
  serviceAccountName: sa-1
  serviceAccountNames:
    - taskName: build-task
      serviceAccountName: sa-for-build
```

如果使用此`Pipeline`，`test-task`将会使用`服务账户` `sa-1`，而`build-task`将会使用`sa-for-build`.

```yaml
kind: Pipeline
spec:
  tasks:
    - name: build-task
      taskRef:
        name: build-push
    - name: test-task
      taskRef:
        name: test
```

### Pod模版

指定一个[pod模版](./podtemplates.md)配置，用于`Task`pod，这运行你在`Task`执行时可自定义一些pod字段.

在以下示例中，`Task`定义了一个`volumeMount`
(`my-cache`), 它由`PipelineRun`提供, 通过一个
`persistentVolumeClaim`. 同时，Pod作为non-root用户运行.

```yaml
apiVersion: tekton.dev/v1beta1
kind: Task
metadata:
  name: mytask
spec:
  steps:
    - name: writesomething
      image: ubuntu
      command: ["bash", "-c"]
      args: ["echo 'foo' > /my-cache/bar"]
      volumeMounts:
        - name: my-cache
          mountPath: /my-cache
---
apiVersion: tekton.dev/v1beta1
kind: Pipeline
metadata:
  name: mypipeline
spec:
  tasks:
    - name: task1
      taskRef:
        name: mytask
---
apiVersion: tekton.dev/v1beta1
kind: PipelineRun
metadata:
  name: mypipelinerun
spec:
  pipelineRef:
    name: mypipeline
  podTemplate:
    securityContext:
      runAsNonRoot: true
    volumes:
    - name: my-cache
      persistentVolumeClaim:
        claimName: my-volume-claim
```

## 持久化卷声明

任何在`PipelineRun`中的持久化卷在运行时被绑定，直到对应的`PipelineRun`或pods被删除. 这适用于任何内部产生的持久化声明.

## 工作区

`PipelineRun`要执行一个定义了`workspaces`的`Pipeline`，需要将`workspaces`映射到实际的物理卷.

这可通过`PipelineRun`配置中相关的`PersistentVolumeClaim`字段来绑定:

```yaml
workspaces:
- name: myworkspace # 必须与Task中的工作区名称一致
  persistentVolumeClaim:
    claimName: mypvc # PVC必须存在
  subPath: my-subdir
```

有关在`PipelineRun`中配置`workspaces`的示例和完整文档，可参考[workspaces.md](./workspaces.md#providing-workspaces-with-pipelineruns).

Tekton`工作区`中可支持多种类型的`Volume`，不同类型的卷列表，可参考
[工作区的`VolumeSources`](workspaces.md#在Workspaces中指定VolumeSources).

_完整示例可参考[PipelineRun工作区](../examples/v1beta1/pipelineruns/workspaces.yaml)
，位于示例目录._

## 取消PipelineRun

为了取消运行中的管道(`PipelineRun`), 你需要更新它的配置，将其设置为取消状态，其关联的`TaskRun`实例将会被标记为取消，正在运行的Pod将会被删除.

```yaml
apiVersion: tekton.dev/v1beta1
kind: PipelineRun
metadata:
  name: go-example-git
spec:
  # […]
  status: "PipelineRunCancelled"
```

## LimitRanges

为了为支持`TaskRun`中步骤执行容器申请最小量的资源，Tekton只会依据TaskRun中`步骤`资源的最大值，包括CPU，内存以及临时存储. 只需要`步骤`中资源请求的最大值，因为`TaskRun` pod中同一时间只会执行一个步骤.
所有请求中没有最大值，将会设置为0.


当在`PipelineRuns`运行的命名空间中设置了[LimitRange](https://kubernetes.io/docs/concepts/policy/limit-range/) 来限制容器的资源请求时（如CPU、内存以及临时存储），Tekton将会搜索命名空间所有的LimitRanges，然后使用其最小值作为资源请求限制，而不是使用0.

有关`PipelineRun`中使用`LimitRange`的示例，可参考[此处](../examples/v1beta1/pipelineruns/no-ci/limitrange.yaml).

---

除非另有说明，本页内容采用[Creative Commons Attribution 4.0 License](https://creativecommons.org/licenses/by/4.0/)授权协议，示例代码采用[Apache 2.0 License](https://www.apache.org/licenses/LICENSE-2.0)授权协议
