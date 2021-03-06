### day2 

```markdown
day 2 代码加锁：不要让琐事成为烦心事
1. 使用synchronized关键字虽然简单，但是要弄清楚共享资源是实例级别的，还是类级别的。
	会被那些线程操作。
2. 加锁尽量考虑锁粒度和场景。
	- 尽量只为必要的代码加锁，降低锁的粒度。
	- 在性能要求高的场景，细化考虑锁的读写场景，以及悲观锁还是乐观锁优先
		考虑ReentrantReadWriteLock,StampedLock等高级工具类
3. 考虑可能遇到的死锁问题
	- 避免无限等待和循环等待
4. 如果业务逻辑中加锁的实现比较复杂，要仔细检查加锁和释放锁的可能性。
```

### day3

```markdown
day 3 线程池: 业务代码最常用也最容易犯错的组件
1. Excutors类提供的一些快捷声明线程池的方法，简单却埋坑，需要更具场景和需求合理配置线程池
2. 既然选择了线程池就要服用线程池，每次new一个线程池出来可能比不用线程池更糟糕。
3. 复用线程池不代表应用始终使用一个线程池，根据任务性质来选择不同的线程池。特别注意IO和Cpu绑定的任务。
4. 最好对线程池等核心组件进行监控
```

### day4

```markdown
day 4 连接池:别让连接池帮了倒忙
业务代码中最常用的三种连接池
    - redis连接池
    - HTTP连接池
    - 数据库连接池
1. 连接池的实现方式
    - 池和链接分离
    - 内部带有连接池
    - 非连接池
2. 使用姿势
    - 确保链接池是复用的
    - 尽可能在程序退出之前显式的关闭连接池释放资源
3. 连接池配置参数
    - 最重要的是最大链接数
```

### day8
```markdown
day 8 判等问题 程序中如何确定你就是你
判等问题上主要涉及到
    - equals
    - compareTo
    - Java 数值缓存
    - Java 字符串驻留
equals 和 == 的区别
    1. ==
        - 对于基本类型来说 ==就是比较值
        - 对于引用类型来说 就是比较指针 而不是比较内容
    2. equals
java 数值缓存
    Integer、Short、Long的valueOf方法中实现了一个[-128,127]的缓存池
    在这个范围内的话，返回的是一个对象
    其中Integer的缓存大小还可以通过-XX:AutoBoxCacheMax=1000来改变，不会小于[-128,127]
java 字符串驻留
    直接用双引号声明的字符串和intern方法都会对字符串进行驻留
    但滥用intern方法会导致性能问题 
        - -XX:PrintStringTableStatistic在关闭程序后打印字符串常量表的信息
        - 字符串常量表(池)是一个固定大小的map，可以通过-XX:StringTableSize来调节大小
实现equals
    1. equals是Object类中定义的方法，如果不重写这个方法，就会用Object中的方法
       而Object类中的equals就是比较地址
    2. 实现equals的要求
        - 考虑到性能，需要先进行指针判等，如果是一个对象直接返回
        - 需要对另外一个对象进行判空，如果为null，直接返回false
        - 判断两个对象的类型是否相同，不同直接返回false
        - 最后强制转换类型，逐一判断各个字段
hashCode方法和equals方法需要配对实现
    hashSet需要hash值来判断是否存在相同对象
compareTo需要与equals逻辑保持一致
lombok中的坑
    @Data会实现equalsAndHashCode方法 可以用@EqualsAndHashCode.Exclude
    来排除指定字段，让其不参与equals和hashCode方法
    @EqualsAndHashCode默认callSuper=false，即不调用父类的方法
    可以设为true来覆盖
不同类加载器加载的对象，肯定不等
结论：
1. 比较值的内容，除了基本类型能用==外，其他类型都用equals
2. 自定义类型，如果要实现Comparable，请记得equals、hashCode、compareTo
   三者逻辑一致

问题:
1.getClass方法和instanceOf有什么区别
    - instanceOf 判断的是你是不是该类或者该类的子类
    - getClass后等值判断，看是不是一个类，更精确
2.HashSet和TreeSet的contains方法有什么区别
    - hashSet底层是HashMap存的Key，判断是否存在的方法就是equals和hashCode
    - treeSet底层是TreeMap的Key，数据结构是红黑树，自带排序功能contains方法
      根据comparator或compareTo判断相等
```

### day9
```
day 9 注意精度,舍入和溢出问题
计算器或计算机得到反直觉计算结果的原因
    - 计算机无法保存精确的浮点数
    - 计算机中的数字不是无限增大的是空间限制的
1.危险的Double
    - 浮点数计算并不精确
    - BigDecimal表示和计算浮点数时,务必使用字符串构造方法来初始化BigDecimal
        - scale 小数位数
        - percision 精度 有效数字长度
        - new BigDecimal(Double.toString(100)) 
            - Double.toString(100) -> 100.0
            - scale 1
            - percision 4
        - new BigDecimal("100")
            - scale 0
            - percision 3
    - 如果一定要用Double来初始化BigDecimal的话,可以使用BigDecimal.valueOf方法
2.考虑浮点数舍入和格式化的方式
    - 浮点数的字符串格式化也要通过BigDecimal进行
3.用equals判等BigDecimal有问题
    - BigDecimal的equals判断 值 和 精度 是都都相同
    - BigDecimal的比较应该用compareTo
    - 值相同,精度不同的BigDecimal的hashCode不同,可以放在同一个hashSet中
        - 使用treeMap treeMap使用compareTo
        - BigDecimal存入hashSet或HashMap前,用StripTrailingZeros去掉尾部的零,
4.小心数值溢出问题
    - 所有基本类型都会有超出表达范围的可能性
    - 默默的溢出,不会报错
        - 用Math类的addExact,subtractExact等方法,在数值溢出时报异常
        - 使用BigInteger来对大数进行科学计算
```

### 加餐1
```markdown
加餐 1 java8中那些重要知识点1
1.java8的新功能
    - Lambda
    - stream
    - parallelStream
    - optional
    - 新日期时间类
2.推荐书籍 <java实战(第二版)>
3.Lambda表达式
    - Lambda表达式的一个初衷是为了简化匿名类的语法,但事实上并不是匿名类的语法糖
    - 第二个初衷是为了使java走向函数式编程
    - Lambda如何匹配Java类型系统呢
        - 函数式接口
        函数式接口是一种只含有单一方法的接口
        使用@FunctionalInterface修饰
        使用Lambda表达式来实现函数式接口,不需要提供雷鸣和方法定义
        有些函数式接口还利用default关键字实现了几个默认方法,提供额外的功能
    - Lambda表达式给我们提供了复用代码的更多可能性
        - 把一大段逻辑中变化的部分抽象出函数式接口,由外部方法提供函数实现,重用方法内的整体逻辑
4.使用java8简化代码
    - 使用stream简化集合操作
    - 使用optional简化判空逻辑
        - 基本类型可空对象
            - OptionalDouble
            - OptionalInt
            - OptionalLong
        - 引用类型可空对象
            - Optional
    - JDK8使用Lambda和Stream对各种类的增强
        - Java8中很多类也实现了函数式的功能
            例如ConcurrentHashMap的computeIfAbsent方法
5.并行流
通过parallel方法一键将Stream转化为并行操作提交到线程池处理
五种方式
    1. 直接使用线程,通过countDownLatch来进行同步
    2. 使用Excutors.newFixedThreadPool来获得固定线程数的线程池
    3. 是否用forkJoinPool
    4. 使用并行流,并行流使用的是ForkJoinPool里面的commonPool
    5. 使用completableFuture.runAsync方法
  2和4常用
  forkJoinPool和ThreadPoolExcutor区别在于
        - forkJoinPool对于n并行度有n个独立队列
        - ThreadPoolExcutor是共享队列
  forkJoinPool的配置需要在启动时设置
6.forEachOrdered会使并行流失去并行度
```

### 加餐2
```markdown
加餐 1 java8中那些重要知识点2
终结操作 terminal operation
中间操作 intermediate operation
```
Stream常见操作

| 方法         | 中文       | 操作类型 | 类比SQL        | 使用的类型/函数式接口           | 作用                                              |
|------------|----------|------|--------------|-----------------------|-------------------------------------------------|
| filter     | 过滤/筛选    | 中间   | where        | Predicate<T>          | 对流过滤，使元素符合传入条件                                  |
| map        | 转换/投影    | 中间   | select       | Function<T,R>         | 使用传入的函数，对流中每一个元素进行转换                            |
| flatMap    | 展开/扁平化   | 中间   | N/A          | Function<T,Stream<R>> | 相当于map\+flat，通过map把每一个元素转换为一个流，然后把所有流连接到一起扁平化展开 |
| sorted     | 排序       | 中间   | order by     | Comparator<T>         | 使用传入的比较器，对流中的元素进行排序                             |
| distinct   | 去重       | 中间   | distinct     | long                  | 对流中元素进行去重\(Objects\.equals判重\)                  |
| skip&limit | 分页       | 中间   | limit,offset | long                  | 跳过流中部分元素以及限制元素数量                                |
| collect    | 收集       | 终结   | N/A          | Collector<T,A,R>      | 对流进行终结操作，把流导出成我们需要的数据结构                         |
| forEach    | 遍历       | 终结   | N/A          | Consumer<T>           | 对每一个元素遍历进行消费                                    |
| anyMatch   | 是否有元素匹配  | 终结   | N/A          | Predicate<T>          | 使用谓词判断是否有任何一个元素满足匹配                             |
| allMatch   | 是否所有元素匹配 | 终结   | N/A          | Predicate<T>          | 使用谓词判断是否有所有元素满足匹配                               |

Collectors中的一些静态方法

| 方法                | 返回类型                 | 作用                                           |
|-------------------|----------------------|----------------------------------------------|
| toList            | List<T>              | 把流中元素收集成为一个List                              |
| toSet             | Set<T>               | 把流中元素收集成为一个Set，去重                            |
| toCollection      | Collection<T>        | 把流中元素收集成为指定集合                                |
| counting          | Long                 | 计算流中元素个数                                     |
| summingInt        | Integer              | 对流中元素的某个整数属性求和                               |
| averagingInt      | Double               | 对流中元素的某个整数属性求平均值                             |
| joining           | String               | 连接流中元素toString后的字符串                          |
| minBy             | Optional<T>          | 使用指定的比较器选出最小的元素                              |
| maxBy             | Optional<T>          | 使用指定的比较器选出最大的元素                              |
| collectingAndThen | 根据收集器返回              | 包裹另一个收集器，对结果进行转换                             |
| groupBy           | Map<K,List<T>>       | 根据元素的一个属性对元素进行分组，属性值作为key                    |
| partitionBy       | Map<Boolean,List<T>> | 根据流中元素应用谓词\(Predicate\)的结果，将元素分为true和false两组 |
