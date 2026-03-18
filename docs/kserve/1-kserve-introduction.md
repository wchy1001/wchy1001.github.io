# 高性能推理框架-Kserve 介绍

Kserve是一个高性能的AI推理框架，看起来是对论文mooncake的工程实践。在新的版本中（2025年下半年开始），Kserve 开始支持 LLMInferenceService CR，相对以往的InferenceService 添加了对生成式模型的高级调度功能，其中包括：PD分离，智能路由等高级功能，因此本文主要介绍的是LLMISvc的资源工作原理，在查看本文之前，去阅读一下[kserve](https://kserve.github.io/website/docs/model-serving/generative-inference/llmisvc/llmisvc-overview)的官方网站将非常有帮助。

Kserve严重依赖LLM-D这个项目，LLM-D这个项目完成了核心的scheduler，路由等功能，参考官网说明

## LLM-D 概览

以下内容完全摘自官网：

llm-d is a Kubernetes-native distributed inference serving stack, providing well-lit paths for anyone to serve large generative AI models at scale, with the fastest time-to-value and competitive performance per dollar for most models across most hardware accelerators.
KServe's generative inference leverages llm-d components to scale and schedule traffic efficiently:

- Router and Scheduler: The router exposes a stable endpoint and uses a scheduler to select the best backend replica based on precise prefix-cache aware routing and customizable scheduling policies to decrease latency and increase throughput.
- Inference Pool: A group of worker pods (for example, vLLM) serving your model. The pool scales independently from the router.
- EndpointPickerConfig: Configures the scheduler with pluggable scoring and picking strategies (for example, prefix-cache-scorer, load-aware-scorer).
- LeaderWorkerSet (LWS): Ensures reliable leadership and coordination for the pool in multi node setup.
- Prefill/Decode Disaggregation: Reduce time to first token (TTFT) and get more predictable time per output token (TPOT) by splitting inference into prefill servers handling prompts and decode servers handling responses, primarily on large models such as Llama-70B and when processing very long prompts.
- Wide Expert-Parallelism: Deploy very large Mixture-of-Experts (MoE) models like DeepSeek-R1 and significantly reduce end-to-end latency and increase throughput by scaling up with Data Parallelism and Expert Parallelism over fast accelerator networks.

总而言之，Kserve是针对LLMisvc CR 创建出符合LLm-D项目要求的一些资源，使LLM-D的配置，简单的抽象成在LLMisvc资源，相当于对LLm-D的使用 做了一层符合k8s风格的包装。而至于 P & D 分离，router，scheduler，inferencepool 等资源 都是LLM-D 项目里面的范畴，其实单纯的使用LLM-D项目也可以完成这些功能

gateway-api-inference-extension是一个specification，定义了 在k8s里面实现AI inference的一些参考，kserve主要是用了 其 定义的两方面内容，一个是scheduler， 一个是AI gateway

kserve中选型的LLM-D scheduler 是对gateway-api-inference-extension scheduler部分的础上扩展了一些新的插件。而选型的AI gateways 则是由 envoy ai gateway 来实现，项目代码分别如下

1. LLM-D 实现了 拓展了 gateway-api-inference-extension EPP，[项目地址](https://github.com/llm-d/llm-d-inference-scheduler)：

2. AI gateway 则是由envory ai gateway来实现，[项目地址](https://github.com/envoyproxy/ai-gateway)

## 总结 kserve中PD分离的工作原理

如果对后续的代码实现没有兴趣，那么直接看结论即可：

该图对inferencepool做了高度的抽象，直接省略了如何挑选vllm后端的过程 ![llm-d 流程图](https://github.com/llm-d/llm-d/raw/main/docs/assets/images/llm-d-arch.svg)

在kserve中创建PD分离的 LLmisvc资源后，对应的controller，会创建出对应的deployment或者leaderworkset工作负载，确保vllm能够开始推理。一个LLmisvc，一般包括如下几个资源，deployment，httproute，inferencepool，scheduler deployment， 而request的请求路径是：首先通过httprouter，然后找到 inferencepool这个backend，inferencepool配置了scheduler的信息，然后把请求转发到scheduler上，scheduler调度后，把请求发送到 decode节点中的sidecar 服务，而sidecar（本质上是个代理）根据请求head中的prefill 等信息，决定要不要先转发请求到 prefill节点，还是直接到decode节点

httproute的配置类似是： 当符合固定前缀，比如 `/default/qwen3` 那么转发到对应inferencepool 这个backend中，而inferencepool 这个资源需要配置在envoy的配置文件中的extension部分中，把inferencepool的资源处理请求转移到 envoy-ai-gateway-service中，而inferencepool中会包括scheduler信息，最后把路由请求转发到scheduler组件中

scheduler组件，主要实现了后端pods的指标采集，kv cache manager， envoy extension的grpc 服务，还有 schedler的核心调度插件的注册和运行。当请求从httprouter来到scheduler之后，envory extension的rpc 服务被调用，开始处理请求，执行reqeust请求处理相关的插件（比如是否需要修改头部，增加特别的header，是否是get接口，压根不需要调度的这种），然后执行scheduler的调度器，依次执行，filters插件，然后score插件，最后执行pick插件 最后根据调度结果，决定是否把host+ip 写到request的header中，最后把请求发送给decode节点中的sidecar服务上

decode pod & prefill pod阶段： 在所有的decode pods中会运行sidecar容器，这个sidecar是一个代理，当scheduler选择最终的decode节点之后，请求来到decode节点中的sidecar服务中，sidecar查看请求头中是否增加了prefill信息，如果增加特定的头部信息（比如： prefill-host： 1.1.1.1:8000） sidecar 开始执行 PD分离流程，大概如下，先暂存当前请求的参数，然后构建prefill的reqeust，设置max_len=1，设置do_remote_decode=true 等参数，设置好之后把请求丢给prefill节点，prefill节点返回之后，从response中 获取 kv_transfer_params 等参数信息，然后根据上面暂存的参数恢复原始的请求，之后把请求在丢给decode节点。