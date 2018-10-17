### 目录
* 前言
* 什么是工作流
* Activiti简介
* 监控
* p.s.
### 前言
因公司在其产品中需要加入一类日常办公用的业务功能，而这些功能都离不开工作流引擎的支持。我花了大概一周的时间去研究了几个工作流引擎，最终决定用activiti引擎完成这些业务。  
现在回忆下activiti的学习过程，还是比较痛苦，但我仍愿意分享一下我的一些学习经历。本来计划用一个流程图来讲述一下引擎的结构，但翻遍官方文档未找到相关资料，无奈只能靠自己的一些使用体会进行一次说明。文档将分几方面阐述笔者对activiti的理解，由于笔者水平有限，文中难免出现一些错误或不准确或前言不搭后语或逻辑混乱或借鉴其他文档的地方，如果发现，你TM来打我啊。
### 什么是工作流
首先了解下什么是工作流，一言以蔽之，以任务的形式驱动人或系统去完成某个作业。有了工作流引擎后，我们不再需要等待其他人的工作进度，只需要关心当前我们有多少代办即可。而activiti就是这样的一个工作流引擎，它基于BPMN2.0开发，具备稳定、快速轻量的特点。  
工作流的生命周期，一个工作流主要分为5个部分，分别为定义、发布、执行、监控和优化  
![](https://github.com/FeiMin/Activiti6.0Learn/blob/master/ddf91248-9ca5-443e-8d50-a9e3d523ea6e.png)
* 定义：将业务定义为具体流程，再交由程序人员制定成可识别的文件。定义需根据BPMN协议来进行执行，下免简单描述下什么是BPMN。Business Process Modeling Notation，简称BPMN，官方文档500多页，有兴趣的可以去看。BPMN简单的来说就是业务流程建模标注。
* 发布：由业务人员打包资源（包括bpmn文件）等信息并提交到工作流引擎。
* 执行：流程引擎按照事先定义的流程来处理业务。
* 监控：在办理业务中对比task执行结果。
* 优化：对业务结果进行优化，包括对流程图的修改或重新设计  
### Activiti简介
什么是activiti？activiti是一个基于BPMN2.0协议实现的工作流引擎。接下来我们将从以下几个方面谈下笔者对它的了解：
* 持久化：activiti将数据通过MyBatis和数据库进行数据交换，目前支持以下几种数据库
![](https://github.com/FeiMin/Activiti6.0Learn/blob/master/10fd850b-e739-4f94-b921-d75f5bddc7e9.png)
* 接口：activiti提供了七大Service接口，通过引擎获取，支持链式编程
![](https://github.com/FeiMin/Activiti6.0Learn/blob/master/0.8454358355627953.png)
    * RepositoryService：流程仓库Service，用于流程文件的部署、删除和读取
    * TaskService：用户管理和查询任务
    * IdentityService：管理和查询用户、用户组以及用户和任务之前的关系
    * FormService：用户管理相关的表单数据（没用过）
    * RuntimeService：查询正在运行的流程实例和任务
    * ManagementService：管理引擎的、可以查询引擎配置、执行的作业等信息
    * HistoryService：可以查询所有的历史数据、包含流程实例、任务、活动、变量、批注等
* 数据分离：个人认为此部分是activiti的优秀设计。将运行时数据和历史数据分离，这样可以快速读取运行时数据，仅当使用历史数据时才进行查询。面对日积月累的数据，此种设计可以保证activiti的高效运行。
* 架构与组件：activiti最重要的是引擎，下图为activiti架构图
![](https://github.com/FeiMin/Activiti6.0Learn/blob/master/f5f0e243-d0aa-4749-aae1-e9b8c0e547e9.png)
   * Activiti Engine：最核心模块、提供针对BPMN2.0规范的解析、执行、创建、管理、查询历史等操作。
   * Activiti Modeler：模型设计器
   * Activiti Explorer：管理仓库、用户、组、启动流程和任务办理等
### 监控
可以使用标准Java Management Extensions（JMX）技术连接到Activiti引擎，以获取信息或更改其行为。任何标准JMX客户端都可用于此目的。启用和禁用Job Executor，部署新的流程定义文件并删除它们只是使用JMX可以完成的示例，而无需编写单行代码。  
```
<dependency>
  <groupId>org.activiti</groupId>
  <artifactId>activiti-jmx</artifactId>
  <version>latest.version</version>
</dependency>
```
JMX  
`service:jmx:rmi:///jndi/rmi://localhost:1099/jmxrmi/activiti`
### p.s.
activiti入门较简单，跑通demo也很简单。但想要根据业务系统进行灵活运用，还是需要不少的学习时间，至少要熟练使用系统提供的API(核心的几十上百个)。

#### 参考
https://www.activiti.org/userguide/#_introduction
