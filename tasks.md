<!--
---
linkTitle: "Tasks"
weight: 1
---
-->
# Tasks

- [概览](#概览)
- [配置`Task`](#配置Task)
  - [对比`Task` 与 `ClusterTask`](#对比Task与ClusterTask)
  - [定义 `Steps`](#定义-Steps)
    - [保留目录](#保留目录)
    - [在`Steps`中运行脚本](#在Steps中运行脚本)
  - [详述 `Parameters`](#详述-Parameters)
  - [详述 `Resources`](#详述-Resources)
  - [详述 `Workspaces`](#详述-Workspaces)
  - [存储执行结果](#存储执行结果)
  - [详述 `Volumes`](#详述-Volumes)
  - [详述  `Step` 模板](#详述-Step-模板)
  - [详述 `Sidecars`](#详述-Sidecars)
  - [添加描述](#添加描述)
  - [使用便利替换](#使用便利替换)
    - [替换参数与资源](#替换参数与资源)
    - [替换 `Array` 参数](#替换-Array-参数)
    - [替换 `Workspace` 路径](#替换-Workspace-路径)
    - [替换 `Volume` 名称和类型](#替换-Volume-名称和类型)
- [代码示例](#code-examples)
  - [构建和推送Docker镜像](#构建和推送Docker镜像)
  - [挂载多个`Volumes`](#挂载多个Volumes)
  - [挂载`ConfigMap` 为`Volume` 资源](#挂载ConfigMap-为Volume-资源)
  - [使用  `Secret` 作为环境变量源](#使用-Secret-作为环境变量源)
  - [在`Task`中使用  `Sidecar`](#在Task中使用-Sidecar)
- [调试](#调试)
  - [审视文件结构](#审视文件结构)
  - [审视`Pod`](#审视Pod)

## 概览

`Task`作为持续集成工作流的一部分，由一些列按特定顺序执行的步骤组成。`Task`在指定的命名空间中有效，`ClusterTask`在集群全局有效。

`Task`由以下元素组成:

- [参数(Parameters)](#详述-Parameters)
- [资源(Resources)](#详述-Resources)
- [步骤(Steps)](#定义-Steps)
- [工作区(Workspaces)](#详述-Workspaces)
- [结果(Results)](#存储执行结果)

## 配置`Task`

`Task`定义支持以下字段:

- 必须字段:
  - [`apiVersion`][kubernetes-overview] - 指定API版本. 例如,
    `tekton.dev/v1beta1`.
  - [`kind`][kubernetes-overview] - 定义此资源对象为`Task`对象。
  - [`metadata`][kubernetes-overview] - 指定`Task`资源对象的元数据，例如，名称(`name`).
  - [`spec`][kubernetes-overview] - 指定此`Task`的配置信息.
  - [`steps`](#defining-steps) - 指定一个或多个容器镜像来运行`Task`.
- 可选字段:
  - [`description`](#adding-a-description) - 有关该`Task`资源对象的描述.
  - [`params`](#specifying-parameters) - 指定执行`Task`所需参数.
  - [`resources`](#specifying-resources) - **alpha版本** 指定所需的或由`Task`创建的
    [`PipelineResources`](resources.md) .
    - [`inputs`](#specifying-resources) - 指定由`Task`引入的资源.
    - [`outputs`](#specifying-resources) - 指定由`Task`产生的资源.
  - [`workspaces`](#specifying-workspaces) - 指定`Task`所需的卷路径.
  - [`results`](#storing-execution-results) - 指定`Task`写入其执行结果的文件.
  - [`volumes`](#specifying-volumes) - 指定一个或多个在`Task`中的`Steps`步骤中有效的卷.
  - [`stepTemplate`](#specifying-a-step-template) - 指定一个`Container`步骤定义，它作为`Task`中所有`Steps`使用的基础容器.
  - [`sidecars`](#specifying-sidecars) - 指定与`Task`中`Step`一起运行的边车`Sidecar`容器.

[kubernetes-overview]:
  https://kubernetes.io/docs/concepts/overview/working-with-objects/kubernetes-objects/#required-fields

以下非功能性示例Demo使用了上述多个字段:

```yaml
apiVersion: tekton.dev/v1beta1
kind: Task
metadata:
  name: example-task-name
spec:
  params:
    - name: pathToDockerFile
      type: string
      description: The path to the dockerfile to build
      default: /workspace/workspace/Dockerfile
  resources:
    inputs:
      - name: workspace
        type: git
    outputs:
      - name: builtImage
        type: image
  steps:
    - name: ubuntu-example
      image: ubuntu
      args: ["ubuntu-build-example", "SECRETS-example.md"]
    - image: gcr.io/example-builders/build-example
      command: ["echo"]
      args: ["$(params.pathToDockerFile)"]
    - name: dockerfile-pushexample
      image: gcr.io/example-builders/push-example
      args: ["push", "$(resources.outputs.builtImage.url)"]
      volumeMounts:
        - name: docker-socket-example
          mountPath: /var/run/docker.sock
  volumes:
    - name: example-volume
      emptyDir: {}
```

### 对比`Task`与`ClusterTask`

`ClusterTask`是范围为整个集群的`Task`, 而`Task`的范围为单个命名空间。
`ClusterTask`行为与`Task`完全一致，本文档中所有内容都适合于这两种任务。

**注意:** 当你使用`ClusterTask`时, 你必须将`taskRef`中的`kind`子字段设置为`ClusterTask`.
如果不特别指定，`kind`默认为`Task`

以下为在Pipeline中使用`ClusterTask`的示例:

```yaml
apiVersion: tekton.dev/v1beta1
kind: Pipeline
metadata:
  name: demo-pipeline
  namespace: default
spec:
  tasks:
    - name: build-skaffold-web
      taskRef:
        name: build-push
        kind: ClusterTask
      params: ....
```

### 定义 `Steps`

`Step`通过引用一个容器镜像通过输入资源执行特定的工具并产生特定的输出。要添加步骤`Steps`到`Task`你需要定义`steps`字段（必须）并包含一些列的步骤`Steps`，在`Steps`中呈现的顺序即为执行的顺序。

`steps`字段中的容器镜像必须满足以下要求：

- 容器镜像需要遵循[容器契约](./container-contract.md).
- 每一个容器运行完成或者直到发生了首次错误.
- CPU,内存以及临时存储资源请求应该设置为0，或者，如果指定，最小值通过`Namespace`的`LimitRanges`设置，如果容器镜像无法获得`Task`中所有容器镜像的最大资源请求. 这确保了执行`Task`的Pod只会请求运行一个单独镜像所需的资源，而不是一次申请运行所有容器镜像所需的资源。

#### 保留目录

以下为Tekton运行所有`Tasks`所保留的特殊目录：

* `/workspace` - 此目录是[资源](#resources) 以及 [工作区](#workspaces)
  的挂载位置. 这些路径可在`Task`中通过 [变量替换](variables.md)来使用
* `/tekton` - 这个目录用于Tekton的特定功能:
    * `/tekton/results` 存储任务[结果](#results) 的位置.
      这些路径可在`Task`中通过 [变量替换](variables.md)来使用
    * 还有一些与[Tekton内部实现细节](developers/README.md#reserved-directories)相关的子目录
       **用于不应该依赖于这些目录结果，因为它们可能在未来被改变**

#### 在`Steps`中运行脚本

步骤可以指定一个`script`字段，它可以包含一段脚本。这个脚本可以直接调用，就像他在容器镜像中一样，并且可以直接传递任何参数.

**注意:** 如果设置了`script`字段, 步骤就不能再设置`command`字段.

不是以[shebang](https://en.wikipedia.org/wiki/Shebang_(Unix))
行开始的脚本将使用以下默认前缀:

```bash
#!/bin/sh
set -xe
```

你可以覆盖此默认前缀来指定特定的解析，此解析器必须包含在`Step's`容器镜像中.

以下示例执行一段基础脚本:

```yaml
steps:
- image: ubuntu  # contains bash
  script: |
    #!/usr/bin/env bash
    echo "Hello from Bash!"
```
以下示例执行一段Python脚本:

```yaml
steps:
- image: python  # contains python
  script: |
    #!/usr/bin/env python3
    print("Hello from Python!")
```

以下示例执行一段Node脚本:

```yaml
steps:
- image: node  # contains node
  script: |
    #!/usr/bin/env node
    console.log("Hello from Node!")
```

你可以直接执行工作区中的脚本:

```yaml
steps:
- image: ubuntu
  script: |
    #!/usr/bin/env bash
    /workspace/my-script.sh  # provided by an input resource
```

你也可以执行容器镜像中的脚本:

```yaml
steps:
- image: my-image  # contains /bin/my-binary
  script: |
    #!/usr/bin/env bash
    /bin/my-binary
```

### 详述 `Parameters`

你可以指定参数，例如编译参数或者工件名称，然后再`Task`执行时被使用.
`Parameters`通过`TaskRun`传递给所关联的`Task`.

参数名称:
- 只能由数字、字母、连字符 (`-`), 以及下划线 (`_`)组成.
- 必须由字母或者下划线(`_`)开始.

例如`fooIs-Bar_`是一个有效的名称，但`barIsBa$` 或 `0banana` 就是无效的.

定义的每一个参数都具有`type`字段，它可设置为`array`或者`string`.`array`主要用于传递`Task's`的执行参数，如果没有指定，`type`默认为`string`. 当指定了参数实际值时，将解析为`type`字段指定的类型.

以下示例阐述了在`Task`中使用`Parameters`，`Task`定义了两个名称为`flags`（类型为`array`)及`someURL`(类型为`string`)的参数，然后在`steps.args`列表中使用，你可以通过星号操作符来提取`array`中所有参数，在此例中，`flags`包含星号操作`$(params.flags[*])`.


**注意:** 输入参数可以在`Task`中作为变量来使用，通过[变量替换]](#using-variable-substitution)的方式.

```yaml
apiVersion: tekton.dev/v1beta1
kind: Task
metadata:
  name: task-with-parameters
spec:
  params:
    - name: flags
      type: array
    - name: someURL
      type: string
  steps:
    - name: build
      image: my-builder
      args: ["build", "$(params.flags[*])", "url=$(params.someURL)"]
```

下列`TaskRun`提供了参数的具体指:

```yaml
apiVersion: tekton.dev/v1beta1
kind: TaskRun
metadata:
  name: run-with-parameters
spec:
  taskRef:
    name: task-with-parameters
  params:
    - name: flags
      value:
        - "--set"
        - "arg1=foo"
        - "--randomflag"
        - "--someotherflag"
    - name: someURL
      value: "http://google.com"
```

### 详述 `Resources`

`Task`中可使用由[`PipelineResources`](resources.md#using-resources)定义的输入或输出资源.

使用`input`字段提供执行`Task`所需的上下文或者/以及其数据. 例如:

```yaml
resources:
  outputs:
    name: storage-gcs
    type: gcs
steps:
  - image: objectuser/run-java-jar #https://hub.docker.com/r/objectuser/run-java-jar/
    command: [jar]
    args:
      ["-cvf", "-o", "/workspace/output/storage-gcs/", "projectname.war", "*"]
    env:
      - name: "FOO"
        value: "world"
```

**注意**: 如果`Task`依赖于输出资源功能，那么在`Task's` `steps`字段中的容器不可挂载路径`/workspace/output`.

以下示例中`tar-artifact`资源同时被用于输出以及输出。因此，输入资源通过指定`targetPath`字段，被拷贝到了`customworkspace`目录，`untar` 步骤解压文件到`tar-scratch-space`目录，`edit-tar`步骤添加一个新文件，然后`tar-it-up`步骤创建新的压缩文件并将其放到`/workspace/customworkspace/`目录。当`Task`执行完成，它将结果放于`/workspace/customworkspace`目录，并上传它到由`tar-artifact`字段定义的桶中.

```yaml
resources:
  inputs:
    name: tar-artifact
    targetPath: customworkspace
  outputs:
    name: tar-artifact
steps:
 - name: untar
    image: ubuntu
    command: ["/bin/bash"]
    args: ['-c', 'mkdir -p /workspace/tar-scratch-space/ && tar -xvf /workspace/customworkspace/rules_docker-master.tar -C /workspace/tar-scratch-space/']
 - name: edit-tar
    image: ubuntu
    command: ["/bin/bash"]
    args: ['-c', 'echo crazy > /workspace/tar-scratch-space/rules_docker-master/crazy.txt']
 - name: tar-it-up
   image: ubuntu
   command: ["/bin/bash"]
   args: ['-c', 'cd /workspace/tar-scratch-space/ && tar -cvf /workspace/customworkspace/rules_docker-master.tar rules_docker-master']
```

### 详述 `Workspaces`

`工作区`[workspaces.md#declaring-workspaces-in-tasks]允许你指定一个或多个在`Task`执行时所需的卷，例如:

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

有关更多详情, 请参考 [在`Tasks`中使用`工作区`](workspaces.md#using-workspaces-in-tasks)
以及[`TaskRun`中的`工作区`](../examples/v1beta1/taskruns/workspace.yaml) 示例YAML文件.

### 存储执行结果

使用`results`字段来指定一个或多个文件来存储`Task`的执行结果，这些文件存储在`/tekton/results`目录中，如果在results中至少有一个文件，这个目录将会被自动创建，要指定一个文件，提供其`name`和`description`字段.

在以下示例中，`Task`在`results`中指定了两个文件：`current-date-unix-timestamp` 和 `current-date-human-readable`.

```yaml
apiVersion: tekton.dev/v1beta1
kind: Task
metadata:
  name: print-date
  annotations:
    description: |
      A simple task that prints the date
spec:
  results:
    - name: current-date-unix-timestamp
      description: The current date in unix timestamp format
    - name: current-date-human-readable
      description: The current date in human readable format
  steps:
    - name: print-date-unix-timestamp
      image: bash:latest
      script: |
        #!/usr/bin/env bash
        date +%s | tee /tekton/results/current-date-unix-timestamp
    - name: print-date-humman-readable
      image: bash:latest
      script: |
        #!/usr/bin/env bash
        date | tee /tekton/results/current-date-human-readable
```

**注意:** `Task's`结果的最大尺寸的限制由Kubernetes容器终端日志功能限制, 结果返回到controller的过程通过此机制完成，此限制为["2048 字节或 80 行"](https://kubernetes.io/docs/tasks/debug-application-cluster/determine-reason-pod-failure/#customizing-the-termination-message).
写到终端的日志编码为JSON对象，Tekton使用这些对象传递附加信息到控制器。由此，`Task`结果对于保持小量数据最实用，如提交SHAs，分支名称，临时空间名称等.

如果你的`Task`写入大量的小结果，你可以通过分割结果到不同的步骤来绕过此限制，但是如果结果大于1Kb，可以使用[`工作区`](#specifying-workspaces) 来在`Pipeline` `Tasks`之间传递数据.

### 详述 `Volumes`

除了由输入或输出资源隐式创建的卷外，你还可为`Task`步骤指定一个或多个执行时所需的[`卷`](https://kubernetes.io/docs/concepts/storage/volumes/).

例如，你可以使用`Volumes`做以下事情:

- [挂载一个Kubernetes`Secret`](auth.md).
- 创建一个`emptyDir`持久化`卷`来缓存跨步骤的数据.
- 挂载[Kubernetes `ConfigMap`](https://kubernetes.io/docs/tasks/configure-pod-container/configure-pod-configmap/).
- 挂载宿主Docker socket来构建`Dockerfile`为容器镜像.
  **注意:** 在集群中通过`docker build`来构建容器镜像是**非常不安全的**, 通常只在示例时使用，你应该使用[kaniko](https://github.com/GoogleContainerTools/kaniko) 来替代.

### 详述  `Step` 模板

`stepTemplate`字段指定一个[`容器`](https://kubernetes.io/docs/concepts/containers/)配置，这些配置将对`Task`中的所有步骤有效，在单独的`Steps`中的独立配置将覆盖掉`stepTemplate`配置.

在以下示例中，`Task`指定了`stepTemplate`字段，设置环境变量`FOO`为`bar`，在第一个步骤中将使用此值，但第二个步骤中将值覆盖为`baz`.

```yaml
stepTemplate:
  env:
    - name: "FOO"
      value: "bar"
steps:
  - image: ubuntu
    command: [echo]
    args: ["FOO is ${FOO}"]
  - image: ubuntu
    command: [echo]
    args: ["FOO is ${FOO}"]
    env:
      - name: "FOO"
        value: "baz"
```

### 详述 `Sidecars`

`sidecars`字段指定一系列的[`容器`](https://kubernetes.io/docs/concepts/containers/)
，这些容器与`Task`中的`Steps`一起运行. 你可以使用`Sidecars`来提供附加的功能，例如[Docker in Docker](https://hub.docker.com/_/docker)或运行一个模拟API服务来执行测试任务.
`Sidecars`在`Task`执行时保持运行，在`Task`执行完成后被删除.
想了解更多信息, 请参考 [`TaskRuns`中的`Sidecars`](taskruns.md#sidecars).

在以下示例中，`Step`使用Docker-in-Docker `Sidecar`来构建一个Docker镜像:

```yaml
steps:
  - image: docker
    name: client
    script: |
        #!/usr/bin/env bash
        cat > Dockerfile << EOF
        FROM ubuntu
        RUN apt-get update
        ENTRYPOINT ["echo", "hello"]
        EOF
        docker build -t hello . && docker run hello
        docker images
    volumeMounts:
      - mountPath: /var/run/
        name: dind-socket
sidecars:
  - image: docker:18.05-dind
    name: server
    securityContext:
      privileged: true
    volumeMounts:
      - mountPath: /var/lib/docker
        name: dind-storage
      - mountPath: /var/run/
        name: dind-socket
volumes:
  - name: dind-storage
    emptyDir: {}
  - name: dind-socket
    emptyDir: {}
```

与`Steps`类似，Sidecars也可以运行脚本:

```yaml
sidecars:
  image: busybox
  name: hello-sidecar
  script: |
    echo 'Hello from sidecar!'
```
**注意:** Tekton当前的`Sidecar`实现存在一个bug，Tekton使用一个名称为`nop`的镜像来终止`Sidecars`.此镜像通过传递给Tekton控制器的标记来配置. 如果配置的`nop`的镜像容器包含一个明确命令在接收到"stop"信号之前保持运行，那么`Sidecar`将会保持执行，直到`TaskRun`引发超时错误.
更多信息, 请参考 [issue 1347](https://github.com/tektoncd/pipeline/issues/1347).

### 添加描述

`description`字段是可选字段，通过此字段可允许你为`Task`设置描述信息.

### 使用便利替换

`Task`允许你为以下实体替换变量名称:

- [参数和资源]](#substituting-parameters-and-resources)
- [`Array` 参数](#substituting-array-parameters)
- [`工作区`](#substituting-workspace-paths)
- [`卷` 名称和类型](#substituting-volume-names-and-paths)

可参考 [任务完整的变量替换列表](./variables.md#variables-available-in-a-task).

#### 替换参数与资源

[`参数`](#specifying-parameters) 和 [`资源`](#specifying-resources) 属性可替换以下变量值:

- 要引用`Task`中的一个参数，使用以下语法，`<name>`为参数名称:
  ```shell
  $(params.<name>)
  ```
- 要从资源中访问参数，参考[变量替换]](resources.md#variable-substitution)

#### 替换 `Array` 参数

你可以通过使用星号操作符来展开`array`类型的参数，要实现此操作，添加星号操作符 (`[*]`)到参数名称并放置到需要插入数组元素的字符串位置.

例如，以下`params`字段包含列表，你可以展开
`command: ["first", "$(params.array-param[*])", "last"]` 到 `command: ["first", "some", "array", "elements", "last"]`:

```yaml
params:
  - name: array-param
    value:
      - "some"
      - "array"
      - "elements"
```

你 **必须** 引用`array`类型的参数到一个大的`string`数组中完全独立的字符串元素.
引用`array`参数到任务其他位置将会导致错误，例如，如果`build-args`是一个类型为`array`的参数，以下示例是无效的`step`，因为字符串不是独立的:

```yaml
 - name: build-step
      image: gcr.io/cloud-builders/some-image
      args: ["build", "additionalArg $(params.build-args[*])"]
```

同样，引用`build-args`到非`array`字段，也是无效的:

```yaml
 - name: build-step
      image: "$(params.build-args[*])"
      args: ["build", "args"]
```

一个有效引用`build-args`参数的方式是在一个合适的字段中独立引用(此例中为`args`):

```yaml
 - name: build-step
      image: gcr.io/cloud-builders/some-image
      args: ["build", "$(params.build-args[*])", "additonalArg"]
```

#### 替换 `Workspace` 路径

你可以通过以下方式替换`工作区`路径:

```yaml
$(workspaces.myworkspace.path)
```

当`Volume`名称随机化以及智能在`Task`执行时设置时，你可以通过以下方式替换卷名称:

```yaml
$(workspaces.myworkspace.volume)
```

#### 替换 `Volume` 名称和类型

你可以替换`Volume`名称和[类型](https://kubernetes.io/docs/concepts/storage/volumes/#types-of-volumes)
来参数化它们. Tekton支持常见的`Volume`类型，如`ConfigMap`, `Secret`, 以及 `PersistentVolumeClaim`.
参考此 [示例](#using-kubernetes-configmap-as-volume-source)来了解详情

## 代码示例

学习以下代码示例可更改地了解如何配置你的`Tasks`::

- [构建和推送Docker镜像](#构建和推送Docker镜像)
- [挂载多个`Volumes`](#挂载多个Volumes)
- [挂载`ConfigMap` 为`Volume` 资源](#挂载ConfigMap-为Volume-资源)
- [使用  `Secret` 作为环境变量源](#使用-Secret-作为环境变量源)
- [在`Task`中使用  `Sidecar`](#在Task中使用-Sidecar)

_提示: 查看
[示例](https://github.com/tektoncd/pipeline/tree/master/examples) 来获取更多代码示例._

### 构建和推送Docker镜像

以下示例`Task`构建并推送一个`Dockerfile`构建的镜像.

**注意:** 在集群中通过`docker build`构建镜像是**非常不安全的**，这里只作为演示目的，请使用[kaniko](https://github.com/GoogleContainerTools/kaniko) 来代替.

```yaml
spec:
  params:
    # These may be overridden, but provide sensible defaults.
    - name: directory
      type: string
      description: The directory containing the build context.
      default: /workspace
    - name: dockerfileName
      type: string
      description: The name of the Dockerfile
      default: Dockerfile
  resources:
    inputs:
      - name: workspace
        type: git
    outputs:
      - name: builtImage
        type: image
  steps:
    - name: dockerfile-build
      image: gcr.io/cloud-builders/docker
      workingDir: "$(params.directory)"
      args:
        [
          "build",
          "--no-cache",
          "--tag",
          "$(resources.outputs.image.url)",
          "--file",
          "$(params.dockerfileName)",
          ".",
        ]
      volumeMounts:
        - name: docker-socket
          mountPath: /var/run/docker.sock

    - name: dockerfile-push
      image: gcr.io/cloud-builders/docker
      args: ["push", "$(resources.outputs.image.url)"]
      volumeMounts:
        - name: docker-socket
          mountPath: /var/run/docker.sock

  # 作为示例，此Task挂载宿主的docker socket.
  volumes:
    - name: docker-socket
      hostPath:
        path: /var/run/docker.sock
        type: Socket
```

#### 挂载多个`Volumes`

以下示例演示挂载多个`卷`:

```yaml
spec:
  steps:
    - image: ubuntu
      script: |
        #!/usr/bin/env bash
        curl https://foo.com > /var/my-volume
      volumeMounts:
        - name: my-volume
          mountPath: /var/my-volume

    - image: ubuntu
      script: |
        #!/usr/bin/env bash
        cat /etc/my-volume
      volumeMounts:
        - name: my-volume
          mountPath: /etc/my-volume

  volumes:
    - name: my-volume
      emptyDir: {}
```

#### 挂载`ConfigMap` 为`Volume` 资源

以下示例演示挂载`ConfigMap` 为`Volume`:

```yaml
spec:
  params:
    - name: CFGNAME
      type: string
      description: Name of config map
    - name: volumeName
      type: string
      description: Name of volume
  steps:
    - image: ubuntu
      script: |
        #!/usr/bin/env bash
        cat /var/configmap/test
      volumeMounts:
        - name: "$(params.volumeName)"
          mountPath: /var/configmap

  volumes:
    - name: "$(params.volumeName)"
      configMap:
        name: "$(params.CFGNAME)"
```

#### 使用  `Secret` 作为环境变量源

以下示例演示使用`Secret`作为环境变量源:

```yaml
apiVersion: tekton.dev/v1beta1
kind: Task
metadata:
  name: goreleaser
spec:
  params:
  - name: package
    type: string
    description: base package to build in
  - name: github-token-secret
    type: string
    description: name of the secret holding the github-token
    default: github-token
  resources:
    inputs:
    - name: source
      type: git
      targetPath: src/$(params.package)
  steps:
  - name: release
    image: goreleaser/goreleaser
    workingDir: /workspace/src/$(params.package)
    command:
    - goreleaser
    args:
    - release
    env:
    - name: GOPATH
      value: /workspace
    - name: GITHUB_TOKEN
      valueFrom:
        secretKeyRef:
          name: $(params.github-token-secret)
          key: bot-token
```

#### 在`Task`中使用  `Sidecar`

以下示例演示在`Task`中使用`Sidecar`:

```yaml
apiVersion: tekton.dev/v1beta1
kind: Task
metadata:
  name: with-sidecar-task
spec:
  params:
  - name: sidecar-image
    type: string
    description: Image name of the sidecar container
  - name: sidecar-env
    type: string
    description: Environment variable value
  sidecars:
  - name: sidecar
    image: $(params.sidecar-image)
    env:
    - name: SIDECAR_ENV
      value: $(params.sidecar-env)
  steps:
  - name: test
    image: hello-world
```

## 调试

此节描述了调试大多数`Tasks`问题的技术.

### 审视文件结构

一个常见的问题是当配置`Tasks`根无法知道你的数据的位置时，在多数情况下输入文件或由你`Task`输出的文件位于`/workspace`目录，但是实际细节可能有所不同，要审视`Task`的文件结构，你可以添加一个步骤输出所有存储在`/workspace`目录的文件列表到构建日志，例如:

```yaml
- name: build-and-push-1
  image: ubuntu
  command:
  - /bin/bash
  args:
  - -c
  - |
    set -ex
    find /workspace
```

你也可以选择检查由你`Task`使用的文件的*内容*:

```yaml
- name: build-and-push-1
  image: ubuntu
  command:
  - /bin/bash
  args:
  - -c
  - |
    set -ex
    find /workspace | xargs cat
```

### 审视`Pod`

要审视由你`Task`任务所使用的`Pod`的内容，你可以添加一个步骤来在某个阶段暂停，例如:

```yaml
- name: pause
  image: docker
  args: ["sleep", "6000"]

```

---

除非另有说明，本页内容采用[Creative Commons Attribution 4.0 License](https://creativecommons.org/licenses/by/4.0/)授权协议，示例代码采用[Apache 2.0 License](https://www.apache.org/licenses/LICENSE-2.0)授权协议