### JVM类加载机制

JVM会把一个Java文件编译成.class文件，再把这个.class文件根据当前运行的硬件平台翻译成对应的机器码进行执行。JVM有一种特殊的加载模式叫做双亲委派模式。

---


#### 1. 加载流程

下面这张图一定见过非常多次了，大致上分为5个步骤，大概了解下每个步骤的作用就行了。


![4](https://github.com/nemolpsky/note/raw/master/file/jvm/images/4.png)

- 加载，把Class文件里面的内容加载到JVM中。
- 验证，准备，解析
  
  验证，Class文件格式是有严格要求的，所以校验Class文件的格式，同时校验Java语法以及代码引用等等。
  
  准备，开始给静态变量分配内存，初始化默认值。
  
  解析，开始给代码中使用到的对象生成引用，并指向对象的地址。
  
- 初始化，给对象中的各种变量初始化值，和上面准备步骤不同的是赋予代码中指定的值，比如int=3。
- 使用，就是正常调用
- 卸载，从JVM中移除，释放内存

---

#### 2. 双亲委派模式

上面的加载流程就像是加载特定格式的文件一样，重点在于Java的加载机制，Java有下面3个类加载器。
- 启动类加载器，负责加载存放在<JAVA_HOME>\lib目录下或者是被-Xbootclasspath参数所指定的路径中的类库。
- 扩展类加载器，负责加载存放在<JAVA_HOME>\lib\ext目录或者是java.ext.dirs系统变量所指定的路径中的类库，这个加载器可以直接使用。
- 应用程序类加载器，也称为系统类加载器。负责加载用户类路径(ClassPath)上所指定的类库，这个加载器我们也可以直接使用。

除了这3个类加载器之外，还可以自定义类加载器，要求符合这种父子关系，双亲委派模式就是加载一个类时，会传递给自己的父类加载器，这样依次传到最顶端的类加载器上，然后开始执行加载操作，如果失败再往下传，这样就可保证同一个Class文件一定会是由同一个类加载器加载出来的。

![5](https://github.com/nemolpsky/note/raw/master/file/jvm/images/5.png)

因为JVM是会判断是否由同一个类加载器进行加载，同一个Class文件还必须由同一个类加载器加载才能是同一个类，比如下面这个例子，都是加载Test类，会发现自定义类加载器加载出来的类会和默认加载的属于不同类。

```
    public static void main(String[] args) throws Exception {
        ClassLoader myClassLoader = new ClassLoader(){
            @Override
            public Class<?> loadClass(String name) throws ClassNotFoundException {
                try {
                    String fileName = name.substring(name.lastIndexOf(".") + 1)+".class";
                    InputStream is = getClass().getResourceAsStream(fileName);
                    if (is == null) {
                        return super.loadClass(name);
                    }
                    byte[] b = new byte[is.available()];
                    is.read(b);
                    return defineClass(name, b, 0, b.length);
                } catch (IOException e) {
                    e.printStackTrace();
                }
                return null;
            }
        };
        Object object = myClassLoader.loadClass("com.lp.chain.Test").newInstance();
        System.out.println(object.getClass().getClassLoader());
        System.out.println(object instanceof Test);

        Object object1 = Launcher.getLauncher().getClassLoader().loadClass("com.lp.chain.Test").newInstance();
        System.out.println(object1.getClass().getClassLoader());
        System.out.println(object1 instanceof Test);
    }
```

---

#### 3. 类隔离
在一个大型项目中经常会遇到一个问题，有些依赖jar包是被多个模块所用到，但是刚好这些jar包版本不一致，会产生冲突，这个时候就可以通过破坏双亲委托模式，自定义类加载器，不同版本的jar包用不同的类加载器去加载，这样就不会产生冲突了。

JVM中还有一个规则，一个类和它的引用类必定会是同一个类加载器加载，基于这些特性可以更好的实现不同模块使用不同版本的jar包，其实tomcat就是有这种机制。

OSGI(Open Service gateway initactive)这类技术也是这个原理，动态的加载模块就是依赖于自定义的类加载器。
