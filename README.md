<!--
---
title: "Tasks and Pipelines"
linkTitle: "Tasks and Pipelines"
weight: 2
description: >
  Building Blocks of Tekton CI/CD Workflow
---
-->
# Tekton管道

Tekton管道是在Kubernetes集群上安装和运行的Kubernetes扩展。
它定义了一系列Kubernetes[自定义资源](https://kubernetes.io/docs/concepts/extend-kubernetes/api-extension/custom-resources/)，通过这些对象可让你组装CI/CD管道。一旦安装，Tekton管道可通过Kubernetes命令行(kubectl)和API调用来管理，就像Pod或其他kubernetes资源一样。Tekton是 [CD基金会](https://cd.foundation/),
和 [Linux基金会](https://www.linuxfoundation.org/projects/) 项目。


## Tekton管道实体

Tekton管道定义了以下实体:

<table>
  <tr>
    <th>实体</th>
    <th>描述</th>
  </tr>
  <tr>
    <td><code>Task</code></td>
    <td>任务定义一系列步骤来运行特定的构建与交付工具，以此通过特定的输入来产生特定的输出</td>
  </tr>
  <tr>
    <td><code>TaskRun</code></td>
    <td><code>Task</code>的实例，可指定实际的输入、输出以及执行参数，可通过自身或作为<code>Pipeline</code>的一部分来调用</td>
  </tr>
  <tr>
    <td><code>Pipeline</code></td>
    <td>定义一系列的<code>Tasks</code>来完成特定的构建或交付目标。可通过事件或<code>PipelineRun</code>来运行</td>
  </tr>
  <tr>
    <td><code>PipelineResource</code></td>
    <td>定义<code>Tasks</code>中步骤需要的输入以及产生的输出位置</td>
  </tr>
  <tr>
    <td><code>PipelineRun</code></td>
    <td><code>Pipeline</code>的实例，可指定特定的输入、输出及执行参数来运行</td>
  </tr>
</table>

## 开始

你可以通过完成[Tekton管道手册](https://github.com/tektoncd/pipeline/blob/master/docs/tutorial.md)以及跟随[示例](https://github.com/tektoncd/pipeline/tree/master/examples)来快速入门。


## 理解Tekton管道

通过以下主题你可以学习如何在你的项目中使用Tekton管道:

- [创建任务](tasks.md)
- [运行独立的任务](taskruns.md)
- [创建管道](pipelines.md)
- [运行管道](pipelineruns.md)
- [定义工作区](workspaces.md)
- [定义管道资源](resources.md)
- [配置认证](auth.md)
- [使用标签](labels.md)
- [查看日志](logs.md)
- [管道指标](metrics.md)
- [变量替换](variables.md)

## 参与Tekton管道项目

如果你想要参与到Tekton管道项目中，请查看[Tekton管道项目贡献指南](https://github.com/tektoncd/pipeline/blob/master/CONTRIBUTING.md)

---

除非另有说明，本页内容采用[Creative Commons Attribution 4.0 License](https://creativecommons.org/licenses/by/4.0/)授权协议，示例代码采用[Apache 2.0 License](https://www.apache.org/licenses/LICENSE-2.0)授权协议
