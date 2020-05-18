<!--
---
linkTitle: "Labels"
weight: 10
---
-->
# 标签

为了更容易地区分与定义管道中的对象，被Tekton管道使用的自定义
[标签](https://kubernetes.io/docs/concepts/overview/working-with-objects/labels/)
集将会从一般资源传递到更具体的资源，另一些标签将会被自动添加，以便于标识资源之间的关系.

---

- [传递明细](#传递明细)
- [自动添加标签](#自动添加标签)
- [示例](#示例)

---

## 传递明细

针对通过`PipelineRun`来执行的`Pipelines`，标签自动从`Pipelines`传递到`PipelineRun`到`TaskRuns`，然后到`Pods`，另外，来自于被`TaskRuns`引用的`Tasks`的标签将会被传递到`TaskRuns`，最后到`Pods`.

针对于直接执行的`TaskRuns`，未作为`Pipeline`的一部分，标签从`Task`(如果存在，参考[指定Task](taskruns.md#指定Task)文档)传递到`TaskRun`,最后到`Pod`.

对于`条件`，标签自动传递到关联的`TaskRuns`，最后到`Pods`.

## 自动添加标签

以下标签被自动添加到资源:

- `tekton.dev/pipeline` 自动添加到`PipelineRuns`(然后传递到`TaskRuns`及`Pods`)，值为`PipelineRun`关联的`Pipeline`名称.
- `tekton.dev/pipelineRun` 自动添加到`TaskRuns` (然后传递到`TaskRuns`及`Pods`), 他在`PipelineRun`运行期间自动创建，包含自动触发创建`TaskRun`的`PipelineRun`的名称.
- `tekton.dev/task` 自动添加到`TaskRuns` (然后传递到`Pods`)，它指向一个已存在的`Task`（参考[指定Task](taskruns.md#指定Task)文档），包含`TaskRun`相关的`Task`的名称.
- `tekton.dev/clusterTask` 自动添加到 `TaskRuns` (然后传递到`Pods`) ，它指向一个已经存在的`ClusterTask`, 包含`TaskRun`关联的`ClusterTask`的名称，为了向前兼容，`TaskRuns`引用的`ClusterTask`也会接收`tekton.dev/task`标签.
- `tekton.dev/taskRun` 自动添加到`Pods`, 它包含创建`Pods`的`TaskRun`的名称.

## 示例

- [找到PipeRun对应的Pods](#找到PipeRun对应的Pods)
- [找到Task对应的TaskRuns](#找到Task对应的TaskRuns)

### 找到PipeRun对应的Pods

为了找到名称为test-pipelinerun的`PipelineRun`所创建的`Pods`，你可以使用以下命令:

```shell
kubectl get pods --all-namespaces -l tekton.dev/pipelineRun=test-pipelinerun
```

### 找到Task对应的TaskRuns

为了找到名称为test-task的`Task`所关联的所有`TaskRuns`，你可以使用以下命令:

```shell
kubectl get taskruns --all-namespaces -l tekton.dev/task=test-task
```

### 找到ClusterTask对应的TaskRuns

为了找到名称为test-clustertask`ClusterTask`所关联的所有`TaskRuns`，你可以使用以下命令:

```shell
kubectl get taskruns --all-namespaces -l tekton.dev/clusterTask=test-clustertask
```
