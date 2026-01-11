### 一、先明确：Spring 中循环依赖的前提

首先要清楚，Spring 仅能解决 **「单例 Bean」的构造器之外的循环依赖**（即字段注入、setter 注入），无法解决「多例 Bean」「构造器注入」的循环依赖，这是三级缓存生效的前提。

循环依赖的核心场景：两个或多个单例 Bean 互相引用，如 `A -> B -> A`，或 `A -> B -> C -> A`。

### 二、三级缓存的组成（核心存储结构）

Spring 容器内部通过 **三个 `Map` 实现三级缓存**，均存储在 `DefaultSingletonBeanRegistry` 类中，各自承担不同的职责，层层递进解决循环依赖：

| 缓存级别 |  缓存名称（Map 字段）   |                    存储内容                     |                           核心作用                           |
| :------: | :---------------------: | :---------------------------------------------: | :----------------------------------------------------------: |
| 一级缓存 |   `singletonObjects`    | 「完全初始化完成」的单例 Bean 实例（成品 Bean） |    对外提供可用的 Bean，缓存最终状态的 Bean，避免重复创建    |
| 二级缓存 | `earlySingletonObjects` |   「提前暴露」的单例 Bean 实例（半成品 Bean）   | 存储已经完成实例化、但尚未完成初始化（未执行 `afterPropertiesSet`、未注入其他依赖、未执行初始化方法）的 Bean，避免重复生成代理对象 |
| 三级缓存 |  `singletonFactories`   |   单例 Bean 的「对象工厂」（`ObjectFactory`）   | 存储创建 Bean 实例的工厂对象，用于在需要时「延迟创建」半成品 Bean（或代理对象），是解决循环依赖的核心 |

补充说明：

1. 三个缓存均为「单例缓存」，仅存储单例 Bean 相关数据；
2. `ObjectFactory` 是一个函数式接口，核心方法 `getObject()` 用于创建 / 获取 Bean 实例，其实现逻辑是 Spring 提前暴露 Bean 的关键。

### 三、三级缓存的底层实现流程（以 `A -> B -> A` 为例）

我们以最典型的 `A` 和 `B` 相互依赖（字段注入）为例，拆解整个流程，清晰看到三级缓存的协作过程：

#### 前置准备

Spring 容器启动时，扫描并解析 Bean 定义，对于单例 Bean，会通过 `getBean()` 方法触发 Bean 的创建，核心流程包含「实例化」和「初始化」两个大阶段：

- 实例化：通过反射创建 Bean 的原始实例（给对象分配内存，完成属性默认初始化），对应 `AbstractAutowireCapableBeanFactory` 的 `createBeanInstance()` 方法；
- 初始化：完成依赖注入（给 Bean 的字段赋值）、执行 `@PostConstruct`、`afterPropertiesSet()`、AOP 代理创建等操作，最终得到成品 Bean。

#### 具体流程（分步拆解）

1. **第一步：创建 Bean A，存入三级缓存**

   

   - 容器调用 `getBean(A)`，尝试从三级缓存中获取 A；

   - 三级、二级、一级缓存均无 A，开始创建 A 的流程；

   - 执行「实例化」：通过反射创建 A 的原始实例（半成品，仅分配内存，字段为默认值）；

   - 「关键步骤」：将 A 的 

     ```
     ObjectFactory
     ```

      存入 

     三级缓存 `singletonFactories`

     ，该工厂的 

     ```
     getObject()
     ```

      方法用于后续获取 A 的半成品实例（若需要代理，会在此处生成代理对象）；

     > 这一步是「提前暴露 Bean」的核心：在 A 尚未完成初始化时，就将其创建工厂暴露出去，让其他 Bean 可以获取到 A 的实例，打破循环依赖的死锁。
     >
     > 

     

   - 执行「初始化」阶段的第一步：依赖注入，发现 A 依赖 B，需要调用 `getBean(B)` 获取 B 实例，此时 A 的创建流程暂停，转而创建 B。

   

2. **第二步：创建 Bean B，存入三级缓存，触发循环依赖解决**

   

   - 容器调用 `getBean(B)`，尝试从三级缓存中获取 B；
   - 三级、二级、一级缓存均无 B，开始创建 B 的流程；
   - 执行「实例化」：通过反射创建 B 的原始实例（半成品）；
   - 「关键步骤」：将 B 的 `ObjectFactory` 存入 **三级缓存 `singletonFactories`**；
   - 执行「初始化」阶段的第一步：依赖注入，发现 B 依赖 A，需要调用 `getBean(A)` 获取 A 实例；

   

3. **第三步：从三级缓存中提取 A，完成 B 的依赖注入**

   

   - 容器再次调用 

     ```
     getBean(A)
     ```

     ，开始依次查询三级缓存：

     1. 先查一级缓存 `singletonObjects`：无 A（A 尚未完成初始化）；
     2. 再查二级缓存 `earlySingletonObjects`：无 A；
     3. 最后查三级缓存 `singletonFactories`：存在 A 的 `ObjectFactory`；

     

   - 「关键步骤」：取出 A 的 `ObjectFactory`，调用 `getObject()` 方法，获取 A 的半成品实例（若 A 需要 AOP 代理，此处会生成代理对象并返回，而非原始实例）；

   - 将 A 的半成品实例从「三级缓存」移除，存入「二级缓存 `earlySingletonObjects`」（避免后续其他 Bean 依赖 A 时，重复调用 `ObjectFactory` 生成实例 / 代理）；

   - 将 A 的半成品实例返回给 B，完成 B 的依赖注入（B 的字段 `a` 赋值为 A 的半成品）；

   - B 继续完成后续初始化流程（执行 `@PostConstruct`、`afterPropertiesSet()`、AOP 代理等）；

   - B 初始化完成后，将 B 的成品实例存入「一级缓存 `singletonObjects`」，并从二级、三级缓存中移除 B 相关数据；

   - B 的创建流程完成，将 B 的成品实例返回给 A，此时 A 的依赖注入完成（A 的字段 `b` 赋值为 B 的成品）。

   

4. **第四步：完成 A 的初始化，存入一级缓存**

   

   - A 恢复暂停的初始化流程，继续执行后续操作（`@PostConstruct`、`afterPropertiesSet()`、AOP 代理等）；
   - 注意：若 A 已经在第三步生成了代理对象，此处不会重复生成，保证代理对象的唯一性；
   - A 初始化完成后，将 A 的成品实例（或代理实例）存入「一级缓存 `singletonObjects`」，并从二级、三级缓存中移除 A 相关数据；
   - 整个循环依赖解决流程完成，后续获取 A 或 B，直接从一级缓存中获取成品 Bean。

   

### 四、核心关键：为何需要三级缓存？二级缓存不够吗？

这是理解三级缓存的核心难点，答案的关键在于 **「支持 AOP 代理，避免生成重复的代理对象，同时保证 Bean 的延迟初始化」**。

1. 先假设：只有一级 + 二级缓存，没有三级缓存

   

   - 若 Bean 不需要 AOP 代理：仅存半成品 Bean 到二级缓存，确实可以解决循环依赖；

   - 若 Bean 需要 AOP 代理：会出现两个严重问题：

     - 问题 1：代理对象提前生成，破坏 Bean 的延迟初始化原则。Spring 的 AOP 代理默认是在 Bean 「初始化完成后」生成的（即 `initializeBean()` 方法中），若没有三级缓存，为了让循环依赖的 Bean 获取到代理对象，必须在实例化后立即生成代理，提前消耗资源，且不符合 Spring 的设计逻辑；
     - 问题 2：可能生成重复的代理对象。若多个 Bean 同时依赖需要代理的 A，二级缓存直接存储半成品 Bean（或早期代理），可能导致不同 Bean 获取到 A 的不同代理实例，破坏单例原则。

     

   

2. 三级缓存的核心价值（`singletonFactories` 存储 `ObjectFactory`）

   

   - 延迟创建代理对象：`ObjectFactory` 的 `getObject()` 方法是「懒执行」的，只有当真正需要解决循环依赖时，才会调用该方法生成代理对象，否则不会提前生成，符合 Spring 延迟初始化的设计；
   - 保证代理对象唯一性：当第一个依赖 A 的 Bean（如 B）触发 `getObject()` 后，会将生成的代理对象存入二级缓存 `earlySingletonObjects`，后续其他依赖 A 的 Bean 会直接从二级缓存获取，不会再调用 `ObjectFactory`，从而保证所有 Bean 获取到的是同一个代理实例，符合单例原则；
   - 解耦 Bean 实例化和代理创建：将代理创建的逻辑封装在 `ObjectFactory` 中，不侵入 Bean 的正常实例化和初始化流程，保证 Spring 容器的灵活性。

   

### 五、补充：三级缓存的限制（哪些场景无法解决循环依赖）

1. 多例（原型）Bean：多例 Bean 每次 `getBean()` 都会创建新实例，不会存入三级缓存，因此无法解决循环依赖；
2. 构造器注入的单例 Bean：构造器注入时，Bean 尚未完成实例化（无法生成半成品），无法提前暴露到三级缓存，因此无法解决；
3. 非 Spring 容器管理的 Bean：未被 `@Component`、`@Bean` 等注解标记，不进入 Spring 容器的 Bean，三级缓存无法生效；
4. 手动破坏缓存：若通过自定义逻辑修改或移除三级缓存中的数据，可能导致循环依赖解决失败。

### 总结

1. Spring 三级缓存的核心是 `singletonObjects`（成品）、`earlySingletonObjects`（半成品）、`singletonFactories`（对象工厂），存储在 `DefaultSingletonBeanRegistry` 中；
2. 核心流程是「实例化后存入三级缓存 → 依赖注入时从三级缓存提取并迁移到二级缓存 → 初始化完成后迁移到一级缓存」，打破 `A -> B -> A` 的循环死锁；
3. 三级缓存的核心价值是支持 AOP 代理的延迟创建和唯一性保证，二级缓存无法满足该需求；
4. 仅支持单例 Bean 的非构造器注入循环依赖，其他场景无法解决。


# 三级缓存的map的不同



你想了解 Spring 三级缓存底层使用的数据结构，核心结论是：**Spring 三级缓存的底层均基于「哈希表（Hash Table）」实现，在 Java 中具体对应 `ConcurrentHashMap` 或 `HashMap`（核心是 `ConcurrentHashMap`，保证并发安全）**，且三级缓存的存储载体都是 `Map` 接口的实现类，具体细节如下：

### 一、三级缓存的底层数据结构详情

Spring 三级缓存的定义均在 `DefaultSingletonBeanRegistry` 类中，三者的底层数据结构及对应实现、初始化方式如下：

| 缓存级别 |        字段名称         |             底层数据结构             | 具体实现（Java 类） |                           核心说明                           |
| :------: | :---------------------: | :----------------------------------: | :-----------------: | :----------------------------------------------------------: |
| 一级缓存 |   `singletonObjects`    |          哈希表（线程安全）          | `ConcurrentHashMap` | 1.  核心作用：存储完全初始化完成的成品单例 Bean，对外提供可用实例，支持高并发场景下的读写操作；2.  初始化方式：直接通过 `new ConcurrentHashMap<>()` 初始化，默认容量遵循 `ConcurrentHashMap` 的默认配置（初始容量 16，负载因子 0.75）；3.  并发安全：依赖 `ConcurrentHashMap` 的分段锁（JDK 1.7）或 CAS + 同步节点（JDK 1.8+）实现线程安全，避免多线程创建 Bean 时出现数据竞争。 |
| 二级缓存 | `earlySingletonObjects` | 哈希表（非线程安全，需手动加锁保护） |      `HashMap`      | 1.  核心作用：存储提前暴露的半成品 Bean（实例化完成、未初始化），仅用于解决循环依赖过程中的临时存储；2.  初始化方式：通过 `new HashMap<>()` 初始化，非线程安全实现；3.  并发保障：Spring 不会直接对 `HashMap` 进行并发操作，而是通过全局锁 `singletonObjectsMutex`（`Object` 类型的互斥锁）包裹所有对二级缓存的操作，保证多线程下的数据一致性，避免 `HashMap` 出现并发修改异常。 |
| 三级缓存 |  `singletonFactories`   | 哈希表（非线程安全，需手动加锁保护） |     `HashMap>`      | 1.  核心作用：存储 Bean 的对象工厂 `ObjectFactory`，用于延迟创建半成品 Bean 或代理对象；2.  初始化方式：通过 `new HashMap<>()` 初始化，键为 Bean 名称（`String` 类型），值为 `ObjectFactory` 函数式接口实现类；3.  并发保障：与二级缓存一致，所有操作均包裹在全局锁 `singletonObjectsMutex` 中，避免多线程下的并发修改问题，保证 `HashMap` 的安全使用。 |

### 二、核心补充说明

1. **为何一级缓存用 `ConcurrentHashMap`，二、三级缓存用 `HashMap`？**

   

   - 一级缓存 `singletonObjects` 是对外提供成品 Bean 的核心缓存，会被大量线程并发读取和写入（Bean 初始化完成后存入、后续获取 Bean 时读取），因此需要 `ConcurrentHashMap` 自带的线程安全能力，且其并发性能优于「`HashMap + 全局锁`」；
   - 二、三级缓存（`earlySingletonObjects`、`singletonFactories`）仅在「解决循环依赖」的过程中被使用，操作频率远低于一级缓存，且操作逻辑集中在 Spring 容器内部的 `getSingleton()` 方法中，通过全局锁 `singletonObjectsMutex` 即可实现高效的线程安全控制，使用 `HashMap` 可以减少 `ConcurrentHashMap` 的额外开销，提升容器启动效率。

   

2. **哈希表的核心优势（为何选择 `Map` 实现三级缓存）**

   

   - 快速查询：哈希表的平均查询、插入、删除时间复杂度均为 O (1)，Spring 容器在获取 Bean 时，需要通过 Bean 名称快速定位缓存中的数据，哈希表能够满足这一高效查询需求；
   - 键值对映射：三级缓存的核心需求是「Bean 名称 → Bean 实例 / 对象工厂」的一一映射，`Map` 接口的键值对结构天然匹配该需求，键为唯一的 Bean 名称（如 `com.example.UserService`），值为对应的数据内容；
   - 实现简单、扩展性好：`HashMap` 和 `ConcurrentHashMap` 是 Java 集合框架的成熟实现，无需 Spring 手动实现哈希表逻辑，且支持动态扩容，能够适配不同规模的 Bean 缓存需求。

   

3. **全局锁 `singletonObjectsMutex` 的作用**

   

   - 定义：在 `DefaultSingletonBeanRegistry` 中，存在一个 `protected final Object singletonObjectsMutex = new Object();`，作为三级缓存操作的全局互斥锁；
   - 作用：主要保护二、三级缓存的 `HashMap` 操作，同时协调三级缓存之间的数据迁移（如从三级缓存迁移到二级缓存、从二级缓存迁移到一级缓存），保证操作的原子性，避免出现数据不一致。

   

### 三、总结

1. Spring 三级缓存底层均基于「哈希表」实现，核心是 Java 中的 `Map` 接口实现类；
2. 一级缓存 `singletonObjects` 用 `ConcurrentHashMap` 保证高并发安全，二、三级缓存用 `HashMap` 配合全局锁实现安全控制；
3. 选择哈希表的核心原因是「O (1) 级高效查询」和「键值对映射」，完美匹配 Spring 容器通过 Bean 名称快速获取缓存数据的需求。
