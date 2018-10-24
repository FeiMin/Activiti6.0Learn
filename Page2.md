### 目录
* 前言
* 引擎架构
* 流程引擎初始化源码分析
* 流程启动源码分析
* 节点流转源码分析
### 前言
为了让立下的flag不至于倒下的这么快，于是有了以下的内容。在说以下内容之前先回顾两种设计模式
1. 命令模式：一种以数据为驱动的设计模式。请求以命令的方式包裹在对象中，并传给调用对象。调用对象巡找可以处理该命令的合适的对象，并把命令传递给这个对象，该对象执行命令。主要用于解耦。
2. 职责链模式：避免请求发送者与接收者耦合在一起，让多个对象都有可能接收请求，将这些对象连接成一条链，并且沿着这条链传递请求，直到有对象处理它为止。
### 引擎架构
Activiti基于Spring和Ibatis作为基础平台进行开发，其引擎框架结构如下  
![](https://github.com/FeiMin/Activiti6.0Learn/blob/master/img/QQ%E6%88%AA%E5%9B%BE20181024093114.png)
* ProcessEngine：流程引擎的抽象，对于开发者来说，它是我们使用Activiti的facade，通过它可以获得我们需要的一切服务。
* XXService（TaskService,RuntimeService,RepositoryService...):Activiti按照流程的生命周期（定义，部署，运行）把不同阶段的服务封装在不同的Service中，用户可以非常清晰地使用特定阶段的接口。通过ProcessEngine能够获得这些Service实例。TaskService,RuntimeService,RepositoryService是非常重要的三个Service：
    * TaskService：流程运行过程中，与每个任务节点相关的接口，比如complete,delete,delegate等等
    * RepositoryService:流程定义和部署相关的存储服务。
    * RuntimeService：流程运行时相关服务，如startProcessInstanceByKey.
* CommandContextIntercepter(CommandExecutor):Activiti使用命令模式作为基础开发模式，上面Service中定义的各个方法都对应相应的命令对象（Cmd1...), Service把各种请求委托给Cmd，Cmd来决定命令的接收者，接收者执行后返回结果。而CommandContextIntercepter顾名思义，它是一个拦截器，拦截所有命令，在命令执行前后执行一些公共性操作。比如保存上下文之类的操作，例如  
```
  public <T> T execute(CommandConfig config, Command<T> command) {
    CommandContext context = Context.getCommandContext();

   省略

    try {
      
      // Push on stack（执行之前保存上下文）
      Context.setCommandContext(context);
      Context.setProcessEngineConfiguration(processEngineConfiguration);
      if (processEngineConfiguration.getActiviti5CompatibilityHandler() != null) {
        Context.setActiviti5CompatibilityHandler(processEngineConfiguration.getActiviti5CompatibilityHandler());
      }
        //执行命令
      return next.execute(config, command);

    } catch (Exception e) {

      context.exception(e);
      
    } finally {
      try {
        if (!contextReused) {
        //关闭上下文.同时close方法会通过flushSession来将数据保存到数据库
          context.close();
        }
      } finally {
        //释放上下文资源
        // Pop from stack
        Context.removeCommandContext();
        Context.removeProcessEngineConfiguration();
        Context.removeBpmnOverrideContext();
        Context.removeActiviti5CompatibilityHandler();
      }
    }

    return null;
  }
  ```
而这样设计的好处是前面的xxxService不需要知道具体怎么做，只需要把任务交给命令就可以了。  
下面重点说下这块的设计，命令模式的本质在于将命令进行封装，发出命令和执行命令分离。职责链模式只需要将请求放入职责链上，其处理细节和传递都不需要考虑。activiti将这两个模式整合在一起，构成了其服务主要的实现方式。  
其核心只有三个部分CommandExecutor（命令执行器，用于执行命令），CommandInterceptor（命令拦截器，用于构建拦截器链），Command（命令自身）。这三个接口是整个核心的部分，下面由这三个接口逐步介绍相关的类和具体实现：  
```buildoutcfg
public interface Command<T> {

  T execute(CommandContext commandContext);

}
```
```buildoutcfg
public interface CommandExecutor {

  /**
   * @return the default {@link CommandConfig}, used if none is provided.
   */
  CommandConfig getDefaultConfig();

  /**
   * Execute a command with the specified {@link CommandConfig}.
   */
  <T> T execute(CommandConfig config, Command<T> command);

  /**
   * Execute a command with the default {@link CommandConfig}.
   */
  <T> T execute(Command<T> command);

}
```
```buildoutcfg
public interface CommandInterceptor {

  <T> T execute(CommandConfig config, Command<T> command);

  CommandInterceptor getNext();

  void setNext(CommandInterceptor next);

}
```
Command的接口中只有一个execute方法，这里才是写命令的具体实现,其包含了一个CommandConfig和一个命令拦截器CommandInterceptor，而执行的execute(command)方法，实际上调用的就是commandInterceptor.execute(commandConfig,command)。CommandInterceptor中包含了一个set和get方法，用于设置next(实际上就是下一个CommandInterceptor)变量。想象一下，这样就能够通过这种形式找到拦截器链的下一个拦截器链，就可以将命令传递下去。  
Service实现服务的其中一个标准方法是在具体服务中调用commandExecutor.execute(new command())(这里的command是具体的命令)。其执行步骤就是命令执行器commandExecutor.execute调用了其内部变量CommandInterceptor first（第一个命令拦截器）的execute方法（加上了参数commandConfig）。CommandInterceptor类中包含了一个CommandInterceptor对象next，用于指向下一个CommandInterceptor，在拦截器的execute方法中，只需要完成其对应的相关操作，然后执行一下next.execute(commandConfig,command)，就可以很简单的将命令传递给下一个命令拦截器，然后在最后一个拦截器中执行command.execute()，调用这个命令最终要实现的内容就行了。
* 核心业务：Task、processinstance和execution其实都是其核心的业务对象，以命令模式的思路去理解，这些是真正可以处理命令的实体对象。
* Context上下文组件：上下文组件用于保存在生命周期中的的一些全局性信息，在类的定义中主要保存如下几类信息
    * commandContext：命令上下文，保存每个命令的必要资源，如session
    * processEnginConfigurationContext：引擎的全局配置信息（）,保存引擎dataSource等信息
    * TransactionContext：事务相关？没找到相关实现
```buildoutcfg
protected static ThreadLocal<Stack<CommandContext>> commandContextThreadLocal
protected static ThreadLocal<Stack<ProcessEngineConfigurationImpl>> processEngineConfigurationStackThreadLocal
protected static ThreadLocal<Stack<TransactionContext>> transactionContextThreadLocal
```
* 持久化：Activiti使用ibatis作为ORMapping工具。在此基础之上Activiti设计了自己的持久化框架
* Event-Listener 组件：Activiti允许客户端代码介入流程的执行。为此提供了一个基础组件。用户可以介入的代码类型包括：TaskListener，JavaDelegate，Expression，ExecutionListener
* Cache：没有仔细看，但在Context中看到的情况是通过Map和List做缓存
### 流程引擎初始化源码分析
如图所示为processEngine方法堆栈，由方法名可以看出其生命周期，init和destroy方法。  
![](https://github.com/FeiMin/Activiti6.0Learn/blob/master/img/processEngine%E5%A0%86%E6%A0%88.png)
* init负责加载类路径下的名为activiti.cfg.xml或activiti-context.xml的资源文件，并用俩种不同的方式来初始化资源文件中所配置的流程引擎  
    * 通过initProcessEngineFromResource私有化方法，调用关系为initProcessEngineFromResource->buildProcessEngine
    * 借助spring，initProcessEngineFromSpringResource
```
private static ProcessEngineInfo initProcessEngineFromResource(URL resourceUrl) {
      //略
      ProcessEngine processEngine = buildProcessEngine(resourceUrl);
     //略
    return processEngineInfo;
  }
```

```buildoutcfg
 protected static void initProcessEngineFromSpringResource(URL resource) {
    try {
      Class<?> springConfigurationHelperClass = ReflectUtil.loadClass("org.activiti.spring.SpringConfigurationHelper");
      Method method = springConfigurationHelperClass.getDeclaredMethod("buildProcessEngine", new Class<?>[] { URL.class });
      ProcessEngine processEngine = (ProcessEngine) method.invoke(null, new Object[] { resource });

      String processEngineName = processEngine.getName();
      ProcessEngineInfo processEngineInfo = new ProcessEngineInfoImpl(processEngineName, resource.toString(), null);
      processEngineInfosByName.put(processEngineName, processEngineInfo);
      processEngineInfosByResourceUrl.put(resource.toString(), processEngineInfo);

    } catch (Exception e) {
      throw new ActivitiException("couldn't initialize process engine from spring configuration resource " + resource.toString() + ": " + e.getMessage(), e);
    }
  }
```
容器初始化的时间，在第一次需要获取到流程引擎实例时进行初始化
```
  public static ProcessEngine getProcessEngine(String processEngineName) {
    if (!isInitialized()) {
      init();
    }
    return processEngines.get(processEngineName);
  }
```
由实例化的方法我们也可以看出，引擎容器只会加载一次
```buildoutcfg
 public synchronized static void init() {
    if (!isInitialized()) {
      //略
  }
```
* destroy,主要做关闭引擎实例，清理容器（对应的实例信息等），并对容器初始化标识（静态变量isInitialized）重置为false，至此一个流程引擎容器一个完整生命周期结束。
```
 public synchronized static void destroy() {
    if (isInitialized()) {
      Map<String, ProcessEngine> engines = new HashMap<String, ProcessEngine>(processEngines);
      processEngines = new HashMap<String, ProcessEngine>();

      for (String processEngineName : engines.keySet()) {
        ProcessEngine processEngine = engines.get(processEngineName);
        try {
          processEngine.close();
        } catch (Exception e) {
          log.error("exception while closing {}", (processEngineName == null ? "the default process engine" : "process engine " + processEngineName), e);
        }
      }

      processEngineInfosByName.clear();
      processEngineInfosByResourceUrl.clear();
      processEngineInfos.clear();

      setInitialized(false);
    }
  }
```
上述只是简单的分析了引擎的启动流程，而真正负责初始化容器的代码为buildProcessEngine，过程较为复杂，此处不再详述
```
  private static ProcessEngine buildProcessEngine(URL resource) {
    InputStream inputStream = null;
    try {
      inputStream = resource.openStream();
      ProcessEngineConfiguration processEngineConfiguration = ProcessEngineConfiguration.createProcessEngineConfigurationFromInputStream(inputStream);
      return processEngineConfiguration.buildProcessEngine();

    } catch (IOException e) {
      throw new ActivitiIllegalArgumentException("couldn't open resource stream: " + e.getMessage(), e);
    } finally {
      IoUtil.closeSilently(inputStream);
    }
  }
```
### 流程启动源码分析
* startProcessInstanceByXXX，典型的命令模式应用，封装StartProcessInstanceCmd命令实体
```
  public ProcessInstance startProcessInstanceByKey(String processDefinitionKey, Map<String, Object> variables) {
    return commandExecutor.execute(new StartProcessInstanceCmd<ProcessInstance>(processDefinitionKey, null, null, variables));
  }
```
* 执行execute方法，execute方法的执行逻辑如下  
 根据processDefinitionKey或proceeDefinitionId在已发布的流程定义中查找，它是先查找缓冲中的流程定义然后再去数据库中查找以便提高效率，如果找不到或找到的流程定义被挂起将抛出运行时异常ActivitiException.  
```buildoutcfg
    // Find the process definition
    ProcessDefinition processDefinition = null;
    if (processDefinitionId != null) {

      processDefinition = deploymentCache.findDeployedProcessDefinitionById(processDefinitionId);
      if (processDefinition == null) {
        throw new ActivitiObjectNotFoundException("No process definition found for id = '" + processDefinitionId + "'", ProcessDefinition.class);
      }

    } else if (processDefinitionKey != null && (tenantId == null || ProcessEngineConfiguration.NO_TENANT_ID.equals(tenantId))) {

      processDefinition = deploymentCache.findDeployedLatestProcessDefinitionByKey(processDefinitionKey);
      if (processDefinition == null) {
        throw new ActivitiObjectNotFoundException("No process definition found for key '" + processDefinitionKey + "'", ProcessDefinition.class);
      }

    } else if (processDefinitionKey != null && tenantId != null && !ProcessEngineConfiguration.NO_TENANT_ID.equals(tenantId)) {

      processDefinition = deploymentCache.findDeployedLatestProcessDefinitionByKeyAndTenantId(processDefinitionKey, tenantId);
      if (processDefinition == null) {
        throw new ActivitiObjectNotFoundException("No process definition found for key '" + processDefinitionKey + "' for tenant identifier " + tenantId, ProcessDefinition.class);
      }

    } else {
      throw new ActivitiIllegalArgumentException("processDefinitionKey and processDefinitionId are null");
    }
```    
创建流程实例createAndStartProcessInstance，包含流程实例创建和启动过程（6.x版本），创建的过程也是比较复杂，此处简述  
创建责任链中的实体（ExecutionEntity）
```buildoutcfg
 ExecutionEntity processInstance = commandContext.getExecutionEntityManager()
    		.createProcessInstanceExecution(processDefinition, businessKey, processDefinition.getTenantId(), initiatorVariableName);
```

填充实例对象属性，在填充过程中可以看到存数据库的部分，此处的executionentity对应表act_ru_execution(后续如果将数据库会说到，后续可能会写吧，呵呵)
```
 public ExecutionEntity createProcessInstanceExecution(ProcessDefinition processDefinition, String businessKey, String tenantId, String initiatorVariableName) {
    ExecutionEntity processInstanceExecution = executionDataManager.create();
    
    i//填充，内容略
    // Inherit tenant id (if any)
    if (tenantId != null) {
      processInstanceExecution.setTenantId(tenantId);
    }
    // Store in database
    insert(processInstanceExecution, false);

    if (initiatorVariableName != null) {
      processInstanceExecution.setVariable(initiatorVariableName, authenticatedUserId);
    }

    // Need to be after insert, cause we need the id
    processInstanceExecution.setProcessInstanceId(processInstanceExecution.getId());
    processInstanceExecution.setRootProcessInstanceId(processInstanceExecution.getId());
    return processInstanceExecution;
  }
```
获取流程变量属性，此处的variable为bpmn协议中重要的元素，作为流程变量处理
```buildoutcfg
  processInstance.setVariables(processDataObjects(process.getDataObjects()));

    // Set the variables passed into the start command
    if (variables != null) {
      for (String varName : variables.keySet()) {
        processInstance.setVariable(varName, variables.get(varName));
      }
    }
    if (transientVariables != null) {
      for (String varName : transientVariables.keySet()) {
        processInstance.setTransientVariable(varName, transientVariables.get(varName));
      }
    }
```
封装流程启动者，在此过程中发现流程启动者是通过threadlocal来管理，所以人员的封装一定是在流程启动之前的。同时还将启动者关联到数据库的act_ru_identitylink表中
```buildoutcfg
  String authenticatedUserId = Authentication.getAuthenticatedUserId();
    if (authenticatedUserId != null) {
      getIdentityLinkEntityManager().addIdentityLink(processInstanceExecution, authenticatedUserId, null, IdentityLinkType.STARTER);
    }
```
流程中启动节点的变量封装，关于启动节点的定义在bpmn相关协议中说明，这里不再赘述
```buildoutcfg
 if (initiatorVariableName != null) {
      processInstanceExecution.setVariable(initiatorVariableName, authenticatedUserId);
    }
```
将流程数据记录到历史表中，act_hi_task
```buildoutcfg
  commandContext.getHistoryManager().recordProcessInstanceStart(processInstance, initialFlowElement);

    processInstance.setVariables(processDataObjects(process.getDataObjects()));
```
启动processInstance
```buildoutcfg
  
    if (startProcessInstance) {
        startProcessInstance(processInstance, commandContext, variables);
      }
```
下面以流程图的形式说明下启动流程实例的过程  
![](https://github.com/FeiMin/Activiti6.0Learn/blob/master/img/%E5%90%AF%E5%8A%A8%E5%AE%9E%E4%BE%8B.png)
### 
最后通过代码说明下的节点如何进行流转，之前说到过节点的流转采用了责任链的模式，所以在代码中我们可以看到通过状态值最终定义了下一个节点的流向ActivitiEventDispatcher。  
此段代码的流程如下，找到该流程实例->节点的权限校验->节点状态封装->通过完成状态->dispatherEvent最终封装下一个节点event
```buildoutcfg
protected void executeTaskComplete(CommandContext commandContext, TaskEntity taskEntity, Map<String, Object> variables, boolean localScope) {
    // Task complete logic

    if (taskEntity.getDelegationState() != null && taskEntity.getDelegationState().equals(DelegationState.PENDING)) {
      throw new ActivitiException("A delegated task cannot be completed, but should be resolved instead.");
    }

    commandContext.getProcessEngineConfiguration().getListenerNotificationHelper().executeTaskListeners(taskEntity, TaskListener.EVENTNAME_COMPLETE);
    if (Authentication.getAuthenticatedUserId() != null && taskEntity.getProcessInstanceId() != null) {
      ExecutionEntity processInstanceEntity = commandContext.getExecutionEntityManager().findById(taskEntity.getProcessInstanceId());
      commandContext.getIdentityLinkEntityManager().involveUser(processInstanceEntity, Authentication.getAuthenticatedUserId(),IdentityLinkType.PARTICIPANT);
    }

    ActivitiEventDispatcher eventDispatcher = Context.getProcessEngineConfiguration().getEventDispatcher();
    if (eventDispatcher.isEnabled()) {
      if (variables != null) {
        eventDispatcher.dispatchEvent(ActivitiEventBuilder.createEntityWithVariablesEvent(ActivitiEventType.TASK_COMPLETED, taskEntity, variables, localScope));
      } else {
        eventDispatcher.dispatchEvent(ActivitiEventBuilder.createEntityEvent(ActivitiEventType.TASK_COMPLETED, taskEntity));
      }
    }

    commandContext.getTaskEntityManager().deleteTask(taskEntity, null, false, false);

    // Continue process (if not a standalone task)
    if (taskEntity.getExecutionId() != null) {
      ExecutionEntity executionEntity = commandContext.getExecutionEntityManager().findById(taskEntity.getExecutionId());
      Context.getAgenda().planTriggerExecutionOperation(executionEntity);
    }
  }
```
流转分析在此处只是简单一讲，具体流转的实现是通过监听器，以后会单独讲这一块。
