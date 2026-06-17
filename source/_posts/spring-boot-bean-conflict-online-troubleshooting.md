---
title: Spring Boot 线上启动报 Bean 冲突的一次排查记录
date: 2026-06-17 11:30:00
tags:
  - Spring Boot
  - Java
  - 线上问题排查
  - 部署
categories:
  - 后端排查
---

## 问题现象

最近在部署一个 Spring Boot 项目时，遇到了一个比较迷惑的问题：

**测试服务器可以正常启动，线上服务器启动失败。**

线上启动时报错如下：

```text
Application run failed
org.springframework.beans.factory.BeanDefinitionStoreException: Failed to parse configuration class [com.muni.VideoTuberApplication]

Caused by: org.springframework.context.annotation.ConflictingBeanDefinitionException:
Annotation-specified bean name 'apiController' for bean class [com.muni.web.api.ApiController]
conflicts with existing, non-compatible bean definition of same name and class [com.muni.web.api.old.ApiController]
```

从错误信息看，Spring 扫描到了两个同名 Bean：

```text
com.muni.web.api.ApiController
com.muni.web.api.old.ApiController
```

因为两个类名都叫 `ApiController`，默认 Bean 名都是：

```text
apiController
```

所以 Spring 启动时报了 Bean 冲突。

## 第一反应：是不是 jar 包有问题？

本地启动没有问题，测试服务器也能启动，只有线上服务器失败。

于是第一步怀疑的是：线上服务器下载到的 jar 包是不是旧的？

线上部署脚本会从 S3 下载：

```bash
aws s3 cp s3://xxx/pockly/video_tuber_admin.jar /workspace/pockly/video_tuber_admin.jar
aws s3 cp s3://xxx/pockly/lib.zip /workspace/pockly/lib.zip
```

后来对比了测试服务器和线上服务器的文件 md5，确认：

```text
video_tuber_admin.jar md5 一样
lib.zip md5 一样
```

这说明两个服务器拿到的包本身是一致的。

## 检查 jar 包内容

接着直接检查线上服务器的 jar 包里有没有旧 class：

```bash
cd /workspace/pockly

jar tf video_tuber_admin.jar | grep 'ApiController.class'
```

输出结果是：

```text
com/muni/web/api/old/ApiController.class
```

继续检查旧路径：

```bash
jar tf video_tuber_admin.jar | grep 'com/muni/web/api/ApiController.class'
```

没有任何输出。

这一步基本可以确定：

**当前 jar 包里没有 `com.muni.web.api.ApiController` 这个旧类。**

也就是说，jar 包本身不是问题。

## 检查启动脚本

测试服务器和线上服务器的启动脚本也做了对比。

测试环境启动脚本里多了远程调试参数：

```bash
-agentlib:jdwp=transport=dt_socket,server=y,suspend=n,address=5005
```

内存参数也略有不同：

```bash
# 测试
-Xms512m -Xmx1024m

# 线上
-Xms1024m -Xmx1024m
```

但核心启动方式都是：

```bash
java -jar /workspace/pockly/video_tuber_admin.jar
```

所以启动脚本本身也不像是导致旧 class 被加载的原因。

## 关键排查：查服务器目录里的 class 文件

既然 jar 没问题，启动脚本也没明显问题，那就继续查线上服务器整个部署目录中是否还有散落的旧 class。

执行：

```bash
cd /workspace/pockly
find . -name '*ApiController.class'
```

结果发现了关键线索：

```text
./com/muni/web/api/ApiController.class
```

也就是说，线上服务器的 `/workspace/pockly` 根目录下面，竟然残留了一份旧的 class 文件：

```text
/workspace/pockly/com/muni/web/api/ApiController.class
```

这份 class 不在 jar 里，而是直接散落在部署目录中。

这就是线上启动失败的根因。

## 为什么散落的 class 会影响启动？

项目启动脚本里有这样的配置：

```bash
export DEFAULT_SEARCH_LOCATIONS="classpath:/,classpath:/conf/,file:./,file:./conf/"
export CUSTOM_SEARCH_LOCATIONS=${DEFAULT_SEARCH_LOCATIONS},file:${BASE_DIR}/conf/
```

其中包含：

```text
file:./
```

如果应用启动时工作目录是：

```text
/workspace/pockly
```

而这个目录下刚好有：

```text
com/muni/web/api/ApiController.class
```

那么 Spring 就可能把这份散落的旧 class 也扫描进去。

于是就出现了：

```text
com.muni.web.api.ApiController
com.muni.web.api.old.ApiController
```

两个类同时存在，最终 Bean 名冲突。

## 解决方式

直接删除线上服务器中残留的旧 class 目录：

```bash
cd /workspace/pockly

rm -rf com
rm -rf BOOT-INF META-INF org
rm -rf classes
```

然后确认旧 class 已经不存在：

```bash
find . -name '*ApiController.class'
```

确认没有旧的：

```text
./com/muni/web/api/ApiController.class
```

之后重新启动服务：

```bash
cd /workspace/pockly/bin
sh startUp.sh
```

## 建议优化部署脚本

这次问题的根源不是代码，也不是 jar 包，而是服务器部署目录不干净。

因此建议在更新脚本中加入清理逻辑：

```bash
#!/bin/bash

WORK_DIR="/workspace/pockly"

cd ${WORK_DIR} || {
    echo "目录不存在: ${WORK_DIR}"
    exit 1
}

echo "清理旧解压文件..."

rm -rf com BOOT-INF META-INF org classes
rm -f video_tuber_admin.jar.old lib.zip.old

echo "备份旧文件..."

if [ -f "lib.zip" ]; then
    mv lib.zip lib.zip.old
fi

if [ -f "video_tuber_admin.jar" ]; then
    mv video_tuber_admin.jar video_tuber_admin.jar.old
fi

echo "开始从 S3 下载新文件..."

aws s3 cp s3://xxx/pockly/video_tuber_admin.jar ${WORK_DIR}/video_tuber_admin.jar
aws s3 cp s3://xxx/pockly/lib.zip ${WORK_DIR}/lib.zip

echo "文件下载完成"
ls -lh ${WORK_DIR}
```

更稳妥的方式是每次部署使用全新的 release 目录，而不是长期复用同一个目录。例如：

```text
/workspace/pockly/releases/202606171130/
```

启动时通过软链接指向当前版本：

```text
/workspace/pockly/current -> /workspace/pockly/releases/202606171130/
```

这样可以减少历史文件残留导致的问题。

## 总结

这次问题一开始看起来像是 Spring Bean 冲突：

```text
apiController conflicts
```

但真正原因不是代码里真的存在两个 Controller，也不是 jar 包错误，而是：

**线上服务器部署目录残留了旧的 `ApiController.class` 文件。**

最终定位路径是：

```text
/workspace/pockly/com/muni/web/api/ApiController.class
```

排查这类问题时，可以按这个顺序来：

```bash
# 1. 确认 jar 里有没有旧 class
jar tf video_tuber_admin.jar | grep 'ApiController.class'

# 2. 确认部署目录有没有散落 class
find /workspace/pockly -name '*ApiController.class'

# 3. 确认其他 jar 有没有旧 class
find /workspace/pockly -name "*.jar" -exec sh -c 'jar tf "$1" 2>/dev/null | grep -q "com/muni/web/api/ApiController.class" && echo "$1"' _ {} \;

# 4. 清理旧解压目录
rm -rf com BOOT-INF META-INF org classes
```

这类线上问题最容易误判成“缓存问题”，但 Java 本身并不会在 jar 更新后凭空缓存旧 class。

如果 jar 是干净的，老进程也不存在，那就重点查：

```text
部署目录
classpath
外部 lib
解压残留文件
```

这次的教训也很简单：

**部署目录一定要干净，尤其不要在应用根目录里留下解压出来的 class 文件。**
