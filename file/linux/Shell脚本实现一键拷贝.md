### Shell实现一键拷贝

Bash Shell的复杂之处在于它的参数展开过于繁琐复杂，而且有些脚本语法有些地方很苛刻，所以写起来很麻烦，但是如果对参数展开了解的够透彻实现起来还是比较容易的，而且脚本也可以从那些复杂繁琐的重复命令里面解脱出来。

详细的可以看[Linux Shell展开解析](https://github.com/nemolpsky/note/blob/master/file/linux/Linux%E6%9D%83%E9%99%90%E4%BD%93%E7%B3%BB.md)。



因为是Win 10安装了WLS(Windows Subsystem for Linux)，平常开发调试都是在Ubuntu上，所以经常会从宿主机上复制文件过去，这个脚本配合alias命令可以实现一个命令复制文件。

注意看下面的例子，其实还是比较简单的，单引号是绝对不展开的字符串，双引号是除了```$```符号不解析，最重要的是```$()```语法，也就是把括号内的命令的执行结果让```echo```输出，使用的反引号也可以实现同样的效果。最后要注意的就是Shell语言对空格要求很严格，尤其是赋值的时候左值跟右值之间不能有空格。


```
#!/bin/bash
# This is a copy shell script.
echo 'start copy.'

# copy from
file_source='/mnt/d/download/host_temp/'*
# copy to
file_target='/home/half/opt/'
# create copy target directory
# target_name=`date +%Y%m%d%H%M%S`
target_name=$(date +%Y%m%d%H%M%S)
echo $(mkdir -v $file_target$target_name)

# ls source directory
files=$(ls /mnt/d/download/host_temp)

# print files which are copied
echo -e "copy file[\n $files \n]."

# execute copy
echo $(cp -v $file_source $file_target$target_name)
```


给脚本设置权限，再添加个别名，直接从绝对路径执行脚本，这样就只需要执行```copyh```就可以一键复制文件到Linux上了。

```
half@Half-PC:~/opt$ chmod 755 copy_from_host
half@Half-PC:~/opt$ alias copyh='/home/half/opt/shell/copy_from_host'
half@Half-PC:~/opt$ alias
alias copyh='/home/half/opt/shell/copy_from_host'
```