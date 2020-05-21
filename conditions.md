<!--
---
linkTitle: "Conditions"
weight: 11
---
-->
# 条件

本文档描述`Conditions`及其能力.

---

- [语法](#语法)
  - [检查](#检查)
  - [参数](#参数)
  - [资源](#资源)
- [标签](#标签)
- [示例](#示例)

## 语法

要定义`Condition`资源的配置文件，你可设置以下字段:

- 必须:
  - [`apiVersion`][kubernetes-overview] - API 版本, 例如
    `tekton.dev/v1alpha1`.
  - [`kind`][kubernetes-overview] - 设定为`Condition`资源对象.
  - [`metadata`][kubernetes-overview] - 设定`Condition`资源对象的元数据, 例如`名称（name）`.
  - [`spec`][kubernetes-overview] - 设置`Condition`资源对象的配置信息，以便于让`Condition`做相应的事情，包括:
    - [`check`](#检查) - 设定一个用于运行评估条件的容器
    - [`description`](#description) - 条件的描述.

[kubernetes-overview]:
  https://kubernetes.io/docs/concepts/overview/working-with-objects/kubernetes-objects/#required-fields

### 检查

检查 `check` 字段是必须的，你需要定义一个单独的check来定义`Condition`的内容, check必须指定一个 [`Step`](./tasks.md#定义-Steps). 镜像容器执行至完成，容器必须成功地退出，例如退出代码为0,表示条件评估成功，其他退出码表示条件检查失败.

### 描述

描述`description`字段是可选字段，用于描述条件.

### 参数

可以为条件定义参数，这些参数必须在PipelineRun中提供. check下的字段可以通过模板语法访问这些参数:

```yaml
spec:
  parameters:
    - name: image
      default: ubuntu
  check:
    image: $(params.image)
```

参数名称必须由数字、字符以及`-` 和 `_`组成，并且只能以字符或`_`开始。例如，`fooIs-Bar_`是有效名称，`barIsBa$`或者`0banana`是无效的.

每一个定义的参数具有类型字段，如果没有设置将默认为字符串类型.
其他可选的类型为数组(array) - 非常有用, 例如，检查推送的分支名称不匹配多个被保护的分支名称. 当实际的参数值被提供时，将会按照提供类型解析与验证.

### 资源

条件可以通过`resources`字段定义输入[`管道资源`](resources.md)，以便于条件容器步骤获取对应的数据或上下文来处理检查.

在条件中的资源与`Tasks`中的使用方式一致，例如可以使用
[变量替换](./resources.md#变量替换) 以及 `targetPath` 字段来[控制资源挂载的位置](./resources.md#控制资源挂载位置)

## 标签

[标签](labels.md) 作为`Condition`元数据的一部分来定义，将会自动传递给`Pod`.

## 示例

查看完整的示例，请参考
[示例目录](https://github.com/tektoncd/pipeline/tree/master/examples).

---

除非另有说明，本页内容采用[Creative Commons Attribution 4.0 License](https://creativecommons.org/licenses/by/4.0/)授权协议，示例代码采用[Apache 2.0 License](https://www.apache.org/licenses/LICENSE-2.0)授权协议
