## Groovy 及 Gradle

### 一、Groovy 基础

Groovy 是基于 JVM 的面向对象的编程语言，即可用于面向对象编程，也可用作纯粹的编程语言。具有其他面编程语言的优秀特性，比如动态类型转换、闭包及元编程支持。Groovy 与 Java 可相互调用并结合编程，相比 Java 更加灵活和简洁。

其变量及方法的定义比较简单，变量及方法都用`def`关键字来定义，如若方法指定返回类型，则`def`可不用，还有一些语法在编写时可忽略的部分（在 Android 中编写 gradle 常见到），比如分号、方法的括号、参数类型、return 可以忽略。

类与  Java 类非常类似，区别是 Groovy 中：默认的修饰符为 public（变量及方法也是），没有可见性修饰符的字段会自动生成 setter 和 getter 方法，类不需要与其源文件有相同名称（建议一样）

语句上，for 循环支持 Java 及类似 Kotlin 的循环使用；switch 兼容 Java 并可处理更多类型的 case 表达式（可以是字符串、列表、范围等）

数据类型上，有：Java 中的基本数据类型、Groovy 中的容器类、闭包。字符串有单引号字符串、双引号字符串，可用`$`进行插值、三引号字符串，可保留文本的换行及缩进格式，不支持插值；List 为 Java 集合类基础的增强简化版，索引 - 1 是拿列表末尾元素，`<<`运算符是表示在列表末尾追加一个元素；Map 的创建需指定键和值：`def name = [one: 'aaa', two: 'bbb']`，当变量作为键值时，需使用括号包裹起来，比如`person = [(key): 'aaa']`。

说下闭包（Closure），闭包是一个开放的、匿名的、可接受参数和返回值的代码块，如下

``` groovy
{ 
    [closureParameters -> ] // 参数部分，可选，若只有一个参数，参数名可选，隐式指定为 it 
    statements              // 语句部分
}

// 调用
def code = { 123 }
assert code() == 123
assert code.call() == 123
def isOddNumber = { int i -> i % 2 != 0}
assert isOddNumber(3) == true
```

其他 I/O 操作及一些语法糖的使用比较简单，如`file.eachLine`、`file.text`、空安全`?`、`with` 等，使用过 Kotlin 的同学应该会感觉到很熟悉。在具体使用时，可以去查看 Groovy 的官方 API 文档。

### 二、Gradle  核心思想

#### APK 构建流程

![img](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/79ea21ad2a9a4b93867a4dd08c51ed45~tplv-k3u1fbpfcp-zoom-in-crop-mark:3024:0:0:0.awebp)

![img](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/ae7b715dd08f40078903f56eeabd5a54~tplv-k3u1fbpfcp-zoom-in-crop-mark:3024:0:0:0.awebp)

![img](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/8890e628fa934782b4162cadf6c198b8~tplv-k3u1fbpfcp-zoom-in-crop-mark:3024:0:0:0.awebp)

### 三、Gradle  特性

轻松的可扩展性、采用了 Groovy、强大的依赖管理、灵活的约定、Gradle Wrapper 简化其本身的下载、安装和构建、可以和其他构建工具集成、底层 API、社区的支持和推动。

``` groovy
task A {
    doLast {
        //...
    }
}

task A << {
    //...
}
```

#### 创建任务

``` groovy
def Task hello = task(hello)
hello.doLast {
    //...
}

def Task hello = task(hello, group:BasePlugin.BUILD_GROUP)
//...

tasks.create(name: 'hello') << {
    //...
}
```

#### 任务依赖

``` groovy
task hello << {
    //...
}
task go(dependsOn: hello) << {
    // 任务 go，指定依赖任务 hello，在 hello 之后执行
}
```

#### 动态定义任务

``` groovy
3.times {number -> // 0, 1, 2
    task "task$number" << {
        //...
    }
}
```

#### 任务的分组和描述

```groovy
task hello {
    group = 'build'
    description = 'hello world'
    doLast {
        //...
    }
}

def Task hello = task(hello)
hello.description = 'hello world'
hello.group = 'build'
//...
```

#### 日志级别

```groovy
ERROR,QUIET,WARNING,LIFECYCLE,INFO,DEBUG

gradle -q\-i\-d\... 任务名称
```

#### 一些命令行

```groovy
gradle -q tasks // 所有任务信息
gradle hello -x go // 排除任务 x
gradle -q help --task hello // 获取 help 任务帮助信息
gradle helloWorld goForIt // 先后执行任务
gradle hW gF // 任务名称缩写（需驼峰缩写唯一）
```

### 四、Gradle Wrapper

一个脚本，可指定 Gradle 版本，使开发任务能快速启动并运行 gradle 项目，标准化了项目，从而提高了开发效率。使用 Gradle Wrapper 使用 gradlew 和 gradlew.bat 脚本。

升级版本可设置属性文件 distributionUrl 属性或执行任务指令

```groovy
gradlew wrapper --gradle-version 5.1.1
```

自定义 Gradle Wrapper

```groovy
task wrapper(type: Wrapper) {
    gradleVersion = '4.2.1' // 将要下载的版本修改为 4.2.1
}

tesk wrapper(type: Wrapper) {
    gradleVersion = '4.2.1'
    distributionUrl = ''
    distributionPath = wrapper/dists
}
```

### 五、Gradle 插件

两种类型：脚本插件和对象插件。脚本插件是额外的构建脚本；对象插件又叫二进制插件，是实现了 Plugin 接口的类。

#### 脚本插件

```groovy
//other.gradle
ext {
    version = 1.0
}
//build.gradle
apply from: 'other.gradle'
task test {
    doLast {
        println "${version}"
    }
}
```

#### 对象插件

``` groovy
apply plugin: 'java'
apply plugin: 'cpp'
```

第三方对象插件通常是 jar 文件，要想让构建脚本知道第三方插件的存在，需要使用 buildScript 来设置依赖。

``` groovy
buildScript {
    repositories {
        maven {
            url "https://plugins.gradle..."
        }
    }
    dependencies {
        classpath "..."
    }
}
apply plugin: "com...."
```

#### 自定义 Gradle 插件

有三种方式：在 build.gradle 中编写、在 buildSrc 工程项目中编写、在独立项目中编写。

### 六、Android Gradle 构建的生命周期

```mermaid
graph LR
初始化阶段 --> 配置阶段 --> 执行阶段
```

#### 6.1 初始化阶段

任务：创建项目的层次结构，并为每一个项目创建一个`Project`实例。
相关脚本文件：`settings.gradle`，对应一个`Settings`对象

```groovy
gradle.addBuildListener(new BuildListener() {
    void buildStarted(Gradle var1) {
        // 开始构建
    }
    void settingsEvaluated(Settings var1) {
        // settings.gradle 执行完毕
    }
    void projectsLoaded(Gradle var1) {
        // 初始化阶段结束（项目结构加载完成）
    }
    void projectsEvaluated(Gradle var1) {
        // 配置阶段结束
    }
    void buildFinished(BuildResult var1) {
        // 构建结束
    }
})
```

#### 6.2 配置阶段

任务：执行各个项目下的`build.gradle`脚本，完成 Project 的配置，并构造 Task 任务依赖关系图以便在执行阶段按照依赖关系执行。
相关脚本文件：`build.gradle`，对应一个`Project`对象。

配置阶段执行的代码包括`build.gradle`中各种语句、闭包及 Task 中的配置段语句。

执行任何 Gradle 命令，**初始化阶段和配置阶段的代码都会被执行**。

排查构建速度，是否可以写成任务 Task？

#### 6.3 执行阶段

根据 Task 依赖关系构建一个有向无环图，`getTaskGraph`，对应的类为`TaskExecutionGraph`，然后通过调用`gradle <任务名>`执行对应任务。

#### 6.4 Hook

![Gradle构建周期中的Hook点](https://p1-jj.byteimg.com/tos-cn-i-t2oaga2asx/gold-user-assets/2018/7/3/1645f7712096f3e6~tplv-t2oaga2asx-zoom-in-crop-mark:3024:0:0:0.awebp)

监听器一定要在回调的生命周期之前添加，比如 settings 相关监听需在`settings.gradle`中添加才是正确的。

```groovy
// 需在 settings.gradle 中设置
gradle.settingsEvaluated { setting ->
  // do something with setting
}

gradle.projectsLoaded { 
  gradle.rootProject.afterEvaluate {
    println 'rootProject evaluated'
  }
}
```

动态改变 Task 依赖关系：1.寻找插入点；2.动态插入自定义任务。

### 七、AAR 依赖 和 module 源码依赖动态切换

*底层模块改动怎么编译，顶层模块不知道做了改动*
*git 编译后回退，怎么编译，aar 已更新代码却已回退*

[Android 编译速度优化黑科技 - RocketX](https://juejin.cn/post/7038157787976695815) 摘录

主要问题：

![RocketX](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/dc82ac8215584839b96201d342286b77~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp?)

![RocketX](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/d331dce355d14aeaaad164631375048e~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp?)

- 如何手动添加 aar 依赖？

  `implement` -> `DynamicAddDependencyMethods$tryInvokeMethod` -> `DirectDependencyAdder$add` -> `DefaultDependencyHandler.this.doAdd`

  ```java
  public interface Project extends Comparable<Project>, ExtensionAware, PluginAware {
       ...
       DependencyHandler getDependencies(); 
       ...
  }
  
  public Dependency add(String configurationName, Object dependencyNotation) {
          return this.add(configurationName, dependencyNotation, (Closure)null);
      }
  
      public Dependency add(String configurationName, Object dependencyNotation, Closure configureClosure) {
         //这里直接调用到了 doAdd 
          return this.doAdd(this.configurationContainer.getByName(configurationName), dependencyNotation, configureClosure);
      }
  ```

   `doAdd` 方法三个参数通过 `debug` 源码发现，`configuration` 就是 `"implementation", "api", "compileOnly"` 这三个字符串生成的对象，`dependencyNotation` 是一个 `LinkHashMap`   有两个键值对，分别是 `name:aarName`, `ext:aar`,最后一个`configureAction` 传 `null` 就可以了，调用  `project.dependencies.add` 最终会调到 `doAdd`  方法，也就是说直接调用 add 即可。

  ```java
  fun addAarDependencyToProject(aarName: String, configName: String, project: Project) {
          //添加 aar 依赖 以下代码等同于 api/implementation/xxx (name: 'libaccount-2.0.0', ext: 'aar'),源码使用 linkedMap
          val map = linkedMapOf<String, String>()
          map.put("name", aarName)
          map.put("ext", "aar")
          project.dependencies.add(configName, map)
      }
  ```

  ```java
  fun flatDirs() {
          val map = mutableMapOf<String, File>()
          map.put("dirs", File(getLocalMavenCacheDir()))
          appProject.rootProject.allprojects {
              it.repositories.flatDir(map)
          }
      }
  ```

- 哪一个 module 做了修改

  遍历整个项目文件的`lastModifyTime`去做实现、每个文件整个 countTime 对比改动

- module 依赖关系获取

  时机需要在 run 编译 task 之间，确保依赖关系获取后替换能生效，而且要在全局 module 依赖图已生成之后（也就是执行完 build.gradle）：`DependencyResolutionListener`和`projectsEvaluated`。

  但是出现的问题是` beforeResolve` 会回调多次，并且执行完毕每一个` module`的  `build.gradle` 把依赖解析出来则会回调。那么如果在业务层处理一下，等待到最后一个`module` 回调完毕，再通过  `project.configurations` 获取到所有 `module` 的依赖图？答案是可以的，但是 时机已经晚了，等到最后一个` module` 解析完毕之后 回调 `beforeResolve` ，再去修改依赖关系会报以下异常（无法修改依赖关系）；

  换个法子通过`project.gradle.projectsEvaluated {}`回调之后拿到所有 `module` 依赖关系并且去修改。依赖图可以拿到，但是修改依赖关系还是会报异常，时机终究还是晚了。

  ```kotlin
  //AppProjectDependencies.kt
      init {
          val projectsEvaluatedList = hookProjectsEvaluatedAction()
          project.gradle.projectsEvaluated {
              //先执行重依赖
              resolveDenpendency()
              //后执行移除的监听（主要调整执行顺序，重依赖才能生效和不报错，可能有AGP 版本兼容问题）
              val clazz = Class.forName("org.gradle.api.invocation.Gradle")
              val method = clazz.getDeclaredMethod("projectsEvaluated", Action::class.java)
              val mMethodInvocation = MethodInvocation(method, arrayOf(it))
              projectsEvaluatedList.forEach {
                  it.dispatch(mMethodInvocation)
              }
  
          }
      }
  
      //把所有 监听了 projectsEvaluated 的匿名内部类移除
      fun hookProjectsEvaluatedAction(): List<BroadcastDispatch<BuildListener>> {
          var removeDispatch = mutableListOf<BroadcastDispatch<BuildListener>>()
          try {
              var buildListenerBroadcast: ListenerBroadcast<BuildListener>? = null
              val fBuildListenerBroadcast =
                      DefaultGradle::class.java.getDeclaredField("buildListenerBroadcast")
              fBuildListenerBroadcast.isAccessible = true
              buildListenerBroadcast =
                      fBuildListenerBroadcast.get(project.gradle) as? ListenerBroadcast<BuildListener>
  
              val fBroadcast = ListenerBroadcast::class.java.getDeclaredField("broadcast")
              fBroadcast.isAccessible = true
              val broadcast: BroadcastDispatch<BuildListener>? =
                      fBroadcast.get(buildListenerBroadcast) as? BroadcastDispatch<BuildListener>
              val fDispatchers = broadcast?.javaClass?.getDeclaredField("dispatchers")
              fDispatchers?.isAccessible = true
              val dispatchers: ArrayList<BroadcastDispatch<BuildListener>>? =
                      fDispatchers?.get(broadcast) as? ArrayList<BroadcastDispatch<BuildListener>>
  
              val clazz =
                      Class.forName("org.gradle.internal.event.BroadcastDispatch\$ActionInvocationHandler")
              val iterator = dispatchers?.iterator()
              iterator?.let {
                  while (iterator.hasNext()) {
                      try {
                          val next = iterator.next()
                          val fDispatch = next.javaClass.getDeclaredField("dispatch")
                          fDispatch.isAccessible = true
                          val dispatch: Any? = fDispatch.get(next)
                          val fMethodName = clazz.getDeclaredField("methodName")
                          fMethodName.isAccessible = true
                          val methodName = fMethodName.get(dispatch) as? String
                          if (methodName?.contains("projectsEvaluated") == true) {
                              removeDispatch.add(next)
                              iterator.remove()
                          }
                      } catch (ignore: Exception) {
                      }
                  }
              }
          } catch (ignore: Exception) {
          }
          return removeDispatch
      }
  ```

- 如何获取每个`module` 的依赖，依赖就藏在 `Configuration.dependencies`，那么通过`project.configurations.maybeCreate(configName)` 找到所有的 `Configuration` 对象，就能得到每个`module`的 `dependencies`

- module 依赖关系 project 替换成 aar 

- `hook` 编译流程，完成后置换 `loacal maven` 中被修改的 `aar`。需要在 `assembleDebug` 后面补一个 `uploadLocalMavenTask`, 通过 `finalizedBy` 把我们的 `task` 运行起来去同步修改后的 `aar` 。

  