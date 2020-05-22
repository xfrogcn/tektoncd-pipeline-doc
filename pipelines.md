<!--
---
linkTitle: "Pipelines"
weight: 3
---
-->
# 管道

- [概览](#管道)
- [配置`Pipeline`](#配置Pipeline)
  - [指定 `Resources`](#指定-Resources)
  - [指定 `Workspaces`](#指定-Workspaces)
  - [指定 `Parameters`](#指定-Parameters)
  - [添加 `Tasks` 到 `Pipeline`](#添加-Tasks-到-Pipeline)
    - [使用 `from` 参数](#使用-from-参数)
    - [使用 `runAfter` 参数](#使用-runAfter-参数)
    - [使用 `retries` 参数](#使用-retries-参数)
    - [指定执行条件 `Conditions`](#指定执行条件-Conditions)
    - [配置失败超时时间](#配置失败超时时间)
    - [在`Task`级别配置执行结果](#在Task级别配置执行结果)
  - [在`Pipeline`级别配置执行结果](#在Pipeline级别配置执行结果)
  - [配置`Task`执行顺序](#配置Task执行顺序)
  - [添加描述](#添加描述)
  - [代码示例](#代码示例)

## 概览

`Pipeline`包含一系列排列为指定顺序的`Tasks`，通过它来执行持续集成流. 在`Pipeline`中的每一个`Task`作为`Pod`在你的Kubernetes集群中运行. 你可以配置多个条件来满足你的业务需求.

## 配置`Pipeline`

`Pipeline`定义支持以下字段:

- 必须:
  - [`apiVersion`][kubernetes-overview] - API版本, 例如
    `tekton.dev/v1beta1`.
  - [`kind`][kubernetes-overview] - 定义资源对象类型，固定为 `Pipeline`.
  - [`metadata`][kubernetes-overview] - `Pipeline`对象相关的元数据，例如名称`name`.
  - [`spec`][kubernetes-overview] - `Pipeline对象的配置信息, 它包括: 
    - [`tasks`](#添加-Tasks-到-Pipeline) - 指定`Task`，它构成了`Pipeline`的执行流程与功能.
- 可选:
  - [`resources`](#指定-Resources) - **alpha** 指定组成`Pipeline`的`Tasks`所需要的
    [`PipelineResources`](resources.md) .
  - [`tasks`](#添加-Tasks-到-Pipeline):
      - `resources.inputs` / `resource.outputs`
        - [`from`](#使用-from-参数) - 指定[`PipelineResource`](resources.md)的数据来源于之前`Task`的输出.
      - [`runAfter`](#使用-runAfter-参数) - 指定一个`Task`应该在某个或多个其他`Task`之后执行.
      - [`retries`](#使用-retries-参数) - 当错误发生后，`Task`可重试的次数. 而不会取消任务执行.
      - [`conditions`](#指定执行条件-Conditions) - 指定任务执行的`条件`,当评估结果为`true`时才执行任务.
      - [`timeout`](#配置失败超时时间) - `Task`失败前的超时时间. 
  - [`results`](#在Pipeline级别配置执行结果) - 指定`Pipeline`执行结果存放的位置.
  - [`description`](#添加描述) - `Pipeline`对象的描述信息.

[kubernetes-overview]:
  https://kubernetes.io/docs/concepts/overview/working-with-objects/kubernetes-objects/#required-fields

## 指定 `Resources`
`Pipeline`使用[`PipelineResources`](resources.md)来为组成它的`Task`提供输入或存储输出，你可以通过`spec`下的`resources`字段来定义，其中的每一项资源中必须指定唯一的`name`及其`type`。例如:

```yaml
spec:
  resources:
    - name: my-repo
      type: git
    - name: my-image
      type: image
```

## 指定 `Workspaces`

`工作区`允许你为`Pipeline`中每一个`Task`指定在其执行期间所需的一个或多个卷. 你可以通过`workspaces`字段指定一个或多个`工作区`.
例如:

```yaml
spec:
  workspaces:
    - name: pipeline-ws1 # Pipeline中工作区的名称
  tasks:
    - name: use-ws-from-pipeline
      taskRef:
        name: gen-code # gen-code包含一个名称为`output`的工作区
      workspaces:
        - name: output
          workspace: pipeline-ws1
    - name: use-ws-again
      taskRef:
        name: commit # commit包含一个名称为`src`的工作区
      runAfter:
        - use-ws-from-pipeline # 重要: use-ws-from-pipeline任务首先需要写入工作区
      workspaces:
        - name: src
          workspace: pipeline-ws1
```

有关更多信息, 参考:
- [在`Pipelines`中使用`工作区`](workspaces.md#在Pipelines中使用工作区)
- [`PipelineRun`中的工作区](../examples/v1beta1/pipelineruns/workspaces.yaml)代码示例

## 指定-Parameters

你可以指定需要在`Pipeline`执行时传递的全局参数，如编译标识或者制品名称. `参数`通过`PipelineRun`传递给`Pipeline`，可替换`Pipeline`中每一个`Task`的模板值.

参数名称:
- 必须只能使用字符或数字，及连字符(`-`)或下划线(`_`).
- 必须以字母或下划线开始(`_`).

例如, `fooIs-Bar_`是有效的参数名称, 但 `barIsBa$` 或者 `0banana` 无效.

每一个定义的参数包含一个`type`字段，它可以设置为`array`或者`字符串`. `array`可用于传递`Pipeline`中所需要的多个编译标识，如果未指定`type`字段，默认为`string`. 当应用实际的参数值时，将会验证`type`值. 参数的`description`及`default`字段是可选的.

以下示例展示了如何在`Pipeline`中使用`Parameters`. 

以下`Pipeline`定义了一个输入参数，名称为`context`，然后传递值到`Task`，以便于设置`Task`中的`pathToContext`参数.
如果你设置了`default`字段，然后在调用`Pipeline`的`PipelineRun`中未设置值，那么将会使用此默认值.

**注意:** 输入参数值在整个`Pipeline`中可作为变量使用，通过[变量替换](variables.md#Pipeline中有效的变量)的方式.

```yaml
apiVersion: tekton.dev/v1beta1
kind: Pipeline
metadata:
  name: pipeline-with-parameters
spec:
  params:
    - name: context
      type: string
      description: Path to context
      default: /some/where/or/other
  tasks:
    - name: build-skaffold-web
      taskRef:
        name: build-push
      params:
        - name: pathToDockerFile
          value: Dockerfile
        - name: pathToContext
          value: "$(params.context)"
```

以下`PipelineRun`提供了`context`参数的值:

```yaml
apiVersion: tekton.dev/v1beta1
kind: PipelineRun
metadata:
  name: pipelinerun-with-parameters
spec:
  pipelineRef:
    name: pipeline-with-parameters
  params:
    - name: "context"
      value: "/workspace/examples/microservices/leeroy-web"
```

## 添加 `Tasks` 到 `Pipeline`

`Pipeline`定义中必须引用至少一个[`Task`](tasks.md).
`Pipeline`中的每一个`Task`必须具有一个[有效](https://kubernetes.io/docs/concepts/overview/working-with-objects/names/#names)的`name`以及`taskRef`。例如:

```yaml
tasks:
  - name: build-the-image
    taskRef:
      name: build-push
```

你可以使用[`PipelineResources`](#指定-Resources)作为`Tasks`的输入和输出。例如:

```yaml
spec:
  tasks:
    - name: build-the-image
      taskRef:
        name: build-push
      resources:
        inputs:
          - name: workspace
            resource: my-repo
        outputs:
          - name: image
            resource: my-image
```

你也可以为任务提供[`参数`](tasks.md#详述-Parameters):

```yaml
spec:
  tasks:
    - name: build-skaffold-web
      taskRef:
        name: build-push
      params:
        - name: pathToDockerFile
          value: Dockerfile
        - name: pathToContext
          value: /workspace/examples/microservices/leeroy-web
```

### 使用 `from` 参数

如果`Pipeline`中的一个`Task`需要使用之前的`Task`的输出作为它的输入，可使用`from`参数来指定一个`Tasks`列表，指明这边任务必须在目标任务执行**之前**执行。当你的目标任务执行时，只有任务列表中最后一个`Task`输出的结果`PipelineResource`被使用。此输出`PipelineResource`的名称必须与目标`Task`输入`PipelineResource`名称一致. 

在以下示例中，`deploy-app` `Task`使用`build-app`任务名称为`my-image`资源的输出作为其输入.  因此, `build-app` `Task`将会在`deploy-app` `Task`之前执行，而不管任务在`Pipeline`中的顺序.

```yaml
- name: build-app
  taskRef:
    name: build-push
  resources:
    outputs:
      - name: image
        resource: my-image
- name: deploy-app
  taskRef:
    name: deploy-kubectl
  resources:
    inputs:
      - name: image
        resource: my-image
        from:
          - build-app
```

### 使用 `runAfter` 参数

如果你需要`Pipeline`中的任务按照指定的顺序执行，而没有需要通过`from`参数指定的依赖资源，可使用`runAfter`参数来指定任务需要在一个或多个其他`Task`之后执行.

在以下示例中，我们需要在构建之前先进行代码测试。由于构建任务不会使用`test-app`任务的任何输出，所以`build-app`任务使用`runAfter`来指定`test-app`必须在之前运行，而不管任务在`Pipeline`中的实际顺序.

```yaml
- name: test-app
  taskRef:
    name: make-test
  resources:
    inputs:
      - name: workspace
        resource: my-repo
- name: build-app
  taskRef:
    name: kaniko-build
  runAfter:
    - test-app
  resources:
    inputs:
      - name: workspace
        resource: my-repo
```

### 使用 `retries` 参数

在`Pipeline`中的每一个`Task`，你可以指定Tekton在任务失败后可以自动重试的次数. 当`Task`失败了， 其关联的`TaskRun`将会设置`成功` `条件`为`False`. `retries`参数命令Tekton在此条件发生时重试.

如果你知道`Task`在执行过程中可能会有些问题（例如，你可能知道有一些网络连接问题或者依赖丢失），那么你可以将`retries`参数值设置为大于0的数。如果你没有设置此参数，Tekton不会在`Task`失败时进行重试的尝试。.

在以下示例中，`build-the-image` `Task`将会在失败后重试一次，如果重试失败，`Task`执行将会失败.

```yaml
tasks:
  - name: build-the-image
    retries: 1
    taskRef:
      name: build-push
```

### 指定执行条件 `Conditions`

有时你只需要在某些条件满足时才执行任务，`condition`字段允许你应用一系列的[`条件`](./conditions.md)，这些条件在任务执行前运行，如果所有的条件评估为true，任务将会运行，如果其中任何一个条件返回false，任务将不会被执行。此时status.ConditionSucceeded被设置为`ConditionCheckFailed`, 不过，与其他任务失败不同的是，条件失败不会自动导致整个管道失败 -- 其它不依赖于此任务的任务(通过`from`或者`runAfter`)将会持续运行.

```yaml
tasks:
  - name: conditional-task
    taskRef:
      name: build-push
    conditions:
      - conditionRef: my-condition
        params:
          - name: my-param
            value: my-value
        resources:
          - name: workspace
            resource: source-repo
```

在此示例中，`my-condition`引用一个[条件](conditions.md)自定义资源. `build-push`任务将会在条件评估为true时执行.

条件中的资源同样可以使用[`from`](#使用-from-参数) 字段来使用之前任务的输出作为条件的输入资源. 就像Pipeline中的任务一样，通过`from`会影响任务执行顺序 -- 如果一个条件需要另外一个任务的输出，那么产生资源的任务将会首先被执行:

```yaml
tasks:
  - name: first-create-file
    taskRef:
      name: create-file
    resources:
      outputs:
        - name: workspace
          resource: source-repo
  - name: then-check
    conditions:
      - conditionRef: "file-exists"
        resources:
          - name: workspace
            resource: source-repo
            from: [first-create-file]
    taskRef:
      name: echo-hello
```

### 配置失败超时时间

你可以通过`Pipeline`中的`Task`配置下的`Timeout`参数来设置执行`Pipeline`关联的`PipelineRun`中通过`TaskRun`来执行具体`Task`的超时时间。
`超时时间`值是一个Go语言的`duration`格式，可参考[`ParseDuration`](https://golang.org/pkg/time/#ParseDuration), 有效值为`1h30m`, `1h`, `1m`, 以及 `60s`.  

**注意:** 如果你没有设置`Timeout`值，Tekton将使用[`PipelineRun`](pipelineruns.md#configuring-a-pipelinerun)超时替代.

在以下示例中，`build-the-image` `Task`任务配置的超时时间为90秒:

```yaml
spec:
  tasks:
    - name: build-the-image
      taskRef:
        name: build-push
      Timeout: "0h1m30s"
```

### 在Task级别配置执行结果

任务执行后可以产出[`Results`](tasks.md#存储执行结果)，你可以在`Pipeline`随后的`Task`中以参数的方式使用这些`结果`，通过[变量替换](variables.md#Pipeline中有效的变量), Tekton可推断`Task`的执行顺序，以便于在`Task`使用这些结果之前确保结果对应任务已被执行. 

在以下示例中，`previous-task-name` `Task`任务结果作为`bar-result`公开:

```yaml
params:
  - name: foo
    value: "$(tasks.previous-task-name.results.bar-result)"
```

请参考[`PipelineRun`中的任务结果](../examples/v1beta1/pipelineruns/task_results_example.yaml).

## 在Pipeline级别配置执行结果

你可以配置你的`Pipeline`来产出`结果`, 这些结果由管道中的`Task`所产生. 

在以下示例中，`Pipeline`指定了一个`results`项，名称为`sum`，它是由`second-add` `任务`所产生的.

```yaml
  results:
    - name: sum
      description: the sum of all three operands
      value: $(tasks.second-add.results.sum)
```

有关示例，可参考[`PipelineRun`中的结果](../examples/v1beta1/pipelineruns/pipelinerun-results.yaml).

## 配置`Task`执行顺序

你可以通过`Pipeline`来连接`Task`，所以他们的执行可以构成有向无环图(DAG).
`Pipeline`中的每一个`Task`作为图中的一个节点，他们可以通过边连接起来，所以`Pipeline`可以处理完成而不会死循环.

这可通过使用:

- [`from`](#使用-from-参数) 子句， 由每一个`Task`使用的[`PipelineResources`](resources.md), 以及
- [`runAfter`](#使用-runAfter-参数) 子句.

例如, 如下的`Pipeline`定义：

```yaml
- name: lint-repo
  taskRef:
    name: pylint
  resources:
    inputs:
      - name: workspace
        resource: my-repo
- name: test-app
  taskRef:
    name: make-test
  resources:
    inputs:
      - name: workspace
        resource: my-repo
- name: build-app
  taskRef:
    name: kaniko-build-app
  runAfter:
    - test-app
  resources:
    inputs:
      - name: workspace
        resource: my-repo
    outputs:
      - name: image
        resource: my-app-image
- name: build-frontend
  taskRef:
    name: kaniko-build-frontend
  runAfter:
    - test-app
  resources:
    inputs:
      - name: workspace
        resource: my-repo
    outputs:
      - name: image
        resource: my-frontend-image
- name: deploy-all
  taskRef:
    name: deploy-kubectl
  resources:
    inputs:
      - name: my-app-image
        resource: my-app-image
        from:
          - build-app
      - name: my-frontend-image
        resource: my-frontend-image
        from:
          - build-frontend
```

执行顺序图如下:

```none
        |            |
        v            v
     test-app    lint-repo
    /        \
   v          v
build-app  build-frontend
   \          /
    v        v
    deploy-all
```

详细说明:

1. `lint-repo` 和 `test-app` `任务` 没有 `from` 或 `runAfter` 子句，他们将会被立即同时执行.
2. 一旦 `test-app` 完成,  `build-app` 和 `build-frontend` 任务将会同时开始，因为他们都有`runAfter`子句，并指向`test-app`任务.
3. `deploy-all` `任务`在`build-app` 和 `build-frontend`任务完成后执行，因为他会使用这两个任务的输出资源.
4. 整个`Pipeline`将会在`lint-repo` 和 `deploy-all`任务完成时完成.

## 添加描述

`description`字段是可选字段，可通过此字段设置`Pipeline`的描述信息.

## 代码示例

为了更好地理解`Pipelines`, 可通过学习[我们的示例](https://github.com/tektoncd/pipeline/tree/master/examples).

---

除非另有说明，本页内容采用[Creative Commons Attribution 4.0 License](https://creativecommons.org/licenses/by/4.0/)授权协议，示例代码采用[Apache 2.0 License](https://www.apache.org/licenses/LICENSE-2.0)授权协议
