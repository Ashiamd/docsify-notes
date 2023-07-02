# SkyWalking8.7.0源码分析-学习笔记

> B站视频教程：[SkyWalking8.7.0源码分析_哔哩哔哩_bilibili](https://www.bilibili.com/video/BV1dy4y1V7ck/?spm_id_from=333.788.recommend_more_video.3&vd_source=ba4d176271299cb334816d3c4cbc885f)
>
> 教程发布者UP主分享的思维导图：[SkyWalking 源码分析-ProcessOn](https://www.processon.com/view/link/611fc4c85653bb6788db4039)

# 1. SkyWalking premain入口

> 我这里下载 [**https://dlcdn.apache.org/skywalking/java-agent/8.16.0/apache-skywalking-java-agent-8.16.0-src.tgz**](https://dlcdn.apache.org/skywalking/java-agent/8.16.0/apache-skywalking-java-agent-8.16.0-src.tgz
>
> 对应apache skywalking官网的Java Agent部分

SkyWalking premain方法(java agent)主要逻辑分5个步骤：

1. 初始化配置
2. 加载插件
3. 定制Agent（ByteBuddy字节码操作，类似的有ASM）
4. 加载服务
5. 注册关闭钩子

```java
/**
  * Main entrance. Use byte-buddy transform to enhance all classes, which define in plugins.
  */
// 下面这边为方便理解，只保留部分代码逻辑, 顺带删除一些 try-catch-finally块，方便整体阅读
public static void premain(String agentArgs, Instrumentation instrumentation) throws PluginException {
  final PluginFinder pluginFinder;
  // 1. 初始化配置 
  SnifferConfigInitializer.initializeCoreConfig(agentArgs);
 	
  // 2. 加载插件
  pluginFinder = new PluginFinder(new PluginBootstrap().loadPlugins());
	
  // 3. 定制Agent（ByteBuddy字节码操作，类似的有ASM）
  final ByteBuddy byteBuddy = new ByteBuddy().with(TypeValidation.of(Config.Agent.IS_OPEN_DEBUGGING_CLASS));
  AgentBuilder agentBuilder = new AgentBuilder.Default(byteBuddy).ignore(
    nameStartsWith("net.bytebuddy.")
    .or(nameStartsWith("org.slf4j."))
    .or(nameStartsWith("org.groovy."))
    .or(nameContains("javassist"))
    .or(nameContains(".asm."))
    .or(nameContains(".reflectasm."))
    .or(nameStartsWith("sun.reflect"))
    .or(allSkyWalkingAgentExcludeToolkit())
    .or(ElementMatchers.isSynthetic()));
	// ...
  JDK9ModuleExporter.EdgeClasses edgeClasses = new JDK9ModuleExporter.EdgeClasses();
  agentBuilder = BootstrapInstrumentBoost.inject(pluginFinder, instrumentation, agentBuilder, edgeClasses);
  agentBuilder = JDK9ModuleExporter.openReadEdge(instrumentation, agentBuilder, edgeClasses);
  if (Config.Agent.IS_CACHE_ENHANCED_CLASS) {
    try {
      agentBuilder = agentBuilder.with(new CacheableTransformerDecorator(Config.Agent.CLASS_CACHE_MODE));
      LOGGER.info("SkyWalking agent class cache [{}] activated.", Config.Agent.CLASS_CACHE_MODE);
    } catch (Exception e) {
      LOGGER.error(e, "SkyWalking agent can't active class cache.");
    }
  }
  agentBuilder.type(pluginFinder.buildMatch())
    .transform(new Transformer(pluginFinder))
    .with(AgentBuilder.RedefinitionStrategy.RETRANSFORMATION)
    .with(new RedefinitionListener())
    .with(new Listener())
    .installOn(instrumentation);
  PluginFinder.pluginInitCompleted();

	// 4. 加载服务
  ServiceManager.INSTANCE.boot();
	
  // 5. 注册关闭钩子
  Runtime.getRuntime()
    .addShutdownHook(new Thread(ServiceManager.INSTANCE::shutdown, "skywalking service shutdown thread"));
}
```



