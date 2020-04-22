# Tekton管道手册

本手册通过一个简单的`Hello World`示例来展示如何：
- 创建一个`任务(Task)`
- 创建一个包含你`任务(Task)`的`管道(Pipeline)`
- 使用`TaskRun`来实例化并执行`任务(Task)`
- 使用`PipelineRun`来实例化并运行包含你`任务(Task)`的`管道(Pipeline)`

本手册包含一下章节：
- [创建并运行一个`Task`](#创建并运行一个`任务`)
- [创建并运行一个`Pipeline`](#创建并运行一个`Pipeline`)

**注意:** 必须要配置的项通过`#配置`注释来标注，可能包括Docker仓库，日志输出位置以及其他配置项，你必须替换为实际配置。

## 开始之前

在开始之前，请确保你已经在Kubernetes集群中[安装和配置](install.md)了最新的Tekton，包括[Tekton命令行](https://github.com/tektoncd/cli)

如果你想要在本地工作站完成此手册，请参考[在本地运行](#在本地运行)，想要获取更多本手册相关的Tekton实体，请参考[延伸阅读](#延伸阅读)

## 创建并运行一个`任务`

[`Task`](tasks.md)定义一系列的`步骤`以预期的顺序来完成构建工作。每一个`Task`在Kubernetes集群中作为一个Pod来运行，每一个`步骤`作为Pod中独立的容器。例如，以下示例`Task`，输出"Hello World":

```yaml
apiVersion: tekton.dev/v1beta1
kind: Task
metadata:
  name: echo-hello-world
spec:
  steps:
    - name: echo
      image: ubuntu
      command:
        - echo
      args:
        - "Hello World"
```

通过以下命令应用`Task` YAML文件：

```bash
kubectl apply -f <name-of-task-file.yaml>
```

要查看`Task`的详细信息, 可使用以下命令:  
```bash
tkn task describe echo-hello-world
```

命令输出结果如下:

```
Name:        echo-hello-world
Namespace:   default

📨 Input Resources

 No input resources

📡 Output Resources

 No output resources

⚓ Params

 No params

🦶 Steps

 ∙ echo

🗂  Taskruns

 No taskruns
```

要运行此`Task`, 通过使用[`TaskRun`](taskruns.md)来实例化:

```yaml
apiVersion: tekton.dev/v1beta1
kind: TaskRun
metadata:
  name: echo-hello-world-task-run
spec:
  taskRef:
    name: echo-hello-world
```

通过以下命令来应用`TaskRun` YAML文件:

```bash
kubectl apply -f <name-of-taskrun-file.yaml>
```

通过以下命令查询`TaskRun`是否执行成功:

```bash
tkn taskrun describe echo-hello-world-task-run
```

命令输出结果如下:

```
Name:        echo-hello-world-task-run
Namespace:   default
Task Ref:    echo-hello-world

Status
STARTED         DURATION    STATUS
4 minutes ago   9 seconds   Succeeded

Input Resources
No resources

Output Resources
No resources

Params
No params

Steps
NAME
echo
```

状态为`Succeeded`表示`TaskRun`成功无错误地完成.

要查看`TaskRun`运行的详细日志, 可通过如下命令:

```bash
tkn taskrun logs echo-hello-world-task-run
```

此命令输出结果如下:

```
[echo] hello world
```

### 指定`Task`的输入与输出

在更为复杂的场景中，你需要为`Task`定义输入与输出，例如`Task`实现从GitHub仓库拉取代码然后构建Docker镜像的任务

通过使用[`PipelineResources`](resources.md)来定义你需要传入或由`Task`输出的物料，以下示例为常见的资源定义

[`git`资源](resources.md#git-resource) 指定一个git仓库和特定的版本来指明`Task`将要拉取代码的位置:

```yaml
apiVersion: tekton.dev/v1alpha1
kind: PipelineResource
metadata:
  name: skaffold-git
spec:
  type: git
  params:
    - name: revision
      value: master
    - name: url
      value: https://github.com/GoogleContainerTools/skaffold #配置: 如果你要构建其他的仓库，请修改为你的本地git仓库地址
```

[`image`资源](resources.md#image-resource) 定义`Task`构建的镜像将要push到的镜像仓库位置:

```yaml
apiVersion: tekton.dev/v1alpha1
kind: PipelineResource
metadata:
  name: skaffold-image-leeroy-web
spec:
  type: image
  params:
    - name: url
      value: gcr.io/<use your project>/leeroy-web #配置: 替换为实际的仓库路径: 可能是你的本地仓库或Dockerhub，并且已通过service account配置了密文
```

在以下示例中，你可以看到一个`Task`定义了`git`输入以及`image`输出，`Task`命令的参数支持变量替换，所以`Task`定义是固定的，但它的值可在运行时改变

```yaml
apiVersion: tekton.dev/v1beta1
kind: Task
metadata:
  name: build-docker-image-from-git-source
spec:
  params:
    - name: pathToDockerFile
      type: string
      description: The path to the dockerfile to build
      default: $(resources.inputs.docker-source.path)/Dockerfile
    - name: pathToContext
      type: string
      description: |
        The build context used by Kaniko
        (https://github.com/GoogleContainerTools/kaniko#kaniko-build-contexts)
      default: $(resources.inputs.docker-source.path)
  resources:
    inputs:
      - name: docker-source
        type: git
    outputs:
      - name: builtImage
        type: image
  steps:
    - name: build-and-push
      image: gcr.io/kaniko-project/executor:v0.17.1
      # 必须指定DOCKER_CONFIG，以便kaniko获取到docker凭证
      env:
        - name: "DOCKER_CONFIG"
          value: "/tekton/home/.docker/"
      command:
        - /kaniko/executor
      args:
        - --dockerfile=$(params.pathToDockerFile)
        - --destination=$(resources.outputs.builtImage.url)
        - --context=$(params.pathToContext)
```

### 配置`Task`执行凭证

在你执行`TaskRun`之前，你必须创建一个`secret`来push镜像到你的仓库:

```bash
kubectl create secret docker-registry regcred \
                    --docker-server=<your-registry-server> \
                    --docker-username=<your-name> \
                    --docker-password=<your-pword> \
                    --docker-email=<your-email>
```

然后你可以通过`ServiceAccount`来关联此`secret`，通过此服务账户来执行你的`TaskRun`:

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: tutorial-service
secrets:
  - name: regcred
```

保存上述`ServiceAccount`定义为文件，然后通过kubectl命令应用此YAML文件:

```bash
kubectl apply -f <name-of-file.yaml>
```

### 运行`Task`

到此，你已经为第一个`TaskRun`做好准备了!

`TaskRun`绑定了通过`PipelineResources`定义的输入和输出，然后通过变量替换设置参数值，最后执行`Task`中的`Steps`

```yaml
apiVersion: tekton.dev/v1beta1
kind: TaskRun
metadata:
  name: build-docker-image-from-git-source-task-run
spec:
  serviceAccountName: tutorial-service
  taskRef:
    name: build-docker-image-from-git-source
  params:
    - name: pathToDockerFile
      value: Dockerfile
    - name: pathToContext
      value: $(resources.inputs.docker-source.path)/examples/microservices/leeroy-web #配置: 你可以修改为自己的源码路径
  resources:
    inputs:
      - name: docker-source
        resourceRef:
          name: skaffold-git
    outputs:
      - name: builtImage
        resourceRef:
          name: skaffold-image-leeroy-web
```

保存此YAML文件，文件包含你的`Task`, `TaskRun`, 以及 `PipelineResource`定义，然后通过以下命令应用:

```bash
kubectl apply -f <name-of-file.yaml>
```

通过以下命令可检查以上动作所创建的资源:

```bash
kubectl get tekton-pipelines
```

命令输出如下:

```
NAME                                                   AGE
taskruns/build-docker-image-from-git-source-task-run   30s

NAME                                          AGE
pipelineresources/skaffold-git                6m
pipelineresources/skaffold-image-leeroy-web   7m

NAME                                       AGE
tasks/build-docker-image-from-git-source   7m
```

通过以下命令查看`TaskRun`执行结果:

```bash
tkn taskrun describe build-docker-image-from-git-source-task-run
```

命令输出结果如下:

```
Name:        build-docker-image-from-git-source-task-run
Namespace:   default
Task Ref:    build-docker-image-from-git-source

Status
STARTED       DURATION     STATUS
2 hours ago   56 seconds   Succeeded

Input Resources
NAME            RESOURCE REF
docker-source   skaffold-git

Output Resources
NAME         RESOURCE REF
builtImage   skaffold-image-leeroy-web

Params
NAME               VALUE
pathToDockerFile   Dockerfile
pathToContext      /workspace/docker-source/examples/microservices/leeroy-web

Steps
NAME
build-and-push
create-dir-builtimage-wtjh9
git-source-skaffold-git-tck6k
image-digest-exporter-hlbsq
```

`Succeeded`状态代表`Task`任务成功无错误地完成。你也可以确认输出的Docker镜像已经push到了资源定义的位置。

要查看`TaskRun`执行过程中的详细信息, 可查看其日志:

```bash
tkn taskrun logs build-docker-image-from-git-source-task-run
```

## 创建并运行一个`Pipeline`

[`Pipeline`](pipelines.md) 包含一组想要执行的有序的`Task`，以及每一个`Task`相关的输入与输出。通过[`from`](pipelines.md#from) 属性, 你可以将一个`Task`的输出作为另一个`Task`的输入.
`Pipelines`同样提供与`Task`一致的变量替换功能

以下为一个`Pipeline`示例:

```yaml
apiVersion: tekton.dev/v1beta1
kind: Pipeline
metadata:
  name: tutorial-pipeline
spec:
  resources:
    - name: source-repo
      type: git
    - name: web-image
      type: image
  tasks:
    - name: build-skaffold-web
      taskRef:
        name: build-docker-image-from-git-source
      params:
        - name: pathToDockerFile
          value: Dockerfile
        - name: pathToContext
          value: /workspace/docker-source/examples/microservices/leeroy-web #配置: 可修改为你的实际代码位置
      resources:
        inputs:
          - name: docker-source
            resource: source-repo
        outputs:
          - name: builtImage
            resource: web-image
    - name: deploy-web
      taskRef:
        name: deploy-using-kubectl
      resources:
        inputs:
          - name: source
            resource: source-repo
          - name: image
            resource: web-image
            from:
              - build-skaffold-web
      params:
        - name: path
          value: /workspace/source/examples/microservices/leeroy-web/kubernetes/deployment.yaml #配置: 可修改为你的实际代码位置
        - name: yamlPathToImage
          value: "spec.template.spec.containers[0].image"
```

以上`Pipeline`引用了一个名称为`deploy-using-kubectl`的`Task`，其定义如下:

```yaml
apiVersion: tekton.dev/v1beta1
kind: Task
metadata:
  name: deploy-using-kubectl
spec:
  params:
    - name: path
      type: string
      description: Path to the manifest to apply
    - name: yamlPathToImage
      type: string
      description: |
        The path to the image to replace in the yaml manifest (arg to yq)
  resources:
    inputs:
      - name: source
        type: git
      - name: image
        type: image
  steps:
    - name: replace-image
      image: mikefarah/yq
      command: ["yq"]
      args:
        - "w"
        - "-i"
        - "$(params.path)"
        - "$(params.yamlPathToImage)"
        - "$(resources.inputs.image.url)"
    - name: run-kubectl
      image: lachlanevenson/k8s-kubectl
      command: ["kubectl"]
      args:
        - "apply"
        - "-f"
        - "$(params.path)"
```

### 配置`Pipeline`的执行凭证

以上`run-kubectl`步骤需要附加权限，你必须赋予这些权限到你的`ServiceAccount`

首先，创建一个新的角色`tutorial-role`:

```bash
kubectl create clusterrole tutorial-role \
               --verb=* \
               --resource=deployments,deployments.apps
```

然后, 将此角色赋给`ServiceAccount`:

```bash
kubectl create clusterrolebinding tutorial-binding \
             --clusterrole=tutorial-role \
             --serviceaccount=default:tutorial-service
```

要运行`Pipeline`, 可通过[`PipelineRun`](pipelineruns.md) 来实例化:

```yaml
apiVersion: tekton.dev/v1beta1
kind: PipelineRun
metadata:
  name: tutorial-pipeline-run-1
spec:
  serviceAccountName: tutorial-service
  pipelineRef:
    name: tutorial-pipeline
  resources:
    - name: source-repo
      resourceRef:
        name: skaffold-git
    - name: web-image
      resourceRef:
        name: skaffold-image-leeroy-web
```

`PipelineRun`将会自动为每一个的`Task`定义相关的`TaskRun`，然后收集每一个执行`Task`的结果， 在上述示例中，`TaskRun`的顺序如下:

1. `tutorial-pipeline-run-1-build-skaffold-web` 运行 `build-skaffold-web`,
   因为他们没有[`from` 或者 `runAfter` 子句](pipelines.md#ordering).
1. `tutorial-pipeline-run-1-deploy-web` 运行 `deploy-web` 因为其[input](tasks.md#inputs) `web-image` 来自于 [`from`](pipelines.md#from)
   `build-skaffold-web`. 因此, `build-skaffold-web` 必须在`deploy-web`任务之前运行

保存上述`Task`, `Pipeline`, 以及 `PipelineRun`定义到YAML文件，然后使用以下命令应用:

```bash
kubectl apply -f <name-of-file.yaml>
```
**注意:** Also apply the `deploy-task` or the `PipelineRun` will not execute.

你可以通过以下命令监控`PipelineRun`的运行状态:

```bash
tkn pipelinerun logs tutorial-pipeline-run-1 -f
```

要查看`PipelineRun`运行的详细信息, 可使用以下命令:

```bash
tkn pipelinerun describe tutorial-pipeline-run-1
```

命令输出结果如下:

```bash
Name:           tutorial-pipeline-run-1
Namespace:      default
Pipeline Ref:   tutorial-pipeline

Status
STARTED       DURATION   STATUS
4 hours ago   1 minute   Succeeded

Resources
NAME          RESOURCE REF
source-repo   skaffold-git
web-image     skaffold-image-leeroy-web

Params
No params

Taskruns
NAME                                               TASK NAME            STARTED       DURATION     STATUS
tutorial-pipeline-run-1-deploy-web-jjf2l           deploy-web           4 hours ago   14 seconds   Succeeded
tutorial-pipeline-run-1-build-skaffold-web-7jgjh   build-skaffold-web   4 hours ago   1 minute     Succeeded
```

`Succeded`状态表示`PipelineRun`无错误完成。
你也可以查看每个独立的`TaskRuns`的状态.

## 在本地运行

本节提供在本地工作站完成此向导的步骤

### 先决条件

要在本地完成此手册，需先完成以下事项:

- 安装[必要工具](https://github.com/tektoncd/pipeline/blob/master/DEVELOPMENT.md#requirements).
- 安装 [Docker for Desktop](https://www.docker.com/products/docker-desktop) 并配置其使用6个CPU,
  10 GB内存以及2GB的交换空间
- 设置`host.docker.local:5000`为不安全docker仓库. 参考[Docker非安全仓库文档](https://docs.docker.com/registry/insecure/)了解更多详情.
- 传入 `--insecure` 参数到Kaniko任务，以便可以push镜像到非安全仓库.
- 运行本地(非安全)Docker仓库:

  `docker run -d -p 5000:5000 --name registry-srv -e REGISTRY_STORAGE_DELETE_ENABLED=true registry:2`

- (可选) 安装Docker仓库查看来验证镜像是否被成功push:

`docker run -it -p 8080:8080 --name registry-web --link registry-srv -e REGISTRY_URL=http://registry-srv:5000/v2 -e REGISTRY_NAME=localhost:5000 hyper/docker-registry-web`

- 为了验证，你可以push镜像到 `host.docker.internal:5000/myregistry/<image_name>`.

### 重新配置 `image` 资源

你必须重新配置`PipelineResources`中的`image`资源:

- 设置url为 `host.docker.internal:5000/myregistry/<image_name>`
- 设置 `KO_DOCKER_REPO` 变量为`localhost:5000/myregistry`
- 设置你的应用(如部署定义)push到
  `localhost:5000/myregistry/<image name>`.

### 重新配置日志

- 你可以在内存中保留日志，而不将其发送到日志服务如[Stackdriver](https://cloud.google.com/logging/).
- 你可以部署Elasticsearch, Beats, 或者 Kibana 来查询日志. 你可以从以下位置查看配置示例 <https://github.com/mgreau/tekton-pipelines-elastic-tutorials>.
- 要了解日志的更详细信息，参考[日志](logs.md).

## 延伸阅读

要想详细了解本手册调用的Tekton管道实体信息，可参考以下主题:

- [`Tasks`](tasks.md)
- [`TaskRuns`](taskruns.md)
- [`Pipelines`](pipelines.md)
- [`PipelineResources`](resources.md)
- [`PipelineRuns`](pipelineruns.md)

---

除非另有说明，本页内容采用[Creative Commons Attribution 4.0 License](https://creativecommons.org/licenses/by/4.0/)授权协议，示例代码采用[Apache 2.0 License](https://www.apache.org/licenses/LICENSE-2.0)授权协议
