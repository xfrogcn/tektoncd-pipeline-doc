<!--
---
linkTitle: "Pod Templates"
weight: 12
---
-->
# Pod模板

pod模板是[`PodSpec`](https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.15/#pod-v1-core)的子集，可以配置为`Task` pod的基础模板.

这允许你自定义`Task`执行的Pod特定字段, 即`TaskRun`. 
This allows to customize some Pod specific field per `Task` execution, aka `TaskRun`.

另外，你也可以在tekton配置中定义一个默认的pod模板，参考[此处](https://github.com/tektoncd/pipeline/blob/master/docs/install.md), 如果在`PipelineRun`或`TaskRun`中指定了pod模板，默认pod模板将会被忽略. 所以在都存在的情况下，模板**不会**被合并, 它总是只会使用一个.

---

当前支持的字段如下:

- `nodeSelector`: 节点选择器，pod调度必须符合此选择器设置, 参考 [此处](https://kubernetes.io/docs/concepts/configuration/assign-pod-node/).
- `tolerations`: 允许 (但不是必须)pod调度到匹配的节点.
- `affinity`: 强制pod调度到符合条件的节点, 依据节点的标签.
- `securityContext`: pod级别的安全属性以及通用的容器设置，向`runAsUser`或者`selinux` .
- `volumes`: 需要挂载到pod中容器内的卷列表, 这可以让任务用户定义任务使用哪种类型的卷，通过`volumeMount`
- `runtimeClassName`: 用于运行pod的
  [运行时类](https://kubernetes.io/docs/concepts/containers/runtime-class/).
- `automountServiceAccountToken`:是否自动将服务账户所对应的令牌挂载到容器的预定义路径下，默认为`true`.
- `dnsPolicy`: pod的
  [DNS 策略](https://kubernetes.io/docs/concepts/services-networking/dns-pod-service/#pod-s-dns-policy), 可以为`ClusterFirst`, `Default`, 或者 `None`,默认为`ClusterFirst`，注意，Tekton不支持`ClusterFirstWithHostNet`，tekton的pod不能运行为宿主网络模式.
- `dnsConfig`: pod的
  [DNS附加配置](https://kubernetes.io/docs/concepts/services-networking/dns-pod-service/#pod-s-dns-config), 例如域名服务器以及搜索域名.
- `enableServiceLinks`: 是否在相同命名空间中的pod内服务导出为pod的环境变量，类似于Docker的服务链接(service links). 默认为`true`.
- `priorityClassName`: 当pod运行时所使用的
  [优先级类](https://kubernetes.io/docs/concepts/configuration/pod-priority-preemption/). 例如， 有选择地使用低优先级的工作负载.
- `schedulerName` 在调度pod时所使用的 
  [调度器](https://kubernetes.io/docs/tasks/administer-cluster/configure-multiple-schedulers/). 当特定的工作负载需要特定的调度器时，可以设置此字段，例如，如果你针对机器学习工作负载使用volcano.sh，你可以指定vocano.sh调度器的名称来完成.

可以为`TaskRun`或者`PipelineRun`资源指定pod模板，参考[此处](./taskruns.md#pod-template) 或 [此处](./pipelineruns.md#pod-template) 了解使用pod模板的示例.

---

除非另有说明，本页内容采用[Creative Commons Attribution 4.0 License](https://creativecommons.org/licenses/by/4.0/)授权协议，示例代码采用[Apache 2.0 License](https://www.apache.org/licenses/LICENSE-2.0)授权协议
