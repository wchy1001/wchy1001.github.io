# EPP 加载过程

EPP这里指的是scheduler， 要介绍kserve 生成的EPP 配置文件，以及EPP如何加载这些配置文件的。当请求进入到epp之后，epp是如何根据配置的插件来挑选最终的pod的呢？

## Kserve 默认的配置文件

kserver生成的epp 配置文件如下

```yaml
apiVersion: inference.networking.x-k8s.io/v1alpha1
kind: EndpointPickerConfig
plugins:
- type: pd-profile-handler
parameters:
    threshold: 100
- type: prefill-header-handler
- type: prefill-filter
- type: decode-filter
- type: prefix-cache-scorer
- type: load-aware-scorer
- type: max-score-picker
schedulingProfiles:
- name: prefill
plugins:
- pluginRef: prefill-filter
- pluginRef: prefix-cache-scorer
    weight: 2.0
- pluginRef: load-aware-scorer
    weight: 1.0
- pluginRef: max-score-picker
- name: decode
plugins:
- pluginRef: decode-filter
- pluginRef: prefix-cache-scorer
    weight: 2.0
- pluginRef: load-aware-scorer
    weight: 1.0
- pluginRef: max-score-picker
```

## EPP 加载过程

```go
func main() {
    // Register llm-d-inference-scheduler plugins
    plugins.RegisterAllPlugins()

    if err := runner.NewRunner().
        WithCustomCollectors(metrics.GetCollectors()...).
        Run(ctrl.SetupSignalHandler()); err != nil {
        os.Exit(1)
    }
}
```

上一步注册了所有的插件，接下来开始运行Run 初始化函数，这些函数代码的主要 是在 [gateway-api-inference-extension](https://github.com/kubernetes-sigs/gateway-api-inference-extension)中实现的

### 加载runner

代码在[这里](https://github.com/kubernetes-sigs/gateway-api-inference-extension/blob/main/cmd/epp/runner/runner.go#L150)

runner中的Run函数的加载，首先处理log handler 以及 flag的解析，接下来开始加载epp的配置文件

### 1.第一阶段解析配置文件

```go
    ........
    // 处理配置文件，把string转换成config对象，并且注册内部插件
    rawConfig, err := r.parseConfigurationPhaseOne(ctx)
    ........
```

这个过程中，会注册InTreeplugins，也就是默认的插件，以及featuresGate 对象，之后把配置文件从bytes转换成EndpointPickerConfig对象，也就是配置文件的struct，这时候如果没有配置SaturationDetector，那个这个阶段会自动生成一个

```go
    if cfg.SaturationDetector == nil {
        cfg.SaturationDetector = &configapi.SaturationDetector{}
    }
    if cfg.SaturationDetector.QueueDepthThreshold == 0 {
        cfg.SaturationDetector.QueueDepthThreshold = saturationdetector.DefaultQueueDepthThreshold
    }
    if cfg.SaturationDetector.KVCacheUtilThreshold == 0.0 {
        cfg.SaturationDetector.KVCacheUtilThreshold = saturationdetector.DefaultKVCacheUtilThreshold
    }
    if cfg.SaturationDetector.MetricsStalenessThreshold.Duration == 0.0 {
        cfg.SaturationDetector.MetricsStalenessThreshold =
            metav1.Duration{Duration: saturationdetector.DefaultMetricsStalenessThreshold}
    }
```

SaturationDetector 是什么？ 饱和检测，是用来检测后端的vllm系统中是否还可以接受新的请求，主要通过判断如下几个指标： waiting queue 有没有超过了阈值，kvcache的使用率是否超过阈值，以及默认的的stale指标，如果stale指标太久了，那么也会认为不可调度。

### 2.加载metrics collection

metrics collection 是用来收集pods指标的，可以来判断pods的状态

```go
epf, err := r.setupMetricsCollection(setupLog, r.featureGates[datalayer.FeatureGate])
```

如果没有启动experiment 的数据采集的话，默认用的是metrics指标，如果用其他的插件收集数据也可以，结果放到podsmetrics(包括pods和指标的信息)中即可

```go
func setupMetricsV1(setupLog logr.Logger) (datalayer.EndpointFactory, error) {
    // 把vllm的指标映射成为别的名字，方便后续处理
    mapping, err := backendmetrics.NewMetricMapping(
        *totalQueuedRequestsMetric,
        *kvCacheUsagePercentageMetric,
        *loraInfoMetric,
        *cacheInfoMetric,
    )
......
    var metricsHttpClient *http.Client
......
        metricsHttpClient = http.DefaultClient
    }
    // 实现了 获取指标和停止接口，其中获取指标是通过 PodMetricsClientImpl 这个来实现，其中的fetch metrics，调用backendmetrics这个来获取
    pmf := backendmetrics.NewPodMetricsFactory(&backendmetrics.PodMetricsClientImpl{
        MetricMapping:            mapping,
        ModelServerMetricsPath:   *modelServerMetricsPath,
        ModelServerMetricsScheme: *modelServerMetricsScheme,
        Client:                   metricsHttpClient,
    },
        *refreshMetricsInterval)
    return pmf, nil
```

首先把vllm metrics中的指标 映射成自定义的名字方便后续使用（比如cache_info metrics或者lora相关的），最后返回pmd 也就是pod metrics factory，pmd 需要实现 NewEndpoint 接口和 ReleaseEndpoint接口，NewEndpoint 主要是用来开启 定期的采集指标任务，也就是通过http请求/metrics接口，然后把结果存入到podsmetrics中，这样后续可以随时取到pods的数据

### 3. 设置datastore

datastore是epp的核心存储功能，存储了，pods，pool，还有其他的objective，相当于epp的数据库了， poolGet的用来get pool的信息，这里的pool指的是inferencepool CR。需要实现如下几个接口

```go
type Datastore interface {
    // InferencePool operations
    // PoolSet sets the given pool in datastore. If the given pool has different label selector than the previous pool
    // that was stored, the function triggers a resync of the pods to keep the datastore updated. If the given pool
    // is nil, this call triggers the datastore.Clear() function.
    PoolSet(ctx context.Context, reader client.Reader, endpointPool *datalayer.EndpointPool) error
    PoolGet() (*datalayer.EndpointPool, error)
    PoolHasSynced() bool
    PoolLabelsMatch(podLabels map[string]string) bool

    // InferenceObjective operations
    ObjectiveSet(infObjective *v1alpha2.InferenceObjective)
    ObjectiveGet(objectiveName string) *v1alpha2.InferenceObjective
    ObjectiveDelete(namespacedName types.NamespacedName)
    ObjectiveGetAll() []*v1alpha2.InferenceObjective

    // PodList lists pods matching the given predicate.
    PodList(predicate func(backendmetrics.PodMetrics) bool) []backendmetrics.PodMetrics
    PodUpdateOrAddIfNotExist(pod *corev1.Pod) bool
    PodDelete(podNAme string)

    // Clears the store state, happens when the pool gets deleted.
    Clear()
}
```

在上一步中的采集metrics指标，一旦后续开始被采集，那么都会存在这个datastore中，datastore里面的PodList方法很常用，返回pods和指标的相关信息

### 4.处理配置文件的第二阶段

```go
eppConfig, err := r.parseConfigurationPhaseTwo(ctx, rawConfig, ds)
```

这一步 首先加载plugin handler(epp plugin handler)，用来处理plugin相关的， 初始化所有在 配置文件中配置的plugins,然后添加到handler中，之后开始处理scheduler profile相关内容, profile 是 scheduler最小调度单位，实际上scheduler是根据profile来调度一堆插件来执行的, 对profile主要做如下的配置:

- 首先看profile有没有配置，如果没配置就生成个默认的
- 如果profile 是1个，检查plugins里面有没有 profilehandler，没有的话就加一个，
- 在配置的plugin中，检查有一个picker plugin，这个类型的plugin是用来挑选最终的结果的，默认就是maxScorePicker，筛选最多分数，没有就加一个
- 接下来开始检查每一个profile里面的plugins配置，score 类型的plugins 是不是没有配置weight，没配置就用默认值：1， 查看profile的plugins有没有picker，没有的话就加一个maxScorePicker

profile会添加配置文件里面的plugin，在添加的时候，就会分类好,filter类插件，scorer类插件，还有picker类插件

```go
func (p *SchedulerProfile) AddPlugins(pluginObjects ...plugins.Plugin) error {
    for _, plugin := range pluginObjects {
        if weightedScorer, ok := plugin.(*WeightedScorer); ok {
            p.scorers = append(p.scorers, weightedScorer)
            plugin = weightedScorer.Scorer // if we got WeightedScorer, unwrap the plugin
        } else if scorer, ok := plugin.(Scorer); ok { // if we got a Scorer instead of WeightedScorer that's an error.
            return fmt.Errorf("failed to register scorer '%s' without a weight. follow function documentation to register a scorer", scorer.TypedName())
        }
        if filter, ok := plugin.(Filter); ok {
            p.filters = append(p.filters, filter)
        }
        if picker, ok := plugin.(Picker); ok {
            if p.picker != nil {
                return fmt.Errorf("failed to set '%s' as picker, already have a registered picker plugin '%s'", picker.TypedName(), p.picker.TypedName())
            }
            p.picker = picker
        }
    }
    return nil
}
```

下面开始处理schedulercfg 用来 后续初始化scheduler使用

```go
 // 返回schedulerconfig 包括profiles 和 profilehandler
    config.SchedulerConfig, err = loadSchedulerConfig(rawConfig.SchedulingProfiles, handle)
    if err != nil {
        return nil, err
    }
    // 设置饱和检测的配置
    config.SaturationDetectorConfig = loadSaturationDetectorConfig(rawConfig.SaturationDetector)
```

scedulerconfig 包括两个字段，一个是 profiles的handler，一个是profiles的内容，profile handler是后续用来挑选执行哪个profile使用的，主要的接口如下

```go
type ProfileHandler interface {
    plugins.Plugin
    // Pick selects the SchedulingProfiles to run from a list of candidate profiles, while taking into consideration the request properties
    // and the previously executed SchedluderProfile cycles along with their results.
    Pick(ctx context.Context, cycleState *types.CycleState, request *types.LLMRequest, profiles map[string]*SchedulerProfile,
        profileResults map[string]*types.ProfileRunResult) map[string]*SchedulerProfile

    // ProcessResults handles the outcome of the profile runs after all profiles ran.
    // It may aggregate results, log test profile outputs, or apply custom logic. It specifies in the SchedulingResult the
    // key of the primary profile that should be used to get the request selected destination.
    // When a profile run fails, its result in the profileResults map is nil.
    ProcessResults(ctx context.Context, cycleState *types.CycleState, request *types.LLMRequest,
        profileResults map[string]*types.ProfileRunResult) (*types.SchedulingResult, error)
}
```

Pick用来挑选哪个profile用来执行，processResult 用来汇总多个profile结束后的结果，比如多个profile都筛选出了不同的pods,那么需要这个函数来处理。

profile主要是包括了插件列表，filters类型的插件，scorer 类型的插件，以及picker类型的插件 最后构建了结构体就是:

```go
scedulerconfig{
    profilehandler 实现了ProfileHandler接口的plugin
    profiels: 一个列表，包括了profile，每个profile都保存了 []filter plugins,[]score plugins，以及picker plugin
```

然后设置饱和检测的配置，之后要对plugins进行分类: plugins分两类：

- 第一类: 是实现了，PreRequest，responsereceived, responseStreaming, ResponseComplete 或者PrepareDataplugin  这种类型的插件叫做request controller 插件，他们只会作用在request请求上，而不参与调度，比如修改request中的头部内容,
- 第二类： 实现filter, score, pick类型的插件，这类是参与调度循环的，属于scheduler管理

### 5.设置 epp 服务自身的 metrics

添加自定义的collect，如： InferencePoolMetricsCollector， 收集inferencepool相关的指标，主要指标是：inference_pool_per_pod_queue_size，inferencepool 对应的pods，每个排队情况，最后设置epp本身的metrics 认证方式，随后传入到后续的manager中。 注意这个metrics不是收集pods，而是epp服务本身的metrics，比如 epp 在整个调度中，消耗多少时间，最终都可以从这里获取到。

### 6. 初始化controller manager

metrics的相关配置，以及leader配置等信息传入到manager，开始初始化manager object，之后运行leader 选举（如果enable的话）

### 7.初始化scheduler以及saturationDector

```go
    // 初始化scheduler，其中的Schedule方法为核心调度算法，在下文会调用到
    scheduler := scheduling.NewSchedulerWithConfig(r.schedulerConfig)
    // 初始化 detector， 用来检测候选pods中是否 至少有一个满足 在 kv使用率，排队 等信息都满足的情况
    saturationDetector := saturationdetector.NewDetector(eppConfig.SaturationDetectorConfig, setupLog)
    .......
    admissionController = requestcontrol.NewLegacyAdmissionController(saturationDetector)
```

用上文的schedulerConfig 初始化scheduler对象，这个是个重要的结构体，后续会再次用到。然后使用saturationDetector创建admissionController ，如果系统已经满了的话，就拒绝掉请求.

### 8.初始化director

```go
    director := requestcontrol.NewDirectorWithConfig(
        ds,
        scheduler,
        admissionController,
        r.requestControlConfig)
```

这个资源对象也非常重要，会在后续的ext-proc server中被调度到，director中的HandleRequest 会拦截envoy的request请求，并由它调度 request 类型的plugins来 处理 request，调度scheduler 选择pods等业务逻辑

### 9.注册manager

```go
    serverRunner := &runserver.ExtProcServerRunner{
        GrpcPort:                         *grpcPort,
        GKNN:                             *gknn,
        Datastore:                        ds,
        DisableK8sCrdReconcile:           disableK8sCrdReconcile,
        SecureServing:                    *secureServing,
        HealthChecking:                   *healthChecking,
        CertPath:                         *certPath,
        RefreshPrometheusMetricsInterval: *refreshPrometheusMetricsInterval,
        MetricsStalenessThreshold:        *metricsStalenessThreshold,
        Director:                         director,
        SaturationDetector:               saturationDetector,
        UseExperimentalDatalayerV2:       r.featureGates[datalayer.FeatureGate], // pluggable data layer feature flag
    }
    if err := serverRunner.SetupWithManager(ctx, mgr); err != nil {
        setupLog.Error(err, "Failed to setup EPP controllers")
        return err
    }
```

这里主要注册了3个controller，inferencepool， InferenceObjectiveReconciler（本地环境没有配置这个），还有pods controller。其中inferencepool如果更新，会调用datestore的poolset方法，这个方法会触发pods的全量metrics收集和更新，收集来源是通过inferencepool中的label设置的来收集，只收集符合labels的pods，pods controller 则检测是不是需要从datastore中删除和更新，如果从datastore中获取pods等信息 而没有获取到，那么也会触发metrics收集的定时任务，后续就会定时收集

### 10.继续注册其他servers,healthcheck & ext-proc

```go
    // --- Add Runnables to Manager ---
    // Register health server.
    if err := registerHealthServer(mgr, ctrl.Log.WithName("health"), ds, *grpcHealthPort, isLeader, *haEnableLeaderElection); err != nil {
        return err
    }

    // Register ext-proc server.
    if err := registerExtProcServer(mgr, serverRunner, ctrl.Log.WithName("ext-proc")); err != nil {
        return err
    }
```

其中最主要的是 registerExtProcServer，查看rannable的定义如下

```go
func (r *ExtProcServerRunner) AsRunnable(logger logr.Logger) manager.Runnable {
    return runnable.NoLeaderElection(manager.RunnableFunc(func(ctx context.Context) error {
            // 定期刷新指标，刷新metrics指标，或者打印log用来debug
            backendmetrics.StartMetricsLogger(ctx, r.Datastore, r.RefreshPrometheusMetricsInterval, r.MetricsStalenessThreshold)
        .......
            srv = grpc.NewServer()
        //tProServer 注册envory 外部扩展，需要实现其中的process方法，核心处理逻辑，其中调用Directores.HandleRequest 最为重要，处理request的不同阶段
        extProcServer := handlers.NewStreamingServer(r.Datastore, r.Director)
        extProcPb.RegisterExternalProcessorServer(srv, extProcServer)
```

StremingServer需要实现 process接口，详情查看 envoy的[proto文件](https://www.envoyproxy.io/docs/envoy/latest/api-v3/service/ext_proc/v3/external_processor.proto)，StreamingServer中的process太长，其中大概处理就是如果是get等请求，没有body，则随机路由到一个vllm pod上，如果有request body，则调用director中的handlereqeust

```go
reqCtx, err = s.director.HandleRequest(ctx, reqCtx)
```

接下来重点查看 这个 方法，这个director在上文已经初始化了：大概执行如下：

```go
// Get candidate pods for scheduling
    // 获取inferencepool下面的label的pods，包括了worker节点和prefill节点
    candidatePods := d.getCandidatePodsForScheduling(ctx, reqCtx.Request.Metadata)
    if len(candidatePods) == 0 {
        return reqCtx, errutil.Error{Code: errutil.ServiceUnavailable, Msg: "failed to find candidate pods for serving the request"}
    }
    // 查看系统是否饱和，优先级小于0并且系统饱和的话，直接拒绝掉
    if err := d.admissionController.Admit(ctx, reqCtx, candidatePods, *infObjective.Spec.Priority); err != nil {
        logger.V(logutil.DEFAULT).Info("Request rejected by admission control", "error", err)
        return reqCtx, err
    }
    // 转换了类型，拷贝了当前的值
    snapshotOfCandidatePods := d.toSchedulerPodMetrics(candidatePods)

    // Prepare per request data by running PrepareData plugins.
    // 执行prerequest的plugin，传入了request 和 pods参数
    if d.runPrepareDataPlugins(ctx, reqCtx.SchedulingRequest, snapshotOfCandidatePods) != nil {
        // Don't fail the request if PrepareData plugins fail.
        logger.V(logutil.DEFAULT).Error(err, "failed to prepare per request data")
    }

    // Run admit request plugins
    // 执行admission plugins， 是否允许通过
    if !d.runAdmissionPlugins(ctx, reqCtx.SchedulingRequest, snapshotOfCandidatePods) {
        logger.V(logutil.DEFAULT).Info("Request cannot be admitted")
        return reqCtx, errutil.Error{Code: errutil.Internal, Msg: "request cannot be admitted"}
    }
    // 接下来执行 scheduler 插件
    result, err := d.scheduler.Schedule(ctx, reqCtx.SchedulingRequest, snapshotOfCandidatePods)
    if err != nil {
        return reqCtx, errutil.Error{Code: errutil.InferencePoolResourceExhausted, Msg: fmt.Errorf("failed to find target pod: %w", err).Error()}
    }

    // Prepare Request (Populates RequestContext and call PreRequest plugins)
    // Insert target endpoint to instruct Envoy to route requests to the specified target pod and attach the port number.
    // Invoke PreRequest registered plugins.
    reqCtx, err = d.prepareRequest(ctx, reqCtx, result)
    if err != nil {
        return reqCtx, err
    }
```

获取候选pods，这些pods是通过inferencepool中配置的labels，筛选出来的，然后判断系统是否饱和，看看是不是要拒绝掉请求，然后执行 preparedata plugin，这种类型的插件，在环境中，并没有配置，接下来执行admissionplugins，然后最核心的部分是执行scheduler 插件，最后执行preparerequest插件

```go
  result, err := d.scheduler.Schedule(ctx, reqCtx.SchedulingRequest, snapshotOfCandidatePods)
```

接下来查看schedule是如何调度的，

```go
    for { // get the next set of profiles to run iteratively based on the request and the previous execution results
        loggerVerbose.Info("Running profile handler, Pick profiles", "plugin", s.profileHandler.TypedName())
        before := time.Now()
        // 选择一个profile去运行，cycleState保存了plugins之间的共享数据，profileRunResult 返回了 一组 pods， 由于是for,因此会一直选
        profiles := s.profileHandler.Pick(ctx, cycleState, request, s.profiles, profileRunResults)
        // 记录延迟
        metrics.RecordPluginProcessingLatency(framework.ProfilePickerExtensionPoint, s.profileHandler.TypedName().Type, s.profileHandler.TypedName().Name, time.Since(before))
        loggerVerbose.Info("Completed running profile handler Pick profiles successfully", "plugin", s.profileHandler.TypedName(), "result", profiles)
        if len(profiles) == 0 { // profile picker didn't pick any profile to run
            break
        }
        // 从一组profiles中开始运行，每个profiles 都运行run方法
        for name, profile := range profiles {
            loggerVerbose.Info("Running scheduler profile", "profile", name)
            // run the selected profiles and collect results (current code runs all profiles)
            profileRunResult, err := profile.Run(ctx, request, cycleState, candidatePods)
            if err != nil {
                loggerVerbose.Info("failed to run scheduler profile", "profile", name, "error", err.Error())
            } else {
                loggerVerbose.Info("Completed running scheduler profile succuessfully", "profile", name)
            }
            // filter， scorer，pick 运行完成之后， 结果报错了在profileRunResults，其中是一组targets pods
            profileRunResults[name] = profileRunResult // if profile failed to run, the run result is nil
        }
    }
```

通过profilehandler pick中挑选一个profile开始运行，然后记录指标，开始运行 profile.Run 函数

```go
func (p *SchedulerProfile) Run(ctx context.Context, request *types.LLMRequest, cycleState *types.CycleState, candidatePods []types.Pod) (*types.ProfileRunResult, error) {
    // 一次运行filters plugins
    pods := p.runFilterPlugins(ctx, request, cycleState, candidatePods)
    if len(pods) == 0 {
        return nil, errutil.Error{Code: errutil.Internal, Msg: "no pods available for the given request"}
    }
    // if we got here, there is at least one pod to score
    // 运行打分 plugins
    weightedScorePerPod := p.runScorerPlugins(ctx, request, cycleState, pods)

    // 运行pick plugin
    result := p.runPickerPlugin(ctx, cycleState, weightedScorePerPod)

    return result, nil
}
```

调度过程是： 先运行filters plugins，然后运行打分plugins，然后进行score插件的分数加权计算，最后运行picker 插件来挑选最终的一个或多个（通过配置控制）
当所有profile全部执行后，执行profilehandler中的ProcessResults方法，用来合并数据或者打印测试等

当scheduler插件运行之后，最后还需要运行prepareRequest插件，这个插件处理request的相关信息，比如在PD分离设置成功之后，那么需要在header中设置 prefill的host-ip等信息

## EPP 筛选过程

根据最开始的配置文件，来看看这些配置是如何实现pd分离的

### scheduler 选择顺序

先从plugin中找到实现Pick接口的plugin，这个插件用来挑选下面的profiles来执行，有且只有一个plugin实现profile Pick接口，其他的不应该存在，本例中是 pd-profile-handler，参数 threshold是100，那么首先执行这个，来找profile，这个plugin在一开始已经外部注册了。这个pick plugin执行了如下功能：

1.首先查看 decodeprofile 也就是 名字是 decode的profile有没有执行过，如果没执行直接返回 decode的profile， 让scheduler先执行，如果执行了继续下面的

2.判断threshold值，如果大于0，那么就执行pd分离

3.执行pd分离的时候，先计算前缀命中率

- 计算过程中，首先从上一轮的decode执行的结果选择一个key，大概是prefix-cache-scorer/prefix-cache-scorer这个样子的，查看这个key的状态，然后从profileResults中获取decodeprofiles 执行的结果，并且选择第一个targetpods，因为profiles执行后，经历过picker选择，那么第一个一定是最高分的pods
- 然后通过上一步中 'prefix-cache/scorer'拿到的状态，，开始匹配出多少个hash block，hash block默认是第一个block是model name 因此还要去掉第一个hash block  然后 和 0 做比较（后文有写）
- 接下来计算百分比，通过 命中的 block数量* hashBlockSize / 用户的input长度，得出百分比
- 如果 (1 - prefixpercent) * input_lenght  也就是未命中的百分比 和 threshold比较，如果低于配置的百分比，那么直接就不要执行 prefill profile了，而是直接运行 decoder
- 如果大于配置的阈值，则返回prefill profile 在scheduler继续选择，
- 最后通过metrics 记录一下 是否pd 分离了

### decode profile执行

上文选择了decode profile执行，接下来的看一下 decode profile是如何执行，首先查看kserver配置decoder profile里面的plugin

```yaml
      - name: decode
        plugins:
        - pluginRef: decode-filter
        - pluginRef: prefix-cache-scorer
          weight: 2.0
        - pluginRef: load-aware-scorer
          weight: 1.0
        - pluginRef: max-score-picker

```

共包括了 4个插件，一个 filters 插件，2个 score 插件 一个picker 插件

#### decoder-filter

实现很简单，从targetPods中返回符合标签的，标签是根据llm-d.ai/role来筛选出 值是both 或者 decode的角色 或者没有标签的

在个人的实验环境中是：

```bash
#筛选的是所有的候选pods，这三个标签是inferencepool中配置的
root@worker-1:~# kubectl  get pods -l app.kubernetes.io/name=small-llm-pd-single-gpu,app.kubernetes.io/part-of=llminferenceservice,kserve.io/component=workload
NAME                                                      READY   STATUS    RESTARTS   AGE
small-llm-pd-single-gpu-kserve-7dfd57c8bf-559hz           2/2     Running   0          84d
small-llm-pd-single-gpu-kserve-7dfd57c8bf-fzjm2           2/2     Running   0          84d
small-llm-pd-single-gpu-kserve-prefill-559cfd8df9-9ng4n   1/1     Running   0          84d
small-llm-pd-single-gpu-kserve-prefill-559cfd8df9-q8rcf   1/1     Running   0          84d
root@worker-1:~#
# 筛选出role是decode的插件，在基础上继续过滤decode标签，由于我的环境是pd分离，因此role是decode而不是both
root@worker-1:~# kubectl  get pods -l app.kubernetes.io/name=small-llm-pd-single-gpu,app.kubernetes.io/part-of=llminferenceservice,kserve.io/component=workload,llm-d.ai/role=decode
NAME                                              READY   STATUS    RESTARTS   AGE
small-llm-pd-single-gpu-kserve-7dfd57c8bf-559hz   2/2     Running   0          84d
small-llm-pd-single-gpu-kserve-7dfd57c8bf-fzjm2   2/2     Running   0          84d
root@worker-1:~#
```

#### prefix-cache-score插件

这个插件是GIE中默认注册的插件，而不是llm-scheduler实现的plugin，这个插件一共实现两个接口，1个是Scorer，一个是PreRequest，scorer先执行，然后在执行prerequest

这里先看score 插件，来看这个插件的工作过程：

- 首先给prompt做hash， 如果prompt太小，小于block size 直接返回，如果太大，则裁剪prompt 去掉超出最大值的部分，然后用input/cacheblocksize 得出 hash快的数量，然后在第一个快 插入model name的hash，也就是为什么上一步要减去第一个block，之后在slice中存储了prompts的hash block

- 初始化sate对象，后续给prefill阶段使用，里面存入了，prefixhashes 也就是刚刚计算的结果，还有每个server cache了多少个 hashes，这个是通过从plugin index里面的存取的，那么plugin index是如何实现的呢?

```go
// An indexer maintains an LRU cache of prompt prefix hashes and the server(s) that might have that
// prefix cached.
type indexer struct {
    mu             sync.RWMutex
    hashToPods     map[BlockHash]podSet                         // the lookup data structure to find pods that have the BlockHash cached
    podToLRU       map[ServerID]*lru.Cache[BlockHash, struct{}] // key is pod namespacedName, value is an LRU cache
    defaultLRUSize int
}
```

index会定时的发送lru cache的指标，其中指标的名字是 prefix_indexer_size， 值是所有pods的总共缓存条目，最后打印一些用于debug的log，用于调试很有用

```go
        log.FromContext(ctx).V(logutil.TRACE).Info("Prefix cache state",
            "total entries", totalEntries,
            "# pods", numPods,
            "avg entries per pod", avg,
            "pod with max cache", maxPodName,
            "max pod size", maxPodEntries,
            "global max LRU cache capacity per pod", i.defaultLRUSize,
        )
```

indexer中的数据是在prerequest的过程中调用存储（下文），而在score过程中（当前阶段）用来查询，先执行score，也就是第一次request，那么缓存应该是空的，从 index的get方法中获取blockhash 那么也应该返回空的pod

之后通过 index 中获取 包括hashblock的servers,通过调用plugin中的matchLongestPrefix方法，这个从index去搜寻，然后返回每个server-id中 最长匹配了多少个block，最终初始化了如下的struct

```go
    state := &SchedulingContextState{
        PrefixHashes:       hashes,
        PrefixCacheServers: p.matchLongestPrefix(ctx, hashes),
    }
```

这种结构体会在plugins中传递，第一个prefixhashes 就是一个slice，index 0 是 model name，其余的是prompt 根据blocksize 切分后的对应的hash值， prefixcacheservers 是 一个map 结构是map[server-id]int, 这里的int 是指 这个server-id 匹配了多少个prefix hashes，越大的话，那么说明这个pod里面缓存了更多的hashes，当第一个请求 走到这个里面的时候，应该都是空的，等这个请求执行完，才能有缓存。

之后把state 保存到 cyclestate 中，这个struct 是在各个plugins中相互传递的，key的话是plugin的typename，比如‘prefix-cache-scorer/prefix-cache-scorer’，之后开始打分，打分的规则是 `每个pod 匹配的 hashblock 长度 / 总的prompt hash长度`, 最后存在score中，是 个map{pods: 分数}， 至此打分完成， 在打分完成之后，还是执行其他的score和picker plugin，但是这里直接介绍这个plugin的prerequest阶段的处理过程，这个步骤要等到所有plugins处理完成才会调用到

#### prerequest阶段

上面的score执行完成之后，score里面已经有pod和其分数了，而且hash blockes也都存在公共变量里面了，接下来处理prereequest，先拿到上文中的调度结果，从主profile（也就是decode profile）运行结果中，挑选第一个pod，然后从pods的指标中获取CacheNumGPUBlocks  也就是调用vllm中的cache_config_info里面的num_gpu_blocks指标，如果配置了自动调整每个pods的gpublocks，这时候按照metrics中的指标大小设计lru的大小，否则就是默认值31250，这个默认值的设计很有意思，可以查看注释说明，其中的gpublocks是一个lru 结构体的大小，设定了大小之后，如果后续满了的话，就会按照lru算法替换新的数据。默认情况下不会使用这个默认值，而是动态的从vllm节点的metrics中获取。

```go
    // The indexer is an approximation to the actual prefix LRU cache state on the model servers per server (pod).
    // A small capacity ensures a high accuracy of cache hit on the model server, but it will
    // increase the chance of false negatives. A high capacity does the opposite.
    // To properly size this, consider the sum of the total number of cache entries on all model
    // servers. Consider the llama3 8B model on a H100 80GB GPUs. The size of the model weight is
    // about 16GB. The remaining HBM used for caching prefixes is 64GB. Each
    // token is about 128KB in size, so we can cache 500K tokens. Using the default block size of 16
    // in vLLM, we will have 250K / 16 = 31.25K blocks.
    DefaultLRUCapacityPerServer = 31250
```

之后取出score过程中的state内容，然后把hashes block 添加到 index下面的podstoLRU数据结构中，类型于：

index中的podstoLRU结构体里面包括了server_id 和 该server的lrucaches，lrucache 内容大概是是{hashes_digest: struct{}}，记录hashes存在这个server的cache中就可以，那么等到下一次请求在来的时候，在上面的score过程中就可以读到这里面的内容了，最后计算一下当前一个pods中的match len，和 server的blocksize，然后记录到prefix_indexer_hit_bytes 这个promtheus指标中，

#### load-aware-scorer 插件

load-aware scorer插件是lld-m外部注册的，并不是intree指标，该指标实现了一个score接口，因此可以认为是 score 插件，机制比较简单

```go
// Score scores the given pod in range of 0-1
// Currently metrics contains number of requests waiting in the queue, there is no information about number of requests
// that can be processed in the given pod immediately.
// Pod with empty waiting requests queue is scored with 0.5
// Pod with requests in the queue will get score between 0.5 and 0.
// Score 0 will get pod with number of requests in the queue equal to the threshold used in load-based filter
// In the future, pods with additional capacity will get score higher than 0.5
func (s *LoadAware) Score(_ context.Context, _ *types.CycleState, _ *types.LLMRequest, pods []types.Pod) map[types.Pod]float64 {
    scoredPods := make(map[types.Pod]float64)

    for _, pod := range pods {
        waitingRequests := float64(pod.GetMetrics().WaitingQueueSize)

        if waitingRequests == 0 {
            scoredPods[pod] = 0.5
        } else {
            if waitingRequests > s.queueThreshold {
                waitingRequests = s.queueThreshold
            }
            scoredPods[pod] = 0.5 * (1.0 - (waitingRequests / s.queueThreshold))
        }
    }
    return scoredPods
}
```

算法就是，如果没有请求在waiting，那么得分就是0.5分，如果 大于配置的 阈值，wating_queue 就是用阈值的数，最后用 0.5 * (1.0 - (waitingRequests / s.queueThreshold)) 数学意义是，使用0.5的因子乘以队列的空闲程度，最终把分数控制在 0-0.5之间

#### max-score-picker plugin

该plugin 实现了pick 方法，比较简单，首先把scoredspods 打散，然后在排序，可能会有相等的，设置最大返回pods数量默认是1， 选择分数最高的，如果最大返回2个，就返回2个分最高的，可以配置。
注意在score插件中返回的scorepods 会和weight 在 scheduler中进一步计算的，最后才会把聚合的pods和分数传入到 pick plugin中，因此这里直接返回评分即可

#### decode profile 总结

使用filters插件，过滤出decoder节点，然后根据 prefix cache 进行pods 加权打分，然后和 负载的加权打分相加，聚合出总分，最后放到max-score-picker 中提取最高分

### prefill profile

这个和decode profile几乎相同，无非是过滤的标签不一样，过滤出prefill的节点，依旧是从candidatepods中开始筛选，不同的profile 也共享了cyclestate这个变量

### 最后数据处理

当decode prefill profiles 都执行完成后，那么开始处理最后的数据，也就是profilehandler 接口中的ProcessResults 函数，我们使用的是pd profiles，接下来查看这个函数如何处理

核心逻辑就是判断是否设置了DateParallel并行，在我们PD分离场景中，并没有设置，因此就保存了decode profile 运行结果 和 prefill profile运行的结果，然后设置primary profile 是 decode

当profilehandler处理之后，prefill-header-handler plugin 在prerequest阶段执行，把prefill的信息添加到request的header中

```go
// 添加到http的头部中
targetPod := prefillProfileRunResult.TargetPods[0].GetPod()
prefillHostPort := net.JoinHostPort(targetPod.Address, targetPod.Port)
request.Headers[common.PrefillPodHeader] = prefillHostPort // in the form of <ip:port>
```

## director 处理

director 是拦截 envory请求的，在处理完scheduler之后，开始从结果中选择pods，选择的规则就是：从primay的profile结果中选择所有targetsPods（其实就一个，除非在maxscorepicker中配置了返回多个），最后就把targetpod 和 targetendpoint（IP：port） 保存到reqCtx对象中，返回给 envory。注意这中间的port  是inferencepool中配置的，默认是8000，也就是 llm 的 sidecar 端口，接下来请求进入到sidecar 配置中，查看是如何处理pd分离的