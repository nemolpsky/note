## Java基础概念

---

1. 8种基本类型

    遵循byte/short/char->int->long->float->double的顺序，容量小的类型转换大类型的时候是自动转换，反过来则需要强制转换，因为可能丢失精度。

    - boolean
      
      只有true和false两个取值。

    - byte
    
      8位，最大存储数据量是255，存放的数据范围是-128~127之间。

    - short
      
      16位，最大数据存储量是65536，数据范围是-32768~32767之间。
    - char
      
      16位，存储Unicode码，用单引号赋值。
    - int
      
      32位，最大数据存储容量是2的32次方减1，数据范围是负的2的31次方到正的2的31次方减1。

    - long
    
      64位，最大数据存储容量是2的64次方减1，数据范围为负的2的63次方到正的2的63次方减1。

    - float
      
      32位，数据范围在3.4e-45~1.4e38，直接赋值时必须在数字后加上f或F。

    - double
      
      64位，数据范围在4.9e-324~1.8e308，赋值时可以加d或D也可以不加。



2. 继承

    - 子类可以拥有父类的私有属性和方法，但只是在内存中JVM运行层面上，实际上是没办法访问和调用的，所以从表像来看可以认为子类不会继承父类的私有属性和方法。

    - 子类无论调用哪一个构造方法，都会隐式默认的调用父类的无参构造方法。

    - 如果子类和父类都有用代码块和静态代码块的情况下，调用顺序会是，父类静态代码块(只有第一次会调用) -> 子类静态代码块(只有第一次会调用) -> 父类代码块  -> 父类无参构造方法 -> 子类代码块 -> 子类构造方法
    
    - 如果父类声明了一个有参构造方法，而子类又没有显式指明要调用哪一个父类的构造方法，会编译失败，因为子类会默认调用父类的无参构造方法，但是因为已经给父类添加了有参构造方法，所以JVM就不会默认的给父类添加无参构造方法了。 


3. 接口和抽象类

   - 接口中所有的方法都是默认由public修饰，也只能由public修饰，从Java 8开始可以允许接口中有静态实现方法。抽象类可以允许有非抽象的方法，而抽象方法可以由public、protected和default来修饰。

   - 接口中所有的属性都是被final和static修饰的，抽象方法可以任意选择。
   - 抽象类更像是一种模板设计，供其他类继承，而接口则是一种规范，规定所有实现类必须按照同一种规则。
   - 抽象类和接口都不能够实例化，下面这种创建方式看上去很像是实例化，其实是使用了匿名内部类。

     ```
        Interface i = new Interface() {
            @Override
            public void method() {
                
            }
        };
        
        Abstract a = new Abstract() {
            @Override
            protected void method() {
                
            }
        };
     ```

4. 局部变量和成员变量

    - 局部变量的生命周期是伴随着方法，而成员变量则是伴随着类。
    - 局部变量只能被final修饰，而成员变量可以被public、protected、private和static所修饰。
    - 局部变量是不会被自动赋值的，所以方法种的局部变量如果不初始化赋值调用的时候会出现编译错误提示，而成员变量则会自动初始化赋值。

5. 静态方法和实例方法

    - 静态方法除了能和实例方法一样使用"实例名.方法名"的方式调用外，还可以使用"类名.方法名"的方式调用。
    - 静态方法内部只能访问静态的对象和实例，因为实例方法必须是新建完对象后才可以调用，而静态方法则是JVM一加载字节码的时候就会加载，所以有可能静态方法已经加载好了，但是实例方法还没有被创建呢。

6. ==和equals()
    
    - 对于基本类型来说，==就是比较值，而且基本类型也只能使用==来比较
    - 对于对象来说==比较的是对象引用的内存地址值，也就是说如果两个引用指向同一个对象实例，使用==比较就会返回true。而equals()方法则是在Object类中声明的，而Object类是Java中所有类的顶级父类，也就是说所有类如果没有重写自己的equals()方法就会默认使用Object类里面的equals()方法，而Object类中对equals()方法的实现就是使用==比较，所以如果比较的对象没有重写equals()方法，那就相当于使用==来比较。

      ```
        // t1==t2，false
        // t1==t3，false
        Test t1 = new Test();
        Test t2 = new Test();
        Test t3 = t1;
      ```
    - String类是重写了equals()方法，所以String类的equals()方法是比较两个String的内容是否一样。

    - 所有的包装类也必须使用equals()方法来比较，因为对于Double和Float这样的浮点型数据用==会有精度缺失，而对于Long和Integer这样的整数类型，它们的源码内部有一个缓存池，范围是[-128，127]，所以如果是在这个数值范围内使用==两个相同的数字会返回true，在这个范围之外则会返回false，所以要统一使用equals()方法。

7. hashCode()与equals()

    - hashCode()方法也是Object类中定义的方法，它会返回一个int值，该值是Java计算出来的这个对象的哈希值。
    - 对象的哈希值通常是使用在HashSet等集合存储中，因为在HashS的源码中是使用散列表来存储元素，它是根据哈希值来判断元素是否重复，如果存储的是对象就会调用hashCode()来获取哈希值。
    - 实际上在HashSet中检测到有对象哈希值重复了还会调用equals()方法来再检测一遍，两者都检测到相同才会显示相同。所以必须对hashCode()也复写，否则每个对象计算出来的哈希值一定不一样，因为hashCode()是个本地方法，默认是根据对象在内存中的位置来计算哈希值。

8. 值传递

    Java中所有的方法参数都是将当前参数的值拷贝一份然后传递进来，如果是基本类型数据这样没有任何问题，比如下列的方法中int变量a传入之后又修改了值，但是a没有任何影响，因为只是拷贝了a的值再传到方法中。

    ```
    public static void main(String[] args){

        int a = 5;
        method(5);
        System.out.println(a);

    }

    public static void method(int i){
        i = 3;
        System.out.println(i);
    }
    ```
    但是引用类型的参数就不一样了，传递参数虽然也是拷贝，但是它拷贝的是引用本身，也就是说两个引用本身确实是分开的，但是都指向了堆内存中同一个对象，所以在方法里面的修改会影响到外部的对象。

    ```
    public static void main(String[] args){

        List<Integer> list = new ArrayList<>();
        list.add(3);
        method1(list);
        System.out.println(list.toString());

    }

    public static void method1(List<Integer> list){
        list.add(1);
        System.out.println(list.toString());
    }
    ```
    可以使用Java提供的拷贝的工具类方法类解决这个问题，对象也可以使用了类似的拷贝工具类。

    ```
        List<Integer> list = new ArrayList<>();
        list.add(3);
        // 拷贝旧集合中的值到新集合中
        List<Integer> list2 = new ArrayList<>(list.size());
        Collections.copy(list,list2);

        int[] arr = new int[5];
        arr[0] = 1;
        // 拷贝一个旧数组的值到新数组中
        method1(Arrays.copyOf(arr,5));
        System.out.println(Arrays.toString(arr));

    ```
9. 浅拷贝和深拷贝

    这两种拷贝是复制一个新的对象出来，浅拷贝是指被复制对象中的引用类型数据只拷贝了引用，深拷贝则是指连引用类型数据也完全拷贝了一个新的对象出来，引用指向的对象都不是同一个。

    - 浅拷贝

      首先浅拷贝要在所有被拷贝的对象上实现Cloneable接口，然后重写clone()方法，最后调用clone()方法即可，可以看到浅拷贝对内部的引用类型数据是拷贝的引用，所以修改新对象会影响到旧对象。

      ```
      public class CloneTest implements Cloneable{

          int id;
          InnerClass innerClass;

          public CloneTest(int id, InnerClass innerClass) {
              this.id = id;
              this.innerClass = innerClass;
          }

          @Override
          public String toString() {
              return "CloneTest{" +
                      "id=" + id +
                      ", innerClass=" + innerClass +
                      '}';
          }

          @Override
          protected Object clone() throws CloneNotSupportedException {
              return super.clone();
          }
      }

      public class InnerClass implements Cloneable{
          int id;

          public InnerClass(int id) {
              this.id = id;
          }

          @Override
          public String toString() {
              return "InnerClass{" +
                      "id=" + id +
                      '}';
          }
      }

      public static void main(String[] args) throws Exception{

        InnerClass innerClass = new InnerClass(10);
        CloneTest cloneTest = new CloneTest(15,innerClass);

        CloneTest cloneTest1 = (CloneTest)cloneTest.clone();
        System.out.println("新对象-"+cloneTest1);
        cloneTest1.id = 5;
        cloneTest1.innerClass.id=20;
        System.out.println("修改新对象-"+cloneTest1);
        System.out.println("旧对象-"+cloneTest);

      }

      新对象-CloneTest{id=15, innerClass=InnerClass{id=10}}
      修改新对象-CloneTest{id=5, innerClass=InnerClass{id=20}}
      旧对象-CloneTest{id=15, innerClass=InnerClass{id=20}}
      ```
    - 深拷贝

      第一种方法是在重写的Clone()方法里面，自己手动去调用引用类型数据的clone()方法，再手动赋值。

      ```
      @Override
      protected Object clone() throws CloneNotSupportedException {
          CloneTest cloneTest = (CloneTest)super.clone();
          cloneTest.innerClass = (InnerClass)innerClass.clone();
          return cloneTest;
      }
      ```

      第二个方法是序列化后再反序列化，这样也可以解决问题，但是要实现Serializable接口。

      ```
      public static void main(String[] args) throws Exception{

          InnerClass innerClass = new InnerClass(10);
          CloneTest cloneTest = new CloneTest(15,innerClass);

          // 序列化
          ByteArrayOutputStream bos = new ByteArrayOutputStream();
          ObjectOutputStream oos = new ObjectOutputStream(bos);
          oos.writeObject(cloneTest);
          // 反序列化
          ByteArrayInputStream bis = new ByteArrayInputStream(bos.toByteArray());
          ObjectInputStream ois = new ObjectInputStream(bis);

          CloneTest cloneTest1 = (CloneTest)cloneTest.clone();
          System.out.println("新对象-"+cloneTest1);
          cloneTest1.id = 5;
          cloneTest1.innerClass.id=20;
          System.out.println("修改新对象-"+cloneTest1);
          System.out.println("旧对象-"+cloneTest);

      }
      ```
10. final关键字

    final关键字可以修饰类、方法和变量。

    - 修饰基本类型变量，这个变量的值就不可以改变，修饰引用类型变量，这个变量指向的对象就不可以改变。
    - 修改类，这个类就不可以被继承，而类中所有的方法都会被隐式的使用final修饰。
    - 修饰方法，这个方法就不可以被重写。

11. transient

      只能用来修饰变量，作用是序列化的时候忽略这些变量。