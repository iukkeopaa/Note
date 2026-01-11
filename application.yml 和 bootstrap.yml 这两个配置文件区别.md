### 一、`application.yml` 和 `bootstrap.yml` 的核心区别

这两个配置文件均用于 Spring 应用的配置，但**加载时机、核心用途、优先级、适用场景**存在本质差异，其中 `bootstrap.yml` 是「引导级配置」，`application.yml` 是「应用级配置」，具体区别如下：

|     对比维度     |           `bootstrap.yml`（/bootstrap.properties）           |         `application.yml`（/application.properties）         |
| :--------------: | :----------------------------------------------------------: | :----------------------------------------------------------: |
|     加载时机     |             更早（Spring 应用「引导阶段」加载）              | 较晚（Spring 应用「应用阶段」加载，在 `bootstrap.yml` 之后） |
|     核心用途     | 1.  配置「应用引导过程中所需的固定配置」（如服务注册中心地址、配置中心地址）；2.  加载「外部配置源」（如 Nacos、Config Server 的远程配置）；3.  配置「不随应用重启 / 环境变化的核心参数」（如加密 / 解密密钥、服务名称） | 1.  配置「应用运行时的业务相关配置」（如端口、数据库连接、日志级别）；2.  配置「随环境变化的动态参数」（如开发 / 测试 / 生产环境的数据库地址）；3.  定义应用内部的业务配置（如接口超时时间、限流阈值） |
|    配置优先级    |      更高（引导级配置，不会被 `application.yml` 覆盖）       | 更低（应用级配置，无法覆盖 `bootstrap.yml` 中相同的配置项）  |
|     生效范围     | 主要作用于「Spring 上下文的引导容器（Bootstrap Context）」，该容器是应用容器的父容器 | 主要作用于「Spring 应用容器（Application Context）」，依赖引导容器的结果启动 |
|     适用场景     | 主要用于 **Spring Cloud 项目**（如 Nacos、Eureka、Spring Cloud Config 集成），纯 Spring Boot 项目可省略 | 适用于 **所有 Spring Boot/Spring Cloud 项目**，是应用配置的核心文件，纯 Spring Boot 项目优先使用该文件 |
| 配置是否可被覆盖 | 自身配置具有「不可替代性」，相同配置项不会被 `application.yml` 覆盖（特殊配置除外） | 可被外部配置源（如远程配置中心、命令行参数）覆盖，无法覆盖 `bootstrap.yml` 的配置 |

### 二、核心补充：加载顺序与优先级逻辑

1. 加载顺序：`bootstrap.yml` → 外部远程配置（如 Nacos 从配置中心拉取的配置） → `application.yml` → 命令行参数；
2. 优先级原则：**先加载的核心配置（尤其是 `bootstrap.yml`）优先级高于后加载的配置，后加载的配置无法覆盖先加载的「同类型核心配置项」（如 `server.port`）**；
3. 本质原因：`bootstrap.yml` 用于配置「应用启动的基础环境」，如果其配置能被 `application.yml` 覆盖，会导致引导阶段的配置不稳定（如配置中心地址被覆盖，无法正常拉取远程配置），因此 Spring 设计时赋予 `bootstrap.yml` 更高的优先级。

### 三、配置冲突时的端口访问结果及原因

#### 1.  最终访问结果

工程启动后，**应该访问 `8080` 端口**（即 `bootstrap.yml` 中配置的端口），`application.yml` 中配置的 `8090` 不会生效。

#### 2.  关键原因解释

1. 优先级层面：`bootstrap.yml` 的配置优先级高于 `application.yml`，`server.port` 作为核心的应用启动配置，在引导阶段就会被 `bootstrap.yml` 确定，后续 `application.yml` 中相同的 `server.port` 配置无法覆盖该值；
2. 加载时机层面：`server.port` 是应用启动时绑定端口的核心参数，Spring 在引导阶段（加载 `bootstrap.yml`）就会读取端口配置并准备端口绑定，而 `application.yml` 是在应用阶段加载，此时端口绑定已经完成，即使配置了不同端口，也无法修改已绑定的端口；
3. 补充验证：启动应用后，可通过日志查看端口绑定信息，会明确打印 `Tomcat started on port(s): 8080 (http) with context path ''`，证实 `8080` 端口生效。

### 四、额外说明（纯 Spring Boot 项目的特殊情况）

1. 纯 Spring Boot 项目（未引入 Spring Cloud 相关依赖）中，**默认不会加载 `bootstrap.yml`**，此时即使创建 `bootstrap.yml` 配置端口，也不会生效，应用会优先读取 `application.yml` 中的 `8090` 端口；
2. 若纯 Spring Boot 项目需要启用 `bootstrap.yml`，需引入 Spring Cloud Context 依赖，才能触发引导阶段的配置加载：

xml











```
<!-- Maven 依赖 -->
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-context</artifactId>
    <!-- 版本需与 Spring Boot 版本兼容 -->
    <version>合适的版本</version>
</dependency>
```

### 总结

1. 核心区别：`bootstrap.yml` 早加载、高优先级、用于引导配置（Spring Cloud 项目）；`application.yml` 晚加载、低优先级、用于应用运行配置（所有项目）；
2. 端口结果：配置冲突时，优先访问 `bootstrap.yml` 中的 `8080` 端口（Spring Cloud 项目中）；
3. 关键逻辑：先加载的引导级配置不会被后加载的应用级配置覆盖，端口绑定在引导阶段完成，后续配置无法修改。

这两个配置文件的优先级是如何确定的？

除了端口，还有哪些配置项会受到优先级的影响？

如何在不同的环境中使用这两个配置文件？
