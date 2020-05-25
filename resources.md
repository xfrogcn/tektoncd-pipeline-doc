<!--
---
linkTitle: "PipelineResources"
weight: 6
---
-->
# 管道资源

管道中的`PipelineResources`是一组对象，可用于[`Task`](tasks.md)的输入及作为`Task`
的输出.

`Task`可以有多个输入和输出.

例如:

-   `Task`输入可以为GitHub源，包含你应用的代码.
-   `Task`的输出可以是你的应用镜像，可用于部署到集群中.
-   `Task`可以为jar文件，可上传到存储桶.

> **注意**: 管道资源尚未提升到Beta版本，这表示`PipelineResources`的支持还是Alpha状态，并不保证此类型在未来的支持. Tekton开发人员对其进行了重新评估，但是仍然存在很多已知问题.
>
> 有关PipeResources的可选Beta支持，可参考[v1alpha1 到 v1beta1 合并向导](./migrating-v1alpha1-to-v1beta1.md#pipelineresources-and-catalog-tasks)
> ，这列出了每一个类型的管道资源以及建议的替换建议.
>
> 有关管道资源为什么还是alpha状态的更多信息[参考这些问题的描述，以及接下来的步骤](#why-arent-pipelineresources-in-beta).

--------------------------------------------------------------------------------

-   [语法](#语法)
-   [使用资源](#使用资源)
    -   [变量替换](#变量替换)
    -   [控制资源挂载位置](#控制资源挂载位置)
    -   [覆盖资源复制到的位置](#覆盖资源复制到的位置)
    -   [资源状态](#资源状态)
    -   [可选的资源](#可选的资源)
-   [资源类型](#资源类型)
    -   [Git 资源](#git-资源)
    -   [Pull Request 资源](#pull-request-资源)
    -   [Image 资源](#image-资源)
    -   [Cluster 资源](#cluster-资源)
    -   [Storage 资源](#storage-资源)
        -   [GCS存储资源](#GCS存储资源)
        -   [BuildGCS存储资源](#BuildGCS存储资源)
    -   [云事件资源](#云事件资源)
-   [为何PipelineResources未提升到Beta?](#为何PipelineResources未提升到Beta)

## 语法

要定义一个`PipelineResource`配置文件，你可以指定以下字段:

-   必须:
    -   [`apiVersion`][kubernetes-overview] - 指定API版本, 如`tekton.dev/v1alpha1`.
    -   [`kind`][kubernetes-overview] - 指定为`PipelineResource`资源对象
    -   [`metadata`][kubernetes-overview] - 指定管道资源对象的元数据，如`名称(name)`.
    -   [`spec`][kubernetes-overview] - 管道资源的配置信息.
        -   [`type`](#资源类型) - 管道资源的类型
-   可选:
    -   [`description`](#描述) -  资源的描述.
    -   [`params`](#资源类型) - 针对每一种管道资源类型的参数设置
    -   [`optional`](#可选的资源) - 标记资源是否为可选(默认下, `optional` 设置为 `false`, 标记资源为必须).

[kubernetes-overview]:
  https://kubernetes.io/docs/concepts/overview/working-with-objects/kubernetes-objects/#required-fields

## 使用资源

资源可用于[Tasks](./tasks.md) 以及
[Conditions](./conditions.md#resources).

输入资源，像源代码(git)或者工件，被转储到路径`/workspace/task_resource_name`, 在挂载的[volume](https://kubernetes.io/docs/concepts/storage/volumes/) 中，并在所有[`steps`](#步骤)中有效. 资源挂载的路径可以通过[覆盖`targetPath`来进行设置](./resources.md#控制资源挂载位置).
步骤可通过[变量替换](#variable-substitution) 使用`path`键获取资源挂载路径.

### 变量替换

在`Task`和`Condition`配置中可通过变量替换引用资源预定义的参数，以下`<name>`表示资源的`name`以及`<key>`是有关资源类型的`params`:

#### 在Task配置中:

对于`Task`配置中的输入资源: `shell $(resources.inputs.<name>.<key>)`

或者输出资源:

```shell
$(resources.outputs.<name>.<key>)
```

#### 在Condition配置中:

输入资源可通过以下访问:

```shell
$(resources.<name>.<key>)
```

#### 访问资源的本地路径

`path`键表示资源挂载的本地路径`shell $(resources.inputs.<name>.path)`

### 控制资源挂载位置

可选的字段`targetPath`可用于指定资源到特定的目录, 如果`targetPath`被设置，资源将会初始化到`/workspace/targetPath`下，如果`targetPath`没有指定，资源将初始化到`/workspace`目录中，以下示例展示了git输入仓储初始化到`$GOPATH`来运行测试:

```yaml
apiVersion: tekton.dev/v1beta1
kind: Task
metadata:
  name: task-with-input
  namespace: default
spec:
  resources:
    inputs:
      - name: workspace
        type: git
        targetPath: go/src/github.com/tektoncd/pipeline
  steps:
    - name: unit-tests
      image: golang
      command: ["go"]
      args:
        - "test"
        - "./..."
      workingDir: "/workspace/go/src/github.com/tektoncd/pipeline"
      env:
        - name: GOPATH
          value: /workspace/go
```

### 覆盖资源复制到的位置

当指定输入和输出`管道资源`时，你可以选择设置每一个资源的`paths`。`paths`将会被`TaskRun`作为新的的源路径，例如，从一个指定的列表路径中拷贝资源。`TaskRun`期望文件夹和内容在指定路径下已经存在。`paths`功能可以提供额外的文件或者在执行步骤之前修改已存在的资源.

输出资源包含名称来引用管道资源以及可选的`paths`。`paths`将会被用于`TaskRun`作为新的路径，例如拷贝资源到指定的路径。`TaskRun`将会负责创建必要的路径以及进行内容的传递. `paths`功能可以用来在`TaskRun`执行完成步骤后来获取其结果.

对于输入和输出资源的`paths`功能常用于在`PipelineRun`里面的任务之间传递相同版本的资源.

在以下示例中，`Task`和`TaskRun`定义了一个输入资源，输出资源以及步骤，它将会编译一个war包。在`TaskRun`（`volume-taskrun`）执行之后，`custom`卷将包含完整的`java-git-resource`（包括war包）拷贝到目标路径`/custom/workspace/`.

```yaml
apiVersion: tekton.dev/v1beta1
kind: Task
metadata:
  name: volume-task
  namespace: default
spec:
  resources:
    inputs:
      - name: workspace
        type: git
    outputs:
      - name: workspace
  steps:
    - name: build-war
      image: objectuser/run-java-jar #https://hub.docker.com/r/objectuser/run-java-jar/
      command: jar
      args: ["-cvf", "projectname.war", "*"]
      volumeMounts:
        - name: custom-volume
          mountPath: /custom
```

```yaml
apiVersion: tekton.dev/v1beta1
kind: TaskRun
metadata:
  name: volume-taskrun
  namespace: default
spec:
  taskRef:
    name: volume-task
  resources:
    inputs:
      - name: workspace
        resourceRef:
          name: java-git-resource
    outputs:
      - name: workspace
        paths:
          - /custom/workspace/
        resourceRef:
          name: java-git-resource
  podTemplate:
    volumes:
      - name: custom-volume
        emptyDir: {}
```

### 资源状态

当资源绑定到`TaskRun`内后，它们将会在`TaskRun` Status.ResourcesResult字段中包含额外的信息。这个信息有助于随后的`TaskRun`审查准确的资源. 当前镜像以及Git资源使用了此机制.

有关此输出的示例如下:

```yaml
resourcesResult:
- key: digest
  value: sha256:a08412a4164b85ae521b0c00cf328e3aab30ba94a526821367534b81e51cb1cb
  resourceRef:
    name: skaffold-image-leeroy-web
```

### 描述

通过`description`字段可以设置对资源的描述，此字段是可选的.

### 可选资源

默认情况下，资源是必须强制提供的，除非`optional`设置为`true`。在`Task`中定义为`optional`的资源，在`TaskRun`中可以不用指定.

```yaml
apiVersion: tekton.dev/v1beta1
kind: Task
metadata:
  name: task-check-optional-resources
spec:
  resources:
    inputs:
      - name: git-repo
        type: git
        optional: true
```
同样，在`Pipeline`中资源也可定义为`optional`，在`PipelineRun`中也可以不用指定.

```yaml
apiVersion: tekton.dev/v1beta1
kind: Pipeline
metadata:
  name: pipeline-build-image
spec:
  resources:
    - name: workspace
      type: git
      optional: true
  tasks:
    - name: check-workspace
...
```

你可以从以下示例中查看`Task`、`Condition`、`Pipeline`中使用可选资源的说明:

-   [Task](../examples/v1beta1/taskruns/optional-resources.yaml)
-   [Cluster Task](../examples/v1beta1/taskruns/optional-resources-with-clustertask.yaml)
-   [Condition](../examples/v1beta1/pipelineruns/conditional-pipelinerun-with-optional-resources.yaml)
-   [Pipeline](../examples/v1beta1/pipelineruns/demo-optional-resources.yaml)

## 资源类型

### Git 资源

`git`资源来表示一个[git](https://git-scm.com/)仓库, 它包含管道需要构建的代码. 为`Task`添加`git` 资源，将会拷贝此仓库并允许`Task`执行必要的动作.

通过`PipelineResource` CRD来创建git资源:

```yaml
apiVersion: tekton.dev/v1alpha1
kind: PipelineResource
metadata:
  name: wizzbang-git
  namespace: default
spec:
  type: git
  params:
    - name: url
      value: https://github.com/wizzbangcorp/wizzbang.git
    - name: revision
      value: master
```

可以使用以下参数:

1.  `url`: 表示git仓库的位置，你可以使用此参数来改变仓库，例如[使用fork](#使用fork)
1.  `revision`: 需要拷贝的Git [版本][git-rev] (分支, 标签, 提交 SHA 或者 ref). 你可以使用此来控制那个提交[或分支](#使用分支)被使用，[git checkout][git-checkout]用来切换版本，在多数情况下的结果是分离HEAD。通过使用refspec来指定revision的方式来迁出具体的分支而不用分离HEAD. _如果没有revision被指定，将使用默认的`master`._
1.  `refspec`: (可选)指定一个git[refspec][git-refspec]传递到git-fetch.
     如果此字段被指定，它必须指定所有refs，分支以及标签或者提交来签出指定的`revision`。附加的提取不用运行来获得revision字段. 如果没有refspec被指定，`revision`字段将会被直接提取. refspec可在以下几种情况下用于操作仓库:
     * 当服务器不支持通过提交SHA（如不允许`uploadpack.allowReachableSHA1InWant`）而你又需要提取和签出某个ref链对应的提交SHA时.
     * 当你需要提取多个其他的refs版本时（例如标签）
     * 当你想要签出指定的分支，revision和refspec可以配合使用来设置进入的分支目标以及切换到分支.

        例如:
         - 在提取ref（分离）后签出指定的版本提交SHA1<br>
           &nbsp;&nbsp;`revision`: cb17eba165fe7973ef9afec20e7c6971565bd72f <br>
           &nbsp;&nbsp;`refspec`: refs/smoke/myref <br>
         - 提取refs/heads/master所有的标签，然后切换到master分支
           (没有分离) <br>
           &nbsp;&nbsp;`revision`: master <br>
           &nbsp;&nbsp;`refspec`: "refs/tags/\*:refs/tags/\* +refs/heads/master:refs/heads/master"<br>
         - 提取develop分支然后切换到此分支(没有分离) <br>
           &nbsp;&nbsp;`revision`: develop <br>
           &nbsp;&nbsp;`refspec`: refs/heads/develop:refs/heads/develop <br>
         - 提取 refs/pull/1009/head 到master 分支，然后切换到此分支(没有分离) <br>
           &nbsp;&nbsp;`revision`: master <br>
           &nbsp;&nbsp;`refspec`: refs/pull/1009/head:refs/heads/master <br>

1.  `submodules`: 定义资源是否需要初始化或获取子模块，值可以为`true`或`false`，_如果未指定，默认值为true_
1.  `depth`: 提供[前复制][git-depth]功能，只提取最近的提交. 此设置同样应用于子模块. 如果设置为`'0'`, 所有的提交都将被提取. _如果未设置, 默认值为1._
1.  `sslVerify`: 定义git配置中的[http.sslVerify][git-http.sslVerify]可设置为`true`或者`false`. _如果未设置，默认值为`true`._

[git-rev]: https://git-scm.com/docs/gitrevisions#_specifying_revisions
[git-checkout]: https://git-scm.com/docs/git-checkout
[git-refspec]: https://git-scm.com/book/en/v2/Git-Internals-The-Refspec
[git-depth]: https://git-scm.com/docs/git-clone#Documentation/git-clone.txt---depthltdepthgt
[git-http.sslVerify]: https://git-scm.com/docs/git-config#Documentation/git-config.txt-httpsslVerify

如果作为输入，Git资源会在`taskRun`对象状态下的`resourceResults`节点记录拉取的确切提交信息:

```yaml
resourceResults:
- key: commit
  value: 6ed7aad5e8a36052ee5f6079fc91368e362121f7
  resourceRef:
    name: skaffold-git
```

#### 使用fork

`Url`参数可以用来指定任何的git仓库，例如使用GitHub fork master仓库:

```yaml
spec:
  type: git
  params:
    - name: url
      value: https://github.com/bobcatfish/wizzbang.git
```

#### 使用分支

`revision`可以是任何
[git commit-ish (revision)](https://git-scm.com/docs/gitrevisions#_specifying_revisions).
你可以通过此来创建一个`PipelineResource`资源，并指定一个分支，例如:

```yaml
spec:
  type: git
  params:
    - name: url
      value: https://github.com/wizzbangcorp/wizzbang.git
    - name: revision
      value: some_awesome_feature
```

要指定一个pull request，你可以使用
[pull requests's 分支](https://help.github.com/articles/checking-out-pull-requests-locally/):

```yaml
spec:
  type: git
  params:
    - name: url
      value: https://github.com/wizzbangcorp/wizzbang.git
    - name: revision
      value: refs/pull/52525/head
```

#### 使用HTTP/HTTPS代理

通过`httpProxy` 和 `httpsProxy`参数可以指定代理访问非SSL或SSL请求, 例如使用企业代理服务器来处理SSL请求：

```yaml
spec:
  type: git
  params:
    - name: url
      value: https://github.com/bobcatfish/wizzbang.git
    - name: httpsProxy
      value: "my-enterprise.proxy.com"
```

#### 不使用代理

通过`noProxy`参数可以用来指定不走代理的请求，例如，访问`no.proxy.com`的时候不走代理:

```yaml
spec:
  type: git
  params:
    - name: url
      value: https://github.com/bobcatfish/wizzbang.git
    - name: noProxy
      value: "no.proxy.com"
```

注意：`httpProxy`, `httpsProxy`, 及 `noProxy`都是可选的，但是如果三个都设置了，也不会做有效性验证.

### Pull Request Resource

The `pullRequest` resource represents a pull request event from a source control
system.

Adding the Pull Request resource as an input to a `Task` will populate the
workspace with a set of files containing generic pull request related metadata
such as base/head commit, comments, and labels.

The payloads will also contain links to raw service-specific payloads where
appropriate.

Adding the Pull Request resource as an output of a `Task` will update the source
control system with any changes made to the pull request resource during the
pipeline.

Example file structure:

```shell
/workspace/
/workspace/<resource>/
/workspace/<resource>/labels/
/workspace/<resource>/labels/<label>
/workspace/<resource>/status/
/workspace/<resource>/status/<status>
/workspace/<resource>/comments/
/workspace/<resource>/comments/<comment>
/workspace/<resource>/head.json
/workspace/<resource>/base.json
/workspace/<resource>/pr.json
```

More details:

Labels are empty files, named after the desired label string.

Statuses describe pull request statuses. It is represented as a set of json
files.

References (head and base) describe Git references. They are represented as a
set of json files.

Comments describe a pull request comment. They are represented as a set of json
files.

Other pull request information can be found in `pr.json`. This is a read-only
resource. Users should use other subresources (labels, comments, etc) to
interact with the PR.

For an example of the output this resource provides, see
[`example`](../cmd/pullrequest-init/example).

To create a pull request resource using the `PipelineResource` CRD:

```yaml
apiVersion: tekton.dev/v1alpha1
kind: PipelineResource
metadata:
  name: wizzbang-pr
  namespace: default
spec:
  type: pullRequest
  params:
    - name: url
      value: https://github.com/wizzbangcorp/wizzbang/pulls/1
  secrets:
    - fieldName: authToken
      secretName: github-secrets
      secretKey: token
---
apiVersion: v1
kind: Secret
metadata:
  name: github-secrets
type: Opaque
data:
  token: github_personal_access_token_secret # in base64 encoded form
```

Params that can be added are the following:

1.  `url`: represents the location of the pull request to fetch.
1.  `provider`: represents the SCM provider to use. This will be "guessed" based
    on the url if not set. Valid values are `github` or `gitlab` today.
1.  `insecure-skip-tls-verify`: represents whether to skip verification of certificates
    from the git server. Valid values are `"true"` or `"false"`, the default being
    `"false"`.

#### Statuses

The following status codes are available to use for the Pull Request resource:
https://godoc.org/github.com/jenkins-x/go-scm/scm#State

#### Pull Request

The `pullRequest` resource will look for GitHub or GitLab OAuth authentication
tokens in spec secrets with a field name called `authToken`.

URLs should be of the form: https://github.com/tektoncd/pipeline/pull/1

#### Self hosted / Enterprise instances

The PullRequest resource works with self hosted or enterprise GitHub/GitLab
instances. Simply provide the pull request URL and set the `provider` parameter.
If you need to skip certificate validation set the `insecure-skip-tls-verify`
parameter to `"true"`.

```yaml
apiVersion: tekton.dev/v1alpha1
kind: PipelineResource
metadata:
  name: wizzbang-pr
  namespace: default
spec:
  type: pullRequest
  params:
    - name: url
      value: https://github.example.com/wizzbangcorp/wizzbang/pulls/1
    - name: provider
      value: github
```

### Image Resource

An `image` resource represents an image that lives in a remote repository. It is
usually used as [a `Task` `output`](tasks.md#outputs) for `Tasks` that build
images. This allows the same `Tasks` to be used to generically push to any
registry.

Params that can be added are the following:

1.  `url`: The complete path to the image, including the registry and the image
    tag
1.  `digest`: The
    [image digest](https://success.docker.com/article/images-tagging-vs-digests)
    which uniquely identifies a particular build of an image with a particular
    tag. _While this can be provided as a parameter, there is not yet a way to
    update this value after an image is built, but this is planned in
    [#216](https://github.com/tektoncd/pipeline/issues/216)._

For example:

```yaml
apiVersion: tekton.dev/v1alpha1
kind: PipelineResource
metadata:
  name: kritis-resources-image
  namespace: default
spec:
  type: image
  params:
    - name: url
      value: gcr.io/staging-images/kritis
```

#### Surfacing the image digest built in a task

To surface the image digest in the output of the `taskRun` the builder tool
should produce this information in a
[OCI Image Layout](https://github.com/opencontainers/image-spec/blob/master/image-layout.md)
`index.json` file. This file should be placed on a location as specified in the
task definition under the default resource directory, or the specified
`targetPath`. If there is only one image in the `index.json` file, the digest of
that image is exported; otherwise, the digest of the whole image index would be
exported. For example this build-push task defines the `outputImageDir` for the
`builtImage` resource in `/workspace/buildImage`

```yaml
apiVersion: tekton.dev/v1beta1
kind: Task
metadata:
  name: build-push
spec:
  resources:
    inputs:
      - name: workspace
        type: git
    outputs:
      - name: builtImage
        type: image
        targetPath: /workspace/builtImage
  steps: ...
```

If no value is specified for `targetPath`, it will default to
`/workspace/output/{resource-name}`.

_Please check the builder tool used on how to pass this path to create the
output file._

The `taskRun` will include the image digest in the `resourcesResult` field that
is part of the `taskRun.Status`

for example:

```yaml
status:
    ...
    resourcesResult:
    - key: "digest"
      value: "sha256:eed29cd0b6feeb1a92bc3c4f977fd203c63b376a638731c88cacefe3adb1c660"
      resourceRef:
        name: skaffold-image-leeroy-web
    ...
```

If the `index.json` file is not produced, the image digest will not be included
in the `taskRun` output.

### Cluster Resource

A `cluster` resource represents a Kubernetes cluster other than the current
cluster Tekton Pipelines is running on. A common use case for this resource is
to deploy your application/function on different clusters.

The resource will use the provided parameters to create a
[kubeconfig](https://kubernetes.io/docs/tasks/access-application-cluster/configure-access-multiple-clusters/)
file that can be used by other steps in the pipeline `Task` to access the target
cluster. The kubeconfig will be placed in
`/workspace/<your-cluster-name>/kubeconfig` on your `Task` container

The Cluster resource has the following parameters:

-   `url` (required): Host url of the master node
-   `username` (required): the user with access to the cluster
-   `password`: to be used for clusters with basic auth
-   `namespace`: The namespace to target in the cluster
-   `token`: to be used for authentication, if present will be used ahead of the
    password
-   `insecure`: to indicate server should be accessed without verifying the TLS
    certificate.
-   `cadata` (required): holds PEM-encoded bytes (typically read from a root
    certificates bundle).
-   `clientKeyData`: contains PEM-encoded data from a client key file 
        for TLS 
-   `clientCertificateData`: contains PEM-encoded data from a client cert file for TLS


Note: Since only one authentication technique is allowed per user, either a
`token` or a `password` should be provided, if both are provided, the `password`
will be ignored.

`clientKeyData` and `clientCertificateData` are only required if `token` or 
`password` is not provided for authentication to cluster.

The following example shows the syntax and structure of a `cluster` resource:

```yaml
apiVersion: tekton.dev/v1alpha1
kind: PipelineResource
metadata:
  name: test-cluster
spec:
  type: cluster
  params:
    - name: url
      value: https://10.10.10.10 # url to the cluster master node
    - name: cadata
      value: LS0tLS1CRUdJTiBDRVJ.....
    - name: token
      value: ZXlKaGJHY2lPaU....
```

For added security, you can add the sensitive information in a Kubernetes
[Secret](https://kubernetes.io/docs/concepts/configuration/secret/) and populate
the kubeconfig from them.

For example, create a secret like the following example:

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: target-cluster-secrets
data:
  cadatakey: LS0tLS1CRUdJTiBDRVJUSUZ......tLQo=
  tokenkey: ZXlKaGJHY2lPaUpTVXpJMU5pSXNJbX....M2ZiCg==
```

and then apply secrets to the cluster resource

```yaml
apiVersion: tekton.dev/v1alpha1
kind: PipelineResource
metadata:
  name: test-cluster
spec:
  type: cluster
  params:
    - name: url
      value: https://10.10.10.10
    - name: username
      value: admin
  secrets:
    - fieldName: token
      secretKey: tokenKey
      secretName: target-cluster-secrets
    - fieldName: cadata
      secretKey: cadataKey
      secretName: target-cluster-secrets
```

Example usage of the `cluster` resource in a `Task`, using
[variable substitution](tasks.md#variable-substitution):

```yaml
apiVersion: tekton.dev/v1beta1
kind: Task
metadata:
  name: deploy-image
  namespace: default
spec:
  resources:
    inputs:
      - name: workspace
        type: git
      - name: dockerimage
        type: image
      - name: testcluster
        type: cluster
  steps:
    - name: deploy
      image: image-with-kubectl
      command: ["bash"]
      args:
        - "-c"
        - kubectl --kubeconfig
          /workspace/$(resources.inputs.testcluster.name)/kubeconfig --context
          $(resources.inputs.testcluster.name) apply -f /workspace/service.yaml'
```

To use the `cluster` resource with Google Kubernetes Engine, you should use the
`cadata` authentication mechanism.

To determine the caData, you can use the following `gcloud` commands:

```shell
gcloud container clusters describe <cluster-name> --format='value(masterAuth.clusterCaCertificate)'
```

To create a secret with this information, you can use:

```shell
CADATA=$(gcloud container clusters describe <cluster-name> --format='value(masterAuth.clusterCaCertificate)')
kubectl create secret generic cluster-ca-data --from-literal=cadata=$CADATA
```

To retrieve the URL, you can use this gcloud command:

```shell
gcloud container clusters describe <cluster-name> --format='value(endpoint)'
```

Then to use these in a resource, reference the cadata from the secret you
created above, and use the IP address from the gcloud command as your url
(prefixed with https://):

```yaml
spec:
  type: cluster
  params:
    - name: url
      value: https://<ip address determined above>
  secrets:
    - fieldName: cadata
      secretName: cluster-ca-data
      secretKey: cadata
```

### Storage Resource

The `storage` resource represents blob storage, that contains either an object
or directory. Adding the storage resource as an input to a `Task` will download
the blob and allow the `Task` to perform the required actions on the contents of
the blob.

Only blob storage type
[Google Cloud Storage](https://cloud.google.com/storage/)(gcs) is supported as
of now via [GCS storage resource](#gcs-storage-resource) and
[BuildGCS storage resource](#buildgcs-storage-resource).

#### GCS Storage Resource

The `gcs` storage resource points to
[Google Cloud Storage](https://cloud.google.com/storage/) blob.

To create a GCS type of storage resource using the `PipelineResource` CRD:

```yaml
apiVersion: tekton.dev/v1alpha1
kind: PipelineResource
metadata:
  name: wizzbang-storage
  namespace: default
spec:
  type: storage
  params:
    - name: type
      value: gcs
    - name: location
      value: gs://some-bucket
    - name: dir
      value: "y" # This can have any value to be considered "true"
```

Params that can be added are the following:

1.  `location`: represents the location of the blob storage.
1.  `type`: represents the type of blob storage. For GCS storage resource this
    value should be set to `gcs`.
1.  `dir`: represents whether the blob storage is a directory or not. By default
    a storage artifact is not considered a directory.

    -   If the artifact is a directory then `-r`(recursive) flag is used, to
        copy all files under the source directory to a GCS bucket. Eg: `gsutil
        cp -r source_dir/* gs://some-bucket`
    -   If an artifact is a single file like a zip or tar, then the copy will be
        only 1 level deep(not recursive). It will not trigger a copy of sub
        directories in the source directory. Eg: `gsutil cp source.tar
        gs://some-bucket.tar`.

Private buckets can also be configured as storage resources. To access GCS
private buckets, service accounts with correct permissions are required. The
`secrets` field on the storage resource is used for configuring this
information. Below is an example on how to create a storage resource with a
service account.

1.  Refer to the
    [official documentation](https://cloud.google.com/compute/docs/access/service-accounts)
    on how to create service accounts and configuring
    [IAM permissions](https://cloud.google.com/storage/docs/access-control/iam-permissions)
    to access buckets.

1.  Create a Kubernetes secret from a downloaded service account json key

    ```bash
    kubectl create secret generic bucket-sa --from-file=./service_account.json
    ```

1.  To access the GCS private bucket environment variable
    [`GOOGLE_APPLICATION_CREDENTIALS`](https://cloud.google.com/docs/authentication/production)
    should be set, so apply the above created secret to the GCS storage resource
    under the `fieldName` key.

    ```yaml
    apiVersion: tekton.dev/v1alpha1
    kind: PipelineResource
    metadata:
      name: wizzbang-storage
      namespace: default
    spec:
      type: storage
      params:
        - name: type
          value: gcs
        - name: location
          value: gs://some-private-bucket
        - name: dir
          value: "y"
      secrets:
        - fieldName: GOOGLE_APPLICATION_CREDENTIALS
          secretName: bucket-sa
          secretKey: service_account.json
    ```

--------------------------------------------------------------------------------

#### BuildGCS Storage Resource

The `build-gcs` storage resource points to a
[Google Cloud Storage](https://cloud.google.com/storage/) blob like
[GCS Storage Resource](#gcs-storage-resource), either in the form of a .zip
archive, or based on the contents of a source manifest file.

In addition to fetching an .zip archive, BuildGCS also unzips it.

A
[Source Manifest File](https://github.com/GoogleCloudPlatform/cloud-builders/tree/master/gcs-fetcher#source-manifests)
is a JSON object, which is listing other objects in a Cloud Storage that should
be fetched. The format of the manifest is a mapping of the destination file path
to the location in a Cloud Storage, where the file's contents can be found. The
`build-gcs` resource can also do incremental uploads of sources via the Source
Manifest File.

To create a `build-gcs` type of storage resource using the `PipelineResource`
CRD:

```yaml
apiVersion: tekton.dev/v1alpha1
kind: PipelineResource
metadata:
  name: build-gcs-storage
  namespace: default
spec:
  type: storage
  params:
    - name: type
      value: build-gcs
    - name: location
      value: gs://build-crd-tests/rules_docker-master.zip
    - name: artifactType
      value: Archive
```

Params that can be added are the following:

1.  `location`: represents the location of the blob storage.
1.  `type`: represents the type of blob storage. For BuildGCS, this value should
    be set to `build-gcs`
1.  `artifactType`: represent the type of `gcs` resource. Right now, we support
    following types:

*   `ZipArchive`:
    *   ZipArchive indicates that the resource fetched is an archive file in the
        zip format.
    *   It unzips the archive and places all the files in the directory, which
        is set at runtime.
    *   `Archive` is also supported and is equivalent to `ZipArchive`, but is
        deprecated.
*   `TarGzArchive`:
    *   TarGzArchive indicates that the resource fetched is a gzipped archive
        file in the tar format.
    *   It unzips the archive and places all the files in the directory, which
        is set at runtime.
*   [`Manifest`](https://github.com/GoogleCloudPlatform/cloud-builders/tree/master/gcs-fetcher#source-manifests):
    *   Manifest indicates that the resource should be fetched using a source
        manifest file.

Private buckets other than the ones accessible by a
[TaskRun Service Account](./taskruns.md#service-account) can not be configured
as `storage` resources for the `build-gcs` storage resource right now. This is
because the container image
[gcr.io/cloud-builders//gcs-fetcher](https://github.com/GoogleCloudPlatform/cloud-builders/tree/master/gcs-fetcher)
does not support configuring secrets.

### Cloud Event Resource

The `cloudevent` resource represents a
[cloud event](https://github.com/cloudevents/spec) that is sent to a target
`URI` upon completion of a `TaskRun`. The `cloudevent` resource sends Tekton
specific events; the body of the event includes the entire `TaskRun` spec plus
status; the types of events defined for now are:

-   dev.tekton.event.task.unknown
-   dev.tekton.event.task.successful
-   dev.tekton.event.task.failed

`cloudevent` resources are useful to notify a third party upon the completion
and status of a `TaskRun`. In combinations with the
[Tekton triggers](https://github.com/tektoncd/triggers) project they can be used
to link `Task/PipelineRuns` asynchronously.

To create a CloudEvent resource using the `PipelineResource` CRD:

```yaml
apiVersion: tekton.dev/v1alpha1
kind: PipelineResource
metadata:
  name: event-to-sink
spec:
  type: cloudEvent
  params:
  - name: targetURI
    value: http://sink:8080
```

The content of an event is for example:

```yaml
Context Attributes,
  SpecVersion: 0.2
  Type: dev.tekton.event.task.successful
  Source: /apis/tekton.dev/v1beta1/namespaces/default/taskruns/pipeline-run-api-16aa55-source-to-image-task-rpndl
  ID: pipeline-run-api-16aa55-source-to-image-task-rpndl
  Time: 2019-07-04T11:03:53.058694712Z
  ContentType: application/json
Transport Context,
  URI: /
  Host: my-sink.default.my-cluster.containers.appdomain.cloud
  Method: POST
Data,
  {
    "taskRun": {
      "metadata": {...}
      "spec": {
        "inputs": {...}
        "outputs": {...}
        "serviceAccount": "default",
        "taskRef": {
          "name": "source-to-image",
          "kind": "Task"
        },
        "timeout": "1h0m0s"
      },
      "status": {...}
    }
  }
```

## Why Aren't PipelineResources in Beta?

The short answer is that they're not ready to be given a Beta level of support by Tekton's developers. The long answer is, well, longer:

- Their behaviour can be opaque.

    They're implemented as a mixture of injected Task Steps, volume configuration and type-specific code in Tekton
    Pipeline's controller. This means errors from `PipelineResources` can manifest in quite a few different ways
    and it's not always obvious whether an error directly relates to `PipelineResource` behaviour. This problem
    is compounded by the fact that, while our docs explain each Resource type's "happy path", there never seems to
    be enough info available to explain error cases sufficiently.

- When they fail they're difficult to debug.

    Several PipelineResources inject their own Steps before a `Task's` Steps. It's extremely difficult to manually
    insert Steps before them to inspect the state of a container before they run.

- There aren't enough of them.

    The six types of existing PipelineResources only cover a tiny subset of the possible systems and side-effects we
    want to support with Tekton Pipelines.

- Adding extensibility to them makes them really similar to `Tasks`:
    - User-definable `Steps`? This is what `Tasks` provide.
    - User-definable params? Tasks already have these.
    - User-definable "resource results"? `Tasks` have `Task` Results.
    - Sharing data between Tasks using PVCs? `workspaces` provide this for `Tasks`.
- They make `Tasks` less reusable.
    - A `Task` has to choose the `type` of `PipelineResource` it will accept.
    - If a `Task` accepts a `git` `PipelineResource` then it's not able to accept a `gcs` `PipelineResource` from a
      `TaskRun` or `PipelineRun` even though both the `git` and `gcs` `PipelineResources` fetch files. They should
      technically be interchangeable: all they do is write files from somewhere remote onto disk. Yet with the existing
      `PipelineResources` implementation they aren't interchangeable.

They also present challenges from a documentation perspective:

- Their purpose is ambiguous and it's difficult to articulate what the CRD is precisely for.
- Four of the types interact with external systems (git, pull-request, gcs, gcs-build).
- Five of them write files to a Task's disk (git, pull-request, gcs, gcs-build, cluster).
- One tells the Pipelines controller to emit CloudEvents to a specific endpoint (cloudEvent).
- One writes config to disk for a `Task` to use (cluster).
- One writes a digest in one `Task` and then reads it back in another `Task` (image).
- Perhaps the one thing you can say about the `PipelineResource` CRD is that it can create
  side-effects for your `Tasks`.

### Next steps

So what are PipelineResources still good for?  We think we've identified some of the most important things:

1. You can augment `Task`-only workflows with `PipelineResources` that, without them, can only be done with `Pipelines`.
    - For example, let's say you want to checkout a git repo for your Task to test. You have two options. First, you could use a `git` PipelineResource and add it directly to your test `Task`. Second, you could write a `Pipeline` that has a `git-clone` `Task` which checks out the code onto a PersistentVolumeClaim `workspace` and then passes that PVC `workspace` to your test `Task`. For a lot of users the second workflow is totally acceptable but for others it isn't. Some of the most noteable reasons we've heard are:
      - Some users simply cannot allocate storage on their platform, meaning `PersistentVolumeClaims` are out of the question.
      - Expanding a single `Task` workflow into a `Pipeline` is labor-intensive and feels unnecessary.
2. Despite being difficult to explain the whole CRD clearly each individual `type` is relatively easy to explain.
    - For example, users can build a pretty good "hunch" for what a `git` `PipelineResource` is without really reading any docs.
3. Configuring CloudEvents to be emitted by the Tekton Pipelines controller.
    - Work is ongoing to get notifications support into the Pipelines controller which should hopefully be able to replace the `cloudEvents` `PipelineResource`.

For each of these there is some amount of ongoing work or discussion. It may be that
`PipelineResources` can be redesigned to fix all of their problems or it could be
that the best features of `PipelineResources` can be extracted for use everywhere in
Tekton Pipelines. So given this state of affairs `PipelineResources` are being kept
out of beta until those questions are resolved.

For Beta-supported alternatives to PipelineResources see
[the v1alpha1 to v1beta1 migration guide](./migrating-v1alpha1-to-v1beta1.md#pipelineresources-and-catalog-tasks)
which lists each PipelineResource type and a suggested option for replacing it.

Except as otherwise noted, the content of this page is licensed under the
[Creative Commons Attribution 4.0 License](https://creativecommons.org/licenses/by/4.0/),
and code samples are licensed under the
[Apache 2.0 License](https://www.apache.org/licenses/LICENSE-2.0).
