> # PageHelper分页插件使用指南
>
> > 本文档全面介绍PageHelper插件的核心功能、使用方法和注意事项，帮助开发者高效实现MyBatis分页
>
> ## 一、插件概述
>
> PageHelper是基于MyBatis的**分页拦截器**，通过动态改写SQL实现分页功能：
>
> ![微信截图_20250729183016](C:\Users\admin\OneDrive\博客文章准备\img\微信截图_20250729183016.png)
>
> ## 二、快速开始
>
> ### 1. 添加依赖
>
> xml
>
> 复制
>
> ```xml
> <dependency>
>     <groupId>com.github.pagehelper</groupId>
>     <artifactId>pagehelper</artifactId>
>     <version>5.3.2</version> <!-- 使用最新版本 -->
> </dependency>
> ```
>
> ### 2. 基本配置
>
> **XML配置 (mybatis-config.xml):**
>
> xml
>
> 复制
>
> ```xml
> <plugins>
>     <plugin interceptor="com.github.pagehelper.PageInterceptor">
>         <property name="helperDialect" value="mysql"/>
>         <property name="reasonable" value="true"/>
>         <property name="supportMethodsArguments" value="true"/>
>     </plugin>
> </plugins>
> ```
>
> **Spring Boot配置 (application.yml):**
>
> yaml
>
> 复制
>
> ```yaml
> pagehelper:
>   helper-dialect: mysql
>   reasonable: true
>   support-methods-arguments: true
> ```
>
> ## 三、核心使用方式
>
> ### 1. 基础分页
>
> ```java
> // 分页查询：第2页，每页10条
> PageHelper.startPage(2, 10);
> 
> List<User> users = userMapper.selectAll(); 
> 
> // 转换为PageInfo对象
> PageInfo<User> pageInfo = new PageInfo<>(users);
> ```
>
> ### 2. 使用PageInfo（推荐）
>
> ```java
> pageInfo.getTotal();      // 总记录数
> pageInfo.getList();       // 当前页数据 
> pageInfo.getPageNum();    // 当前页码
> pageInfo.getPages();      // 总页数
> pageInfo.getNavigatePages(5); // 获取5页导航页
> ```
>
> ### 3. 高级查询模式
>
> ```java
> PageHelper.startPage(1, 20)
>          .setOrderBy("create_time DESC")
>          .doSelectPage(() -> {
>              userMapper.findByConditon(condition);
>          });
> ```
>
> ## 四、分页对象对比
>
> ### Page与PageInfo核心区别
>
> |     **特性**     |    `Page`对象    |   `PageInfo`对象    |
> | :--------------: | :--------------: | :-----------------: |
> |   **继承关系**   |  ArrayList子类   |      独立POJO       |
> |   **包含数据**   | 本身就是数据列表 | 通过`getList()`获取 |
> |   **导航信息**   |       ❌ 无       |   ✅ 完整导航信息    |
> | **序列化友好度** |   需要特殊处理   |     ✅ 开箱即用      |
> |   **内存占用**   |      轻量级      | 较高（含导航数据）  |
> |   **适用场景**   |   简单分页展示   |  带导航的复杂分页   |
>
> ## 五、关键注意事项
>
> ### 1. 执行顺序要求（最重要）
>
> ```java
> // ✅ 正确示例
> PageHelper.startPage(pageNum, pageSize);
> List<User> users = userMapper.selectUsers();
> 
> // ❌ 错误示例
> List<User> users = userMapper.selectUsers();
> PageHelper.startPage(pageNum, pageSize); // 无效
> ```
>
> ### 2. 线程安全处理
>
> 在多线程环境下务必及时清理ThreadLocal：
>
> ```java
> try {
>     PageHelper.startPage(1, 10);
>     return userMapper.selectAll();
> } finally {
>     // 确保清除ThreadLocal数据
>     PageHelper.clearPage(); 
> }
> ```
>
> ### 3. count优化策略
>
> 针对大数据表优化count查询：
>
> ```xml
> <select id="selectWithLargeData" resultType="User">
>     SELECT /*! 分页不统计字段 */ name, email FROM large_table
> </select>
> ```
>
> ```java
> PageHelper.startPage(1, 100, false) // 关闭自动count
>           .setCount(() -> customCountMethod());
> ```
>
> ### 4. 特殊分页场景处理
>
> **物理分页VS逻辑分页：**
>
> ```java
> // 内存分页（小数据集）
> PageHelper.startPage(1, 100, false, true, true); 
> 
> // 物理分页（大数据集）
> PageHelper.startPage(1, 1000, true); 
> ```
>
> **获取所有列总数但仅返回部分列：**
>
> ```sql
> /* 需要统计的字段 */
> SELECT SQL_CALC_FOUND_ROWS id, name FROM users;
> 
> /* 获取实际总数 */
> SELECT FOUND_ROWS() AS total;
> ```
>
> ## 六、高级功能指南
>
> ### 1. 多表联合查询分页
>
> ```sql
> <!-- 需在SQL中明确指定主表字段 -->
> SELECT u.*, d.name as dept_name 
> FROM users u 
> LEFT JOIN department d ON u.dept_id = d.id
> ```
>
> 配置参数：
>
> ```properties
> pagehelper.count-sql-func=count(0)
> ```
>
> ### 2. 分页拦截器扩展点
>
> 自定义拦截器实现：
>
> ```java
> public class CustomPageInterceptor extends PageInterceptor {
>     @Override
>     protected void afterAll() {
>         // 自定义分页后处理逻辑
>     }
> }
> ```
>
> ## 七、常见问题解决方案
>
> ### Q1：分页后返回的总数错误
>
> **原因排查：**
>
> 1. 多表JOIN统计不准确
> 2. 自动生成的COUNT语句错误
> 3. SQL中存在GROUP BY
>
> **解决方案：**
>
> ```java
> PageHelper.startPage(1, 10)
>     .doCount(true) // 强制使用自定义count
>     .setCountSqlParser(new CustomCountParser()); 
> ```
>
> ### Q2：分页页码跳转异常
>
> **配置参数：**
>
> ```yaml
> pagehelper:
>   reasonable: true       # 修正异常页码
>   page-size-zero: true   # 允许pageSize=0返回全部结果
>   params: count=countSql 
> ```
>
> ### Q3：性能问题（千万级数据）
>
> 优化策略：
>
> ```java
> // 1. 禁用自动count
> PageHelper.startPage(1, 100, false);
> 
> // 2. 使用并行count查询
> CompletableFuture<Long> countFuture = supplyAsync(() -> userMapper.count());
> 
> // 3. 分批次获取数据
> PageHelper.offsetPage(0, 1000);
> while (hasMore) {
>     List<User> batch = userMapper.selectLarge();
>     PageHelper.offsetPage(batch.size(), 1000);
> }
> ```
>
> ## 八、最佳实践建议
>
> 1. **统一分页返回格式**
>
> ```java
> public class PageResult<T> {
>     private int page;
>     private int pageSize;
>     private long total;
>     private List<T> items;
>     
>     public static <T> PageResult<T> of(PageInfo<T> pageInfo) {
>         return new PageResult<>(
>             pageInfo.getPageNum(),
>             pageInfo.getPageSize(),
>             pageInfo.getTotal(),
>             pageInfo.getList()
>         );
>     }
> }
> ```
>
> 1. **Controller层统一处理**
>
> ```java
> @GetMapping("/users")
> public PageResult<User> getUsers(
>     @RequestParam(defaultValue = "1") int page,
>     @RequestParam(defaultValue = "10") int size
> ) {
>     return userService.getPagedUsers(page, size);
> }
> ```
>
> 1. **使用AOP简化分页**
>
> ```java
> @Around("execution(* *..*Service.*Paged(..))")
> public Object handlePagination(ProceedingJoinPoint joinPoint) {
>     Object[] args = joinPoint.getArgs();
>     PageParam param = extractPageParam(args); // 自定义参数提取
>     
>     try {
>         PageHelper.startPage(param.getPage(), param.getSize());
>         return joinPoint.proceed();
>     } finally {
>         PageHelper.clearPage();
>     }
> }
> ```
>
> ## 九、版本升级说明
>
> | 版本  |       重要变更        |    迁移建议     |
> | :---: | :-------------------: | :-------------: |
> | v4.x  |        旧版API        | 建议升级到v5.x  |
> | v5.0  | 重构为PageInterceptor | 需修改配置类名  |
> | v5.1+ | 增强对Spring Boot支持 | 推荐使用starter |
> | v5.3+ |     支持JDK8+特性     |  要求JDK8+环境  |
>
> > 官方文档：https://github.com/pagehelper/Mybatis-PageHelper
>
> 通过本文档，您应该能够掌握PageHelper的核心使用方法和关键优化策略。在实际项目中，请根据数据量和业务场景选择合适的分页策略。