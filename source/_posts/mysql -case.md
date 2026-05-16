> markdown
>
> 复制
>
> ~~~markdown
># MySQL `CASE` 函数详解
> 
> ## 📌 核心功能
> `CASE` 是 MySQL 中的**条件表达式**，用于在查询中实现逻辑分支（类似编程中的 `if-else`/`switch`）
> 
>## 📐 两种语法形式
> 
>### 1. 简单 CASE 表达式（精确匹配）
> ```sql
> CASE 字段名
>     WHEN 值1 THEN 结果1
>       WHEN 值2 THEN 结果2
>     ...
>     ELSE 默认结果
> END
> ~~~
> 
> **应用场景**：字段值精确匹配时
> ​**示例**​：
> 
>  sql
> 
> 复制
> 
> ```sql
> SELECT 
>     product_id,
>     CASE category_id 
>         WHEN 1 THEN '电子产品'
>         WHEN 2 THEN '家居用品'
>         ELSE '其他'
>       END AS category
> FROM products;
> ```
> 
> ### 2. 搜索 CASE 表达式（复杂条件）
> 
> sql
> 
> 复制
> 
> ```sql
> CASE
>     WHEN 条件1 THEN 结果1
>      WHEN 条件2 THEN 结果2
>     ...
>     ELSE 默认结果
> END
> ```
> 
> **应用场景**：需要使用比较运算符（`>`, `<`, `BETWEEN`）或逻辑运算时
> ​**示例**​：
> 
> sql
> 
>   复制
> 
> ```sql
> SELECT 
>     student_name,
>     CASE
>         WHEN score >= 90 THEN 'A'
>         WHEN score >= 80 THEN 'B'
>              WHEN score >= 60 THEN 'C'
>         ELSE 'D'
>     END AS grade
> FROM exam;
> ```
> 
> ## ⚠️ 关键特性
> 
>      1. **执行顺序**：从上到下判断，**首个满足条件的分支**生效
> 
> 2. 
> 
>    返回值
> 
>         ：
> 
>    - 所有结果应有兼容的数据类型（统一返回字符串/数字）
>    - 未匹配且未指定 `ELSE` 时返回 `NULL`
>     
> 3. **命名**：使用 `AS` 给结果列命名（如：`END AS result`）
> 
> ## 🔍 解析分组统计示例
> 
> sql
>     
> 复制
> 
> ```sql
>     SELECT
>     CASE job 
>         WHEN 1 THEN '班主任' 
>         WHEN 2 THEN '讲师' 
>         WHEN 3 THEN '学工主管' 
>         WHEN 4 THEN '教研主管' 
>             WHEN 5 THEN '咨询师' 
>         ELSE '其他' 
>     END AS pos,
>     COUNT(*) total
> FROM emp 
> GROUP BY job  -- ✅ 分组依据是原始job值
> ORDER BY total;  -- 按计数结果升序排序
> ```
> 
>### ✨ 代码解析：
> 
>|          组件           |                      说明                       |
> | :---------------------: | :---------------------------------------------: |
> | `CASE job...END AS pos` |            将数字职位ID转为中文名称             |
> |    `COUNT(*) total`     |              统计每个分组的记录数               |
> |     `GROUP BY job`      | **关键点**：按原始`job`值分组（非转换后的`pos`) |
>     |    `ORDER BY total`     |               按统计数量升序排列                |
> 
> ### 🔥 典型应用场景：
>     
> 1. 数据透视表统计
> 
>    sql
> 
>    复制
> 
>    ```sql
>    -- 统计各成绩段人数
>    SELECT
>      COUNT(CASE WHEN score >= 90 THEN 1 END) AS 'A',
>      COUNT(CASE WHEN score >= 80 THEN 1 END) AS 'B'
>    FROM students;
>    ```
> 
> 2. 条件排序
> 
>    sql
> 
>    复制
> 
>    ```sql
>         ORDER BY CASE status 
>         WHEN '紧急' THEN 1 
>         WHEN '高' THEN 2 
>         ELSE 3 
>    END
>    ```
>      
> 3. 字段值转换显示
> 
>   sql
> 
>   复制
> 
>   ```sql
>    SELECT 
>     order_id,
>      CASE status_code 
>         WHEN 'P' THEN '已付款' 
>         WHEN 'S' THEN '已发货'
>      END AS status
>    FROM orders;
>   ```
>     
> ## 💡 重要注意事项
> 
>     1. **分组陷阱**：使用 `GROUP BY` 时，确保理解分组依据是**原始字段值**而非转换结果
> 2. **性能优化**：对大数据表避免在 `WHERE` 子句中使用 `CASE`
> 3. **多分支合并**：相同逻辑结果可合并分支（如 `WHEN 4 OR 5 THEN '管理岗'`）
> 
> > 通过 `CASE` 可实现灵活的**数据清洗、分类统计和结果转换**，是 SQL 查询中最强大的工具之一！
> 
>     复制
> 
> ```markdown
> undefined
> ```

#### IF用法

```XML
<!-- 统计员工的性别信息 -->
<select id="countEmpGenderData" resultType="java.util.Map">
    select
    if(gender = 1, '男', '女') as name,
    count(*) as value
    from emp group by gender ;
</select>
```

if函数语法：`if(条件, 条件为true取值, 条件为false取值)`

ifnull函数语法：`ifnull(expr, val1)`    如果expr不为null，取自身，否则取val1
