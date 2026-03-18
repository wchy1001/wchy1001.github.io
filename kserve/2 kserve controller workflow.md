---
layout: default
title: kserve llmisvc controller
---

## kserve 资源介绍

本章通过自己手动创建llmisvc 来查看代码是如何运行的

## 创建测试示例

参考官网的demo，首先创建如下的CR：

```yaml
apiVersion: serving.kserve.io/v1alpha1
kind: LLMInferenceService
metadata:
  name: small-llm-pd-single-gpu
  namespace: default
spec:

  model:
    uri: pvc://nfs-pvc/hf/8b_instruction_tuned 
    name: 8b_instruction_tuned

  # Decode 工作负载（主服务）
  replicas: 2
  template:
    containers:
      - name: main
        resources:
          limits:
            nvidia.com/gpu: "1"
            cpu: "8"
            memory: 32Gi

  # Prefill 工作负载（同样也在单卡上跑，主要为了架构上 PD 分离）
  prefill:
    replicas: 2
    template:
      containers:
        - name: main
          resources:
            limits:
              nvidia.com/gpu: "1"
              cpu: "8"
              memory: 32Gi

  # 路由配置：使用默认网关和 HTTPRoute（空对象即可让 KServe 自动生成）[web:1]
  router:
    gateway: {}
    route: {}
    scheduler: {}
```

## Kserver处理LLMInferenceService

在controller中的reconcile方法中，主要完成如下功能

```go
func (r *LLMISVCReconciler) reconcile(ctx context.Context, llmSvc *v1alpha1.LLMInferenceService) error {
    logger := log.FromContext(ctx).WithName("reconcile")
    ctx = log.IntoContext(ctx, logger)

    // Load global configuration from KServe configmap
    // TODO(ctrl): add watch on CfgMap with predicate and cache tuning to trigger reconcile when it changes
    config, configErr := LoadConfig(ctx, r.Clientset)
    if configErr != nil {
        return fmt.Errorf("failed to load ingress config: %w", configErr)
    }

    // Combine base configurations with service-specific overrides
    // This includes default configs based on deployment pattern (single node, multi-node, etc.)
    baseCfg, err := r.combineBaseRefsConfig(ctx, llmSvc, config)
    if err != nil {
        llmSvc.MarkPresetsCombinedNotReady("CombineBaseError", err.Error())
        return fmt.Errorf("failed to combine base-configurations: %w", err)
    }
    llmSvc.MarkPresetsCombinedReady()

    logger.V(2).Info("Reconciling with combined base configurations", "combined.spec", baseCfg.Spec, "original.spec", llmSvc.Spec)
    // Replace the spec with the merged configuration for reconciliation
    // We are only writing to status, so we can safely use the original object.
    llmSvc.Spec = baseCfg.Spec

    if err := r.reconcileWorkload(ctx, llmSvc, config); err != nil {
        return fmt.Errorf("failed to reconcile workload: %w", err)
    }

    if err := r.reconcileRouter(ctx, llmSvc); err != nil {
        return fmt.Errorf("failed to reconcile networking: %w", err)
    }

    return nil
}
```

主要做了如下几件事：

1. 合并模板文件，如果在LLMisvc 中引用了其他的配置文件，那么会在这里统一合并,如果没有，那么合并默认的配置文件模版。
2.reconcile workload： 处理workload，workload主要分为lws（用于多机模型）和deployment（用于单机模型）
3.reconcileRouter：处理router

### 合并模版文件

LLMinferencesvc和其他的资源一样，需要合并配模板置选项，配置资源'llminferenceserviceconfigs.serving.kserve.io'，安装完kserver会自动生成默认的配置文件，内容如下：

```bash
root@worker-1:~# kubectl  -n kserve get llminferenceserviceconfigs.serving.kserve.io
NAME                                             AGE
kserve-config-llm-decode-template                95d
kserve-config-llm-decode-worker-data-parallel    95d
kserve-config-llm-prefill-template               95d
kserve-config-llm-prefill-worker-data-parallel   95d
kserve-config-llm-router-route                   95d
kserve-config-llm-scheduler                      95d
kserve-config-llm-template                       95d
kserve-config-llm-worker-data-parallel           95d
```

合并的规则如下：

```go
    if resolvedSpec.Prefill != nil { // P/D
        // Prefill
        switch {
        case resolvedSpec.Prefill.Worker == nil:
            // single-node prefill
            refs = append(refs, corev1.LocalObjectReference{Name: configPrefillTemplateName})
        case resolvedSpec.Prefill.Worker != nil && resolvedSpec.Prefill.Parallelism.IsDataParallel():
            // multi-node Data Parallel prefill
            refs = append(refs, corev1.LocalObjectReference{Name: configPrefillWorkerDataParallelName})
        case resolvedSpec.Prefill.Worker != nil && resolvedSpec.Prefill.Parallelism.IsPipelineParallel():
            // multi-node Pipeline Parallel prefill
            refs = append(refs, corev1.LocalObjectReference{Name: configPrefillWorkerPipelineParallelName})
        }
        // Decode
        switch {
        case resolvedSpec.Worker == nil:
            // single-node decode
            refs = append(refs, corev1.LocalObjectReference{Name: configDecodeTemplateName})
        case resolvedSpec.Worker != nil && resolvedSpec.Parallelism.IsDataParallel():
            // multi-node Data Parallel decode
            refs = append(refs, corev1.LocalObjectReference{Name: configDecodeWorkerDataParallelName})
        case resolvedSpec.Worker != nil && resolvedSpec.Parallelism.IsPipelineParallel():
            // multi-node Pipeline Parallel decode
            refs = append(refs, corev1.LocalObjectReference{Name: configDecodeWorkerPipelineParallelName})
        }
    } else { // Non P/D
        switch {
        case resolvedSpec.Worker == nil:
            // single-node
            refs = append(refs, corev1.LocalObjectReference{Name: configTemplateName})
        case resolvedSpec.Worker != nil && resolvedSpec.Parallelism.IsDataParallel():
            // multi-node Data Parallel
            refs = append(refs, corev1.LocalObjectReference{Name: configWorkerDataParallelName})
        case resolvedSpec.Worker != nil && resolvedSpec.Parallelism.IsPipelineParallel():
            // multi-node Pipeline Parallel
            refs = append(refs, corev1.LocalObjectReference{Name: configWorkerPipelineParallelName})
        }
    }
```

先判断是不是 PD分离（通过判断prefill是否指定），如果是，然后判断prefill的worker，如果没有，就用 `kserve-config-llm-prefill-template` 合并，如过prefill下面worker设置了，在看是DP还是PP架构，然后合并对应的配置文件，其他合并规则，可以看代码的解释。最后如果有手动指定其他的config reference，则继续合并,然后还需要合并router和scheduler等相关信息。如果需要自定义某些配置，可以修改模板里面的配置，或者新建模板然后在llmisvc中引入即可

### workload的处理

reconcile workload 主要工作如下：

```go
    // Handle multi-node deployments using LeaderWorkerSets
    if err := r.reconcileMultiNodeWorkload(ctx, llmSvc, config); err != nil {
        llmSvc.MarkWorkerWorkloadNotReady("ReconcileMultiNodeWorkloadError", err.Error())
        return fmt.Errorf("failed to reconcile multi node workload: %w", err)
    }

    // Handle single-node deployments using standard Deployments
    if err := r.reconcileSingleNodeWorkload(ctx, llmSvc, config); err != nil {
        llmSvc.MarkMainWorkloadNotReady("ReconcileSingleNodeWorkloadError", err.Error())
        return fmt.Errorf("failed to reconcile single node workload: %w", err)
    }

    // Create Service to expose workload pods
    if err := r.reconcileWorkloadService(ctx, llmSvc); err != nil {
        llmSvc.MarkMainWorkloadNotReady("ReconcileWorkloadServiceError", err.Error())
        return fmt.Errorf("failed to reconcile workload service: %w", err)
    }
```

不管如何情况，都会调和各种类型的workload，防止变更LLMisvc的时候，修改了拓扑结构（比如从multinode 改成单节点），
mutinode适合一些模型比较大的情况，比如deepseek-r1 这类 在单节点上没有办法部署完整的模型，使用LWS（leaderworkSet）来部署, reconcileMultiNodeWorkload 主要完成如下几个工作：

1.处理llm worker节点的ServiceAccount
2.处理llm-prefill节点的ServiceAccount（不一定有）
3.创建LLm worker节点的workload
4.创建prefill节点的workload

其中创建llm worker 的workload和prefill为主要工作流程：
`spec.worker`为空的时候，会删除workload, 换言之，只有当`llmisvc.spec.worker`不为空的时候，触发multinode LWS的创建：
创建worker节点的时候，需要查看prefill是否设置，如果设置了就是PD分离架构，否则就是普通的单实例架构，接下来创建leaderworkerset，其中设置 tag 如： `"llm-d.ai/role"` 为 decode 还是 both，如果prefil没有设置就是both，设置了就是role节点，后续会设置prefill节点，leaderworkerset 会去合并 llmsvc下面的template内容，自动复制到lws中，之后 挂载 模型，然后设置routing sidecar， routing sidecar用来后续分发流量到 prefill节点，还是 decode节点，如果有routing sidecar 那么则设置pod的环境变量`
INFERENCE_POOL_NAME`, decode节点的参数设置来源为：

```bash
kubectl get llminferenceserviceconfigs.serving.kserve.io -n kserve kserve-config-llm-decode-template  -o yaml
apiVersion: serving.kserve.io/v1alpha1
kind: LLMInferenceServiceConfig
metadata:
......
  name: kserve-config-llm-decode-template
  namespace: kserve
spec:
  template:
    containers:
    - args:
      - --served-model-name
      - '{{ .Spec.Model.Name }}'
      - --port
      - "8001"
      - --disable-log-requests
      command:
      - vllm
      - serve
      - /mnt/models
      env:
......
    initContainers:
    - args:
      - --port=8000
      - --vllm-port=8001
      - --connector=nixlv2
      - --secure-proxy=false
      env:
      - name: INFERENCE_POOL_NAMESPACE
        valueFrom:
          fieldRef:
            fieldPath: metadata.namespace
      image: ghcr.io/llm-d/llm-d-routing-sidecar:v0.3.0
      imagePullPolicy: IfNotPresent
      livenessProbe:
        failureThreshold: 3
        initialDelaySeconds: 10
        periodSeconds: 10
        tcpSocket:
          port: 8000
        timeoutSeconds: 10
      name: llm-d-routing-sidecar
  .......
```

其中明确设置了 llm-d-routing-sidecar 作为sidecar容器，因此，在创建worker的时候，内容如上，注意 sidecar容器仅仅在decode中使用，并不会在prefill节点，后面流量会根据k-v cache 或者 负载，优先发送到decode节点上，由decode节点，决定是否需要转发到prefill节点，以及如何转发到prefill节点。至此decode节点设置完成。

prefill节点： 当spec.prefill设置之后，触发prefill的lws创建：创建prefill 的lws基本和decode大差不差，区别在于引用了不同的 llminferenceserviceconfigs.serving.kserve.io 对象，以及缺少了sidecar等相关配置 内容如下:

```yaml
.......
spec:
  prefill:
    template:
      containers:
      - args:
        - --served-model-name
        - '{{ .Spec.Model.Name }}'
        - --port
        - "8000"
        - --disable-log-requests
        command:
        - vllm
        - serve
        - /mnt/models
        env:
        - name: HOME
          value: /home
        - name: VLLM_LOGGING_LEVEL
          value: INFO
        - name: HF_HUB_CACHE
          value: /models
        image: ghcr.io/llm-d/llm-d-dev:v0.2.2
        imagePullPolicy: IfNotPresent
        name: main
.......
```

之后处理单节点的情况，其实单节点和multinode的流程大致相似，只不过把LWS的资源对象，替换成deployment，其他大致相同，流程为：
判断是否单节点（根据llmsvc.spec.worker是否设置），如果设置，则删除当前的deployment，说明已经被上一步的multinode创建了，否则创建deployment, deployment 根据 kserve-config-llm-template 来生成deployment资源，设置role为decode还是prefill，创建deployment，处理serviceAccount  处理template中的合并内容，然后是否添加sidecar，挂载 model位置

单节点和多节点，唯一的区别在于使用的lws还是deployment，在PD分离的场景中，几乎是一样的配置和流程。

### 路由处理

处理路由 如下：
1.创建scheduler，包括：

- 创建scheduler service account
- 创建inference model
- 创建scheduler deployment
- 创建scheduler service
- 创建inference pool

2.创建http route

- 设置http route 把api 请求路由到 inference pool 上面

至此，kserve的工作结束，至于流量是如何路由的，以及epp如何pick的，则属于llm-d的领域，下文将会分析

## 创建的结果

```bash
root@worker-1:~/wchy/kubeflow-workspaces# kubectl  get llminferenceservice
NAME                      URL   READY   REASON   AGE
small-llm-pd-single-gpu         True             72d
```

worker & prefill  deployment，其中两个decode节点，2个prefill节点，1个 scheduler节点

``` bash
root@worker-1:~/wchy/kubeflow-workspaces# kubectl  get deployments.apps
NAME                                              READY   UP-TO-DATE   AVAILABLE   AGE
small-llm-pd-single-gpu-kserve                    2/2     2            2           72d
small-llm-pd-single-gpu-kserve-prefill            2/2     2            2           72d
small-llm-pd-single-gpu-kserve-router-scheduler   1/1     1            1           72d
```

inference pool 和 inference model 如下：

```bash
root@worker-1:~# kubectl  get inferencemodels.inference.networking.x-k8s.io
NAME                                      MODEL NAME             INFERENCE POOL                           CRITICALITY   AGE
small-llm-pd-single-gpu-inference-model   8b_instruction_tuned   small-llm-pd-single-gpu-inference-pool   Critical      72d
root@worker-1:~# kubectl  get inferencepools.inference.networking.x-k8s.io
NAME                                     AGE
small-llm-pd-single-gpu-inference-pool   72d
root@worker-1:~#
```

httprouter：

```
root@worker-1:~# kubectl  get httproute -o yaml
apiVersion: v1
items:
- apiVersion: gateway.networking.k8s.io/v1
  kind: HTTPRoute
  metadata:
    creationTimestamp: "2025-12-16T07:22:09Z"
    generation: 1
    labels:
      app.kubernetes.io/component: llminferenceservice-router
      app.kubernetes.io/name: small-llm-pd-single-gpu
      app.kubernetes.io/part-of: llminferenceservice
    name: small-llm-pd-single-gpu-kserve-route
    namespace: default
    ownerReferences:
    - apiVersion: serving.kserve.io/v1alpha1
      blockOwnerDeletion: true
      controller: true
      kind: LLMInferenceService
      name: small-llm-pd-single-gpu
      uid: 81661342-6dcd-441e-a548-37fa43fd0257
    resourceVersion: "94566617"
    uid: fa36b62d-81ae-4793-a47c-a24369121c7d
  spec:
    parentRefs:
    - group: gateway.networking.k8s.io
      kind: Gateway
      name: kserve-ingress-gateway
      namespace: kserve
    rules:
    - backendRefs:
      - group: inference.networking.x-k8s.io
        kind: InferencePool
        name: small-llm-pd-single-gpu-inference-pool
        port: 8000
        weight: 1
      filters:
      - type: URLRewrite
        urlRewrite:
          path:
            replacePrefixMatch: /
            type: ReplacePrefixMatch
      matches:
      - path:
          type: PathPrefix
          value: /default/small-llm-pd-single-gpu
      timeouts:
        backendRequest: 0s
        request: 0s
```