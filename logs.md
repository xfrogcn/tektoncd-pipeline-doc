<!--
---
linkTitle: "Logs"
weight: 9
---
-->
# 日志

[`PipelineRuns`](pipelineruns.md) 以及 [`TaskRuns`](taskruns.md)的日志是关联到相关的Pod的.

要访问日志，现在有以下几种方式:

- [你可以从pod中获取日志](https://kubernetes.io/docs/reference/kubectl/cheatsheet/#interacting-with-running-pods)
  例如使用 `kubectl`:

  ```bash
  # 获取TaskRun实例关联的pod名称
  kubectl get taskruns -o yaml | grep podName

  # 或者从PipelineRun获取关联的pod名称
  kubectl get pipelineruns -o yaml | grep podName

  # 使用kubectl访问pod中所有容器的日志
  kubectl logs $POD_NAME --all-containers

  # 或者获取pod中特定容器的日志
  kubectl logs $POD_NAME -c $CONTAINER_NAME
  kubectl logs $POD_NAME -c step-run-kubectl
  ```

- 你可以直接使用[`tkn` 命令行工具](https://github.com/tektoncd/cli)来访问日志
- 你可以使用
  [web仪表盘](https://github.com/tektoncd/dashboard)来访问日志
- 你可以设置外部服务来使用及显示日志, 例如
  [Elasticsearch, Beats and Kibana](https://github.com/mgreau/tekton-pipelines-elastic-tutorials)
