<!--
---
linkTitle: "Workspaces"
weight: 5
---
-->
# 工作区

- [概览](#概览)
  - [`Tasks`和`TaskRuns`中的`工作区（Workspaces）`](#Tasks和TaskRuns中的工作区（Workspaces）)
  - [`Pipelines`和`PipelineRuns`中的`工作区`](#Pipelines和PipelineRuns中的工作区)
- [配置`工作区`](#配置工作区)
  - [在`Tasks`中使用`工作区`](#在Tasks中使用工作区)
    - [在`TaskRuns`中使用`工作区`变量](#在TaskRuns中使用工作区变量)
    - [映射`Tasks`中的`工作区`到`TaskRuns`](#映射Tasks中的工作区到TaskRuns)
    - [使用`工作区`的`TaskRun`定义示例](#使用工作区的TaskRun定义示例)
  - [在`Pipelines`中使用`工作区`](#在Pipelines中使用工作区)
    - [在`Pipeline`中指定`工作区`顺序](#在Pipeline中指定工作区顺序)
    - [在`PipelineRuns`中指定`工作区`](#在PipelineRuns中指定工作区)
    - [使用`工作区`的`PipelineRun`定义示例](#使用工作区的PipelineRun定义示例)
  - [在`Workspaces`中指定`VolumeSources`](#在Workspaces中指定VolumeSources)
- [更多示例](#更多示例)

## 概览

`工作区`允许`Tasks`定义一部分文件系统，然后通过`TaskRuns`在运行时提供。`TaskRun`可以通过多种方式来指定：使用只读的`ConfigMap`或`Secret`，已存在的`PersistentVolumeClaim`，由提供的`VolumeClaimTemplate`模版创建新的`PersistentVolumeClaim`，或则简单的`emptyDir`.

`工作区`与`Volumes`相似，除了它允许`Task`作者让用户和`TaskRuns`运行是决定使用哪一个存储类.

工作区可用于以下目的:

- 存储输入或输出
- 在`Tasks`之间分享数据
- 挂载`Secrets`中的凭证
- 挂载`ConfigMaps`中的配置
- 挂载组织中的常用工具
- 缓存工件来加速构建过程

### `Tasks`和`TaskRuns`中的`工作区（Workspaces）`

`Tasks`为起步骤指明一个`工作区`存在于磁盘中，在运行时，`TaskRun`提供`Volume`明细来挂载到`工作区`中.

关注点的分离可以提供更多的灵活性，例如，隔离的`TaskRun`可以通过提供`emptyDir`卷来更快地挂载，然后在执行完成后自动清除. 在更复杂的系统中，`TaskRun`可能使用`PersistentVolumeClaim`提前准备好数据供`Task`处理。这两种场景可能使用同一个`Task's`及`工作区`，然后在`TaskRun`中根据情况来指定.

### `Pipelines`和`PipelineRuns`中的`工作区`

`Pipeline`可以使用`工作区`来在属于它的`Tasks`之间共享存储。例如，`Task`可以克隆源代码仓储到一个`工作区`，然后另一个`Task`可以从`工作区`中编译代码。`Pipeline's`保证两个`Tasks`使用相同的工作区，更重要的时，采用正确的顺序来访问工作区.

`PipelineRuns`处理方式与`TaskRuns`一样 —— 它们提供特定的`Volume`信息来让每一个`Pipeline`使用`工作区`.
`PipelineRuns`另外的职责是确保无论何种类型的`Volume`都可以在`Tasks`间安全和正确的共享.

## 配置`工作区`

本节描述如何在`TaskRun`中配置一个或多个`工作区`.

### 在`Tasks`中使用`工作区`

要在`Task`中配置一个或多个`工作区`,添加一个`workspaces`，并指定包含以下字段的列表:

- `name` -  (**必须**) 一个 **唯一** 的字符串标识，通过它可以引用此工作区
- `description` - 描述此工作区的信息
- `readOnly` - 定义`Task`是否可写入此`工作区`.
- `mountPath` - 工作区挂载的磁盘位置，`Steps`中可通过此路径访问。路径相对于`/workspace`，如果未指定`mountPath`，将会默认为`/workspace/<name>`，`<name>`为工作区唯一的名称.
  
注意:
  
- `Task`可以包含多个`工作区`. 
- `readOnly` `Workspace`将会通过read-only方式挂载，尝试写入只读工作区将导致`TaskRuns`失败.
- `mountPath`可以是绝对路径或相对路径，绝对路径必须以`/`开始，相对路径开始于目录名称，例如， `mountPath` 为 `"/foobar"`是绝对路径，导出的工作区路径为`/foobar`，但是`mountPath` 为 `"foobar"`是相对路径，其实际路径为`/workspace/foobar`.

以下`Task`示例包含一个名称为`messages`的`工作区`，`Task`将消息写入此位置:

```yaml
spec:
  steps:
  - name: write-message
    image: ubuntu
    script: |
      #!/usr/bin/env bash
      set -xe
      echo hello! > $(workspaces.messages.path)/message
  workspaces:
  - name: messages
    description: The folder where we write the message to
    mountPath: /custom/path/relative/to/root
```

#### 在`TaskRuns`中使用`工作区`变量

`Tasks`中可通过以下变量获取`工作区`信息:

- `$(workspaces.<name>.path)` - 工作区的路径，`<name>`是`工作区`的名称.
- `$(workspaces.<name>.volume)`- 工作区的`Volume`，`<name>`是`工作区`的名称.

#### 映射`Tasks`中的`工作区`到`TaskRuns`

`TaskRun`在执行包含`工作区`列表的`Task`时，必须将`工作区`绑定到真实的物理`Volumes`，由此，`TaskRun`包含自己的`workspaces`列表，每一项包含以下字段:

- `name` - (**必须**) `工作区`的名称
- `subPath` - 可选的`Volume`中的子路径，这里面的数据将会挂载到`Workspace`

项中也必须包含一个`VolumeSource`,参考[与`工作区`一起使用 `VolumeSources`](#specifying-volumesources-in-workspaces) 获取更多信息.
               
**注意:**
-  在`TaskRun`执行之前，`subPath` *必须* 存在在`卷`中，否则`TaskRun`将会执行失败.
- `工作区`必须在执行与`Task`关联的`TaskRun`之前有效，否则`TaskRun`将会失败.

#### 使用`工作区`的`TaskRun`定义示例

以下示例展示了如何在`TaskRun`中指定`工作区`,有关更多信息, 参考 [在`TaskRun`中的`工作区`](../examples/v1beta1/taskruns/workspace.yaml).

在以下示例中，存在一个名称为`PersistentVolumeClaim`的持久化存储声明，它被用于名称为`myworkspace`的`工作区`中，它只导出PVC中的`my-subdir`子目录:

```yaml
workspaces:
- name: myworkspace
  persistentVolumeClaim:
    claimName: mypvc
  subPath: my-subdir
```

在以下示例中，提供一个[`emptyDir`](https://kubernetes.io/docs/concepts/storage/volumes/#emptydir)给任务的名称为`myworkspace`的`工作区`:

```yaml
workspaces:
- name: myworkspace
  emptyDir: {}
```

以下示例中，为`Task`中名称为`myworkspace`的工作区指定一个名称为`my-configmap`的配置映射:

```yaml
workspaces:
- name: myworkspace
  configmap:
    name: my-configmap
```

以下示例，为`Task`中名称为`myworkspace`的工作区指定一个名称为`my-secret`的`密文`:

```yaml
workspaces:
- name: myworkspace
  secret:
    secretName: my-secret
```

有关更深入的示例，请参考[workspace.yaml](../examples/v1beta1/taskruns/workspace.yaml).

### 在`Pipelines`中使用`工作区`

独立的`Tasks`定义其运行所需`工作区`，`Pipeline`决定需要在其`Tasks`之间共享的工作区。要定义共享的工作区，你必须在`Pipeline`中定义以下信息:

- 需要由`PipelineRuns`提供的工作区列表，使用`workspaces`字段定义工作区列表，列表中每一项具有唯一的名称.
- 映射`Pipeline`与`Task`中工作区名称的映射.

以下示例定义了一个具有一个工作区的`Pipeline`，名称为`pipeline-ws1`.此工作区绑定到了两个`Tasks` —— 第一个`gen-code`任务作为`输出`工作区，然后作为第二个`commit`任务的`src`
工作区. 如果`PipelineRun`提供了`PersistentVolumeClaim`作为工作区，则可在`Task`之间共享数据.

```yaml
spec:
  workspaces:
    - name: pipeline-ws1 # Pipeline中的工作区名称
  tasks:
    - name: use-ws-from-pipeline
      taskRef:
        name: gen-code # gen-code 任务指定工作区名称为"output"
      workspaces:
        - name: output
          workspace: pipeline-ws1
    - name: use-ws-again
      taskRef:
        name: commit # commit 任务中工作区名称为 "src"
      workspaces:
        - name: src
          workspace: pipeline-ws1
      runAfter:
        - use-ws-from-pipeline # 重要: 实现执行use-ws-from-pipeline任务来写入数据
```

#### 在`Pipeline`中指定`工作区`顺序

在`Tasks`之间共享一个`工作区`需要你定义这些`Tasks`访问`工作区`的顺序，因为不同的存储类具有不同的并发读写的限制. 例如，`PersistentVolumeClaim`可能只允许一个`Task`写入一次.

**警告:** 你 *必须* 确保顺序的正确性，不正确的顺序可能导致死锁，比如在多个`Task` `Pods`同时尝试挂载一个`PersistentVolumeClaim`来在同一时间写入时，这将导致`Tasks`超时.

要定义顺序，在`Pipeline`定义中使用`runAfter`字段，更多信息，参见[`runAfter` 文档](pipelines.md#runAfter).

#### 在`PipelineRuns`中指定`工作区`

要使用`PipelineRun`运行一个包含`工作区`的`Pipeline`，必须先将工作区绑定到真实的物理卷，这可以通过`PipelineRun` `workspaces`字段来设置工作区列表，列表中每一项包含以下字段:

- `name` - (**必须**) `Pipeline`中定义的`工作区`名称.
- `subPath` - (可选) 卷中的子路径，该路径中存储工作区需要使用的数据，该路径必须存在，否则`TaskRun`将执行失败.

项中必须包含一种`VolumeSource`，参考[与`工作区`一起使用`VolumeSources`](#specifying-volumesources-in-workspaces).

**注意:** 如果`Pipeline`中定义的工作区，在`PipelineRun`中未绑定，`PipelineRun`将会失败.

#### 使用`工作区`的`PipelineRun`定义示例

以下示例阐述了如何在`PipelineRuns`中指定工作区，更深入的示例请参考[`PipelineRun`中的工作区](../examples/v1beta1/pipelineruns/workspaces.yaml) YAML 示例.

在以下示例中，存在一个名称为`mypvc`的`PersistentVolumeClaim`，用于名称为`myworkspace`的工作区，它只导出了`mypvc`中的`my-subdir`子路径: 

```yaml
workspaces:
- name: myworkspace
  persistentVolumeClaim:
    claimName: mypvc
  subPath: my-subdir
```

以下示例中, 将名称为`my-configmap`的`配置映射`用于`Pipeline`中定义的名称为`myworkspace`的工作区中:

```yaml
workspaces:
- name: myworkspace
  configmap:
    name: my-configmap
```

以下示例中, 将名称为`my-secret`的`密文`用于`Pipeline`中定义的名称为`myworkspace`的工作区中:

```yaml
workspaces:
- name: myworkspace
  secret:
    secretName: my-secret
```

### 在`Workspaces`中指定`VolumeSources`

你在每一个工作区中可使用一种类型的`VolumeSource`. 每一种类型具有不同的配置项`Workspaces` 支持以下字段:

#### `emptyDir`


`emptyDir` 字段指向[`emptyDir` 卷](https://kubernetes.io/docs/concepts/storage/volumes/#emptydir) ，它存储`TaskRun`运行的临时数据. `emptyDir` 卷**不** 适合于在管道任务之间共享数据.
但是，它适用于在`TaskRuns`内部步骤之间共享数据.

#### `persistentVolumeClaim`

`persistentVolumeClaim`指向[`persistentVolumeClaim`卷](https://kubernetes.io/docs/concepts/storage/volumes/#persistentvolumeclaim).
`PersistentVolumeClaim`卷是在`Pipeline`中`Tasks`之间共享数据的好选择.

#### `volumeClaimTemplate`

`volumeClaimTemplate`是[`persistentVolumeClaim`卷](https://kubernetes.io/docs/concepts/storage/volumes/#persistentvolumeclaim)的模版, 为每一个`PipelineRun`或者`TaskRun`创建，由`PipelineRun`或`TaskRun`从模版创建的PVC，将在`PipelineRun`或`TaskRun`删除后被删除.

当需要在运行`PipelineRun`或`TaskRun`时共享数据时，`volumeClaimTemplate`卷是好的选择.

#### `configMap`

`configMap`指向[`configMap`卷](https://kubernetes.io/docs/concepts/storage/volumes/#configmap).
使用`configMap`作为工作区，具有以下限制:

- `configMap`卷总是只读，步骤不能进行写入.
- `configMap`指定的配置映射必须在`TaskRun`运行时存在.
- `configMaps`[最大不超过1MB](https://github.com/kubernetes/kubernetes/blob/f16bfb069a22241a5501f6fe530f5d4e2a82cf0e/pkg/apis/core/validation/validation.go#L5042).

#### `secret`

`secret`字段指向[`secret`卷](https://kubernetes.io/docs/concepts/storage/volumes/#secret).
使用`secret`卷作为工作区，具有以下限制:

- `secret`卷总是只读，步骤不能进行写入.
- `secret`指定的密文必须在`TaskRun`运行时存在.
- `secret`[最大不超过1MB](https://github.com/kubernetes/kubernetes/blob/f16bfb069a22241a5501f6fe530f5d4e2a82cf0e/pkg/apis/core/validation/validation.go#L5042).

如果你需要一个目前尚未存在的`VolumeSource`类型, 可在github上[提交需求](https://github.com/tektoncd/pipeline/issues) 或者 [pull request](https://github.com/tektoncd/pipeline/blob/master/CONTRIBUTING.md).

## 更多示例

以下包含有关配置`工作区`的更详细的示例:

- [`TaskRun`中的工作区](../examples/v1beta1/taskruns/workspace.yaml)
- [`PipelineRun`中的工作区](../examples/v1beta1/pipelineruns/workspaces.yaml)
