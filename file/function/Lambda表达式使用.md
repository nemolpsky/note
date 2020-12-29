### Lambda表达式

在[Java函数式编程](https://github.com/nemolpsky/note/blob/master/file/function/Java%E5%87%BD%E6%95%B0%E5%BC%8F%E7%BC%96%E7%A8%8B.md)中说过，```Lambda```表达式其实就是匿名函数，使用它最大的好处就是在于可以直接在使用时实现接口中的函数方法，不需要事先定义实现方法。


---

#### 1. 函数式接口介绍

在上一篇文章中讲到在原有的```Comparator```等接口上使用了```@FunctionalInterface```注解，就相当于标注这个接口是函数式接口，如果不符合函数式接口的条件，这个注解会提示异常，其实函数式接口的标准就是有且仅有一个抽象方法，默认实现方法则不会有任何限制。

```
@FunctionalInterface
public interface StrFilter {

    /**
     *  抽象过滤方法
     * @param str1 被过滤的字符串
     * @param str2 需要过滤的关键字
     * @return
     */
    boolean filter(String str1,String str2);

    /**
     *  默认过滤方法
     * @param str1 被过滤的字符串
     * @param str2 需要过滤的关键字
     * @return
     */
    static boolean defaultFilter(String str1, String str2){
        return str1.contains(str2);
    }
}
```

其实Java的函数式编程都是基于这种函数式接口，定义好接口的抽象方法，自己再使用```Lambda```表达式来实现具体行为，本质上就还是以前说的，顶层尽量使用接口来描述行为，而不是大量的使用具体的代码来描述实现，更加便于后续的扩展。因此要想使用```Lambda```表达式就有一个前提，那就是要有对应的函数式接口来定义好了抽象行为，因此Java 8中提供了大量已经定义好的函数式接口，都在```java.util.function```包下，比如下列的```Function```接口。

```
@FunctionalInterface
public interface Function<T, R> {

    R apply(T t);

    default <V> Function<V, R> compose(Function<? super V, ? extends T> before) {
        Objects.requireNonNull(before);
        return (V v) -> apply(before.apply(v));
    }

    default <V> Function<T, V> andThen(Function<? super R, ? extends V> after) {
        Objects.requireNonNull(after);
        return (T t) -> after.apply(apply(t));
    }

    static <T> Function<T, T> identity() {
        return t -> t;
    }
}
```

---

#### 2. 函数式接口使用

可能会有很多人对于Java提供的函数式接口比较迷惑，到底该怎么用，上面也说了函数式接口就是定义了抽象方法，提前定义好行为，使用者自己来实现，这种说法是比较宏观抽象的说法，那么具体到代码逻辑层面来看，其实就是定义抽象方法，定义好参数和返回类型，具体实现则是通过提供的参数进行逻辑运算最后返回指定类型的结果，比如```Predicate```接口其实就和S```trFilter```接口一样，接受参数，返回一个```boolean```值，所以它也可以实现过滤的功能，下面这段代码只是修改为使用```Predicate```类型的引用，其它地方基本没动，对比```StrFilter```就很容易明白Java提供的这一对函数式接口该怎么用了。

```
public class MainClass2 {

    public static void main(String[] args) {

        List<String> list = new ArrayList<>(Arrays.asList("aaa", "bbb", "ccc", "ddd", "abd", "abc"));

        List<String> newList = filterStr(list,s -> s.contains("a"));
        System.out.println(newList);
    }

    /**
     *
     * @param list 需要筛选的集合
     * @param filter 筛选器
     * @return
     */
    public static List<String> filterStr(List<String> list,Predicate<String> filter) {
        List<String> newList = new ArrayList<>();
        // 遍历集合中的每个元素
        for (String str1 : list) {
            // 通过筛选器判断元素是否要被过滤
            if (filter.test(str1)) {
                newList.add(str1);
            }
        }
        return newList;
    }
}
```

可以看一下```Predicate```接口，只有一个```test```抽象方法，其它的都是默认实现方法，还有一个静态方法，其它方法会起到更好的扩展作用，后续再介绍。

```
@FunctionalInterface
public interface Predicate<T> {

    boolean test(T t);

    default Predicate<T> and(Predicate<? super T> other) {
        Objects.requireNonNull(other);
        return (t) -> test(t) && other.test(t);
    }

    default Predicate<T> negate() {
        return (t) -> !test(t);
    }

    default Predicate<T> or(Predicate<? super T> other) {
        Objects.requireNonNull(other);
        return (t) -> test(t) || other.test(t);
    }

    static <T> Predicate<T> isEqual(Object targetRef) {
        return (null == targetRef)
                ? Objects::isNull
                : object -> targetRef.equals(object);
    }
```

---

#### 3. 类型匹配

在Java中会对使用的```Lambda```表达式进行校验，判断是否符合条件，它会分为下列步骤。

- 先检查filterStr方法第二个参数是否为一个函数式接口
- 再检查接口中Predicate参数绑定的类型，这里是String类型
- 再检查Predicate接口中是否有一个抽象方法
- 最后检查Lambda表达式的参数和表达式的返回类型是否符合抽象方法的定义

```
List<String> newList = filterStr(list,s -> s.contains("a"));
```

所这也造成一个影响，```Lambda```有时候看起来很不直观，必须查看方法的定义才能了解你到底使用哪个函数式接口，因为有时候不同的接口看起来```Lambda```表达式是一样的。

```
Callable<Integer> callable = () -> 1;
Supplier<Integer> supplier = () -> 1;

ExecutorService executor = Executors.newCachedThreadPool();
Future<Integer> result = executor.submit(callable);
System.out.println(result.get());
```

所以在使用```Lambda```表达式的时候推荐使用类型推断，也就是在参数那里写上类型，这样极大程度的加强了阅读性，否则在很复杂的```Lambda```表达式中很难看懂参数的含义。

```
List<String> newList = filterStr(list,(String s) -> s.contains("a"));
```

---

#### 4. 局部变量

```Lambda```表达式也可以使用外部的变量，但是需要注意的是方法内部的变量，要求这个变量是```final```类型，这个设计是因为```Lambda```表达式使用局部变量的时候实际是复制了变量的副本来使用，所以后续对该变量的修改对```Lambda```表达式都没有作用，因此会有编译警告提示你使用```final```。

```
public class MainClass2 {

    public static void main(String[] args) {

        List<String> list = new ArrayList<>(Arrays.asList("aaa", "bbb", "ccc", "ddd", "abd", "abc"));
        // 不要对keyWord继续赋值，或者把keyWord赋值给另一个变量
        String keyWord = "a";
        List<String> newList = filterStr(list,(String s) -> s.contains(keyWord));
        System.out.println(newList);
        keyWord = "b";
    }
}
```


#### 5. 构造函数引用

前面讲过::的格式使用方法引用，构造函数也支持这么用，比如下面的Supplier接口中的get方法是返回一个T类型的对象，那么```Supplier<User>```声明表示它需要一个```User```对象，使用User::new会自动选择无参的构造方法，因为```get```方法不接受任何参数。这种用法特别容易搞混，尽量少用。

```
@FunctionalInterface
public interface Supplier<T> {

    /**
     * Gets a result.
     *
     * @return a result
     */
    T get();
}

public class User {

    public User() {
    }

    public User(String userName, Integer age) {
        this.userName = userName;
        this.age = age;
    }

    private String userName;
    private Integer age;
}

public class MainClass {
    public static void main(String[] args) {
        Supplier<User> supplier = User::new;
    }
}

```


---

#### 5. 复合Lambda表达式

除此之外，Java 8提供的函数式接口除了一个抽象方法之外还有另外的默认实现方法，其实你可以理解为Java提前给你实现好的一些逻辑，可以更加方便的进行复合操作。


```
@FunctionalInterface
public interface Function<T, R> {

    R apply(T t);

    default <V> Function<V, R> compose(Function<? super V, ? extends T> before) {
        Objects.requireNonNull(before);
        return (V v) -> apply(before.apply(v));
    }

    default <V> Function<T, V> andThen(Function<? super R, ? extends V> after) {
        Objects.requireNonNull(after);
        return (T t) -> after.apply(apply(t));
    }

    static <T> Function<T, T> identity() {
        return t -> t;
    }
}

```

比如```andThen```方法就是说在进行第一次操作之后还需要进行的操作，下面的例子就是先跟集合中所有的元素加一个前缀，再加两次后缀，可以声明一个```Function```引用再进行连缀写法，阅读性好，最好不要直接匿名函数的方式写成一整条，阅读性极差。

```
public class MainClass {
    public static void main(String[] args) {
        List<String> list = new ArrayList<>(Arrays.asList("aaa", "bbb", "ccc", "ddd", "abd", "abc"));
        // 匿名写法
        // Function<String,String> function = ((Function<String,String>)(String str) -> "before:"+str).andThen((String str1) -> str1 + ":then1").andThen((String str1) -> str1 + ":then2");
        // System.out.println(filterStr(list,((Function<String,String>)(String str) -> "before:"+str).andThen((String str1) -> str1 + ":then1").andThen((String str1) -> str1 + ":then2")));

        Function<String,String> function1 = (String str) -> "before:"+str;
        Function<String,String> function2 = function1.andThen((String str1) -> str1 + ":then1").andThen((String str1) -> str1 + ":then2");

        System.out.println(filterStr(list,function2));
    }
}
```

再看看```Comparator```接口，它的抽象方法是```compareTo```方法，实际上还有很多其他的默认实现方法，比如```comparing```方法，而且它还返回的是一个```Comparator```接口类型，所以可以更加方便的使用连缀的方式调用```thenComparing```方法，下列代码实际上就是先按长度排序，再按第一个数字的大小来排序。

```
public class MainClass2 {
    public static void main(String[] args) {
        List<String> list = new ArrayList<>(Arrays.asList("111", "231", "152", "62", "82", "22", "0"));

        list.sort(Comparator.comparing(String::length).thenComparing((String str) -> str.indexOf(0)));

        System.out.println(list);
    }
}
```