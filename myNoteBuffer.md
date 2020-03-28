### myNoteBuffer
#### 在实体机上debug
```shell script
# 服务端需要在启动参数上加
-Xdebug -Xrunjdwp:transport=dt_socket,server=y,suspend=15000
```
```shell script
# attch端需要配置如下启动参数
-agentlib:jdwp=transport=dt_socket,server=y,suspend=n,address=15000
```

#### Lombok
```
    // @AllArgsConstructor没有@NoArgsConstructor
    // 1.18.8
    public ItemQuantityParam(final BigDecimal quantity, final String rootItemId) {
        this.quantity = quantity;
        this.rootItemId = rootItemId;
    }

    // 1.16.18
    @ConstructorProperties({"quantity","rootItemId"})
    public ItemQuantityParam(BigDecimal quantity, String rootItemId) {
        this.quantity = quantity;
        this.rootItemId = rootItemId;
    }
    要么 有@ConstructorProperties注解 要么有 无参构造加setter方法
```

#### java中打印class信息
```
getName()
1. 如果是引用类型，返回该类的二进制名称
2. 如果是基本类型，返回其关键字
3. 如果是数组，在前面加上[来表示数组的维数
    - 引用类型数组 [Lclasname
    - 基本类型数组 [(primitive Encoding)
        - boolean Z
        - char    C
        - byte    B
        - short   S
        - int     I
        - long    J
        - float   F
        - double  D
例子
 String.class.getName()
     returns "java.lang.String"
 byte.class.getName()
     returns "byte"
 (new Object[3]).getClass().getName()
     returns "[Ljava.lang.Object;"
 (new int[3][4][5][6][7][8][9]).getClass().getName()
     returns "[[[[[[[I"
```
