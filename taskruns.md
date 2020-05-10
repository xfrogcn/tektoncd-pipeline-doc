<!--
---
linkTitle: "TaskRuns"
weight: 2
---
-->
# TaskRuns

使用`TaskRun`资源对象来创建及运行需要在集群中完成的处理.

要创建一个`TaskRun`，你必须首先创建一个[`Task`](tasks.md) ，通过它指定一个或多个容器镜像来处理需要完成的任务.

`TaskRun`将会一直执行，知道所有`steps`完成或发生了错误.

---

- [TaskRuns](#taskruns)
  - [语法](#语法)
    - [指定Task](#指定Task)
    - [参数](#参数)
    - [提供资源](#提供资源)
    - [配置默认超时时间](#配置默认超时时间)
    - [服务账户](#服务账户)
  - [Pod模版](#Pod模版)
  - [工作区](#工作区)
  - [状态](#状态)
    - [步骤](#步骤)
    - [结果](#结果)
  - [取消TaskRun](#取消TaskRun)
  - [示例](#取消TaskRun)
    - [TaskRun示例](#TaskRun示例)
    - [嵌入规范示例](#嵌入规范示例)
    - [重复使用Task示例](#重复使用Task示例)
      - [使用`ServiceAccount`](#使用ServiceAccount)
  - [Sidecars](#sidecars)
  - [LimitRanges](#limitranges)

---

## 语法

要为`TaskRun`定义一个配置文件，你可以指定以下字段:

- 必须:
  - [`apiVersion`][kubernetes-overview] - 指定API版本, 例如
    `tekton.dev/v1beta1`.
  - [`kind`][kubernetes-overview] - 指定为`TaskRun`资源对象.
  - [`metadata`][kubernetes-overview] - 指定`TaskRun`资源对象独有的元数据，如一个`名称（name）`.
  - [`spec`][kubernetes-overview] - 指定`TaskRun`资源对象的配置信息.
    - [`taskRef` 或 `taskSpec`](#指定Task) - 指定你想要运行的[`Task`](tasks.md) 详情
- 可选:

  - [`serviceAccountName`](#服务账户) - 指定`ServiceAccount`资源对象，它允许你的构建在特定授权信息下运行。当`ServiceAccount`未指定时，将使用由配置映射`config-defaults`中配置的`default-service-account`的值.
  - [`params`](#parameters) - 指定参数值
  - [`resources`](#提供资源) - 指定 `PipelineResource` 值
    - [`inputs`] - 指定输入资源
    - [`outputs`] - 指定输出资源
  - [`timeout`] - 指定`TaskRun`超时失败时间，如果`timeout`为空，将使用默认的超时时间，如果值设置为0，表示没有超时时间，你可以参考[此处](#配置默认超时时间).
  - [`podTemplate`](#pod模版) - 指定一个[pod 模版](./podtemplates.md) ，它将被用于基础的`Task`pod.
  - [`workspaces`](#工作区) - 为`Task`中定义的[工作区]](workspaces.md#declaring-workspaces-in-tasks)指定真实的卷.

[kubernetes-overview]:
  https://kubernetes.io/docs/concepts/overview/working-with-objects/kubernetes-objects/#required-fields

### 指定Task

由于`TaskRun`是[`Task`](tasks.md)的调用者, 你必须指定那个`Task`要被运行.

你可以通过提供一个已存在`Task`的引用:

```yaml
spec:
  taskRef:
    name: read-task
```

或者你可以直接内嵌`Task`规范来实现:

```yaml
spec:
  taskSpec:
    resources:
      inputs:
        - name: workspace
          type: git
    steps:
      - name: build-and-push
        image: gcr.io/kaniko-project/executor:v0.17.1
        # 定义DOCKER_CONFIG是必须的，以便kaniko来获取docker凭据
        env:
          - name: "DOCKER_CONFIG"
            value: "/tekton/home/.docker/"
        command:
          - /kaniko/executor
        args:
          - --destination=gcr.io/my-project/gohelloworld
```

### 参数

如果一个`Task`具有[`参数`](tasks.md#详述-Parameters), 你可以通过`params`来设置它们的值:

```yaml
spec:
  params:
    - name: flags
      value: -someflag
```

如果一个参数没有默认值，它必须被指定.

### 提供资源

如果一个`Task`需要[输入资源](tasks.md#input-resources) 或者
[输出资源](tasks.md#output-resources), 它们必须提供来运行`Task`.

它们可以通过引用已存在的[`PipelineResources`](resources.md)资源来完成:

```yaml
spec:
  resources:
    inputs:
      - name: workspace
        resourceRef:
          name: java-git-resource
    outputs:
      - name: image
        resourceRef:
          name: my-app-image
```

或者直接内嵌资源对象定义:

```yaml
spec:
  resources:
    inputs:
      - name: workspace
        resourceSpec:
          type: git
          params:
            - name: url
              value: https://github.com/pivotal-nader-ziada/gohelloworld
```

`paths`字段可以被用于[覆盖资源的路径](./resources.md#overriding-where-resources-are-copied-from)

### 配置默认超时时间

你可以通过修改[`config/config-defaults.yaml`](./../config/config-defaults.yaml)中的`default-timeout-minutes`值来修改默认超时时间.

`timeout`的格式为Go语言[`ParseDuration`](https://golang.org/pkg/time/#ParseDuration)`时长`，例如:

- `1h30m`
- `1h`
- `1m`
- `60s`

如果`default-timeout-minutes`无效，默认的超时时间为60分钟，如果`default-timeout-minutes`设置为0，则没有默认超时时间.

### 服务账户

指定`ServiceAccount`资源对象的名称。使用`serviceAccountName`字段指定的服务账户来运行你的任务. 如果未指定`serviceAccountName`字段，你的`Task`将会使用配置映射`configmap-defaults`中设置的值，如果没有，将使用`TaskRun`资源对象所在[命名空间](https://kubernetes.io/docs/concepts/overview/working-with-objects/namespaces/)的
[`default`服务账户](https://kubernetes.io/docs/tasks/configure-pod-container/configure-service-account/#use-the-default-service-account-to-access-the-api-server).

有关指定服务账户的示例和更多信息，可参考[`服务账户`](./auth.md) 相关主题.

## Pod模版

指定一个[pod模版](./podtemplates.md) 配置，它将由`Task`Pod作为基础模版来使用. 这允许为每一个`Task`自定义一些Pod规则字段.

在以下示例中，任务定义了一个`volumeMount`(`my-cache`),它由TaskRun来指定，使用了一个持久化卷。通过SchedulerName字段指定使用哪一个调度器来调度Pod。Pod将作为非根用户运行，HostNetwork设置为true将使用宿主的网络空间.

```yaml
apiVersion: tekton.dev/v1beta1
kind: Task
metadata:
  name: mytask
  namespace: default
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
kind: TaskRun
metadata:
  name: mytaskrun
  namespace: default
spec:
  taskRef:
    name: mytask
  podTemplate:
    schedulerName: volcano
    hostNetwork: true
    securityContext:
      runAsNonRoot: true
    volumes:
    - name: my-cache
      persistentVolumeClaim:
        claimName: my-volume-claim
```

## 工作区

要执行定义有`工作区`的`Task`,必须将`工作区`映射到真实物理卷.

以下是`TaskTun`中使用`PersistentVolumeClaim`作为工作区的相关字段:

```yaml
workspaces:
- name: myworkspace # 必须和Task中定义的工作区名称一致
  persistentVolumeClaim:
    claimName: mypvc # 此PVC必须存在
  subPath: my-subdir
```

有关在`TaskRun`中配置`工作区`的更多示例及完整文档，请参考[workspaces.md](./workspaces.md#providing-workspaces-with-taskruns).

Tekton支持多种不同`Volume`类型的`工作区`，有关工作区的不同类型，参考[工作区的`VolumeSources`](workspaces.md#volumesources-for-workspaces).


_有关完整示例参考示例目录下的[TaskRun工作区](../examples/v1beta1/taskruns/workspace.yaml)._

## 状态

当一个TaskRun完成，它的`status`字段中将填充整个运行期间的信息，
并且包含每一个步骤.

以下示例展示了一个完成的TaskRun及其`status`字段:

```yaml
completionTime: "2019-08-12T18:22:57Z"
conditions:
- lastTransitionTime: "2019-08-12T18:22:57Z"
  message: All Steps have completed executing
  reason: Succeeded
  status: "True"
  type: Succeeded
podName: status-taskrun-pod-6488ef
startTime: "2019-08-12T18:22:51Z"
steps:
- container: step-hello
  imageID: docker-pullable://busybox@sha256:895ab622e92e18d6b461d671081757af7dbaa3b00e3e28e12505af7817f73649
  name: hello
  terminated:
    containerID: docker://d5a54f5bbb8e7a6fd3bc7761b78410403244cf4c9c5822087fb0209bf59e3621
    exitCode: 0
    finishedAt: "2019-08-12T18:22:56Z"
    reason: Completed
    startedAt: "2019-08-12T18:22:54Z"
  ```

字段包含`TaskRun`的开始和结束时间，以及每一个步骤的退出代码，针对每一个步骤同时包含完整的镜像摘要.

如果任何Pod发生[`OOMKilled`](https://kubernetes.io/docs/tasks/administer-cluster/out-of-resource/), `TaskRun`将会标记为失败，尽管它的退出代码可能为0.

### 步骤

如果多个定义在`Task`中的步骤被`TaskRun`调用，我们将会看到`TaskRun`中`status.steps`展示的步骤顺序与`Task`中`spec.steps`中定义顺序一致，当使用`get`命令获取`TaskRun`时，例如`kubectl get taskrun <name> -o yaml`. 替换 \<name\> 为`TaskRun`的名称.

### 结果

如果通过`TaskRun`调用了由`Task`定义的一个或多个`results`,我们将在状态节点中获取到`Task Results`. 例如:

```yaml
Status:
  # […]
  Steps:
  # […]
  Task Results:
    Name:   current-date-human-readable
    Value:  Thu Jan 23 16:29:06 UTC 2020

    Name:   current-date-unix-timestamp
    Value:  1579796946

```

结果将会被逐字被打印，任何新行或则其他空白换行将会包含在输出中

## 取消TaskRun

要取消一个正在执行任务的(`TaskRun`), 你需要更新它的spec，将其状态标记为取消. 正在运行的Pod将会被删除.

```yaml
apiVersion: tekton.dev/v1alpha1
kind: TaskRun
metadata:
  name: go-example-git
spec:
  # […]
  status: "TaskRunCancelled"
```

## 示例

- [TaskRun示例](#TaskRun示例)
- [嵌入规范示例](#嵌入规范示例)
- [重复使用Task示例](#重复使用Task示例)

### TaskRun示例

要运行一个`Task`，创建一个新的`TaskRun`,通过它指定运行`Task`所需的输入、输出。以下示例通过创建`read-repo-run`来运行`read-task`任务，任务`read-task`有一个git输入资源，TaskRun`read-repo-run`包含了`go-example-git`的git资源引用.

```yaml
apiVersion: tekton.dev/v1alpha1
kind: PipelineResource
metadata:
  name: go-example-git
spec:
  type: git
  params:
    - name: url
      value: https://github.com/pivotal-nader-ziada/gohelloworld
---
apiVersion: tekton.dev/v1beta1
kind: Task
metadata:
  name: read-task
spec:
  resources:
    inputs:
      - name: workspace
        type: git
  steps:
    - name: readme
      image: ubuntu
      script: cat workspace/README.md
---
apiVersion: tekton.dev/v1beta1
kind: TaskRun
metadata:
  name: read-repo-run
spec:
  taskRef:
    name: read-task
  resources:
    inputs:
      - name: workspace
        resourceRef:
          name: go-example-git
```

### 嵌入规范示例

运行Task的另一种方式是在taskRun中嵌入Task资源.
这适合于只运行一次的任务，或者调试. TaskRun资源可以包含Task引用或则TaskSpec，当不能同时包含. 以下示例`build-push-task-run-2` 包含 `TaskSpec` 而没有引用Task.

```yaml
apiVersion: tekton.dev/v1alpha1
kind: PipelineResource
metadata:
  name: go-example-git
spec:
  type: git
  params:
    - name: url
      value: https://github.com/pivotal-nader-ziada/gohelloworld
---
apiVersion: tekton.dev/v1beta1
kind: TaskRun
metadata:
  name: build-push-task-run-2
spec:
  resources:
    inputs:
      - name: workspace
        resourceRef:
          name: go-example-git
  taskSpec:
    resources:
      inputs:
        - name: workspace
          type: git
    steps:
      - name: build-and-push
        image: gcr.io/kaniko-project/executor:v0.17.1
        # 必须指定DOCKER_CONFIG，以便kaniko获取docker凭据
        env:
          - name: "DOCKER_CONFIG"
            value: "/tekton/home/.docker/"
        command:
          - /kaniko/executor
        args:
          - --destination=gcr.io/my-project/gohelloworld
```

输入和输出资源同样也可以内嵌而无需创建管道资源. TaskRun资源可以包含管道资源的引用或则管道资源定义，但不能同时包含. 以下示例中，Git管道资源定义直接提供给运行`read-repo`任务的TaskRun.

```yaml
apiVersion: tekton.dev/v1beta1
kind: TaskRun
metadata:
  name: read-repo
spec:
  taskRef:
    name: read-task
  resources:
    inputs:
      - name: workspace
        resourceSpec:
          type: git
          params:
            - name: url
              value: https://github.com/pivotal-nader-ziada/gohelloworld
```

**注意**: TaskRun可以通过内嵌Task定义和资源定义.`TaskRun`也可作为运行`Task`任务的历史记录.

### 重复使用Task示例

有以下几个示例阐述了如何重用任务
[`TaskRuns`](taskruns.md) (包含引用
[`PipelineResources`](resources.md)) 实例化
[`Task` (`dockerfile-build-and-push`) 在`Task` 示例文档中](tasks.md#example-task).

构建 `mchmarny/rester-tester`:

```yaml
# The PipelineResource
metadata:
  name: mchmarny-repo
spec:
  type: git
  params:
    - name: url
      value: https://github.com/mchmarny/rester-tester.git
```

```yaml
# The TaskRun
spec:
  taskRef:
    name: dockerfile-build-and-push
  params:
    - name: IMAGE
      value: gcr.io/my-project/rester-tester
  resources:
    inputs:
      - name: workspace
        resourceRef:
          name: mchmarny-repo
```

构建 `googlecloudplatform/cloud-builder`'的 `wget` 构建器:

```yaml
# The PipelineResource
metadata:
  name: cloud-builder-repo
spec:
  type: git
  params:
    - name: url
      value: https://github.com/googlecloudplatform/cloud-builders.git
```

```yaml
# The TaskRun
spec:
  taskRef:
    name: dockerfile-build-and-push
  params:
    - name: IMAGE
      value: gcr.io/my-project/wget
    # Optional override to specify the subdirectory containing the Dockerfile
    - name: DIRECTORY
      value: /workspace/wget
  resources:
    inputs:
      - name: workspace
        resourceRef:
          name: cloud-builder-repo
```

构建 `googlecloudplatform/cloud-builder`的`docker` 构建器:

```yaml
# The PipelineResource
metadata:
  name: cloud-builder-repo
spec:
  type: git
  params:
    - name: url
      value: https://github.com/googlecloudplatform/cloud-builders.git
```

```yaml
# The TaskRun
spec:
  taskRef:
    name: dockerfile-build-and-push
  params:
    - name: IMAGE
      value: gcr.io/my-project/docker
    # Optional overrides
    - name: DIRECTORY
      value: /workspace/docker
    - name: DOCKERFILE_NAME
      value: Dockerfile-17.06.1
  resources:
    inputs:
      - name: workspace
        resourceRef:
          name: cloud-builder-repo
```

#### 使用`ServiceAccount`

定义一个 `ServiceAccount` 来访问私有`git` 仓储:

```yaml
apiVersion: tekton.dev/v1beta1
kind: TaskRun
metadata:
  name: test-task-with-serviceaccount-git-ssh
spec:
  serviceAccountName: test-task-robot-git-ssh
  resources:
    inputs:
      - name: workspace
        type: git
  steps:
    - name: config
      image: ubuntu
      command: ["/bin/bash"]
      args: ["-c", "cat README.md"]
```

 `serviceAccountName: test-build-robot-git-ssh`引用以下的
`ServiceAccount`:

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: test-task-robot-git-ssh
secrets:
  - name: test-git-ssh
```

 `name: test-git-ssh`, 引用以下的`Secret`:

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: test-git-ssh
  annotations:
    tekton.dev/git-0: github.com
type: kubernetes.io/ssh-auth
data:
  # Generated by:
  # cat id_rsa | base64 -w 0
  ssh-privatekey: LS0tLS1CRUdJTiBSU0EgUFJJVk.....[example]
  # Generated by:
  # ssh-keyscan github.com | base64 -w 0
  known_hosts: Z2l0aHViLmNvbSBzc2g.....[example]
```

指定`ServiceAccount`资源对象的`name`. 使用`serviceAccountName`字段来在指定的服务账户权限下运行你的`Task`. 如果没有指定`serviceAccountName` 字段，你的`Task`将使用与`Task`资源对象相同[命名空间](https://kubernetes.io/docs/concepts/overview/working-with-objects/namespaces/)下的[`default` 服务账户](https://kubernetes.io/docs/tasks/configure-pod-container/configure-service-account/#use-the-default-service-account-to-access-the-api-server)
来运行.

有关指定服务账户的更多示例及信息，请参考[`ServiceAccount`](./auth.md) 相关主题.

## Sidecars

在Kubernetes中已经固定的模式“sidecar”——一种容器可以与你的工作负载同时运行来提供辅助功能.
常见的sidecar模式示例是日志、更新文件的服务以及网络代理.

Tekton很乐意为运行TaskRun的Pod注入sidecars，但是边车的行为不太一样：当TaskRun步骤完成时，任何Pod中包含的边车容器将会被终止. 为了终止sidecars，它将重启一个新的"nop"镜像然后快速退出.这样的结果是你的TaskRun Pod将会包含一个边车容器，它的重试次数为1，以及一个你与你期望不一致的容器镜像.

注意: 这里有一些当前边车实现已知的问题:

- 配置的“nop”镜像不能提供sidecar预期运行的命令，如果提供了命令，它将不会退出，这将导致sidecar一直运行，直接任务超时事件发生。[此问题链接](https://github.com/tektoncd/pipeline/issues/1347)

- 执行`kubectl get pods` 命令，如果sidecar成功退出，TaskRun的Pod将显示为完成状态，如果sidecar发生错误退出，Pod将显示错误状态，而不管步骤容器退出的状态如何，此问题仅仅在`get pods`命令出现，Pod的描述将会显示失败状态，并包含每一个单独容器的状态及退出原因.

## LimitRanges

为了为`TaskRun`中的`steps`容器请求运行是需要的最小资源，Tekton仅请求TaskRun中`steps`中的最大cpu、内存以及临时存储，只需要最大的单个容器资源，因为`TaskRun` Pod中步骤只有一次执行一个。所有不是最大值的请求都将被设置为0.

当在命名空间中通过[LimitRange](https://kubernetes.io/docs/concepts/policy/limit-range/)设置了容器的最小资源请求时（如CPU、内存、临时存储），Tekton将会搜索命名空间所有Limitranges，并将容器资源请求设置为这些值，而不是设置为0.

使用LimitRange的`TaskRun`示例可参考[此处](../examples/v1beta1/taskruns/no-ci/limitrange.yaml).

---

除非另有说明，本页内容采用[Creative Commons Attribution 4.0 License](https://creativecommons.org/licenses/by/4.0/)授权协议，示例代码采用[Apache 2.0 License](https://www.apache.org/licenses/LICENSE-2.0)授权协议
