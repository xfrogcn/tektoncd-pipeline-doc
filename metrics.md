<!--
---
linkTitle: "Pipeline Metrics"
weight: 14
---
-->
# 管道控制器指标

以下管道指标由`controller-service`服务从`9090`端口导出

支持以下类型的导出器：Prometheus、Google Stackdriver等，他们可以通过[可观察性配置](./config/config-observability.yaml)来配置.


| 指标名称 | 指标类型 | 标签 | 状态 |
| ---------- | ----------- | ----------- | ----------- |
| tekton_pipelinerun_duration_seconds_[bucket, sum, count] | 直方图 | `pipeline`=&lt;pipeline_name&gt; <br> `pipelinerun`=&lt;pipelinerun_name&gt; <br> `status`=&lt;status&gt; <br> `namespace`=&lt;pipelinerun-namespace&gt; | 实验性 |
| tekton_pipelinerun_taskrun_duration_seconds_[bucket, sum, count] | 直方图 | `pipeline`=&lt;pipeline_name&gt; <br> `pipelinerun`=&lt;pipelinerun_name&gt; <br> `status`=&lt;status&gt; <br> `task`=&lt;task_name&gt; <br> `taskrun`=&lt;taskrun_name&gt;<br> `namespace`=&lt;pipelineruns-taskruns-namespace&gt;| 实验性 |
| tekton_pipelinerun_count| 计数器 | `status`=&lt;status&gt; | 实验性 |
| tekton_running_pipelineruns_count | 计量器 | | 实验性 | 
| tekton_taskrun_duration_seconds_[bucket, sum, count] | 直方图 | `status`=&lt;status&gt; <br> `task`=&lt;task_name&gt; <br> `taskrun`=&lt;taskrun_name&gt;<br> `namespace`=&lt;pipelineruns-taskruns-namespace&gt; | 实验性 | 
| tekton_taskrun_count | 计数器 | `status`=&lt;status&gt; | 实验性 | 
| tekton_running_taskruns_count | 计量器 | | 实验性 |
| tekton_taskruns_pod_latency | 计量器 | `namespace`=&lt;taskruns-namespace&gt; <br> `pod`= &lt; taskrun_pod_name&gt; <br> `task`=&lt;task_name&gt; <br> `taskrun`=&lt;taskrun_name&gt;<br> | 实验性 |
