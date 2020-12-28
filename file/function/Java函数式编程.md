### Java函数式编程

函数式编程是Java 8引入的一个最重要的特性，其实函数式编程最简单的理解就是函数本身也可以作为参数和返回值来进行使用，当然还有其它一些特性，但是这一点是函数式编程中最重要的特点之一。函数式编程其实早在别的语言中就已经有了，比如C语言就支持函数式编程。现在的趋势也都是慢慢偏向于函数式编程，使用得当的话，代码会很直观，扩展性也会很好，但是如果使用不当，也会是一场灾难。


---


#### 1. 函数传递和Lambda表达式

现在有一个最常见的需求，过滤集合中指定的字符串，例如只保留字符串中包含字母```a```的字符串，并符合要求的字符串集合。最简单的方式肯定就是直接for循环，但是一般这种常见的需求都会使用设计模式方便以后扩展，比如下面其实就是一个策略模式的实现，接口中定义了抽象的过滤方法，集合迭代调用过滤器来进行过滤，使用过滤器的时候匿名类重写的方式进行具体的实现，这也是Java以前很常见的是使用方式，扩展性其实也不错，就是代码会看上去比较长，有很多固定的模板代码。

```
public interface StrFilter {
    
    /**
     *  抽象过滤方法
     * @param str1 被过滤的字符串
     * @param str2 需要过滤的关键字
     * @return
     */
    boolean filter(String str1,String str2);
}

public class MainClass {

    public static void main(String[] args) {
        List<String> list = new ArrayList<>(Arrays.asList("aaa", "bbb", "ccc", "ddd", "abd", "abc"));
        List<String> newList1 = filterStr(list, "a", new StrFilter() {
            @Override
            public boolean filter(String str1, String str2) {
                return str1.contains(str2);
            }
        });
        System.out.println(newList1);
    }

    /**
     *
     * @param list 需要筛选的集合
     * @param str2 筛选的字段
     * @param filter 筛选器
     * @return
     */
    public static List<String> filterStr(List<String> list, String str2, StrFilter filter) {
        List<String> newList = new ArrayList<>();
        // 遍历集合中的每个元素
        for (String str1 : list) {
            // 通过筛选器判断元素是否要被过滤
            if (filter.filter(str1, str2)) {
                newList.add(str1);
            }
        }
        return newList;
    }
}
```

其实上面的匿名方法在Java 8之前算是最简单的方式了，如果想要代码好看，那就只能自己定义实现类，创建对象进行传参，但是工作量又多了，而如果使用Java 8的函数式编程就不一样了，就如同最开始介绍的那样，把函数当做参数使用，首先在接口中定义一个静态的默认方法，其实本质上就理解为类中的正常方法即可。然后直接使用```::```的方式把```defaultFilter```方法当做参数传递进```filterStr```方法中，这种方式被称为方法引用。

注意，```filterStr```方法需要接受一个```StrFilter```类型的引用，然后调用其对应的```filter```实现方法来进行过滤操作，而```StrFilter::defaultFilter```其实就表示把```defaultFilter```方法当做这个实现方法来进行过滤操作，这就是所谓的传递函数。

```
public interface StrFilter {
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

public class MainClass {

    public static void main(String[] args) {
        List<String> list = new ArrayList<>(Arrays.asList("aaa", "bbb", "ccc", "ddd", "abd", "abc"));
        List<String> newList = filterStr(list,"a", StrFilter::defaultFilter);
        System.out.println(newList);
    }
}
```

Lambda表达式其实就是匿名函数的意思，就像最初使用匿名类方式实现```StrFilter```接口一样，按照上面的方法引用方式进行函数传递还需要事先实现好方法才能使用，其实函数式编程也是支持匿名函数，也就是Lambda表达式。比如下面这个例子，前面的```(s1,s2)```表示函数的参数，箭头是固定格式，而```s1.contains(s2)```则是具体实现，这个匿名函数其实就是对```StrFilter```接口中的```filter(String str1,String str2)```方法的匿名实现，只要参数个数、类型和返回结果能符合声明就算是一个正确的匿名函数。

```
public class MainClass {

    public static void main(String[] args) {
        List<String> list = new ArrayList<>(Arrays.asList("aaa", "bbb", "ccc", "ddd", "abd", "abc"));
        List<String> newList = filterStr(list,"a",(s1,s2) -> s1.contains(s2));
        System.out.println(newList);
    }
}
```

甚至还可以在匿名函数的基础上使用方法引用，```filter(String str1,String str2)```方法的两个参数都是```String```类型的，所以```String::contains```的意思其实就是调用```str1```的```contains```方法，```str2```作为```contains```方法的参数。

```
public class MainClass {

    public static void main(String[] args) {
        List<String> list = new ArrayList<>(Arrays.asList("aaa", "bbb", "ccc", "ddd", "abd", "abc"));
        List<String> newList = filterStr(list,"a",String::contains);
        System.out.println(newList);
    }
}
```

---

#### 2. 原有类库使用Lambda表达式

比如```Comparator```接口是原来Java自带的用于排序的接口，使用方式也是匿名类的方式，改成Lambda表达式也是和上面的原则一样，首先两个参数要对应，然后箭头后面则是对这两个参数进行逻辑操作，返回符合接口方法定义的类型。

```
public class MainClass {

    public static void main(String[] args) {
        List<Integer> list1 = new ArrayList<>(Arrays.asList(22,11,25,66,1,63,29));
        list1.sort(new Comparator<Integer>() {
            @Override
            public int compare(Integer o1, Integer o2) {
//                return o1-o2;
                return o1.compareTo(o2);
            }
        });
        System.out.println(list1);

        // use lambda
        List<Integer> list2 = new ArrayList<>(Arrays.asList(22,11,25,66,1,63,29));
//        list2.sort((i1,i2) -> i1-i2);
//        list2.sort((i1,i2) -> i1.compareTo(i2));
        list2.sort(Integer::compareTo);
        System.out.println(list2);
    }
}
```

新建线程时使用```Runnable```接口也是可以使用Lambda表达式，没有参数就是用```()```表示，箭头后面是```run```方法中执行的逻辑。

```
public class MainClass {

    public static void main(String[] args) {
        new Thread(new Runnable() {
            @Override
            public void run() {
                System.out.println(Thread.currentThread().getName());
            }
        }).start();

        new Thread(() -> System.out.println(Thread.currentThread().getName())).start();
    }
}
```

其实查看这两个接口可以发现它们都被```@FunctionalInterface```修饰，这个注解表示该接口可以使用Lambda来使用，不过只是一个标准，就算没有这个注解修饰只要接口符合Lambda的条件，也依然可以使用Lambda表达式。

```
@FunctionalInterface
public interface Runnable {
    ...
}

@FunctionalInterface
public interface Comparator<T> {
    ...
}
```