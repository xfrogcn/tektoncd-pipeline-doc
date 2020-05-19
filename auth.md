<!--
---
linkTitle: "Authentication"
weight: 7
---
-->
# 认证

本文档定义在执行`TaskRun`或`PipelineRun`期间是如何提供认证的(参考本文档的`Runs`).

构建系统支持两种类型的认证，通过使用Kubernetes的`Secret`类型:

- `kubernetes.io/basic-auth`
- `kubernetes.io/ssh-auth`

这些类型的密文可以通过附加到`ServiceAccount`的方式传递到`Run`.

## 导出凭证

按照原有格式，这些密文是无法适用于Git及Docker场景的，对于Git，它们需要转换为`.gitconfig`配置文件，对于Docker，它们需要转换为`~/.docker/config.json`文件，而这些支持具有多个凭证并应用了多个场景中，这些凭证通常需要混合在一个规范的密钥环中.

为了解决此问题，在`PipelineResources`被使用之前，所有`pods`执行一个凭证初始化进程，通过此进程访问它相关的密文并将其转换到`$HOME`下不同的文件中.

## SSH 认证 (Git)

1. 定义一个 `Secret`，包含你的SSH私有秘钥(`secret.yaml`):

   ```yaml
   apiVersion: v1
   kind: Secret
   metadata:
     name: ssh-key
     annotations:
       tekton.dev/git-0: github.com # 详见下面描述
   type: kubernetes.io/ssh-auth
   data:
     ssh-privatekey: <base64 encoded>
     # 这是非标准的，但是可以增强安全性.
     known_hosts: <base64 encoded>
   ```

   `tekton.dev/git-0` 在上述示例中指定凭证所属于哪个网站地址，参考
   [凭证选择指南](#凭证选择指南) 了解详细信息.

1. 通过拷贝（示例）`cat ~/.ssh/id_rsa | base64`内容来获取`ssh-privatekey`的值.

1. 通过拷贝`cat ~/.ssh/known_hosts | base64`的内容作为`known_hosts`字段的值.

1. 接下来, 通过`ServiceAccount`来使用此`Secret` (
   `serviceaccount.yaml`):

    ```yaml
    apiVersion: v1
    kind: ServiceAccount
    metadata:
      name: build-bot
    secrets:
      - name: ssh-key
    ```

1. 然后在`TaskRun`中使用`ServiceAccount`(`run.yaml`):

    ```yaml
    apiVersion: tekton.dev/v1beta1
    kind: TaskRun
    metadata:
      name: build-push-task-run-2
    spec:
      serviceAccountName: build-bot
      taskRef:
        name: build-push
    ```

1. 或者在`PipelineRun`中使用`ServiceAccount`(`run.yaml`):

   ```yaml
   apiVersion: tekton.dev/v1beta1
   kind: PipelineRun
   metadata:
     name: demo-pipeline
     namespace: default
   spec:
     serviceAccountName: build-bot
     pipelineRef:
       name: demo-pipeline
   ```

1. 执行`Run`:

   ```shell
   kubectl apply --filename secret.yaml serviceaccount.yaml run.yaml
   ```

当`Run`执行时，在步骤执行之前，`~/.ssh/config`文件将会被自动生成，包含`Secret`的配置内容. 这个键将会被用于处理`PipelineResources`资源.

## 基本认证 (Git)

1. 定义一个`Secret`，包含`Run`所需要的用于认证Git仓库的用户名和密码(`secret.yaml`):

   ```yaml
   apiVersion: v1
   kind: Secret
   metadata:
     name: basic-user-pass
     annotations:
       tekton.dev/git-0: https://github.com # 详见下面描述
   type: kubernetes.io/basic-auth
   stringData:
     username: <username>
     password: <password>
   ```

   `tekton.dev/git-0` 在上述示例中指定凭证所属于哪个网站地址，参考
   [凭证选择指南](#凭证选择指南) 了解详细信息.

1. 接下来, 通过`ServiceAccount`来使用`Secret` (
   `serviceaccount.yaml`):

   ```yaml
   apiVersion: v1
   kind: ServiceAccount
   metadata:
     name: build-bot
   secrets:
     - name: basic-user-pass
   ```

1. 然后在`TaskRun`中使用`ServiceAccount`(`run.yaml`):

   ```yaml
   apiVersion: tekton.dev/v1beta1
   kind: TaskRun
   metadata:
     name: build-push-task-run-2
   spec:
     serviceAccountName: build-bot
     taskRef:
       name: build-push
   ```

1. 或者在`PipelineRun`中使用`ServiceAccount`(`run.yaml`):

   ```yaml
   apiVersion: tekton.dev/v1beta1
   kind: PipelineRun
   metadata:
     name: demo-pipeline
     namespace: default
   spec:
     serviceAccountName: build-bot
     pipelineRef:
       name: demo-pipeline
   ```

1. 执行`Run`:

   ```shell
   kubectl apply --filename secret.yaml serviceaccount.yaml run.yaml
   ```

当以上`Run`执行时，在执行具体步骤之前，`~/.gitconfig`文件将会根据`Secret`配置自动生成，然后这个凭证将在使用`PipelineResources`资源时被使用.

## 基本认证 (Docker)

1. 定义一个`Secret`,包含构建时Docker仓库认证需要的用户名和密码(`secret.yaml`):

   ```yaml
   apiVersion: v1
   kind: Secret
   metadata:
     name: basic-user-pass
     annotations:
       tekton.dev/docker-0: https://gcr.io # 详见下面描述
   type: kubernetes.io/basic-auth
   stringData:
     username: <username>
     password: <password>
   ```

   `tekton.dev/docker-0` 在上述示例中指定凭证所属于哪个网站地址，参考
   [凭证选择指南](#凭证选择指南) 了解详细信息.

1. 接下来, 定义`ServiceAccount`来使用此`Secret` (
   `serviceaccount.yaml`):

   ```yaml
   apiVersion: v1
   kind: ServiceAccount
   metadata:
     name: build-bot
   secrets:
     - name: basic-user-pass
   ```

1. 然后在`TaskRun`中使用此`ServiceAccount` (`run.yaml`):

```yaml
apiVersion: tekton.dev/v1beta1
kind: TaskRun
metadata:
  name: build-push-task-run-2
spec:
  serviceAccountName: build-bot
  taskRef:
    name: build-push
```

1. 或者在`PipelineRun`中使用`ServiceAccount`(`run.yaml`):

   ```yaml
   apiVersion: tekton.dev/v1beta1
   kind: PipelineRun
   metadata:
     name: demo-pipeline
     namespace: default
   spec:
     serviceAccountName: build-bot
     pipelineRef:
       name: demo-pipeline
   ```

1. 执行`Run`:

   ```shell
   kubectl apply --filename secret.yaml serviceaccount.yaml run.yaml
   ```

当执行`Run`时，在执行具体步骤之前，将先通过`Secret`配置自动产生`~/.docker/config.json` Docker配置文件，然后在使用具体`PipelineResources`资源时自动使用这些凭证.

## Kubernetes Docker仓库密文

Kubernetes为Docker仓库提供两种类型的密文: 旧的`kubernetes.io/dockercfg`格式及新的`kubernetes.io/dockerconfigjson`格式，Tekton同时支持这些密文.

1. 从Docker客户端配置文件定义一个, 可参考文档
   [从私有仓库中拉取镜像](https://kubernetes.io/docs/tasks/configure-pod-container/pull-image-private-registry/)

   ```bash
   kubectl create secret generic regcred \
    --from-file=.dockerconfigjson=<path/to/.docker/config.json> \
    --type=kubernetes.io/dockerconfigjson
   ```

1. 通过`ServiceAccount`使用此`Secret`:

   ```yaml
   apiVersion: v1
   kind: ServiceAccount
   metadata:
     name: build-bot
   secrets:
     - name: regcred
   ```

1. 在`TaskRun`中使用`ServiceAccount`:

   ```yaml
   apiVersion: tetkon.dev/v1beta1
   kind: TaskRun
   metadata:
     name: build-with-basic-auth
   spec:
     serviceAccountName: build-bot
     steps:
     ...
   ```

1. 执行构建:

   ```shell
   kubectl apply --filename secret.yaml --filename serviceaccount.yaml --filename taskrun.yaml
   ```

当此TaskRun执行时，在具体步骤执行之前，将通过`Secret`配置自动生成`~/.docker/config.json`文件，此文件将用于Docker仓库的认证.

如果`kubernetes.io/*`及tekton基础认证密文被同时提供，tekton将合并它们，tekton风格凭证将优先于`kubernetes.io/dockerconfigjson`(或`kubernetes.io/dockercfg`).

## 凭证选择指南

一个`Run`可能需要多种不同的认证类型，例如，一个`Run`可能需要访问多个私有Git仓库，多个私有Docker仓库，你可以使用标注来指定那个密文应该使用到那个资源，例如:

```yaml
apiVersion: v1
kind: Secret
metadata:
  annotations:
    tekton.dev/git-0: https://github.com
    tekton.dev/git-1: https://gitlab.com
    tekton.dev/docker-0: https://gcr.io
type: kubernetes.io/basic-auth
stringData:
  username: <cleartext non-encoded>
  password: <cleartext non-encoded>
```

以上定义一个`基础认证`(用户名和密码)密文将会用于访问github.com及gitlab.com上的git仓库，以及gcr.io Docker镜像仓库.

同样地, 对于SSH:

```yaml
apiVersion: v1
kind: Secret
metadata:
  annotations:
    tekton.dev/git-0: github.com
type: kubernetes.io/ssh-auth
data:
  ssh-privatekey: <base64 encoded>
  # 非标准，但是设置后可增强安全性.
  # 省略将会导致使用ssh-keyscan(参考下面).
  known_hosts: <base64 encoded>
```

此SSH密文定义表示只应用于github.com的Git仓库.

凭证注释的键必须开始于 `tekton.dev/docker-` 或者
`tekton.dev/git-`, 值为使用凭证的URL域名.

## 实现细节

### Docker `basic-auth`

设定URL，用户名和密码，值为: `https://url{n}.com`,
`user{n}`, and `pass{n}`, 产生的Docker配置文件如下:

```json
=== ~/.docker/config.json ===
{
  "auths": {
    "https://url1.com": {
      "auth": "$(echo -n user1:pass1 | base64)",
      "email": "not@val.id",
    },
    "https://url2.com": {
      "auth": "$(echo -n user2:pass2 | base64)",
      "email": "not@val.id",
    },
    ...
  }
}
```

Docker不支持`kubernetes.io/ssh-auth`密文，所以此类型的注释将会被忽略.

### Git `basic-auth`

设定URL，用户名，密码，值为: `https://url{n}.com`,
`user{n}`, and `pass{n}`, 产生如下Git配置文件:

```
=== ~/.gitconfig ===
[credential]
    helper = store
[credential "https://url1.com"]
    username = "user1"
[credential "https://url2.com"]
    username = "user2"
...
=== ~/.git-credentials ===
https://user1:pass1@url1.com
https://user2:pass2@url2.com
...
```

### Git `ssh-auth`

设定宿主域名, 私有秘钥，以及`known_hosts`，值为: `url{n}.com`,
`key{n}`, and `known_hosts{n}`, 产生的Git配置文件如下:

```
=== ~/.ssh/id_key1 ===
{contents of key1}
=== ~/.ssh/id_key2 ===
{contents of key2}
...
=== ~/.ssh/config ===
Host url1.com
    HostName url1.com
    IdentityFile ~/.ssh/id_key1
Host url2.com
    HostName url2.com
    IdentityFile ~/.ssh/id_key2
...
=== ~/.ssh/known_hosts ===
{contents of known_hosts1}
{contents of known_hosts2}
...
```

注意: 由于 `known_hosts` 是一个非标准的`kubernetes.io/ssh-auth`扩展，当没有指定此字段时，将会通过`ssh-keygen url{n}.com`来替代.

### 最少特权

这里所描述的密文将会被存储到`$HOME`路径下（惯例指向`/tekton/home`目录），在所有`Source`和`Steps`中有效.

对于敏感凭证，它可能不能透传给某些步骤，那么请避免使用此文描述的机制，而应该选择从`密文`挂载`卷`，并手动挂载到指定的步骤.

---

除非另有说明，本页内容采用[Creative Commons Attribution 4.0 License](https://creativecommons.org/licenses/by/4.0/)授权协议，示例代码采用[Apache 2.0 License](https://www.apache.org/licenses/LICENSE-2.0)授权协议
