## Collection
Collection是集合框架的根接口。代表一系列对象的集合，对象被称为元素。
一些集合允许重复的元素，另外一些不允许。一些集合元素是有序的，例外一些则不是。
JDK没有提供Collection接口的直接实现类，但提供了Collection的子接口的实现类例如：Set，List。它的主要作用是用来传递集合，在需要最大通用性的场景操作集合。

Bags 即 MultiSets，这种可能有重复元素的无序集合，应该直接实现这个接口。

所有通用的Collection实现类(通常是通过其子接口之一实现)，都应该提供两个标准构造函数。
1. 一个无参构造函数，用于创建空集合
2. 一个参数为Collection的单参数构造函数，用于创建一个元素类型与其参数集合元素类型相同的集合。
实际上，后一个构造函数允许用户复制任何集合，从而生成所需实现类型的等效集合。
没有办法强制执行此约定（因为接口不能包含构造函数），但是Java平台库中的所有通用Collection实现都遵从。

The "destructive" methods contained in this interface, that is, the methods that modify the collection on which they operate, are specified to throw UnsupportedOperationException if this collection does not support the operation. If this is the case, these methods may, but are not required to, throw an UnsupportedOperationException if the invocation would have no effect on the collection. For example, invoking the addAll(Collection) method on an unmodifiable collection may, but is not required to, throw the exception if the collection to be added is empty.
如果此接口不支持该操作，则指定该接口中包含的“破坏性”方法，即修改其操作的集合的方法，以引发UnsupportedOperationException。在这种情况下，如果调用对集合没有影响，则这些方法可能会（但不是必需）引发UnsupportedOperationException。例如，如果要添加的集合为空，则对一个不可修改的集合调用addAll（Collection）方法可能（但并非必须）引发异常。

Some collection implementations have restrictions on the elements that they may contain. For example, some implementations prohibit null elements, and some have restrictions on the types of their elements.  Attempting to add an ineligible element throws an unchecked exception, typically NullPointerException or ClassCastException.  Attempting to query the presence of an ineligible element may throw an exception, or it may simply return false; some implementations will exhibit the former behavior and some will exhibit the latter.  More generally, attempting an operation on an ineligible element whose completion would not result in the insertion of an ineligible element into the collection may throw an exception or it may succeed, at the option of the implementation. Such exceptions are marked as "optional" in the specification for this interface.
一些集合实现对它们可能包含的元素有限制。例如，某些实现禁止使用null元素，而某些实现对其元素类型进行限制。尝试添加不合格元素会引发未经检查的异常，通常为NullPointerException或ClassCastException。尝试查询不合格元素的存在可能会引发异常，或者可能仅返回false；否则，可能会抛出异常。一些实现将表现出前一种行为，而某些将表现出后者。更一般地，尝试对不合格元素进行操作，该操作的完成不会导致将不合格元素插入集合中，这可能会导致异常或成功实现，具体取决于实现方式。此类异常在此接口的规范中标记为“可选”。

It is up to each collection to determine its own synchronization policy.  In the absence of a stronger guarantee by the implementation, undefined behavior may result from the invocation of any method on a collection that is being mutated by another thread; this includes direct invocations, passing the collection to a method that might perform invocations, and using an existing iterator to examine the collection.
由每个集合决定自己的同步策略。在实现没有更强有力的保证的情况下，未定义的行为可能是由于调用另一个线程正在变异的集合上的任何方法而导致的；这包括直接调用，将​​集合传递给可能执行调用的方法，以及使用现有的迭代器检查集合。

Many methods in Collections Framework interfaces are defined in terms of the equals(Object) method.  For example, the specification for the contains(Object) contains(Object o) method says: "returns true if and only if this collection contains at least one element e such that (o==null ? e==null : o.equals(e))."  This specification should not be construed to imply that invoking Collection.contains with a non-null argument o will cause o.equals(e) to be invoked for any element e.  Implementations are free to implement optimizations whereby the equals invocation is avoided, for example, by first comparing the hash codes of the two elements.  (The hashCode() specification guarantees that two objects with unequal hash codes cannot be equal.)  More generally, implementations of the various Collections Framework interfaces are free to take advantage of the specified behavior of underlying Object methods wherever the implementor deems it appropriate.
Collections Framework接口中的许多方法都是根据equals（Object）方法定义的。例如，contains（Object）contains（Object o）方法的规范说：“当且仅当此集合包含至少一个元素e时，返回true，使得（o == null？e == null：o.equals （e））。”此规范不应解释为暗示调用带有非null参数o的Collection.contains会导致对任何元素e调用o.equals（e）。实现可以自由地进行优化，从而避免了等号调用，例如，首先比较两个元素的哈希码。 （hashCode（）规范保证了具有不相等哈希码的两个对象不能相等。）更一般而言，各种Collections Framework接口的实现均可在实现者认为合适的地方自由利用基础Object方法的指定行为。


Some collection operations which perform recursive traversal of the collection may fail with an exception for self-referential instances where the collection directly or indirectly contains itself. This includes the clone(), equals(), hashCode() and toString() methods. Implementations may optionally handle the self-referential scenario, however most current implementations do not do so.
某些执行集合递归遍历的集合操作可能会失败，但对于直接或间接包含其自身的集合的自引用实例则例外。这包括clone（），equals（），hashCode（）和toString（）方法。实现可以有选择地处理自引用场景，但是大多数当前实现不这样做。

This interface is a member of the Java Collections Framework.

The default method implementations (inherited or otherwise) do not apply any synchronization protocol.  If a Collection implementation has a specific synchronization protocol, then it must override default implementations to apply that protocol.
默认方法实现（继承或以其他方式）不应用任何同步协议。如果Collection实现具有特定的同步协议，则它必须覆盖默认实现以应用该协议。


## thread-safe
