<!--
---
linkTitle: "Container Contracts"
weight: 8
---
-->
# 容器规范

每一个用于[`Task`](tasks.md) 步骤中的容器都需要符合特定的标准和规范.

## 入口

当`Task`中的容器运行时，容器的`entrypoint`将会被自定义的运行库所替代，以便于`Task` pod中的容器按照指定的顺序运行. 所以，这建议总是设置一个指定的命令..

当`command`没有被显式设置时，控制器将会尝试从远端仓库中查询入口, 如果镜像是私有仓库，服务账户应该包含一个
[ImagePullSecret](https://kubernetes.io/docs/tasks/configure-pod-container/configure-service-account/#add-imagepullsecrets-to-a-service-account).
Tekton控制器将会使用服务账户中的`ImagePullSecret`，如果服务账户为空，将会假设为`default`账户。 接下来将会将docker配置添加到`.docker/config.json`在`$HOME/.docker/config.json`路径下，如果所有这些凭证都无效，控制器将会尝试匿名方式来访问镜像仓库.

例如，以下任务使用`gcr.io/cloud-builders/gcloud` 和 `gcr.io/cloud-builders/docker`镜像，入口将会从仓库解析，结果任务将执行`gcloud`和`docker`命令.

```yaml
spec:
  steps:
    - image: gcr.io/cloud-builders/gcloud
      command: [gcloud]
    - image: gcr.io/cloud-builders/docker
      command: [docker]
```

但是，如果步骤指定了`命令（command）`，那将会使用指定的命令.

```yaml
spec:
  steps:
    - image: gcr.io/cloud-builders/gcloud
      command:
        - bash
        - -c
        - echo "Hello!"
```

你也可以通过`args`指定镜像`命令`的参数:

```yaml
steps:
  - image: ubuntu
    command: ["/bin/bash"]
    args: ["-c", "echo hello $FOO"]
    env:
      - name: "FOO"
        value: "world"
```

---

除非另有说明，本页内容采用[Creative Commons Attribution 4.0 License](https://creativecommons.org/licenses/by/4.0/)授权协议，示例代码采用[Apache 2.0 License](https://www.apache.org/licenses/LICENSE-2.0)授权协议
