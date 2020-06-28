1.0 实现原理
1.1 采用的数据结构
1.2 采用的设计模式
1.3 值得学习的点

## Collection
Collection是集合框架的根接口。代表一系列对象的集合，对象被称为元素。
一些集合允许重复的元素，另外一些不允许。一些集合元素是有序的，例外一些则不是。
JDK没有提供Collection接口的直接实现类，但提供了Collection的子接口的实现类例如：Set，List。
它的主要作用是用来传递集合，在需要最大通用性的场景操作集合。

Bags 即 MultiSets，这种可能有重复元素的无序集合，应该直接实现这个接口。

所有通用的Collection实现类(通常是通过其子接口之一实现)，都应该提供两个标准构造函数。
1. 一个无参构造函数，用于创建空集合
2. 一个参数为Collection的单参数构造函数，用于创建一个元素类型与其参数集合元素类型相同的集合。
实际上，后一个构造函数允许用户复制任何集合，从而生成所需实现类型的等效集合。
没有办法强制执行此约定（因为接口不能包含构造函数），但是Java平台库中的所有通用Collection实现都遵从。

Collection中的“破坏性方法”需要在不支持此项操作时抛出UnsupportedOperationException，破坏性方法指的是修改正在操作的集合的方法。
在这种情况下，如果调用对集合没有影响，则这些方法可以（但不是必需）抛出UnsupportedOperationException。
例如，对一个不可修改的集合调用addAll（Collection）时，如果要添加的集合为空，方法可以（但并非必须）抛出异常。

一些集合实现对它们可能包含的元素有限制。
例如，
- 某些实现禁止元素为null
- 某些实现对其元素类型进行限制。
    - 尝试添加不合格元素会抛出未经检查的异常，通常为NullPointerException或ClassCastException。
    - 尝试查询不合格元素的存在可能会引发异常，或者可能仅返回false，一些实现将表现出前一种行为，而某些将表现出后者。
更一般地，尝试对不合格元素进行操作，该操作的完成不会导致将不合格元素插入集合中，这可能会导致异常或成功实现，具体取决于实现方式。此类异常在此接口的规范中标记为“可选”。

由每个集合决定自己的同步策略。在实现没有更强有力的保证的情况下，调用另一个线程正在操作的集合上的任何方法可能会导致未知行为。
这包括：
- 直接调用
- 将​​集合传递给可能执行调用的方法
- 使用现有的迭代器检查集合。

Collections Framework接口中的许多方法都是根据equals（Object）方法定义的。
例如，contains（Object）contains（Object o）方法的规范说：“当且仅当此集合包含至少一个元素e时，即（o == null？e == null：o.equals （e）），返回true。”
此规范不应解释为暗示调用带有非空参数o的Collection.contains会导致对任何元素e调用o.equals（e）。实现可以自由地进行优化，从而避免了等号调用，例如，首先比较两个元素的哈希码。 
（hashCode（）规范保证了具有不相等哈希码的两个对象不能相等。）
更一般的，各种Collections Framework接口的实现均可在实现者认为合适的地方自由利用基础Object方法的指定行为。

某些执行集合递归遍历的集合操作可能会失败，但对于直接或间接包含其自身的集合的自引用实例则例外。这包括clone（），equals（），hashCode（）和toString（）方法。实现可以有选择地处理自引用场景，但是大多数当前实现不这样做。

Collection接口是Java集合框架的成员

默认方法实现（继承或以其他方式）不应用任何同步协议。如果Collection实现具有特定的同步协议，则它必须覆盖默认实现以应用该协议。


## Queue & Deque
Queue
1. 向队尾添加元素
    - add     如果是有界队列，满了的话会抛出异常
    - offer   允许添加失败
2. 从队首移除元素
    - remove  如果队首没有元素，抛出异常
    - poll    如果队首没有元素，返回null
3. 查看队首元素
    - element 如果队首没有元素，抛出异常
    - peak    如果队首没有元素，返回null
Deque
1. 向队首添加元素
    - addFirst    报错
    - offerFirst  允许添加失败
2. 向队尾添加元素
    - addLast     报错
    - offerLast   允许添加失败
3. 从队首移除元素
    - removeFirst 报错
    - pollFirst   返回null
4. 从队尾移除元素
    - removeLast  报错
    - pollLast    返回null
5. 查看队首元素
    - getFirst    报错
    - peakFirst   返回null
6. 查看队尾元素
    - getLast     报错
    - peakLast    返回null
7. 移除指定元素
    - removeFirstOccurence 不存在指定元素时，不改变
    - removeLastOccurence  不存在指定元素时，不改变
8. stack方法
    - push        同addFirst
    - pop         同removeFirst

从总体来看集合框架类的组织和实现
- 面向接口
- 通过抽象类来减少重复代码
- 注释清晰
