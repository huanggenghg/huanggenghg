## XML and JSON 解析

### XML 解析

![三种解析方式的比较](https://upload-images.jianshu.io/upload_images/7508328-8c8d09756dc6eb89.png?imageMogr2/auto-orient/strip|imageView2/2/format/webp)

[Android XML解析的三种方式](https://www.jianshu.com/p/4e6eeec47b27)

### JSON 解决

#### Gson

![Gson](https://p1-jj.byteimg.com/tos-cn-i-t2oaga2asx/gold-user-assets/2019/2/16/168f59451f2d9b06~tplv-t2oaga2asx-zoom-in-crop-mark:4536:0:0:0.image)

反序列化过程：

- 反射创建该类型的对象
- 把json中对应的值赋给对象对应的属性
- 返回该对象。 

核心：**TypeAdapter**

适配器模式：

![TypeAdapter](https://p1-jj.byteimg.com/tos-cn-i-t2oaga2asx/gold-user-assets/2019/3/21/169a0e31e960ea5a~tplv-t2oaga2asx-zoom-in-crop-mark:4536:0:0:0.image)

Type 与 TypeAdapter 对应：

![Type 与 TypeAdapter 关系](https://p1-jj.byteimg.com/tos-cn-i-t2oaga2asx/gold-user-assets/2019/3/21/1699c39014be5a66~tplv-t2oaga2asx-zoom-in-crop-mark:4536:0:0:0.image)

每一种基本类型都会创建一个TypeAdapter来适配它们，而所有的复合类型（即我们自己定义的各种JavaBean）都会由ReflectiveTypeAdapter来完成适配

Gson 解析简易流程：

![Gson 解析简易流程](https://p1-jj.byteimg.com/tos-cn-i-t2oaga2asx/gold-user-assets/2019/3/11/1696d4be32a8387f~tplv-t2oaga2asx-zoom-in-crop-mark:4536:0:0:0.image)

ReflectiveTypeAdapter内部会首先创建该类型的对象，然后遍历该对象内部的所有属性，接着把json传的读去委托给了各个属性

找到该类型内部的所有属性，并尝试逐一封装成BoundField。

`gson.fromJson(jsonStr,UserInfo.class)`方法内部真实的代码执行流程大致如下：

- 对jsonStr，UserInfo.class这两个数据进行封装
- 通过UserInfo.class这个Type来获取它对应的TypeAdapter
- 拿到对应的TypeAdapter（ReflectiveTypeAdapterFactor）,并执行读取json的操作
- 返回UserInfo这个类型的对象。

Gson在其构造方法中，就提前把所有的TypeAdapterFactory放在缓存列表中，ReflectiveTypeAdapterFactor最后被添加进去的，ReflectiveTypeAdapterFactory之所以在缓存列表的最后一个，就是因为它能匹配几乎任何类型，因此，我们为一个类型遍历时，只能先判断它是不是基本类型，如果都不成功，最后再使用ReflectiveTypeAdapterFactor进行判断。这就是Gson中用到的工厂模式。

`getAdapter(type)`：**无限递归问题的解决** ————

- `FutureTypeAdapter`，委托模式，代理模式的包装类。委托模式的功能是：隐藏代码具体实现，通过组合的方式同样的功能，避开继承带来的问题。但是在这里使用委派模式似乎并不是基于这些考虑。而是**为了避免陷入无限递归导致对栈溢出的崩溃**。
还要考虑多线程的环境，所以就出现了ThreadLocal这个线程局部变量，这保证了它只会在单个线程中缓存，而且会在单次Json解析完成后移出缓存。

```java
gson.getAdapter(type)  ---> （ReflectiveTypeAdapterFactory）factory.create(this, type) --->   getBoundFields()  --->  createBoundField()  --->  (Gson)context.getAdapter(fieldType)
```

#### fastjson

核心功能：序列化和反序列化

**为什么Fastjson能够做到这么快？**

1. Fastjson中Serialzie的优化实现

- 自行编写类似StringBuilder的工具类SerializeWriter。  
把java对象序列化成json文本，是不可能使用字符串直接拼接的，因为这样性能很差。比字符串拼接更好的办法是使用java.lang.StringBuilder。StringBuilder虽然速度很好了，但还能够进一步提升性能的，fastjson中提供了一个类似StringBuilder的类com.alibaba.fastjson.serializer.SerializeWriter。 
SerializeWriter提供一些针对性的方法减少数组越界检查。例如public void writeIntAndChar(int i, char c) {}，这样的方法一次性把两个值写到buf中去，能够减少一次越界检查。目前SerializeWriter还有一些关键的方法能够减少越界检查的，我还没实现。也就是说，如果实现了，能够进一步提升serialize的性能。 

- 使用ThreadLocal来缓存buf。  
这个办法能够减少对象分配和gc，从而提升性能。SerializeWriter中包含了一个char[] buf，每序列化一次，都要做一次分配，使用ThreadLocal优化，能够提升性能。 

- 使用asm避免反射  
获取java bean的属性值，需要调用反射，fastjson引入了asm的来避免反射导致的开销。fastjson内置的asm是基于objectweb asm 3.3.1改造的，只保留必要的部分，fastjson asm部分不到1000行代码，引入了asm的同时不导致大小变大太多。 

- 使用一个特殊的IdentityHashMap优化性能。  
fastjson对每种类型使用一种serializer，于是就存在class -> JavaBeanSerizlier的映射。fastjson使用IdentityHashMap而不是HashMap，避免equals操作。我们知道HashMap的算法的transfer操作，并发时可能导致死循环，但是ConcurrentHashMap比HashMap系列会慢，因为其使用volatile和lock。fastjson自己实现了一个特别的IdentityHashMap，去掉transfer操作的IdentityHashMap，能够在并发时工作，但是不会导致死循环。 

- 缺省启用sort field输出  
json的object是一种key/value结构，正常的hashmap是无序的，fastjson缺省是排序输出的，这是为deserialize优化做准备。 

- 集成jdk实现的一些优化算法  
在优化fastjson的过程中，参考了jdk内部实现的算法，比如int to char[]算法等等。

2. fastjson的deserializer的主要优化算法（deserializer也称为parser或者decoder）

- 读取 token 基于预测
比如key之后，最大的可能是冒号":"，value之后，可能是有两个，逗号","或者右括号"}"。

- sort field fast match算法
采用一种优化算法，就是假设key/value的内容是有序的，读取的时候只需要做key的匹配，而不需要把key从输入中读取出来。通过这个优化，使得fastjson在处理json文本的时候，少读取超过50%的token，这个是一个十分关键的优化算法。

- 使用 asm 避免反射
deserialize的时候，会使用asm来构造对象，并且做batch set，也就是说合并连续调用多个setter方法，而不是分散调用，这个能够提升性能。

- 对utf-8的json bytes，针对性使用优化的版本来转换编码

- symbolTable算法

[Fastjson内幕](https://blog.csdn.net/zhxdick/article/details/78292033?ops_request_misc=%257B%2522request%255Fid%2522%253A%2522167383863716800188598109%2522%252C%2522scm%2522%253A%252220140713.130102334.pc%255Fblog.%2522%257D&request_id=167383863716800188598109&biz_id=0&utm_medium=distribute.pc_search_result.none-task-blog-2~blog~first_rank_ecpm_v1~rank_v31_ecpm-1-78292033-null-null.article_score_rank_blog&utm_term=fastjson&spm=1018.2226.3001.4450)




























