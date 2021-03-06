### 深入理解JAVA虚拟机读书笔记
-----

#### 关于CMS收集器
CMS用的是`标记-清除`算法，分为四个阶段：初始标记（CMS initial mark），并发标记（CMS concurrent mark），重新标记（CMS remark），并发清除（CMS concurrent sweep）。并发标记和并发清除的速度最慢，但是是与用户线程同时进行的。另外两个阶段比较快，但是需要STW。
CMS的缺点：
1. CPU敏感。默认启动的线程数是（CPU数量+3）/4，如果只有两个，那么就要分出一个核去执行垃圾回收，速度直接慢了一半。
2. 无法处理浮动垃圾。可能出现`Concurrent Mode Failure`，由于并发清除的时候用户线程也在运行，CMS无法在本次GC中清理新的垃圾，只能留待下一次的GC，而且还需要流出足够的空间在收集垃圾的时候给用户使用，如果预留的空间不足就会出现上面说的`CMF`。这是虚拟机将启动备案：临时使用`Serial Old`重新对老年代回收。
3. 会产生很多碎片。不过虚拟机默认每次CMS收集老年代的时候都会进行一次碎片整理。我们可以设置每进行多少次不压缩的CMS GC，跟着进行一次整理的。

#### 关于GC日志
FullGC仅仅表示这次GC过程中发生了STW，而一般的GC则表示没有停顿。新生代的ParNew的日志也会出现Full GC（一般是出现了分配担保失败之类的问题）。
后面紧跟着的就是GC发生的代，这个代的名称跟收集器名称是密切相关的，比如：DefNew -> Serial，ParNew -> Parallel New，PSYoungGen -> Parallel Scavenge。然后就是这个区回收前后的大小，然后是JAVA堆回收前后的大小，再然后表示GC所占时间，单位是秒，有的收集器会给出更具体的时间，比如:\[Times : user=0.01 sys=0.00 real=0.02\]，分别代表用户态、内核态和墙钟时间。一般关注real就可以。这里有个小case，user + sys > real是完全正常的，因为cpu多核的情况下，cpu时间会叠加。

#### 对象晋升
对象优先分配在年轻带，年轻代不足会产生一次YGC，YGC会把eden和survivor中有对象的一个区域一起GC，然后存活的放到survivor的另外一个区域中。如果存活的对象比较多，from或者to区域放不下，会因为分配担保机制，对象提前进入老年代。
对象大小如果超过一个阈值也会进入老年代，PretenureSizeThreshold。但是这个参数在收集器使用Parallel Scavenge的时候不生效。
对象长时间存活也会晋升到老年代，每熬过一次minor gc，年龄加一，默认达到15的时候就晋升到老年代。
如果Survivor空间中相同年龄所有对象大小的综合大于Survivor区域的一半，年龄大于或等于该年龄的对象可以直接进入老年代。

#### 空间分配担保
minor gc之前，虚拟机会先检查`老年代最大可用的连续空间`是否大于`新生代所有对象总大小`，如果大于，那么minor gc就是安全的。如果小于，那么会查看`XX:HandlePromotionFailure`设置值是否允许担保失败，如果允许，那么会检查老年代`最大可用的连续空间`是否大于`历次晋升到老年代对象的平均大小`。如果大于，尝试进行一次minor gc，虽然有风险，如果小于或者设置不允许冒险，改为进行一次Full gc。

#### 类的加载
虚拟机规定5种必须初始化类的情况（当然加载、验证、准备、解析都在初始化之前也必须进行）：
1.遇到new、getstatic、putstatic或invokestatic这4条字节码指令时，没有初始化就必须初始化。
2.使用java.lang.reflect包的方法对类进行反射调用的时候。
3.当初始化一个类的时候，父类未初始化，则先初始化父类
4.main函数的类必须初始化
5.如果一个java.lang.invoke.MethodHandle实例最后的解析结果是REF_getStatic、REF_putStatic、REF_invokeStaitc的时候。这是对动态语言的支持。

#### 关于YGC
这里就不班门弄斧了，直接从狼神的文章中copy一张图下来，可以说是非常的明白：
![ygc](/YGC.png)

#### 关于类加载器的双亲委派模型
破坏类的双亲委派，可以通过给Thread设置contextClassLoader。如果创建线程时没有则从父线程继承一个。这个设计的主要原因是为了让父类加载器能够识别子加载器的类。

#### 关于线上GC问题排查的参数
-XX:+PrintGC 输出GC日志
-XX:+PrintGCDetails 输出GC的详细日志
-XX:+PrintGCTimeStamps 输出GC的时间戳（以基准时间的形式）
-XX:+PrintGCDateStamps 输出GC的时间戳（以日期的形式，如2013-05-04T21:53:59.234+0800）
-XX:+PrintHeapAtGC 在进行GC的前后打印出堆的信息
-Xloggc:../logs/gc.log 日志文件的输出路径