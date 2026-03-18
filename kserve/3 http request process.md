---
layout: default
title: kserve request process
---

## 介绍

本文主要介绍一个request是如何路由的

## 背景

在上文中，Kserver已经完成了重要的资源编排，接下来要看主要的 数据平面

## 请求测试

请求测试以及内容如下

```bash
# 查看body
root@worker-1:~/wchy# cat chat-input-test.json
{
  "model": "8b_instruction_tuned",
  "messages": [
    {
      "role": "system",
      "content": "You are a helpful assistant that provides clear and concise answers."
    },
    {
      "role": "user",
      "content": "Write a short poem about artificial intelligence and machine learning."
    }
  ],
  "max_tokens": 150,
  "temperature": 0.7,
  "stream": false
}

请求如下：
 curl -v http://172.28.0.131:31487/default/small-llm-pd-single-gpu/v1/chat/completions   -H "Content-Type: application/json"  -d @./chat-input-test.json
 # response 如下：
 {"choices":[{"finish_reason":"length","index":0,"logprobs":null,"message":{"annotations":null,"audio":null,"content":"Here is a short poem about artificial intelligence and machine learning:\n\nIn silicon halls, a new mind awakes\nArtificial intelligence, with knowledge it partakes\nAlgorithms dance, with data they play\nLearning from errors, night and day\n\nWith each step, its power grows strong\nPredicting futures, right or wrong\nFrom images to voices, it can create\nA world of possibilities, an endless debate\n\nMachine learning, a tool so fine\nTeaching computers, to think divine\nThrough trial and error, it refines its art\nSolving problems, a brand new start\n\nIn this digital age, we're amazed\nBy the feats AI achieves, in endless ways\nA future unfolds, with endless might\nArtificial intelligence","function_call":null,"reasoning":null,"reasoning_content":null,"refusal":null,"role":"assistant","tool_calls":[]},"stop_reason":null,"token_ids":null}],"created":1772249231,"id":"chatcmpl-23f7a829-3247-4e4* Connection #0 to host 172.28.0.131 left intact
7-bc6c-2f74ee445644","kv_transfer_params":null,"model":"8b_instruction_tuned","object":"chat.completion","prompt_logprobs":null,"prompt_token_ids":null,"service_tier":null,"system_fingerprint":null,"usage":{"completion_tokens":150,"prompt_tokens":38,"prompt_tokens_details":null,"total_tokens":188}}root@worker-1:~/wchy#
```

## 路由过程

请求先到gateway中

```bash
root@worker-1:~# kubectl  get httproutes.gateway.networking.k8s.io  -o yaml
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

资源绑定在了Kserver namespace下面的‘kserve-ingress-gateway’ gateways下面,而gateway 绑定在了envory controller下面

```bash
root@worker-1:~# kubectl  -n kserve get gateways -o yaml
apiVersion: v1
items:
- apiVersion: gateway.networking.k8s.io/v1
  kind: Gateway
  metadata:
.......
  spec:
    gatewayClassName: envoy
    infrastructure:
      labels:
        serving.kserve.io/gateway: kserve-ingress-gateway
    listeners:
    - allowedRoutes:
        namespaces:
          from: All
      name: http
      port: 80
      protocol: HTTP
```

查看 gateway controller如下：

```bash
root@worker-1:~# kubectl  -n envoy-gateway-system  get svc
NAME                                           TYPE           CLUSTER-IP      EXTERNAL-IP   PORT(S)                                            AGE
envoy-gateway                                  ClusterIP      10.43.219.148   <none>        18000/TCP,18001/TCP,18002/TCP,19001/TCP,9443/TCP   92d
envoy-kserve-kserve-ingress-gateway-deaaa49b   LoadBalancer   10.43.255.164   <pending>     80:31487/TCP                                       92d
envoy-ratelimit                                ClusterIP      10.43.162.21    <none>        8081/TCP,19001/TCP                                 92d
root@worker-1:~#
```

backend 为 inference pool，需要走到envory的extension，首先查看配置文件：

```bash
root@worker-1:~# kubectl  -n envoy-gateway-system  get cm envoy-gateway-config  -o yaml
apiVersion: v1
data:
  envoy-gateway.yaml: |
    apiVersion: gateway.envoyproxy.io/v1alpha1
    kind: EnvoyGateway
    gateway:
      controllerName: gateway.envoyproxy.io/gatewayclass-controller
    logging:
      level:
        default: info
    provider:
      kubernetes:
        rateLimitDeployment:
          patch:
            type: StrategicMerge
            value:
              spec:
                template:
                  spec:
                    containers:
                    - imagePullPolicy: IfNotPresent
                      name: envoy-ratelimit
                      image: docker.io/envoyproxy/ratelimit:60d8e81b
      type: Kubernetes
    extensionApis:
      enableEnvoyPatchPolicy: true
      enableBackend: true
    extensionManager:
      backendResources:
        - group: inference.networking.x-k8s.io
          kind: InferencePool
          version: v1alpha2
      hooks:
        xdsTranslator:
          translation:
            listener:
              includeAll: true
            route:
              includeAll: true
            cluster:
              includeAll: true
            secret:
              includeAll: true
          post:
            - Translation
            - Cluster
            - Route
      service:
        fqdn:
          hostname: ai-gateway-controller.envoy-ai-gateway-system.svc.cluster.local
          port: 1063
```

inferencepool的流量会 导入到： ai-gateway-controller.envoy-ai-gateway-system.svc.cluster.local 这里面，这个AI gateways 信息如下：

```bash
root@worker-1:~# kubectl  -n envoy-ai-gateway-system get svc
NAME                    TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)                      AGE
ai-gateway-controller   ClusterIP   10.43.31.1   <none>        9443/TCP,1063/TCP,9090/TCP   92d
root@worker-1:~# kubectl  -n envoy-ai-gateway-system get pods
NAME                                     READY   STATUS    RESTARTS       AGE
ai-gateway-controller-795f8859f7-hfr5m   1/1     Running   10 (18d ago)   92d
root@worker-1:~#
```

AI gateway的代码在 [这里]( https://github.com/envoyproxy/ai-gateway)

在 AI gateways中，主要完成两个工作，一个是启动extension grpc server，另一个是 创建 inferencepool controller， extensionserver 实现了 根据inferencepool去选择EPP，也就是scheduler service，而inferencepool controller 主要用的是 检查EPP 服务 是不是存在，EPP服务如下：

```bash
root@worker-1:~# kubectl  get inferencepools.inference.networking.x-k8s.io  -o yaml
apiVersion: v1
items:
- apiVersion: inference.networking.x-k8s.io/v1alpha2
  kind: InferencePool
......
  spec:
    extensionRef:
      failureMode: FailOpen
      group: ""
      kind: Service
      name: small-llm-pd-single-gpu-epp-service
    selector:
      app.kubernetes.io/name: small-llm-pd-single-gpu
      app.kubernetes.io/part-of: llminferenceservice
      kserve.io/component: workload
    targetPortNumber: 8000
```

检查 extensionRef的服务是不是存在，然后更新InferencePool 的状态, 接下来请求到 EPP 也就是scheduler 服务

## EPP 服务

EPP也就是scheduler，算是核心的模块了，在llm-d scheduler中，通过注册了一堆插件，然后运行对应的EPP服务，插件的内容如下，后续会说到EPP的加载过程。EPP通过注册一堆plugin，在后端的pods中，层层筛选，然后把请求最终转发到合适的pod上，

### EPP 注册插件

EPP 注册了如下插件

```go
// RegisterAllPlugins registers the factory functions of all plugins in this repository.
func RegisterAllPlugins() {
    plugins.Register(filter.ByLabelType, filter.ByLabelFactory)
    plugins.Register(filter.ByLabelSelectorType, filter.ByLabelSelectorFactory)
    plugins.Register(filter.DecodeRoleType, filter.DecodeRoleFactory)
    plugins.Register(filter.PrefillRoleType, filter.PrefillRoleFactory)
    plugins.Register(prerequest.PrefillHeaderHandlerType, prerequest.PrefillHeaderHandlerFactory)
    plugins.Register(profile.DataParallelProfileHandlerType, profile.DataParallelProfileHandlerFactory)
    plugins.Register(profile.PdProfileHandlerType, profile.PdProfileHandlerFactory)
    plugins.Register(scorer.PrecisePrefixCachePluginType, scorer.PrecisePrefixCachePluginFactory)
    plugins.Register(scorer.LoadAwareType, scorer.LoadAwareFactory)
    plugins.Register(scorer.SessionAffinityType, scorer.SessionAffinityFactory)
    plugins.Register(scorer.ActiveRequestType, scorer.ActiveRequestFactory)
    plugins.Register(scorer.NoHitLRUType, scorer.NoHitLRUFactory)
}
```

#### EPP注册的插件介绍

1. bylabel plugin

    主要是根据targets pods中的label 和 value 来筛选 选择哪个，主要是根据key 和 有效的value，比如 key是app，value： test，test1，test2

2. ByLabelSelectorFactory plugin

    和bylabel差不多，只不过多了selector，支持复杂的label 匹配

3. Decoderole plugin

    根据llm-d.ai/role来筛选出 值是both 或者 decode的角色

4. prefillrole plugin

    筛选出llm-d.ai/role 是 prefill的 backend

5. prefillheaderhandler plugin

    主要是删除llm请求头部中的x-prefiller-host-port，通过prefill的profile来筛选出符合条件的pod，然后把pod中的第一个 ip地址+port组合到head中，类似： "x-prefiller-host-port=1.1.1.1:7890"

6. data_parallel_profile_handler plugin

    实现了pick 和 ProcessResults 接口，前者挑选合适的 profile执行，后者处理筛选后的结果，主要功能是在request的头部中添加：x-data-parallel-host-port 值是pod中的 ip+端口，之后会把筛选后的pod中的端口，统一替换成配置的端口，默认是8000, 8000是sidecar的port，vllm是8001， sidecar 会做反向代理，把port转发给 8001

7. pd_profile_handler plugin

    同样实现了pick 和 processresult接口，其中pick 先检查 decode的profile是否运行完成，如果运行完成了，获取prefixstate，然后获取到decode pod，首先转换成server id，然后通过server id 计算 cache命中率百分比，最后决定要不要 运行prefill的profile

8. PrecisePrefixCachePlugin

    实现了Score接口，用来给一堆pod打分，总体流程是 使用kvcacheindexer 根据prompt和模型，来打分，打分的步骤是，先tokenize prompt，然后把token转换成 kv_block.Keys，根据这个key去查找pod，然后在对这帮pods 进行打分，返回一堆pod和分数，最后转换成百分比分数，百分比转换如下：
`scoredPods[pod] = (score - minScore) / (maxScore - minScore)`

9. LoadAware plugin

    根据request waiting 这个metrics 来打分，pod 打分在0-0.5之间，如果没有等待的request 那么pod得分是 0.5， 如果有request在queue中，得分在0-0.5之间

10. SessionAffinity plugin

    根据reqeust header中的 x-session-token 中的pod打分，如果是匹配的pod 就是 1分，其他的是0分

11. ActiveRequest plugin

    根据activerequest打分，如果没有activerequest 意味着最高分 1分。否则拿出所有pod中的最高分，做如下的规则运算 `scoredPodsMap[pod] = float64(maxCount-count) / float64(maxCount)`

12. NoHitLRU plugin

    当没有prefix命中的时候，选择最后一个使用的pod，这里面有很多的新的kv block