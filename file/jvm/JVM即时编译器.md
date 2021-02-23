### JVM的即时编译器

即时编译器，简称JIT(Just-In-Time)编译器，像C和Java这一类的语言都被称为编译型语言，也就是说代码会先进行编译，一次性编译完之后再执行编译后产生的文件。而Python和JavaScript这一类都属于解释型语言，也就是说它们是依靠解释器来逐行读取代码并转换成机器可识别的语言。

Java其实和C语言还略有不同，C语言直接就把代码编译成了机器可以执行的语言，而Java则会先编译成.class文件，这种文件里面都是Java字节码组成的，然后Java虚拟机再把这些.class文件转换成机器可以识别的语言，最后这一步把字节码转换成机器执行的二进制代码就被成为即时编译，之所以叫即时编译就是因为是在程序执行时进行编译的。

其实有一点要注意，当JVM执行代码时，并不会立刻把字节码进行编译，它第一遍只是像解释型语言那样，解释执行字节码，这么做的目的是很多时候有相当一部分数量的代码只会执行很少的次数，甚至只是执行一次，如果第一遍执行就编译，速度比解释执行要慢，还需要内存来存储编译文件，所以JIT存在的意义在于它会识别有哪些代码是频繁执行的，这些代码就会被编译，而且编译的过程会进行优化。


---

#### 1. 编译器类型

编译器可以分为client类型和server类型，两者也被简称为C1编译器和C2编译器。这两者之间的差别在于C1编译器执行编译的时间比C2要早，所以在程序早期运行的时候会启动的更快，C2编译器执行编译的时间要晚一点，但正因为如此它能收集到更多有用的信息来优化编译，所以C2编译器编译出来的代码性能更好。其实这也是有历史原因的，很多年前还有Java GUI这样的程序，它希望启动快速，又不像服务器程序那样长久运行，不过现在的软件开发中Java基本上都是作为后端服务使用了。

下面这张图就是不同的机器上，不同的版本对应的默认的编译器。-d64表示64位的C2编译器，可以看到其实在64位Windows系统和Linux上，如果安装了64位的JVM那其实C1编译器是无法使用的，也就是说一定会使用C2编译器，所以后面所有的例子其实都是基于C2编译器的基础进行讨论。

![2](https://github.com/nemolpsky/note/raw/master/file/jvm/images/2.png)

不过上面的按照-client和-server命令来指定编译器是Java 7的做法，在Java 8的时候默认开启了分层编译，也就是C1和C2共用，使用C1加快程序启动速度，使用C2在运行时编译优化热点代码，所以-client和-server在Java 8已经失效了。查看Java版本的时候其实是有显示当前使用的编译器，mixed mode表示编译使用的是混合模式，也就是分层编译。
```
root@Half-PC:/# java -version
java version "1.8.0_261"
Java(TM) SE Runtime Environment (build 1.8.0_261-b12)
Java HotSpot(TM) 64-Bit Server VM (build 25.261-b12, mixed mode)
root@Half-PC:/# java -client -version
java version "1.8.0_261"
Java(TM) SE Runtime Environment (build 1.8.0_261-b12)
Java HotSpot(TM) 64-Bit Server VM (build 25.261-b12, mixed mode)
```
---


#### 2. 编译优化
在编译的过程中，产生的编译代码会保存在JVM的代码缓存中，-XX:ReservedCodeCacheSize=N和-XX: InitialCodeCachesize=N两个参数分别代表了代码缓存的初始大小和最大值，JVM有设置默认值，可以通过下列命令查看JVM的全局配置，列表会很长。

```
java -XX:+PrintFlagsFinal -version
```

上面说过，C2编译器的作用就是编译优化热点代码，说白了就是把频繁运行的代码进行编译优化，那肯定会有一个阀值，JVM有两种方式来累加阀值，第一个方法的调用次数，第二个则是方法中循环体的执行次数，如果满足阀值就会把方法放入编译队列，这也是所谓的标准编译，阀值由-XX: CompileThreshold=N参数配置。

但是还有另一种情况，比如一个循环永远不结束，这种情况也是有的，比如专门做轮询的方法，这种就是依靠上面说的第二种方式来统计，循环的次数，但是JVM会使用一种叫做栈上替换(On-Stack Replacement, OSR)的编译方式来编译，说白了就是直接把编译好的循环体内部的代码替换掉方法栈上执行的代码，OSR的触发则比较复杂，是由下列三个参数计算。

```
(CompileThreshold*((OnStackReplacePercentage - InterpreterProfilePercentage)/100))
```

不过要注意的是，C2编译器的两个计数器，是会周期性的削减次数，所有如果一个方法的调用频率不够高，那么很可能永远都不会触发编译，可以通过降低阀值来解决。


---

#### 3. 观察编译过程
启动JVM时，可以指定-XX:+PrintCompilation参数打印编译过程，比如下面的代码只有一个main方法，循环调用一个静态的方法。
```
public class MainClass {

    public static void main(String[] args) throws InterruptedException {
        System.out.println("start main class!");
        while (true){
            forEach();
            TimeUnit.MILLISECONDS.sleep(10);
        }
    }

    public static void forEach(){
        int i = 1+1;
    }
}
```

下面是这段代码运行了差不多40分钟后的结果，最多可以看到有7列数据，第一列是时间戳，第二列是编译任务的id，第三列是编译的属性，第四列是分层编译的方式，第五列是编译的方法，第六列是编译后代码的大小，最后一列是逆优化的信息。
```
root@Half-PC:/opt/jar# java -XX:+PrintCompilation -jar JVMDemo-1.0-SNAPSHOT.jar
     67    1       3       java.lang.String::hashCode (55 bytes)
     68    3     n 0       java.lang.System::arraycopy (native)   (static)
     69    2       3       java.lang.String::equals (81 bytes)
     70    4       3       java.lang.String::charAt (29 bytes)
     71    5       3       java.lang.String::length (6 bytes)
     74    6       3       java.lang.Object::<init> (1 bytes)
     76    7       3       java.util.jar.Attributes$Name::isValid (32 bytes)
     78    8       3       java.util.jar.Attributes$Name::isAlpha (30 bytes)
     79   11       3       java.lang.Math::min (11 bytes)
     79    9       3       java.lang.AbstractStringBuilder::ensureCapacityInternal (27 bytes)
     80   10       1       java.lang.ref.Reference::get (5 bytes)
     81   13       3       java.lang.AbstractStringBuilder::append (50 bytes)
     81   14       3       java.lang.CharacterData::of (120 bytes)
     82   15       3       java.lang.CharacterDataLatin1::getProperties (11 bytes)
     82   12       1       java.lang.ThreadLocal::access$400 (5 bytes)
     83   16       3       java.lang.String::indexOf (70 bytes)
start main class!
   1432   17       1       java.util.concurrent.TimeUnit$3::excessNanos (2 bytes)
   2790   18       3       MainClass::forEach (3 bytes)
   2791   19       3       java.util.concurrent.TimeUnit::sleep (27 bytes)
   4139   20       3       java.util.concurrent.TimeUnit$3::toMillis (2 bytes)
   4140   21       3       java.lang.Thread::sleep (61 bytes)
   4150   22     n 0       java.lang.Thread::sleep (native)   (static)
  13594   23       1       MainClass::forEach (3 bytes)
  13595   18       3       MainClass::forEach (3 bytes)   made not entrant
  14937   24       1       java.util.concurrent.TimeUnit$3::toMillis (2 bytes)
  14939   20       3       java.util.concurrent.TimeUnit$3::toMillis (2 bytes)   made not entrant
  57080   25       4       java.util.concurrent.TimeUnit::sleep (27 bytes)
  57083   19       3       java.util.concurrent.TimeUnit::sleep (27 bytes)   made not entrant
 644666   26 %     3       MainClass::main @ 8 (23 bytes)
 655446   27       3       MainClass::main (23 bytes)
 666392   28       4       java.lang.Thread::sleep (61 bytes)
 666394   21       3       java.lang.Thread::sleep (61 bytes)   made not entrant
1080114   29 %     4       MainClass::main @ 8 (23 bytes)
1080118   26 %     3       MainClass::main @ -2 (23 bytes)   made not entrant
```

使用jstat命令也可以看到编译的数量。
```
root@Half-PC:/opt/jar# jstat -compiler 1483
Compiled Failed Invalid   Time   FailedType FailedMethod
      27      0       0     0.02          0
root@Half-PC:/opt/jar# jstat -printcompilation 1483 1000
Compiled  Size  Type Method
      27    118    1 MainClass main
      27    118    1 MainClass main
      27    118    1 MainClass main
      27    118    1 MainClass main
```

---

#### 4. 编译属性和分层编译
上面编译日志第三列的编译属性代表下面的含义，所以可以看到其中线程睡眠调用的本地方法，所以属性为n，main方法内有循环体，所以有%，也就是栈上替换。

```
%表示编译为OSR
s表示方法是同步的
!表示方法有异常处理器
b表示阻塞模式时发生的编译
n表示为封装本地方法所发生的编译
```

分层编译在前面解释过是C1和C2混合编译，因此编译日志上也可以看到分层编译是在哪一层，每一层代表不同的方式。通过上面的日志发现，在过了一段时间后，main方法从3层变为4层，也就是说由C2编译器进行编译。同时可以看到后面紧随其后的一条日志，这条日志表明会把编译层级为3层的main方法代码丢弃，这就是所谓的逆优化，而且还是提供性能的逆优化，不过还有降低性能的逆优化，一般情况会发生在方法执行依赖参数判断的情况下，JVM发觉执行方法的参数变了，不得不把编译好的代码丢弃，编译新的代码出来。

```
0表示解释代码
1表示简单C1编译代码
2表示受限的C1编译代码
3表示完全C1编译代码
4表示C2编译代码
```

```
1080114   29 %     4       MainClass::main @ 8 (23 bytes)
1080118   26 %     3       MainClass::main @ -2 (23 bytes)   made not entrant
```

---

#### 5. 编译优化方式

在JVM的优化中，甚至说很多编译型语言中，最最最重要的优化方式就是内联，其实就是直接略过引用调用，直接复制使用参数值。这种优化方式效率提高很多，但是实际难度也会很复杂。可以配置-XX:-Inline参数关闭内联优化，不过真实环境中没人这么做。

```
public class User {
    
    private int age;

    public int getAge() {
        return age;
    }

    public void setAge(int age) {
        this.age = age;
    }
}

// 源代码
User user = new User();
int age = user.getAge;
// 内联优化后的代码
User user = new User();
int age = user.age;
```


还有一种非常激进的优化方式，叫做逃逸分析，可以配置-XX:+DoEscapeAnalysis参数来控制，默认开启。以下面这段代码为例子，User类中有一个同步的方法，然后在for循环中调用这个方法获得一个结果存入list中。如果JVM发现，代码中只有循环内部使用了对象的引用，那就直接把同步锁给优化掉了，因为不可能会有同步问题，甚至说都不用分配完整的对象出来，直接编译这个方法出来让外部使用即可，诸如此类的，逃逸分析是JVM能做到的最最最复杂的编译，甚至有时候在同步情况下优化可能会造成代码出问题。

```
public class User {
    
    private int age;

    public int getAge() {
        return age;
    }

    public void setAge(int age) {
        this.age = age;
    }

    public synchronized int getCalAge(){
        // do something
    }
}

ArrayList<BigInteger> list = new ArrayList<BigInteger>();
for (int i=0; i < 100; i++) {
    User user = new User();
    list.add(user.getCalAge());
}
```


---

### 总结
- JVM执行代码的实际过程，尤其是字节码到机器语言这个编译过程会有很多内部的优化，甚至可能激进的优化成和源代码完全不一样的语句，但是它能保证结果是最终一致的。
- 编译优化是非常复杂的过程，因为如果代码能够写的简洁，职责清晰，耦合度低，这样对JVM的优化其实会有很大的帮助。
- 也正是由于编译优化的复杂性，在要考虑同步问题的代码中，往往会隐藏着很多问题。