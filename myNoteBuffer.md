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
