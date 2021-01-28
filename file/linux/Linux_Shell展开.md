### Linux Shell展开解析

了解Linux Shell之前很有必要了解下操作系统之间的渊源，首先Linux是GNU计划的产物，是完全开源免费的。而Linux又是一个类Unix系统，也就是界面，操作都很相似，一些软件也可以互相兼容运行，但是Linux的源代码并不是基于Unix的源代码进行开发的，Unix是收费的，所以说是类Unix系统。

Shell也是Unix最早使用的，就是系统用户与Unix之间交互使用的命令行，本质上Shell是一类脚本语言，用户可以通过这个脚本语言来控制Unix系统，Linux既然是类Unix系统，那当然也会有Shell命令来支持交互了。

而其实Shell也有很多变种，这些变种之间的关系可以理解为C和C++关系，Linux中最常用的Shell就是Bash(Bourne-Again SHell)，不过各种Shell基本都能兼容，绝大多数LInux系统都是使用Bash。

```
half@Half-PC:~/test$ echo $SHELL
/bin/bash
half@Half-PC:~/test$ bash --version
GNU bash, version 5.0.17(1)-release (x86_64-pc-linux-gnu)
Copyright (C) 2019 Free Software Foundation, Inc.
License GPLv3+: GNU GPL version 3 or later <http://gnu.org/licenses/gpl.html>

This is free software; you are free to change and redistribute it.
There is NO WARRANTY, to the extent permitted by law.
```

---


#### 1.  Shell展开

也有称作字符展开，参数展开，其实可以理解为Shell语言对参数的处理，比如命令中```~```符号表示当前用户的家目录，```.```表示当前目录，```..```表示父目录，这些符号都有特殊含义，这个解析过程就称作展开。

使用```set```命令可以打开和关闭参数展开的过程，更清晰的看到最终执行的到底是什么参数。
```
set -x // 显示展开过程
set +x // 不显示展开过程
```

现在看看下面这两条命令，第一条可以看到是输出一条文本，第二条则从```*```变为了当前目录下所有文件，因为```*```就是代表当前目录，如果直接输入```*```的话，bash就会把它当成当前目录，而不是一个字符，星号。
```
half@Half-PC:~$ echo a simple txt.
+ echo a simple txt.
a simple txt.

half@Half-PC:~$ echo *
+ echo test
test

half@Half-PC:~$ ls
+ ls --color=auto
test
```

所以有各种各样的参数展开，其实你就理解为各种各样的预定义的符号解析就行，这个时候你可能就明白了为啥都说Linux命令功能强大，但是又特别难学，因为本质上就是一门动态编程语言，当然功能强大上手难了。
```
half@Half-PC:~$ echo $((2 + 2))
+ echo 4
4

half@Half-PC:~$ echo Front-{A,B,C}-Back
+ echo Front-A-Back Front-B-Back Front-C-Back
Front-A-Back Front-B-Back Front-C-Back

half@Half-PC:~$ echo Number_{1..5}
+ echo Number_1 Number_2 Number_3 Number_4 Number_5
Number_1 Number_2 Number_3 Number_4 Number_5

half@Half-PC:~$ echo {Z..A}
+ echo Z Y X W V U T S R Q P O N M L K J I H G F E D C B A
Z Y X W V U T S R Q P O N M L K J I H G F E D C B A

half@Half-PC:~$ echo a{A{1,2},B{3,4}}b
+ echo aA1b aA2b aB3b aB4b
aA1b aA2b aB3b aB4b

half@Half-PC:~$ mkdir {2007..2009}-0{1..9} {2007..2009}-{10..12}
+ mkdir 2007-01 2007-02 2007-03 2007-04 2007-05 2007-06 2007-07 2007-08 2007-09 2008-01 2008-02 2008-03 2008-04 2008-05 2008-06 2008-07 2008-08 2008-09 2009-01 2009-02 2009-03 2009-04 2009-05 2009-06 2009-07 2009-08 2009-09 2007-10 2007-11 2007-12 2008-10 2008-11 2008-12 2009-10 2009-11 2009-12

half@Half-PC:~$ ls
+ ls --color=auto
2007-01  2007-04  2007-07  2007-10  2008-01  2008-04  2008-07  2008-10  2009-01  2009-04  2009-07  2009-10  test
2007-02  2007-05  2007-08  2007-11  2008-02  2008-05  2008-08  2008-11  2009-02  2009-05  2009-08  2009-11
2007-03  2007-06  2007-09  2007-12  2008-03  2008-06  2008-09  2008-12  2009-03  2009-06  2009-09  2009-12

half@Half-PC:~$ echo $USER
+ echo half
half
```

---

#### 2. 命令替换

其中```$```比较重要，可以看作是变量的标识，比如系统中定义的变量，或者是一个Shell脚本中定义的变量。

```
half@Half-PC:~$ echo $USER
+ echo half
half
```

命令替换其实也好理解，就是依赖于```$```符号，可以把要执行的命令先放在```$()```中，这样echo的参数其实就变成这条命令执行完的输出了，也可以直接用反引号代替，比如下面这3条命令就可以看出到底执行了什么参数。

```
half@Half-PC:~$ echo $(ls)
++ ls --color=auto
+ echo 2007-01 2007-02 2007-03 2007-04 2007-05 2007-06 2007-07 2007-08 2007-09 2007-10 2007-11 2007-12 2008-01 2008-02 2008-03 2008-04 2008-05 2008-06 2008-07 2008-08 2008-09 2008-10 2008-11 2008-12 2009-01 2009-02 2009-03 2009-04 2009-05 2009-06 2009-07 2009-08 2009-09 2009-10 2009-11 2009-12 test
2007-01 2007-02 2007-03 2007-04 2007-05 2007-06 2007-07 2007-08 2007-09 2007-10 2007-11 2007-12 2008-01 2008-02 2008-03 2008-04 2008-05 2008-06 2008-07 2008-08 2008-09 2008-10 2008-11 2008-12 2009-01 2009-02 2009-03 2009-04 2009-05 2009-06 2009-07 2009-08 2009-09 2009-10 2009-11 2009-12 test

half@Half-PC:~$ ls -l $(which cp)
++ which cp
+ ls --color=auto -l /usr/bin/cp
-rwxr-xr-x 1 root root 153976 Sep  5  2019 /usr/bin/cp

half@Half-PC:~$ ls -l `which cp`
++ which cp
+ ls --color=auto -l /usr/bin/cp
-rwxr-xr-x 1 root root 153976 Sep  5  2019 /usr/bin/cp

half@Half-PC:~$ ls -l which cp
+ ls --color=auto -l which cp
ls: cannot access 'which': No such file or directory
ls: cannot access 'cp': No such file or directory
```


#### 3. 引用
参数展开的特性确实很强大，也很方便，但是也有个问题，有时候我执行的命令中刚好就包含了特殊字符，但是在命令中只是一个普通字符，bash却给我展开了，怎么办？这个时候就要提到一种叫做引用(Quoting)的特性。

正常情况下，bash会把空格当做参数分隔符，所以多余的空格就会被当做空参数省略掉，使用双引号```""```可以把整个双引号内的字符串当做一整个命令，不再分割解析展开参数，对```~```符号也是如此，但是对```$```符号无效，这就是所谓的引用。
```
half@Half-PC:~/test$ echo this is a                         line.
+ echo this is a line.
this is a line.

half@Half-PC:~/test$ echo "this is a                              line."
+ echo 'this is a                              line.'
this is a                              line.

half@Half-PC:~/test$ echo this is a $S .
+ echo this is a .
this is a .

half@Half-PC:~/test$ echo "this is a $S ."
+ echo 'this is a  .'
this is a  .

half@Half-PC:~/test$ echo this is ~ .
+ echo this is /home/half .
this is /home/half .

half@Half-PC:~/test$ echo "this is ~ ."
+ echo 'this is ~ .'
this is ~ .
```

当然也可以配合转义符号来解决```$```的展开问题。
```
half@Half-PC:~/test$ echo "The balance for user $USER is: \$5.00"
+ echo 'The balance for user half is: $5.00'
The balance for user half is: $5.00

half@Half-PC:~/test$ echo "The balance for user $USER is: $5.00"
+ echo 'The balance for user half is: .00'
The balance for user half is: .00
```

还有就是使用单引号```''```，直接把所有的展开都禁止，输入什么都当做是个字符串。
```
half@Half-PC:~/test$ echo 'The balance for user $USER is: \$5.00'
+ echo 'The balance for user $USER is: \$5.00'
The balance for user $USER is: \$5.00

half@Half-PC:~/test$ echo 'The balance for user $USER is: $5.00'
+ echo 'The balance for user $USER is: $5.00'
The balance for user $USER is: $5.00
```

在管道模式下使用```xargs```则是指定展开模式，也就是把前者的标准输出作为后者的命令参数，而不是作为后者的标准输入。
```
half@Half-PC:~/test$ which cp | xargs file
+ xargs file
+ which cp
sh: 0: getcwd() failed: No such file or directory
/usr/bin/cp: ELF 64-bit LSB shared object, x86-64, version 1 (SYSV), dynamically linked, interpreter /lib64/ld-linux-x86-64.so.2, BuildID[sha1]=421e1abd8faf1cb290df755a558377c5d7def3b1, for GNU/Linux 3.2.0, stripped

half@Half-PC:~/test$ file $(which cp)
++ which cp
sh: 0: getcwd() failed: No such file or directory
+ file /usr/bin/cp
/usr/bin/cp: ELF 64-bit LSB shared object, x86-64, version 1 (SYSV), dynamically linked, interpreter /lib64/ld-linux-x86-64.so.2, BuildID[sha1]=421e1abd8faf1cb290df755a558377c5d7def3b1, for GNU/Linux 3.2.0, stripped
```

---

### 总结

Linux的命令很强大，有时候也确实很难上手，不过最重要的还是要理解其命令背后的本质，知道命令执行模式还有系统的架构模式之后无非就是遇到忘记的命令查看下文档即可。毕竟在系统上操作也是需要谨慎，人云亦云的复制那些千篇一律的都不知道有没有经过验证的命令到服务器上执行也是一种很大的风险。