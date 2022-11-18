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

![外观模式](https://raw.githubusercontent.com/huanggenghg/huanggenghg/main/res/ARouter%20%E8%B7%B3%E8%BD%AC%E6%B5%81%E7%A8%8B%E6%97%B6%E5%BA%8F%E5%9B%BE.drawio.png)

```java
    /**
     * Build postcard by path and default group
     */
    protected Postcard build(String path) {
        if (TextUtils.isEmpty(path)) {
            throw new HandlerException(Consts.TAG + "Parameter is invalid!");
        } else {
            PathReplaceService pService = ARouter.getInstance().navigation(PathReplaceService.class);
            if (null != pService) {
                path = pService.forString(path);
            }
            return build(path, extractGroup(path), true);
        }
    }
```

#### 2.2 Postcard

![Postcard](https://raw.githubusercontent.com/huanggenghg/huanggenghg/main/res/ARouter%20Postcard.drawio.png)

#### Postcard.navigation

![Postcard navigation](https://raw.githubusercontent.com/huanggenghg/huanggenghg/main/res/Arouter%20navigation%20%E6%97%B6%E5%BA%8F.drawio.png)

`LogisticsCenter.completion`：为 Postcard 找到对应的 router，将路由信息填充进 Postcard 对象；找不到的话，抛出`NoRouteFoundException`，对应做降级处理。NavigationCallback or DegradeService `onLost()`。

`_navigation`中根据`postcard.getType()`对应进行跳转动作或提供 Provider 对象。

#### ARouter 中 Annotation

@Autowired：省去了手动编写解析 Intent 参数的代码
@Route：可通过 path 跳转到对应解码或是获取到具体服务
@Interceptor：拦截器处理主要是在`InterceptorServiceImpl`中

Javapoet 解析注解编写对应代码

#### `_ARouter` 介绍

- init : `LogisticsCenter.init`；
- inject : AutowiredServiceImpl `autowird`，反射获得@Autowired注解生成的 Java 类的信息，通过反射方式生成对象，调用`inject`方法；
- `build`
- `navigation`

*组件自动注册插件*

#### `LogisticsCenter` 介绍

- loadRouteMap: 由arouter-gradle-plugin 插件module在编译阶段插入注册Route相关代码。
- init: 调用loadRouteMap完成Warehouse中各个路由Index的加载，如果有plugin帮忙插入代码则直接使用，如果没有则通过运行时扫描dex文件方式加载。
- completion:前面分析navigation时有详细介绍，在Route路由表中根据postcard中的path找到对应的路由信息RouteMeta，利用RouteMeta中信息为Postcard赋值。

**以上来自[可能是最好理解的ARouter源码分析](https://www.jianshu.com/p/c5f3b8d4a746)**

**一篇完整的总结：[探索 ARouter 原理](https://juejin.cn/post/6885932290615509000)**

接下来会结合 Javapoet 及 asm 继续解析 ARouter 原理。

#### @AutoWired

```java
	// AutowiredProcessor

	@Override
    public boolean process(Set<? extends TypeElement> set, RoundEnvironment roundEnvironment) {
        if (CollectionUtils.isNotEmpty(set)) {
            try {
                logger.info(">>> Found autowired field, start... <<<");
                categories(roundEnvironment.getElementsAnnotatedWith(Autowired.class)); // 对注解的列表元素数据根据注解最里层元素进行分类
                generateHelper(); // 解析注解，写 java 文件

            } catch (Exception e) {
                logger.error(e);
            }
            return true;
        }

        return false;
    }
```

```java
private void generateHelper() throws IOException, IllegalAccessException {
        TypeElement type_ISyringe = elementUtils.getTypeElement(ISYRINGE); // 获取 ISyringe 接口元素

}
```
```java
public class Test1Activity$$ARouter$$Autowired implements ISyringe {
  private SerializationService serializationService;

  @Override
  public void inject(Object target) {
    serializationService = ARouter.getInstance().navigation(SerializationService.class);
    Test1Activity substitute = (Test1Activity)target;
    substitute.name = substitute.getIntent().getExtras() == null ? substitute.name : 
            substitute.getIntent().getExtras().getString("name", substitute.name);
    substitute.age = substitute.getIntent().getIntExtra("age", substitute.age);
    substitute.height = substitute.getIntent().getIntExtra("height", substitute.height);
    substitute.girl = substitute.getIntent().getBooleanExtra("boy", substitute.girl);
    
    ...
  }
```

编译阶段即可完成相关类的扫描工作，在 ARouter 初始化时只是完成路由总表的加载，省去在 Dex 文件中扫描类的步骤。

三个注解自动编写的代码是怎样的？用 javapoet 怎么写出来的？
组件自动注入的原理是啥？ ![AutoRegister:一种更高效的组件自动注册方案(android组件化开发)](https://juejin.cn/post/6844903520429162509)
依赖注入框架升级？自动注入，忽略具体内部的依赖注入原理？

扫描需 asm 注入代码的类（用注解去标识？），asm 注入（注入什么？-- 业务）。