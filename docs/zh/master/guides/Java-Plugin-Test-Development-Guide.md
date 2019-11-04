[toc]

## 一、环境要求

1. MacOS/Linux
2. jdk 8+
2. Docker
3. Docker Compose

## 二、用例镜像说明

测试框架提供`JVM-container`和`Tomcat-container`两种类型的镜像。在编写用例时根据实际情况选择适合的镜像，**当两种镜像都适用时建议优先选择`JVM-container`。**

### I. JVM-container镜像说明

[JVM-container](https://github.com/apache/skywalking/tree/master/test/plugin/containers/jvm-container)使用`openjdk:8`作为基础镜像。
需要用例在`mvn package`后生成包含用例jar文件和`startup.sh`脚本且与用例工程同名的zip压缩包。

参考用例:
* [sofarpc-scenario](https://github.com/apache/skywalking/tree/master/test/plugin/scenarios/sofarpc-scenario)
* [webflux-scenario(包含多个子项目的用例)](https://github.com/apache/skywalking/tree/master/test/plugin/scenarios/webflux-scenario)

### II. Tomcat-container镜像说明

[Tomcat-container](https://github.com/apache/skywalking/tree/master/test/plugin/containers/tomcat-container)使用`tomcat:8.5.42-jdk8-openjdk`作为基础镜像，需要用例在执行`mvn package`后生成与用例同名的war包。

参考用例:
* [spring-4.3.x-scenario](https://github.com/apache/skywalking/tree/master/test/plugin/scenarios/spring-4.3.x-scenario)



### III. 工程目录结构

测试用例是一个独立的Maven工程，并且必须能是打包成war包的web工程，或含有完成java工程的zip包，这由用例选择的镜像类型决定。并要求提供一个外部能够访问的Web服务用例测试调用链追踪的链接和心跳检查的链接。这些都需要描述在configuration.yml文件中，关于这个配置文件后面还会继续介绍。

**a. JVM-container类工程目录**

```
[plugin-scenario]
    |- [bin]
        |- startup.sh
    |- [config]
        |- expectedData.yaml
    |- [src]
        |- [main]
            |- ...
        |- [resource]
            |- log4j2.xml
    |- pom.xml
    |- configuration.yaml
    |- support-version.list

[] = directory
```

**b. Tomcat-container类工程目录**

```
[plugin-scenario]
    |- [config]
        |- expectedData.yaml
    |- [src]
        |- [main]
            |- ...
        |- [resource]
            |- log4j2.xml
        |- [webapp]
            |- [WEB-INF]
                |- web.xml
    |- pom.xml
    |- configuration.yaml
    |- support-version.list

[] = directory
```

## 三、用例文件说明
以下是插件测试用例运行时必要的文件：

文件名 | 用途、说明
---|---
`configuration.yml` | 定义用例的基本信息，如: 被测试框架名称、用例入口、运行模式、第三方依赖服务等
`expectedData.yaml` | 期望数据文件用来描述用例生成的Segment数据，文件主要包含两部分内容：注册项 和 Segment数据. Segment数据中包含对Span的校验和Segment个数的校验
`support-version.list` | 描述用例（插件）自动化测试覆盖的版本列表
`startup.sh` |`JVM-container`镜像用例启动脚本（`Tomcat-container`镜像用例请忽略)

`*`support-version.list是固定格式的，每个版本号为一行，以`#`开始的为无效行。

### I. configuration.yml

各个字段描述:

| 字段 | 描述
| --- | ---
| type | 用例镜像类型，可选项: `jvm、tomcat`（必填）
| entryService | 用例访问接口（必填）
| healthCheck | 用例健康检查接口（必填）
| startScript | 用例启动脚本，仅`type=jvm`时有效（选填，`type=jvm`时必填）
| framework | 用例名称（必填）
| runningMode | 用例运行模式，可选项:`default`、 `with_optional`、 `with_bootstrap`（选填，缺省值为`default`）
| withPlugins | 指定具体可选插件，eg:`apm-spring-annotation-plugin-*.jar`（选填，`runningMode=with_optional`或`runningMode=with_bootstrap`时必填）
| environment | 同`docker-compose#environment`，此处为用例容器配置（选填）
| depends_on | 同`docker-compose#depends_on`，此处为用例容器配置（选填）
| dependencies | 同`docker-compose#services`，此处为用例依赖容器配置，且支持`image、links、hostname、environment、depends_on`（选填）

**注意: 且当dependencies不为空时才通过`docker-compose`启动用例**

runningMode可选项描述:

| 可选项 | 描述
| --- | ---
| default | 激活[apm-sdk-plugin](https://github.com/apache/skywalking/tree/master/apm-sniffer/apm-sdk-plugin)目录内所有插件
| with_optional | 激活apm-sdk-plugin目录加[apm-sniffer](https://github.com/apache/skywalking/tree/master/apm-sniffer/optional-plugins)目录内指定插件
| with_bootstrap | 激活apm-sdk-plugin目录加[bootstrap-plugins](https://github.com/apache/skywalking/tree/master/apm-sniffer/bootstrap-plugins)目录内指定插件

**配置文件格式**

```
type:
entryService:
healthCheck:
startScript:
framework:
runningMode:
withPlugins:
environment:
  ...
depends_on:
  ...
dependencies:
  service1:
    image:
    hostname: 
    expose:
      ...
    environment:
      ...
    depends_on:
      ...
    links:
      ...
    entrypoint:
      ...
    healthcheck:
      ...
```

* dependencies支持healthcheck配置，但它所有子项需要以字符串形式表示。
* 如果第三依赖的服务需要与插件的版本一致时，可以${CASE_SERVER_IMAGE_VERSION}在运行时会替换为${test.framework.version}（用例测试插件的版本号）。

> 不支持资源相关配置项，如volumes，ports和ulimits。（当然我们完全不需要向容器的任何端口映射到宿主机上，也不需要额外在挂载目录，所以不支持这些配置。）

**参考示例**
* [dubbo-2.7.x with JVM-container](https://github.com/apache/skywalking/blob/master/test/plugin/scenarios/dubbo-2.7.x-scenario/configuration.yml)
* [jetty with Tomcat-container](https://github.com/apache/skywalking/blob/master/test/plugin/scenarios/jetty-scenario/configuration.yml)
* [gateway with runningMode](https://github.com/apache/skywalking/blob/master/test/plugin/scenarios/gateway-scenario/configuration.yml)
* [canal with docker-compose](https://github.com/apache/skywalking/blob/master/test/plugin/scenarios/canal-scenario/configuration.yml)

### II. expectedData.yaml

#### a. 字段描述符以及检查项说明

**数字型字段校验描述符**

| 描述符 | 描述 |
| :--- | :--- |
| `nq` | 不等于 |
| `eq` | 等于，默认可以不写 |
| `ge` | 大于等于 |
| `gt` | 大于 |

**字符串型字段描述符**

| 描述符 | 描述 |
| :--- | :--- |
| `not null` | 不为null |
| `null` | 空字符或者null |
| `eq` | 精确匹配. 默认可以不写 |


**注册项数据校验格式**
```yml
registryItems:
  applications:
  - APPLICATION_CODE: APPLICATION_ID(int)
  ...
  instances:
  - APPLICATION_CODE: INSTANCE_COUNT(int)
  ...
  operationNames:
  - APPLICATION_CODE: [ SPAN_OPERATION(string), ... ]
  ...
```

以下对各个校验字段的描述:
| 字段 | 描述
| --- | ---
| applications | 注册的Aplication_code和application Id映射关系.目前只需校验不为0即可
| instances | Application生成的实例数
| operationNames | 所有预期生成Span的OperationName列表（LocalSpan不校验）


**Segments数据校验格式**
```yml
segments:
-
  applicationCode: APPLICATION_CODE(string)
  segmentSize: SEGMENT_SIZE(int)
  segments:
  - segmentId: SEGMENT_ID(string)
    spans:
        ....
```

以下对各个校验字段的描述:

| 字段 | 描述 
| --- | ---  
| applicationCode | 待校验Segment的ApplicationCode. 
| segmentSize | 待校验Segment的ApplicationCode生成的Segment的数量. 
| segmentId | segment的trace ID. 
| spans | segment生成的Span列表 

**Span数据校验格式**

**注意**: 期望文件中Segment的Span是按照Span的结束顺序进行排列

```yml
    operationName: OPERATION_NAME(string)
    operationId: SPAN_ID(int)
    parentSpanId: PARENT_SPAN_ID(int)
    spanId: SPAN_ID(int)
    startTime: START_TIME(int)
    endTime: END_TIME(int)
    isError: IS_ERROR(string: true, false)
    spanLayer: SPAN_LAYER(string: DB, RPC_FRAMEWORK, HTTP, MQ, CACHE)
    spanType: SPAN_TYPE(string: Exit, Entry, Local )
    componentName: COMPONENT_NAME(string)
    componentId: COMPONENT_ID(int)
    tags:
    - {key: TAG_KEY(string), value: TAG_VALUE(string)}
    ...
    logs:
    - {key: LOG_KEY(string), value: LOG_VALUE(string)}
    ...
    peer: PEER(string)
    peerId: PEER_ID(int)
    refs:
    - {
       parentSpanId: PARENT_SPAN_ID(int),
       parentTraceSegmentId: PARENT_TRACE_SEGMENT_ID(string),
       entryServiceName: ENTRY_SERVICE_NAME(string),
       networkAddress: NETWORK_ADDRESS(string),
       parentServiceName: PARENT_SERVICE_NAME(string),
       entryApplicationInstanceId: ENTRY_APPLICATION_INSTANCE_ID(int)
     }
   ...
```

以下对各个校验字段的描述:

| 字段 | 描述 
|--- |--- 
| operationName | Span的Operation Name 
| operationId | OperationName对应的Id, 这个值目前为0 
| parentSpanId | Span的父级Span的Id. **注意**: 第一个Span的parentSpanId为-1 
| spanId | Span的Id. **注意**: ID是从0开始. 
| startTime | Span开始时间. 目前不支持精确匹配，只需判断不为0即可 
| endTime | Span的结束时间.目前不支持精确匹配，只需判断不为0即可 
| isError | 是否出现异常. 如果Span抛出异常或者状态码大于400，该值为true, 否则为false 
| componentName | 对应组件的名字。官方提供的[Component](https://github.com/apache/incubator-skywalking/blob/master/apm-protocol/apm-network/src/main/java/org/apache/skywalking/apm/network/trace/component/OfficialComponent.java)，则该值为null. 
| componentId | 组件对应的ID. 。如果是官方提供的[Component](https://github.com/apache/incubator-skywalking/blob/master/apm-protocol/apm-network/src/main/java/org/apache/skywalking/apm/network/trace/component/OfficialComponent.java)，则该值为定义的组件ID 
| tags | Span设置的Tag. **注意**: tag的顺序即为在插件中设置的顺序 
| logs | Span设置的log. **注意**: 顺序为设置Log的顺序 
| SpanLayer | 设置的SpanLayer. 目前可能的值为: DB, RPC_FRAMEWORK, HTTP, MQ, CACHE 
| SpanType | Span的类型. 目前的取值为 Exit, Entry, Local 
| peer | 访问的远端IP. Exit类型的Span, 该值非空 
| peerId | 访问的远端IP的ID，该值目前为0 


以下对SegmentRef各个校验字段的描述:

| 字段 | 描述 
|---- |---- 
| parentSpanId | 调用端的SpanID. 例如HttpClient是由SegmentA的SpanID为1的调用的，所以该值为1 
| parentTraceSegmentId | 调用端的SegmentID. 格式: ${APPLICATION_CODE[SEGMENT_INDEX]}, `SEGMENT_INDEX`是相对于期望文件的INDEX. 例如SegmentB由`httpclient-case`中的第0个Segment调用的，所以这个值为`${httpclient-case[0]}` 
| entryServiceName | 调用链入口的Segment的服务名词. 例如HttpClient的entryServiceName为`/httpclient-case/case/httpclient` 
| networkAddress | 被调用者的网络地址。例如CaseServlet通过127.0.0.1:8080调用到ContextPropagateServlet,所以这个值为127.0.0.1:8080 
| parentServiceName | 调用端的SpanID等于0的OperationName 
| entryApplicationInstanceId | 调用链入口的实例ID。 

#### b. 编写期望数据流程

##### 1. 编写RegistryItems

HttpClient测试用例中运行在Tomcat容器中，所以httpclient的实例数为1, 并且applicationId不为0。HttpClient Span的OperationName和ContextPropagateServlet生成的Span的OperationName一致，所以operationNames中只有两个operationName.
```yml
registryItems:
  applications:
  - {httpclient-case: nq 0}
  instances:
  - {httpclient-case: 1}
  operationNames:
  - httpclient-case: [/httpclient-case/case/httpclient,/httpclient-case/case/context-propagate]
```

##### 2. 编写segmentItems

下面以HttpClient插件为例子，HttpClient插件测试两点: 能否正确的创建HttpClient的Span以及能否正确的传递上下文，基于这两点，设计如下用例：

```
+-------------+         +------------------+            +-------------------------+
|   Browser   |         |  Case Servlet    |            | ContextPropagateServlet |
|             |         |                  |            |                         |
+-----|-------+         +---------|--------+            +------------|------------+
      |                           |                                  |
      |                           |                                  |
      |       WebHttp            +-+                                 |
      +------------------------> |-|         HttpClient             +-+
      |                          |--------------------------------> |-|
      |                          |-|                                |-|
      |                          |-|                                |-|
      |                          |-| <--------------------------------|
      |                          |-|                                +-+
      | <--------------------------|                                 |
      |                          +-+                                 |
      |                           |                                  |
      |                           |                                  |
      |                           |                                  |
      |                           |                                  |
      +                           +                                  +
```

根据HttpClient用例的运行流程，推断httpclient-case产生两个Segment. 第一个Segment是访问CaseServlet所产生的, 暂且叫它`SegmentA`。第二Segment是ContextPropagateServlet所产生的, 暂且叫它`SegmentB`.

```yml
segments:
  - applicationCode: httpclient-case
    segmentSize: 2
```

Skywalking支持Tomcat埋点，所以SegmentA中会包含两个Span，第一个Span是Tomcat的埋点，第二个Span是HttpClient的埋点.

SegmentA的生成的Span数据如下：
```yml
    - segmentId: not null
      spans:
        - operationName: /httpclient-case/case/context-propagate
          operationId: eq 0
          parentSpanId: 0
          spanId: 1
          startTime: nq 0
          endTime: nq 0
          isError: false
          spanLayer: Http
          spanType: Exit
          componentName: null
          componentId: eq 2
          tags:
            - {key: url, value: 'http://127.0.0.1:8080/httpclient-case/case/context-propagate'}
            - {key: http.method, value: GET}
          logs: []
          peer: null
          peerId: eq 0
        - operationName: /httpclient-case/case/httpclient
          operationId: eq 0
          parentSpanId: -1
          spanId: 0
          startTime: nq 0
          endTime: nq 0
          spanLayer: Http
          isError: false
          spanType: Entry
          componentName: null
          componentId: 1
          tags:
            - {key: url, value: 'http://localhost:{SERVER_OUTPUT_PORT}/httpclient-case/case/httpclient'}
            - {key: http.method, value: GET}
          logs: []
          peer: null
          peerId: eq 0
```

SegmentB由于Skywalking对于Tomcat进行埋点会产生一个Span，并且SegmentA传递ContextTrace给SegmentB，对于SegmentB需要校验SegmentRef数据.

SegmentB的Span校验数据格式如下：
```yml
- segmentId: not null
  spans:
  -
   operationName: /httpclient-case/case/context-propagate
   operationId: eq 0
   parentSpanId: -1
   spanId: 0
   tags:
   - {key: url, value: 'http://127.0.0.1:8080/httpclient-case/case/context-propagate'}
   - {key: http.method, value: GET}
   logs: []
   startTime: nq 0
   endTime: nq 0
   spanLayer: Http
   isError: false
   spanType: Entry
   componentName: null
   componentId: 1
   peer: null
   peerId: eq 0
   refs:
   - {parentSpanId: 1, parentTraceSegmentId: "${httpclient-case[0]}", entryServiceName: "/httpclient-case/case/httpclient", networkAddress: "127.0.0.1:8080",parentServiceName: "/httpclient-case/case/httpclient",entryApplicationInstanceId: nq 0 }
```

### III. startup.sh

基础容器为插件测试用例提供运行环境，所以也在测试用例提供必要的环境变量。你可以在你的启动脚本里直接引用如下环境变量：

**环境变量说明列表**

| 变量   | 描述    |
|:----     |:----        |
| agent_opts               |        探针参数         |
| SCENARIO_NAME       |  当前运行用例名称    |
| SCENARIO_VERSION           | 当前运行用例版本 |
| SCENARIO_ENTRY_SERVICE             | 当前运行用例入口接口 |
| SCENARIO_HEALTH_CHECK_URL          | 当前运行用例心跳接口  |


> `type: jvm`要求在启动脚本中显式加入`${agent_opts}`，它包含SkyWalking运行时必要的参数。如果需要覆盖它们，请将参数加在`${agent_opts}`之后。


框架会自动在用例的启动你还可以在你启动脚本里覆盖如下参数，如`jetty-scenario`和`webflux-scenario`都是一个工程中含有两种用例服务的，因此需要额外指定`agent`的服务名。

示例：
```bash
home="$(cd "$(dirname $0)"; pwd)"

java -jar ${agent_opts} "-Dskywalking.agent.service_name=jettyserver-scenario" ${home}/../libs/jettyserver-scenario.jar &
sleep 1

java -jar ${agent_opts} "-Dskywalking.agent.service_name=jettyclient-scenario"  ${home}/../libs/jettyclient-scenario.jar &

```

> 但如非必要请匆覆盖`skywalking.agent.service_name`或者其它agent可配置化参数。

**参考文件**
* [undertow](https://github.com/apache/skywalking/blob/master/test/plugin/scenarios/undertow-scenario/bin/startup.sh)
* [webflux](https://github.com/apache/skywalking/blob/master/test/plugin/scenarios/webflux-scenario/webflux-dist/bin/startup.sh)


### 编写用例代码
pom.xml最佳实践:
1. 测试框架的版本号设置为属性变量
```xml
<test.framework.version>9.0.0.v20130308</test.framework.version>

<dependency>
    <groupId>org.apache.httpcomponents</groupId>
    <artifactId>httpclient</artifactId>
    <version>${test.framework.version}</version>
</dependency>
```
具体请参考[配置]


### 如何做心跳检查

心跳检查是为了感知服务的可用状态，即探针已经激活并完成服务注册以及插件已经初始化。建议并推荐，用例在检查心跳时，先做依赖服务的状态验证。在所有依赖服务的状态验证后，若验证通过需返回`HTTP StatusCode=200`。框架以HTTP状态码`200`作为唯一的成功的信号。


> 注意，由于心跳检查可能会发生多次请求，从而产生多个心跳检查的Segemnt，因此SegmentSize也不再是确定的值（建议使用`ge`符号）。此外，请不要将心跳检查的segment登记在期望数据文件上。

例如：
```yaml
registryItems:
  applications:
    - {canal-scenario: 2}
  instances:
    - {canal-scenario: 1}
  operationNames:
    - canal-scenario: [Canal/example, /canal-scenario/case/canal-case]
  heartbeat: []
segmentItems:
  - applicationCode: canal-scenario
    segmentSize: ge 2
    segments:
      - segmentId: not null
        spans:
          - operationName: Canal/example
...
```

## 本地测试和准备提交


本地测试过程主要两步部分，首先是验证工程可以通过编译，工程目录结构正确，能够部署。开发者在集成测试之前可以先检查脚本是否能够在linux/macos上正常运行，其次entryService/healthcheck接口是否都能正常访问。

然后通过以下命令进行集成测试：

```bash
cd ${SKYWALKING_HOME}
bash ./test/pugin/run.sh -f ${scenario_name}
```

**注意**，如果更新了`./apm-sniffer`目录下的代码，需要重新编译`skywalking-agent`。因为当`skywalking-agent`目录存在时，不会重新编译。

可以通过`${SKYWALKING_HOME}/test/plugin/run.sh -h`了解其所有用法，




### 准备提交

在完成调试之后，开始配置JenkinsFile。如下是JenkinsFile的规则和要求，现在我们有三个JenkinsFile用来配置插件测试任务，分别是`jenkinsfile-agent-test`、`jenkinsfile-agent-test-2`和`jenkinsfile-agent-test-3`三个文件。每个文件分成两组，一共6组并行运行。原则上，希望所有的组能够尽可能同时结束，因此用例加在运行时间比较短的分组上即可。

示例，

```
stage('Test Cases Report (15)') { # 15=12+3 统计两个分组总共有多少个版本
    steps {
        echo "reserve."
    }
}

stage('Run Agent Plugin Tests') {
    when {
        expression {
            return sh(returnStatus: true, script: 'bash tools/ci/agent-build-condition.sh')
        }
    }
    parallel {
        stage('Group1') {
            stages {
                stage('spring-cloud-gateway 2.1.x (3)') { # 此任务一共有多少个版本
                    steps {
                        sh 'bash test/plugin/run.sh gateway-scenario'
                    }
                }
            }
            ...
        }
        stage('Group2') {
            stages {
                stage('solrj 7.x (12)') { # 此任务一共有多少个版本
                    steps {
                        sh 'bash test/plugin/run.sh solrj-7.x-scenario'
                    }
                }
            }
            ...
        }
    }
}
```
