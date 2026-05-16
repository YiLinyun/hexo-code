## MyBatis驼峰映射失效问题分析与修复

### 🐞 问题描述

在接口测试过程中，`Emp`实体类属性（小驼峰命名：`entryDate`, `deptId`）与数据库字段（下划线命名：`entry_date`, `dept_id`）映射失败。测试结果中相关字段始终为`null`：

```java
entryDate=null, deptId=null, createTime=null, updateTime=null, deptName=null
```

### 🔍 根本原因排查

通过分析测试环境配置和代码结构，定位到核心问题：

1. **测试环境配置缺失**
   `test`目录下的专属配置文件**未开启驼峰映射**​：

   yaml

   复制

   ```yaml
   # test/resources/application.yml
   mybatis:
     mapper-locations: classpath:mapper/*.xml
     # 缺少驼峰映射配置！！！
     # map-underscore-to-camel-case未配置
   ```

2. **主配置与测试配置隔离**

   - 主工程配置（`main/resources/`）已正确开启驼峰映射
   - **测试环境配置独立**且未继承主配置，导致映射开关失效

3. **SQL结果集与POJO差异**

   ```sql
   SELECT * FROM emp  -- 实际查询字段为entry_date, dept_id
   ```

   - 未自动映射到POJO的`entryDate`, `deptId`
   - `deptName`字段在emp表中不存在（需关联查询）

### 🛠 解决方案

#### 修复步骤

1. **补充测试环境配置**
   在`test/resources/application.yml`中添加驼峰映射：

   ```yaml
   mybatis:
     configuration:
       map-underscore-to-camel-case: true  # ✅ 关键修复点
     mapper-locations: classpath:mapper/*.xml
     type-aliases-package: com.itheima.pojo
   ```
   
2. **优化SQL查询（可选）**
   解决`deptName`字段映射问题：

   ```sql
   SELECT e.*, d.name AS deptName 
   FROM emp e LEFT JOIN dept d ON e.dept_id = d.id
   ```
   
3. **验证配置优先级**
   确保测试配置覆盖全局配置：

   ```java
   @SpringBootTest(properties = {
       "mybatis.configuration.map-underscore-to-camel-case=true"
   })
   ```

#### 修正前后对比

|     配置位置     | 修复前状态 | 修复后状态 |
| :--------------: | :--------: | :--------: |
| `main/resources` | ✅ 正确配置 |   ✅ 保持   |
| `test/resources` | ❌ 缺失配置 |   ✅ 补全   |

### ✅ 验证结果

修复后重新执行测试：

```java
Emp{
  entryDate=2000-01-01, 
  deptId=2, 
  createTime=2023-10-20T16:35:33,
  updateTime=2023-11-16T16:11:26,
  deptName=研发部  // 需配合SQL优化
}
```

**字段映射成功**，问题解决！

### 📌 经验总结

1. **环境隔离陷阱**
   测试环境配置需显式声明关键参数，不可依赖主配置

2. **驼峰配置规范**

   ```yaml
   # 正确位置（层级关系）
   mybatis.configuration.map-underscore-to-camel-case=true 
   ```
   
3. **防御性配置检查**

   - 新增`@Test`验证映射基础功能：

   ```java
   @Test
   void testCamelMapping() {
       assertThat(mybatisConfig.isMapUnderscoreToCamelCase()).isTrue();
   }
   ```
   
   - SQL日志添加字段名输出验证：
   
   ```sql
   SELECT entry_date AS "entryDate"  -- 显式别名验证
   ```

> **永远记住**：测试环境不是主环境的副本！需要独立验证核心配置的有效性。