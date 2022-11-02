## ARouter

### 一、主要代码结构

- arouter-api：上层主要代码，包括入口类 ARouter，主要逻辑代码类 LogisticsCenter，相关辅助类 ClassUtils 等
- arouter-annotation：ARouter 中主要支持的 annotation（包括 Autowired，Route、Interceptor）的定义，以及 RouteMeta 等基础 model bean 的定义
- arouter-compiler：Arouter 中 annotation 对应的 annotation processor 代码（注解处理器：让这些注解代码起作用，Arouter 中主要是生成相关代码）
- arouter-gradle-plugin：一个 gradle 插件，目的是在 arouter 中插入相关注册代码（代替在 Init 时扫描 dex 文件获取到所有 route 相关类）
- app：demo 代码，包括 Activity 跳转，面向接口服务使用。

### 二、核心源码分析

```java
ARouter.getInstance().build("/test/activity2").navigation();
```

1. ARouter 调用 build 生成 Postcard 的过程
2. Postcard 是什么
3. Postcard 调用 navigation 怎样执行到 startActivity 的

#### 2.1 生成 Postcard 的过程

