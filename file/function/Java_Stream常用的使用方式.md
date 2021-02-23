### Java Stream常用的使用方式

Java 8的Stream配合Lambda可以使用函数式编程，函数式编程有个很大的优点就是可以简化无用的模板代码，代码中尽量只包含逻辑代码，有一些比较常用的使用方式可以极大的简化代码，阅读性也大大的提高了。

---

1.  生成集合

```
// 迭代循环加1，因为是无限流，使用limit方法限制100个元素，配合收集器可以转换成各种各样的集合
// 如果所有元素都是单独生成则可以使用generate代替iterate
List<Integer> list = Stream.iterate(0, i -> i + 1).limit(100).collect(toList());
```

2. flatMap合并流

```
// 生成两个集合
List<Integer> list1 = Stream.iterate(0, i -> i + 1).limit(100).collect(toList());
List<Integer> list2 = Stream.iterate(0, i -> i + 2).limit(100).collect(toList());
// 经常遇到合并多个集合进行再操作，flatMap就是把多个流合并成1个，再进行汇总操作
long count = Stream.of(list1,list2).flatMap(Collection::stream).reduce(0,Integer::sum);
```

3. groupingBy分组

```
public class Student{
    private String grade;
    private Integer age;
    private String subject;
    private String name;
    private Double mark;
}

// 按属性分组
Map<String,List<Student>> map = students.stream().collect(groupingBy(Student::getName));
// 二级分组
Map<Double,Map<String,List<Student>>> map = students.stream().collect(groupingBy(Student::getMark,groupingBy(Student::getSubject)));
// 自定义分组key名
Map<String,List<Student>> map = students.stream().collect(groupingBy(student ->{
        if (student.getMark().intValue() >= 85){
            return "合格";
        }else{
            return "不合格";
        }
    }));

```

4. 数值流

```
List<Integer> list1 = Stream.iterate(0, i -> i + 1).limit(100).collect(toList());
list1.add(null);
// 总共有三种数值流，int，double和long，就是为了解决包装类拆箱装箱的问题，但是要注意，转化过程中的空指针问题要自己解决
long num = list1.stream().filter(Objects::nonNull).mapToInt(i -> i).count();
```