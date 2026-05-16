> 

# MyBatis XML 特殊字符处理：为什么必须使用 CDATA 包裹 `<=`

> **作者**：MyBatis 最佳实践
> ​**最后更新**​：2025 年 7 月 31 日
> ​**关键词**​：MyBatis, XML 解析, CDATA, SQL 特殊字符, 动态 SQL

## 问题背景：看似简单的运算符陷阱

在 MyBatis 的 XML 映射文件中，以下两种写法看似等价：

xml

复制

```xml
<!-- 错误写法 -->
<if test="end != null">
    and e.entrydate <= #{end}
</if>

<!-- 正确写法 -->
<if test="end != null">
    and e.entrydate <![CDATA[ <= ]]> #{end}
</if>
```

然而直接使用 `<=` **会导致 XML 解析错误**，使应用启动失败！

## 根本原因：XML 解析规则冲突

| 运算符 |            问题根源            |             错误示例             |
| :----: | :----------------------------: | :------------------------------: |
|  `<=`  |     `<` 是 XML 标签起始符      | XML 解析器尝试解释 `<=` 为新标签 |
|  `>=`  | `>` 在特定上下文中可能引发歧义 |         结束标签意外终止         |
|  `&`   |       XML 实体引用起始符       |   `&` 后接字符可能被解释为实体   |

**直接后果**：

复制

```markdown
org.xml.sax.SAXParseException: The element type "<=" must be followed by either attribute specifications, ">" or "/>".
```

## 解决方案：CDATA 保护机制

### 什么是 CDATA？

xml

复制

```xml
<![CDATA[ 任何内容都不会被XML解析器解析 ]]>
```

### 工作原理

![11](C:\Users\admin\OneDrive\博客文章准备\img\11.png)









```

```

### 关键保护机制：

1. 将运算符从 **标签语法领域** 转移到 **纯文本领域**
2. 禁用所有特殊字符的 XML 含义
3. 保持 SQL 代码原始形态（无转义需求）

## 替代方案对比（不推荐）

### XML 实体转义写法

xml

复制

```xml
and e.entrydate &lt;= #{end}
```

|      方案      | 可读性 | 维护成本 | 错误风险 |      适用场景      |
| :------------: | :----: | :------: | :------: | :----------------: |
|     CDATA      |  ★★★   |    ★★    |    低    | 含特殊字符的代码块 |
|    实体转义    |   ★    |   ★★★    |    高    |    单个简单字符    |
| 纯文本(无动态) |  ★★★   |    ★     |    无    |    完全静态 SQL    |

> 🚫 转义方案缺点：
>
> - `<=` 破坏 SQL 可读性
> - 需人工确保所有特殊字符都被转义
> - 复杂表达式易遗漏转义（如 `(a < b OR c > d)`）

## 最佳实践指南

### 1. 动态 SQL 片段标准写法

xml

复制

```xml
<select id="findEmployees">
    SELECT * FROM employee
    <where>
        <if test="start != null">
            <![CDATA[ AND e.entrydate >= #{start} ]]>
        </if>
        <if test="end != null">
            <![CDATA[ AND e.entrydate <= #{end} ]]>
        </if>
    </where>
</select>
```

### 2. 完整 SQL 块保护（适用于静态语句）

xml

复制

```xml
<select id="getRecentHires">
    <![CDATA[
        SELECT id, name, 
               entrydate 
        FROM employee 
        WHERE entrydate <= NOW() 
          AND status = 'A'
        ORDER BY entrydate DESC
    ]]>
</select>
```

### 3. 需要保护的特殊字符列表

| 字符 |  在SQL中的常见用途   | 是否必须CDATA保护 |
| :--: | :------------------: | :---------------: |
| `<`  |  小于(<=), 日期比较  |        是         |
| `>`  | 大于(>=), 比较运算符 |        是         |
| `&`  | 位运算(&), UNION ALL |        是         |
| `"`  |      字符串常量      |   视上下文而定    |
| `'`  |      字符串常量      |    通常不需要     |

### 4. 异常检测技巧

当看到以下错误时，立即检查特殊字符保护：

复制

```markdown
Caused by: org.xml.sax.SAXParseException: 
    Element type "=" must be followed by either attribute specifications...
```

## 实战示例对比

### 错误实现导致的问题

xml

复制

```xml
<!-- 漏洞代码：XML解析器将尝试解析 <= -->
<if test="maxSalary != null">
    AND salary <= #{maxSalary}
</if>
```

**运行结果**：应用启动失败，SAXParseException 异常

### CDATA 保护的健壮实现

xml

复制

```xml
<!-- 安全代码：特殊字符被正确隔离 -->
<if test="maxSalary != null">
    <![CDATA[ AND salary <= #{maxSalary} ]]>
</if>
```

**运行结果**：生成预期SQL → `AND salary <= 10000`

## 核心总结

1. **CDATA 不是可选项**
   对于包含 `<`、`>`、`&` 的动态 SQL 片段，CDATA 是必要防护
2. **本质是 XML 特性**
   该要求由 XML 规范驱动（非 MyBatis 限制）
3. **预防 > 修复**
   建立编码规范：所有比较运算符自动加 CDATA 保护
4. **现代 IDE 辅助**
   推荐使用支持 XML 验证的 IDE（如 IntelliJ IDEA），可实时标出未受保护的运算符

> 💡 **专家提示**：即使是经验丰富的开发者也可能忘记此规则。建议在团队代码评审中将此作为必须检查项！