### Arthas3.0 的新特性

#### 在线诊断功能

Arthas3.0 中最重要的特性，不需要登陆机器就可以对应用进行诊断，体验和本地诊断完全一致

##### 使用步骤

TODO

##### 动图演示

TODO

#### 管道支持

Arthas 3.0 开始支持管道, 率先提供了`grep`,`wc`,`plaintext`的支持。

### 去 groovy 依赖

groovy 表达式在 arthas2.0 中大量使用，例如 watch 表达式

```bash
watch com.alibaba.sample.petstore.web.store.module.screen.ItemList add "params + ' ' + returnObj" params.size()==2
```

其中`"params + ' ' + returnObj"`以及`params.size()==2`背后其实都使用了 groovy 来进行表达式求值，如果反复大量的运行这些表达式，groovy 会创建大量的 classloader，打满 perm 区从而触发 FGC。

为了避免这个问题，Arthas 3.0 中使用了 ognl 这个更加轻量的表达式求值库来代替 groovy，彻底解决了 groovy 引起的 FGC 风险。但由于这个替换，导致原来使用 groovy 脚本编写的自定义脚本失效。这个问题留待后续解决。

在 3.0 中，watch 命令的表达式部分的书写有了一些改变，详见[这里](https://arthas.aliyun.com/doc/watch)

#### 提升 rt 统计精度

Arthas 2.0 中，统计 rt 都是以`ms`为单位，对于某些比较小的方法调用，耗时在毫秒以下的都会被认为是 0ms，造成 trace 总时间和各方法的时间相加不一致等问题（虽然这里面确实会有误差，主要 Arthas 自身的开销）。Arthas 3.0 中所有 rt 的单位统一改为使用`ns`来统计，精准捕获你的方法耗时，让 0ms 这样无意义的统计数据不再出现！

```
$ tt -l
 INDEX     TIMESTAMP               COST(ms)    IS-RET    IS-EXP   OBJECT            CLASS                                METHOD
------------------------------------------------------------------------------------------------------------------------------------------------------------
 1000      2017-02-24 10:56:46     808.743525  true      false    0x3bd5e918        TestTraceServlet                     doGet
 1001      2017-02-24 10:56:55     805.799155  true      false    0x3bd5e918        TestTraceServlet                     doGet
 1002      2017-02-24 10:57:04     808.026935  true      false    0x3bd5e918        TestTraceServlet                     doGet
 1003      2017-02-24 10:57:22     805.036963  true      false    0x3bd5e918        TestTraceServlet                     doGet
 1004      2017-02-24 10:57:24     803.581886  true      false    0x3bd5e918        TestTraceServlet                     doGet
 1005      2017-02-24 10:57:39     814.657657  true      false    0x3bd5e918        TestTraceServlet                     doGet
```

#### watch/stack/trace 命令支持按耗时过滤

我们在 trace 的时候，经常会出现某个方法间隙性的 rt 飙高，但是我们只想知道 rt 高的时候，是哪里慢了，对于正常 rt 的方法我们并不关心，Arthas 3.0 支持了按`#cost`(方法执行耗时,单位为`ms`)进行过滤，只输出符合条件的 trace 路径，目前，这三个命令的相关文档已经做了更新，增加了该用法的示例。

#### sysprop 命令操作 SystemProperty

sysprop 命令支持查看所有的系统属性，以及针对特定属性进行查看和修改。

```
$ sysprop
...
 os.arch                                              x86_64
 java.ext.dirs                                        /Users/wangtao/Library/Java/Extensions:/Library/Java/JavaVirtualMachines/jdk1.
                                                      8.0_51.jdk/Contents/Home/jre/lib/ext:/Library/Java/Extensions:/Network/Library
                                                      /Java/Extensions:/System/Library/Java/Extensions:/usr/lib/java
 user.dir                                             /Users/wangtao/work/ali-tomcat-home/ant-develop/output/build
 catalina.vendor                                      alibaba
 line.separator

 java.vm.name                                         Java HotSpot(TM) 64-Bit Server VM
 file.encoding                                        UTF-8
 org.apache.tomcat.util.http.ServerCookie.ALLOW_EQUA  true
 LS_IN_VALUE
 com.taobao.tomcat.info                               Apache Tomcat/7.0.70.1548
 java.specification.version                           1.8
$ sysprop java.version
java.version=1.8.0_51
$ sysprop production.mode true
Successfully changed the system property.
production.mode=true
```

#### thread 命令支持指定采样时间

thread 命令计算线程 cpu 占用的逻辑，默认是采样 100ms 内各个线程的 cpu 使用情况并计算 cpu 消耗占比。有时候 100ms 的时间间隔太短，看不出问题所在，Arthas3.0 中 thread 命令支持设置采样间隔(以`ms`为单位)，可以观察任意时间段内的 cpu 消耗占比情况。

```
$ thread -i 1000
Threads Total: 74, NEW: 0, RUNNABLE: 17, BLOCKED: 0, WAITING: 15, TIMED_WAITING: 42, TERMINATED: 0
ID                 NAME                                                     GROUP                                  PRIORITY           STATE              %CPU               TIME               INTERRUPTED        DAEMON
78                 com.taobao.config.client.timer                           main                                   5                  TIMED_WAITING      22                 0:0                false              true
92                 Abandoned connection cleanup thread                      main                                   5                  TIMED_WAITING      15                 0:2                false              true
361                as-command-execute-daemon                                system                                 10                 RUNNABLE           14                 0:0                false              true
67                 HSF-Remoting-Timer-10-thread-1                           main                                   10                 TIMED_WAITING      12                 0:2                false              true
113                JamScheduleThread                                        system                                 9                  TIMED_WAITING      2                  0:0                false              true
14                 Thread-3                                                 main                                   5                  RUNNABLE           2                  0:0                false              false
81                 com.taobao.remoting.TimerThread                          main                                   5                  TIMED_WAITING      2                  0:0                false              true
104                http-bio-7001-AsyncTimeout                               main                                   5                  TIMED_WAITING      2                  0:0                false              true
123                nioEventLoopGroup-2-1                                    system                                 10                 RUNNABLE           2                  0:0                false              false
127                nioEventLoopGroup-3-2                                    system                                 10                 RUNNABLE           2                  0:0                false              false
345                nioEventLoopGroup-3-3                                    system                                 10                 RUNNABLE           2                  0:0                false              false
358                nioEventLoopGroup-3-4                                    system                                 10                 RUNNABLE           2                  0:0                false              false
27                 qos-boss-1-1                                             main                                   5                  RUNNABLE           2                  0:0                false              true
22                 EagleEye-AsyncAppender-Thread-BizLog                     main                                   5                  TIMED_WAITING      1                  0:0                false              true
```

#### trace 命令自动高亮显示最耗时方法调用

trace 命令现在会自动显示

![Untitled2](TODO /Untitled2.gif)
