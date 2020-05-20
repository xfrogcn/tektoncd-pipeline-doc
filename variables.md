<!--
---
linkTitle: "Variable Substitutions"
weight: 15
---
-->
# 变量替换

本文档主要介绍`Tasks`及`Pipelines`中详细的变量替换功能.

## Pipeline中有效的变量

| 变量  | 描述 |
| -------- | ----------- |
| `params.<param name>` | 运行时参数具体的值. |
| `tasks.<task name>.results.<result name>` | 任务结果值 (**注意**: 与Task在管道中的顺序有关!) |

## Task中的有效变量

| 变量 | 描述 |
| -------- | ----------- |
| `params.<param name>` | 运行时参数具体的值. |
| `resources.inputs.<resource name>.path` | 输入资源的路径. |
| `resources.outputs.<resource name>.path` | 输出资源的路径. |
| `results.<result name>.path` | 任务结果需要保存到的具体文件路径. |
| `workspaces.<workspace name>.path` | 挂载的工作区的路径. |
| `workspaces.<workspace name>.volume` | 工作区对应卷的名称. |
| `credentials.path` | 由`creds-init`初始化容器写入凭证的路径位置. |

### 管道资源变量

每一种类型的管道资源导出一些列的变量，以下按照管道资源类型来进行说明，这些资源在`Tasks`中有效，每个变量可以通过以下方式访问 `resources.inputs.<resource name>.<variable name>` 或者
`resources.outputs.<resource name>.<variable name>`.

#### Git 管道资源

| 变量 | 描述 |
| -------- | ----------- |
| `name` | 资源的名称 |
| `type` | `"git"` |
| `url` | Git仓库的url地址 |
| `revision` | 签出的版本号. |
| `depth` | 资源`depth`参数，整数. |
| `sslVerify` | 资源的 `sslVerify` 参数: `"true"` 或者 `"false"`. |
| `httpProxy` | 资源的 `httpProxy` 参数. |
| `httpsProxy` | 资源的 `httpsProxy` 参数. |
| `noProxy` | 资源的 `noProxy` 参数. |

#### PullRequest 管道资源

| 变量 | 描述 |
| -------- | ----------- |
| `name` | 资源的名称 |
| `type` | `"pullRequest"` |
| `url` | pull request的url地址. |
| `provider` | `"github"` 或者 `"gitlab"`. |
| `insecure-skip-tls-verify` | 资源的 `insecure-skip-tls-verify` 参数: `"true"` 或者 `"false"`. |

#### Image 管道资源

| 变量 | 描述 |
| -------- | ----------- |
| `name` | 资源名称 |
| `type` | `"image"` |
| `url` | 镜像的完整路径. |
| `digest` | 镜像的摘要. |

#### GCS 管道资源

| 变量 | 描述 |
| -------- | ----------- |
| `name` | 资源名称 |
| `type` | `"gcs"` |
| `location` | blob 存储的位置. |

#### BuildGCS 管道资源

| 变量 | 描述 |
| -------- | ----------- |
| `name` | 资源名称 |
| `type` | `"build-gcs"` |
| `location` | blob 存储的位置. |

#### Cluster 管道资源

| 变量 | 描述 |
| -------- | ----------- |
| `name` | 资源名称 |
| `type` | `"cluster"` |
| `url` | 主节点的宿主域名. |
| `username` | 访问集群的用户. |
| `password` | 访问集群的密码. |
| `namespace` | 集群的目标命名空间. |
| `token` | Bearer 令牌. |
| `insecure` | 是否TLS连接验证: `"true"` 或者 `"false"`. |
| `cadata` | 从根证书读取的字符串化的PEM编码字节. |
| `clientKeyData` | TLS客户端秘钥文件字符串化后的PEM编码. |
| `clientCertificateData` | TLS客户端证书文件字符串化后的PEM编码. |

#### CloudEvent 管道资源

| 变量 | 描述 |
| -------- | ----------- |
| `name` | 资源名称 |
| `type` | `"cloudEvent"` |
| `target-uri` | 云事件负载对应的URI. |
