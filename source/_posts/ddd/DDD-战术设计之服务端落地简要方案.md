---
title: DDD 战术设计之 Java 服务端落地简要方案探讨
urlname: ddd_practice_in_java_server_side
date: 2021-11-16 16:04:22
categories:
tags:
---

## 前言

DDD 全称**领域驱动设计**，最初由 Eric Evans 提出。此文章仅探讨 DDD 在 Java 服务端的落地可行性简要方案，以期不需要太过复杂的前期准备也能够快速在项目中运用 DDD 编写代码

注意，此文章：
- 假定读者有一定的 DDD 理论基础
- 假定读者有 Java 服务端开发经验，并掌握主流的开发框架（如 Spring），本文的许多章节将会结合这些框架进行实现
- 以经典的 RBAC 权限模型为示例进行 DDD 建模

### DDD 编写的代码所属层次

我们把 DDD 设计的相关代码放到 Domain 层，这一层是介于经典三层架构中 Service 与 DAO 层之间的特殊的一层，但严格意义上来说还是属于 Service 层（处理业务逻辑），可以想象成在原先的 Service 层上又划分了一层出来。

如下图所示

TODO::

示例
下面是我们在 JAVA 工程中采用的一个 DDD 包结构规范

TODO::

如果是现有项目，可以在与 controller, service 平级的 package 中再创建一个 domain，这样做的好处是不需要改动现有代码，domain 与 service 可以共存

TODO::

也可以设计得更复杂，但成本较高，一般建议在新的、核心的项目中使用

TODO::


## 核心概念的落地

### 实体

> 以标识作为其基本定义的对象称为实体 - Eric Evans 

实体至少有两个字段：唯一标识 `id` 和 负责该实体持久化工作的 `DAO`。

服务端比较流行 ORM 框架（如 MyBatis），这些框架操作的数据对象（DO）往往是一些纯数据模型，不适合将实体与 DO 混用，最好是以实体依赖 DO 的方式设计，将 DO 定位为存储实体属性的承载者

最终得到一个最小的实体定义如下：

```java
public class User {
    private Long id;
    private UserDAO dao;
    
    public User(Long id, UserDAO dao) {
        this.id = id;
        this.dao = dao;
    }
    
    public UserDO data() {
        return this.dao.selectById(this.id);
    }

    public String sayhello() {
        return this.data().getName() + " say: hello.";
    }
}
```

使用时的代码如下

```java
User user = new User(1L);
user.data();
user.sayhello();
```

实际项目中，实体往往还有很多其它依赖，而且随着业务的发展依赖还会不断增多。依赖多了以后，实体的构造就成了一个大问题，不可能再以 new 的方式去创建实体。因此，需要引入工厂的概念，这将在下文讨论

#### 实体与数据对象的关系

初学者的经常存在一个误区是，喜欢把实体与数据对象（或者说数据库表设计）一一对应起来。这种观念一定要纠正过来，在这里给出的建议是：实体的设计要更多从业务概念的角度去考虑（比如从技术的角度可能会细分出 user_info, user_ext, user_login_history 等等数据对象，但从业务的角度其实都应该是对应到用户这一个实体）。

常见的有以下情况
- 一个实体的属性拆分为多张表存储
- 实体之间的关联关系信息

```java
public class UserDAO {
    private UserInfoMapper userInfoMapper;
    private UserExtMapper userExtMapper;
    private UserLoginHistoryMapper userLoginHistoryMapper;
    
    public UserDO selectById(Long id) {
        build(
            userInfoMapper.selectById(id),
            userExtMapper.selectById(id),
            userLoginHistoryMapper.selectById(id),
        )
    }
}
```

多封装一层 DAO 可以屏蔽持久化的底层实现，方便后续替换（比如某天要对现有服务进行拆分，只需将 mapper 替换成 feign client 即可），但同时也会增加工作量。

一种更加简便的方式是直接在实体中引用 mapper 进行操作。

```java
public class User {
    private Long id;
    private UserInfoMapper userInfoMapper;
    private UserExtMapper userExtMapper;
    private UserLoginHistoryMapper userLoginHistoryMapper;

    public UserDO data() {
        ...
    }
}
```

此外，出于高内聚考虑，实体不应该直接操作不属于自己管辖的数据对象。如，User 不应该通过 RoleMapper 直接操作 RoleDO

#### 引用

考虑到在服务端开发中，有状态的对象朝生夕灭的情况非常常见（服务端要管理的对象非常多，不可能将所有实体都存在内存中，一般一个请求过来时会创建对象，请求结束后在下一次 GC 这个对象就会被销毁），而实体之间的关联可能是非常复杂的，每次使用时都构建一个完整的聚合非常不划算，比较建议实体间的聚合采用软关联的方式

可以看到以下两种方式的区别：

##### 硬关联

```java
public class User {
    private Long id;
    private List<Role> roles;
    
    public User(Long id, List<Role> roles) {
        this.id = id;
        this.roles = roles;
    }
    
    public List<Role> listAllRoles() {
        return this.roles;
    }
}
```

##### 软关联

```java
public class User {
    private Long id;
    private RoleRepository roleRepo;
    
    public User(Long id, RoleRepository roleRepo) {
        this.id = id;
        this.roleRepo = roleRepo;
    }
    
    public List<Role> listAllRoles() {
        return this.roleRepo.listAllByUserId(this.getId());
    }
}
```

两者在使用方式并没有区别

```java
for (Role role : user.listAllRoles()) {
    role.dosomething()
}
```


### 工厂

虽然在上面我们采用了软关联的方式建立实体之间的引用关系，但这并不代表要构建一个实体就非常简单了，原因是我们的实体除了依赖其它实体外，往往还需要依赖许多其它对象（如领域服务、仓储、DAO 等），并且随着业务的变化，实体的依赖往往还会随之发生变化，如果还是通过传统的 new 方式去创建一个实体，会产生一些灾难性的问题：

- 使用者必须清楚实体的创建细节，这会大大增加代码的复杂度
- 每当实体的构造方式发生变化时，不得不调整所有创建实体的代码逻辑以解决代码编译问题

这个时候就需要引入工厂（Factory）的概念了，一个通用 Factory 的实现示例如下

```java
@Component
public abstract class Factory {
    @Autowired
    private static UserRepository userRepo;
    
    public User getUser(Long id) {
        return new User(id, userRepo);
    }
}
```

结合 Spring，可以把实体的依赖注入做得更简单，并且在实体 User 的依赖变化后不需要做任何代码变更

```java
public abstract class Factory {
    public User getUser(Long id) {
        User user = new User(id);
        SpringUtils.inject(user);       // 结合 AOP 可以进一步简化装配逻辑
        return user;
    }
}
```

### 实体仓储（Repository） TODO::

仓储可以为使用者提供实体的创建、删除及条件查询操作。在 C/S 中，仓储还要负责内存中的实体的更新（保证数据一致性）及缓存管理（防止频繁地重复创建，影响性能）

在 B/S 架构中，我们将实体与数据对象分离，因此数据一致性问题交给 ORM 去控制即可，仓储

仓储往往依赖 DAO（查询持久化数据）及工厂（创建实体），并且应可以发布领域事件。

```java

```


### 领域服务

领域服务用于处理一些在概念上不属于实体的操作，这些操作本质上往往是一些活动或行为，并且是无状态的。对于这类操作，将其强制进行归类会显得非常别扭，于是便引入了领域服务这一概念。

需要明确的是，其与三层架构的 Service 层（业务逻辑层）并不是一个概念。另外与 Evans 在书中提及的示例不同，为了避免混乱，一般不建议为领域服务的类命名加上 Service 后缀。
可以简单理解为，领域中没有任何实体适合承载该职责，因此我们创造了一个『实体』来承担，只不过这个特殊的『实体』是一个无状态的单例而以。

#### 示例
在某个应用中，由于用户量较大，用户登录历史可能会逐渐变得非常庞大，因此需要需要定时查找超过 15 天的记录并将其清理。

显然，我们需要完成的这个操作无法归类到任何一个实体中，因此我们需要一个名为 LoginHistoryClearer 的领域服务来承接此职责

```java
@Compoment
public class LoginHistoryClearer {
    private UserRepository userRepo;
    
    public void clearOutDated(Integer interval) {
        for (User user : userRepo.listAll()) {
            user.removeOutDatedLoginHistory(interval);
        }
    }
}
```

在其它地方，我们可以直接注入该领域服务，并使用

```java
@Slf4j
@Component
public class UserScheduledTask {
    @Autowired
    private LoginHistoryClearer clearer;

    @Value("${exec.output.interval.days:15}")
    private Integer intervalDays;

    @Scheduled(cron = "0 0 0 * * ?")
    public void deleteExecData() {
        log.info("starting clear out dated data, intervalDays=>{}", intervalDays);
        clearer.clearOutDated(intervalDays);
        log.info("clear out dated data end");
    }
}
```

### 领域事件

在我们的领域活动（实体、仓储等操作）中会出现一系列的重要的事件，而这些事件的订阅者，往往需要对这些事件作出响应（例如，新增用户后，可能会触发一系列动作：发送欢迎信息、发放优惠券等等）。领域事件可以简单地理解为是发布订阅模式在 DDD 中的一种运用。

在我们的实践中，一般采用事件总线来快速地发布一个领域事件。

事件总线的接口定义一般如下

```java
public interface EventBus {
    void post(Event event);
}
```

通过调用 EventBus.post() 方法，我们可以快速发布一个事件。

同时我们还会提供一个抽象类 AbstractEventPublisher 

```java
public class AbstractEventPublisher implements EventPublisher {
    private EventBus eventBus;

    public void setEventBus(EventBus eventBus) {
        this.eventBus = eventBus;
    }

    @Override
    public void publish(Event event) {
        if (eventBus != null) {
            eventBus.post(event);
        } else {
            log.warn("event bus is null. event " + event.getClass() + " will not be published!");
        }
    }
}
```

```java
public interface EventPublisher {
    void publish(Event event);
}
```

这样我们可以让实体或 Manager 继承自 AbstractEventPublisher，其便有了发布事件的能力。至于如何订阅并处理这些事件，取决于 EventBus 的实现方式。举个例子，我们一般使用 Guava 的 EventBus，定义相关的 handler 并注册到 EventBus 中便可方便地处理这些事件

```java
@Component
public class DomainEventBus extends EventBus implements InitializingBean {
    @Autowired
    private FooEventHandler fooEventHandler;

    @Override
    public void afterPropertiesSet() {
        this.register(fooEventHandler);
    }
}

@Component
@Slf4j
public class FooEventHandler implements DomainEventHandler {
    @Override
    @Subscribe
    public void listen(ProjectCreatEvent e) {
        // do something here...
    }
}
```

其它
TODO::
● ddd-support

### DDD 设计

理解了 DDD 中的全部概念，也并不意味着就能做出一个好的设计了。

<!-- DDD 设计中最重要的其实是实体的定义， -->

DDD 的设计最重要的是做好以下几点：
1. 准确地定义实体
2. 准确地定义实体应该有哪些方法
3. 确立实体与实体之间的关系

实体的设计其实是一个建模的过程。面向对象的设计方法本质就是将现实世界的对象关系以简化的形式提炼为模型。关于这一块，已经是一个更大的话题了，不在这里讨论。


案例工程
TODO::



## 其它

### 与现有三层架构是否冲突

并不冲突， 甚至是可以混用的

### 事务问题如何解决

实体操作是可以兼容 JDBC 事务，在编排实体的应用层中加上 Spring 事务注解 @Transaction 即可


### 重复查询的性能问题

如下，会执行三次数据库查询

```java
log.debug(user.data());
log.info(user.data());
log.warn(user.data());
```

实际项目中，应使用带缓存的 ORM 框架（如 MyBatis），这样便可避免同一事务中重复查询带来的性能问题。否则应注意优化编码方式，如下

```java
UserDO data = user.data();
log.debug(data);
log.info(data);
log.warn(data);
```

### 复杂查询场景下的性能问题

DDD 要求将数据对象转换为内存中的实体对象后再进行业务操作，这注定了 DDD 不擅长批量查询以及联表查询等复杂的查询场景。业界采用的方案是 CQRS 模式，将查询操作从领域模型中分离出去。

要实现也很简单，有查询业务时，直接定义一个相应的 Service，在里面操作数据库完成查询即可。

更复杂一点，可以专门为查询操作开一个微服务，实现物理上的分离。


示例如下

不分离，会存在对 dto, vo 等对象的直接引用，导致领域模型被污染

```java
public class User {
    public List<UserLoginHistoryVO> queryLoginHistory(QueryDTO dto) {
        return dao.queryLoginHistory(...)
    }
}
```

分离后，查询操作转移到专门的查询 Service 中

```java
public class UserQueryService {
    public List<UserLoginHistoryVO> queryLoginHistory(QueryDTO dto) {
        return dao.queryLoginHistory(...)
    }
}
```

