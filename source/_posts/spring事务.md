> # Spring事务管理深度笔记
>
> ## 一、核心概念
>
> ### 1. ACID特性在Spring中的体现
>
> - **原子性(Atomicity)**：通过`@Transactional`保证操作要么全部成功，要么全部回滚
> - **一致性(Consistency)**：数据库约束+业务校验双重保障
> - **隔离性(Isolation)**：通过隔离级别控制并发访问
> - **持久性(Durability)**：事务提交后数据持久化到存储
>
> ### 2. 事务关键属性
>
> ```java
> @Transactional(
>     propagation = Propagation.REQUIRED, // 传播行为
>     isolation = Isolation.DEFAULT,      // 隔离级别
>     timeout = 30,                       // 超时时间(秒)
>     readOnly = false,                   // 只读事务
>     rollbackFor = Exception.class,      // 回滚异常
>     noRollbackFor = BusinessException.class // 不回滚异常
> )
> ```
>
> ## 二、事务配置方式
>
> ### 1. 声明式事务（推荐）
>
> ```java
> // 启动类配置
> @EnableTransactionManagement
> public class AppConfig {
>     @Bean
>     public PlatformTransactionManager txManager(DataSource dataSource) {
>         return new DataSourceTransactionManager(dataSource);
>     }
> }
> 
> // 业务层使用
> @Service
> public class UserService {
>     @Transactional
>     public void createUser(User user) {
>         // ...
>     }
> }
> ```
>
> ### 2. 编程式事务
>
> ```java
> @Service
> public class OrderService {
>     private final TransactionTemplate transactionTemplate;
> 
>     public OrderService(PlatformTransactionManager txManager) {
>         this.transactionTemplate = new TransactionTemplate(txManager);
>     }
> 
>     public void processOrder() {
>         transactionTemplate.execute(status -> {
>             try {
>                 // 业务操作...
>                 return true;
>             } catch (Exception e) {
>                 status.setRollbackOnly();
>                 return false;
>             }
>         });
>     }
> }
> ```
>
> ## 三、传播行为详解
>
> |     传播行为      |    值    |                         特点                         |
> | :---------------: | :------: | :--------------------------------------------------: |
> |   **REQUIRED**    |   默认   |               存在事务则加入，否则新建               |
> |   **SUPPORTS**    |   支持   |            存在事务则加入，否则非事务运行            |
> |   **MANDATORY**   |   强制   |               必须存在事务，否则抛异常               |
> | **REQUIRES_NEW**  | 新建事务 |             挂起当前事务，创建独立新事务             |
> | **NOT_SUPPORTED** |  不支持  |               挂起当前事务，非事务执行               |
> |     **NEVER**     |   禁止   |              必须非事务执行，否则抛异常              |
> |    **NESTED**     |   嵌套   | 在当前事务内创建保存点，可部分回滚（需JDBC3.0+驱动） |
>
> **嵌套事务示意图**：
>
> ![1](C:\Users\admin\OneDrive\博客文章准备\img\2.png)
>
> 
>
> 
>
> 
>
> 
>
> 
>
> ```
> 
> ```
>
> ## 四、隔离级别
>
> |       隔离级别       | 脏读 | 不可重复读 | 幻读 |      适用场景      |
> | :------------------: | :--: | :--------: | :--: | :----------------: |
> | **READ_UNCOMMITTED** |  ✓   |     ✓      |  ✓   |      极少使用      |
> |  **READ_COMMITTED**  |  ✗   |     ✓      |  ✓   | 默认级别（Oracle） |
> | **REPEATABLE_READ**  |  ✗   |     ✗      |  ✓   | 默认级别（MySQL）  |
> |   **SERIALIZABLE**   |  ✗   |     ✗      |  ✗   |  高一致性要求场景  |
>
> **设置方式**：
>
> ```java
> @Transactional(isolation = Isolation.REPEATABLE_READ)
> public void updateAccount() {
>     // ...
> }
> ```
>
> ## 五、回滚机制
>
> ### 1. 默认行为
>
> - 对`RuntimeException`和`Error`回滚
> - 对`受检异常(Checked Exception)`不回滚
>
> ### 2. 自定义回滚
>
> ```java
> // 指定回滚异常
> @Transactional(rollbackFor = {IOException.class, SQLException.class})
> 
> // 指定不回滚异常
> @Transactional(noRollbackFor = BusinessException.class)
> ```
>
> ### 3. 手动回滚
>
> ```java
> @Transactional
> public void update() {
>     try {
>         // 业务操作...
>     } catch (Exception e) {
>         // 手动标记回滚
>         TransactionAspectSupport.currentTransactionStatus().setRollbackOnly();
>         throw new ServiceException("操作失败", e);
>     }
> }
> ```
>
> ## 六、常见错误与解决方案
>
> ### 1. 事务不生效（最常见）
>
> **原因**：
>
> - 方法非public修饰
> - 自调用（类内部方法调用）
> - 异常被捕获未抛出
> - 数据库引擎不支持（如MyISAM）
> - 未启用事务管理（缺少`@EnableTransactionManagement`）
>
> **解决方案**：
>
> ```java
> // 正确示例
> @Service
> public class UserService {
>     // 必须通过代理对象调用
>     @Autowired 
>     private UserService selfProxy;
> 
>     public void outerMethod() {
>         // 错误：直接调用
>         // this.innerMethod(); 
>         
>         // 正确：通过代理对象
>         selfProxy.innerMethod();
>     }
> 
>     @Transactional
>     public void innerMethod() {
>         // ...
>     }
> }
> ```
>
> ### 2. 长事务问题
>
> **现象**：
>
> - 数据库连接池耗尽
> - 锁等待超时
> - 系统响应缓慢
>
> **优化方案**：
>
> ```java
> @Transactional(timeout = 5) // 设置合理超时
> public void processBatch() {
>     // 1. 分批处理数据
>     for (int i = 0; i < total; i += BATCH_SIZE) {
>         processChunk(i, BATCH_SIZE);
>     }
>     
>     // 2. 避免在事务中进行IO操作
>     // 3. 只读查询使用@Transactional(readOnly=true)
> }
> ```
>
> ### 3. 事务嵌套混乱
>
> **典型错误**：
>
> ```java
> @Service
> public class OrderService {
>     @Transactional
>     public void createOrder() {
>         // ...
>         paymentService.processPayment(); // REQUIRES_NEW
>         logService.saveLog();            // REQUIRED
>     }
> }
> 
> @Service
> public class PaymentService {
>     @Transactional(propagation = Propagation.REQUIRES_NEW)
>     public void processPayment() {
>         // ...
>     }
> }
> 
> @Service
> public class LogService {
>     @Transactional // 默认REQUIRED
>     public void saveLog() {
>         // ...
>     }
> }
> ```
>
> **风险点**：
>
> - REQUIRES_NEW可能因获取新连接导致连接池耗尽
> - 嵌套事务过多增加数据库压力
> - 异常传播路径复杂
>
> ### 4. 多数据源事务
>
> **解决方案**：
>
> ```java
> @Configuration
> public class TransactionConfig {
>     // 主数据源事务管理器
>     @Bean
>     @Primary
>     public PlatformTransactionManager primaryTxManager(DataSource dataSource) {
>         return new DataSourceTransactionManager(dataSource);
>     }
>     
>     // 次数据源事务管理器
>     @Bean
>     public PlatformTransactionManager secondaryTxManager(
>         @Qualifier("secondaryDataSource") DataSource dataSource) {
>         return new DataSourceTransactionManager(dataSource);
>     }
> }
> 
> // 使用指定事务管理器
> @Service
> public class CrossService {
>     @Transactional(transactionManager = "secondaryTxManager")
>     public void crossOperation() {
>         // ...
>     }
> }
> ```
>
> ## 七、最佳实践
>
> 1. **事务边界**：
>
>    - 在Service层使用事务，Controller层保持无状态
>    - 单个事务不超过5个SQL操作
>
> 2. **异常处理**：
>
>    ```java
>    @Transactional
>    public void businessOperation() {
>        try {
>            // 可能失败的操作
>        } catch (BusinessException e) {
>            // 记录日志但继续执行
>            log.error("业务异常", e);
>        }
>        
>        // 其他操作...
>        // 不要捕获RuntimeException!
>    }
>    ```
>
> 3. **性能优化**：
>
>    ```java
>    // 只读查询优化
>    @Transactional(readOnly = true)
>    public List<User> getUsers() {
>        return userRepository.findAll();
>    }
>    
>    // 短事务原则
>    @Transactional(propagation = Propagation.REQUIRES_NEW, timeout = 3)
>    public void auditLog() {
>        // 快速审计日志记录
>    }
>    ```
>
> 4. **分布式事务**：
>
>    - 跨服务调用使用Seata/TCC模式
>    - 消息队列保证最终一致性
>
>    ```java
>    @Transactional
>    public void placeOrder() {
>        // 1. 本地事务
>        orderDao.create(order);
>        
>        // 2. 发送MQ消息（事务消息）
>        rocketMQTemplate.sendMessageInTransaction(...);
>    }
>    ```
>
> ## 八、调试与监控
>
> ### 1. 事务日志配置
>
> ```properties
> # application.properties
> logging.level.org.springframework.transaction.interceptor=TRACE
> logging.level.org.springframework.jdbc.datasource.DataSourceTransactionManager=DEBUG
> ```
>
> ### 2. 监控指标（Spring Boot Actuator）
>
> ```yaml
> management:
>   endpoints:
>     web:
>       exposure:
>         include: transactions
> ```
>
> ### 3. 事务状态获取
>
> ```java
> @Autowired
> private TransactionSynchronizationManager synchronizationManager;
> 
> public void checkTransaction() {
>     boolean active = synchronizationManager.isActualTransactionActive();
>     TransactionStatus status = TransactionAspectSupport.currentTransactionStatus();
> }
> ```
>
> ## 总结图：Spring事务全景
>
> ![1](C:\Users\admin\OneDrive\博客文章准备\img\1.png)
>
> 
>
> 
>
> 
>
> 
>
> 
>
> ```
> 
> ```
>
> 掌握Spring事务管理的关键在于理解事务边界、合理选择传播行为、正确处理异常，并避免常见陷阱。在实际项目中，建议结合APM工具进行事务链路监控，确保系统在高并发场景下的数据一致性。
