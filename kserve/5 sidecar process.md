---
layout: default
title: sidecar workflow
---

## llm-d sidecar

sidecar是部署在decode节点上的，prefill节点里不需要部署，如果需要prefill的时候，sidecar会把请求转发到prefill上

## sidecar 启动过程

首先来看kserver配置的参数

```text
    Args:
      --port=8000
      --vllm-port=8001
      --connector=nixlv2
      --secure-proxy=false
```

sidecar本质上是个代理，代理prefill或者decode节点

```go
func (s *Server) Start(ctx context.Context, cert *tls.Certificate, allowlistValidator *AllowlistValidator) error {
    s.logger = log.FromContext(ctx).WithName("proxy server on port " + s.port)

    s.allowlistValidator = allowlistValidator

    // Configure handlers
    s.handler = s.createRoutes()
```

查看chatCompletionsHandler 这个route 是如何处理的

```go
func (s *Server) chatCompletionsHandler(w http.ResponseWriter, r *http.Request) {
    var prefillHostPorts []string
    // 这个在 prefill-header-handler 这个插件添加的头，不再schedler过程，
    prefillHostPorts = r.Header.Values(common.PrefillPodHeader)

    // https://datatracker.ietf.org/doc/html/rfc7230#section-3.2.2 specifies proxies
    // may combine multiple header values with a comma. Accept either one host per
    // header line OR one line with multiple header values.
    // 默认情况下就是一个，prefill中只选择一个默认
    if len(prefillHostPorts) == 1 {
        prefillHostPorts = strings.Split(prefillHostPorts[0], ",")
    }

    numHosts := len(prefillHostPorts)
    var prefillHostPort string
    if numHosts > 0 {
        if s.config.EnablePrefillerSampling {
            // Sample a host value from the list
            prefillHostPort = strings.TrimSpace(prefillHostPorts[s.prefillSamplerFn(numHosts)])
        } else if numHosts > 0 {
            // Select only the first header value, consistent with previous behavior
            prefillHostPort = strings.TrimSpace(prefillHostPorts[0])
        }
    }

    if len(prefillHostPort) == 0 {
        s.logger.V(4).Info("skip disaggregated prefill")

        if !s.forwardDataParallel || !s.dataParallelHandler(w, r) {
            s.decoderProxy.ServeHTTP(w, r)
        }
        return
    }
    .......

    s.logger.V(4).Info("SSRF protection: prefill target allowed", "target", prefillHostPort)
    // 连接到 prefill节点，请求转发,如果不需要prefill的话，在上一步就转发给decode节点了
    s.runConnectorProtocol(w, r, prefillHostPort)
}
```

如果在上面的调度器中设置了prefill的头部，那么这里就会转发给prefill，否则直接转发给decode节点，接下来查看连接到prefill的节点步骤， 默认连接协议是 runNIXLProtocolV2，代码如下，注释写在代码里

```go
func (s *Server) runNIXLProtocolV2(w http.ResponseWriter, r *http.Request, prefillPodHostPort string) {
    s.logger.V(4).Info("running NIXL protocol V2", "url", prefillPodHostPort)

    // Read request body
    defer r.Body.Close() //nolint:all
    original, err := io.ReadAll(r.Body)
    if err != nil {
        w.WriteHeader(http.StatusBadRequest) // TODO: check FastAPI error code when failing to read body
        w.Write([]byte(err.Error()))         //nolint:all
        return
    }

    // Parse completion request
    var completionRequest map[string]any
    if err := json.Unmarshal(original, &completionRequest); err != nil {
        if err := errorJSONInvalid(err, w); err != nil {
            s.logger.Error(err, "failed to send error response to client")
        }
        return
    }

    // Generate unique request UUID
    uuid, err := uuid.NewUUID()
    if err != nil {
        if err := errorBadGateway(err, w); err != nil {
            s.logger.Error(err, "failed to send error response to client")
        }
        return
    }
    uuidStr := uuid.String()

    // Prefill Stage

    // 1. Prepare prefill request
    ctx := r.Context()
    preq := r.Clone(ctx)
    // 由于request id 在epp的scheduler中被删除，因此这个要重新生成
    preq.Header.Add(requestHeaderRequestID, uuidStr)
    // 获取request中的stream字段
    streamValue, streamOk := completionRequest[requestFieldStream]
    // 获取request中的 stream_opt字段
    streamOptionsValue, streamOptionsOk := completionRequest[requestFieldStreamOptions]
    // 获取request中的max_token字段
    maxTokensValue, maxTokensOk := completionRequest[requestFieldMaxTokens]
    // 获取request中的max_completion_tokens字段
    maxCompletionTokensValue, maxCompletionTokensOk := completionRequest[requestFieldMaxCompletionTokens]
    // 设置kv_transfer_params参数
    completionRequest[requestFieldKVTransferParams] = map[string]any{
        // 设置 do_remote_decode 
        requestFieldDoRemoteDecode:  true,
        // 设置 do_remote_prefill
        requestFieldDoRemotePrefill: false,
        // remote_engine_id
        requestFieldRemoteEngineID:  nil,
        // remote_block_ids
        requestFieldRemoteBlockIDs:  nil,
        requestFieldRemoteHost:      nil,
        requestFieldRemotePort:      nil,
    }
    // 设置request中的stream 为 false
    completionRequest[requestFieldStream] = false
    // 删除request中的stream_options key
    delete(completionRequest, requestFieldStreamOptions)
    // 添加max_token 和 max_completion_tokens token都为1，这样的话，对端只做prefill
    completionRequest[requestFieldMaxTokens] = 1
    completionRequest[requestFieldMaxCompletionTokens] = 1
    // json 序列化
    pbody, err := json.Marshal(completionRequest)
    if err != nil {
        if err := errorJSONInvalid(err, w); err != nil {
            s.logger.Error(err, "failed to send error response to client")
        }
        return
    }
    preq.Body = io.NopCloser(strings.NewReader(string(pbody)))
    preq.ContentLength = int64(len(pbody))
    // 组装好preq之后，准备丢给prefill节点
    //  prefillerproxyhandler是个lru cache里面存了hostport和对应的proxy
    prefillHandler, err := s.prefillerProxyHandler(prefillPodHostPort)
    if err != nil {
        if err := errorBadGateway(err, w); err != nil {
            s.logger.Error(err, "failed to send error response to client")
        }
        return
    }

    // 2. Forward request to prefiller
    s.logger.V(4).Info("sending prefill request", "to", prefillPodHostPort)
    s.logger.V(5).Info("Prefill request", "body", string(pbody))
    pw := &bufferedResponseWriter{}
    // 发起请求并且把结果放在pw中
    prefillHandler.ServeHTTP(pw, preq)

    if pw.statusCode < 200 || pw.statusCode >= 300 {
        s.logger.Error(err, "request failed", "code", pw.statusCode)
        w.WriteHeader(pw.statusCode)
        return
    }

    // Process response - extract p/d fields
    var prefillerResponse map[string]any
    if err := json.Unmarshal([]byte(pw.buffer.String()), &prefillerResponse); err != nil {
        if err := errorJSONInvalid(err, w); err != nil {
            s.logger.Error(err, "failed to send error response to client")
        }
        return
    }

    // 3. Verify response
    // 从response中，获取 kv_transfer_params 
    pKVTransferParams, ok := prefillerResponse[requestFieldKVTransferParams]
    if !ok {
        s.logger.Info("warning: missing 'kv_transfer_params' field in prefiller response")
    }

    s.logger.V(5).Info("received prefiller response", requestFieldKVTransferParams, pKVTransferParams)

    // Decode Stage

    // 1. Prepare decode request
    // clone原始请求
    dreq := r.Clone(ctx)
    // 添加request-id
    dreq.Header.Add(requestHeaderRequestID, uuidStr)
    // 删除stream字段
    delete(completionRequest, requestFieldStream)
    // 如果一开始配置了stream的话，那么还是用之前的value
    if streamOk {
        completionRequest[requestFieldStream] = streamValue
    }
    if streamOptionsOk {
        completionRequest[requestFieldStreamOptions] = streamOptionsValue
    }
    delete(completionRequest, requestFieldMaxTokens)
    // 恢复之前的值
    if maxTokensOk {
        completionRequest[requestFieldMaxTokens] = maxTokensValue
    }
    // 恢复之前的设置值
    delete(completionRequest, requestFieldMaxCompletionTokens)
    if maxCompletionTokensOk {
        completionRequest[requestFieldMaxCompletionTokens] = maxCompletionTokensValue
    }
    // 使用reponse中返回的 kv_transfer_params 参数
    completionRequest[requestFieldKVTransferParams] = pKVTransferParams

    dbody, err := json.Marshal(completionRequest)
    if err != nil {
        if err := errorJSONInvalid(err, w); err != nil {
            s.logger.Error(err, "failed to send error response to client")
        }
        return
    }
    dreq.Body = io.NopCloser(strings.NewReader(string(dbody)))
    dreq.ContentLength = int64(len(dbody))

    // 2. Forward to local decoder.
    // 发送到decode节点上
    s.logger.V(5).Info("sending request to decoder", "body", string(dbody))
    if !s.forwardDataParallel || !s.dataParallelHandler(w, dreq) {
        s.logger.V(4).Info("sending request to decoder", "to", s.decoderURL.Host)
        s.decoderProxy.ServeHTTP(w, dreq)
    }
}
```