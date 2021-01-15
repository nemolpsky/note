### Java IO流

Java IO是Java中用于读取和写入的API，它可能是Java中最重要却不起眼的API了，简单来说一个程序必定会有外界有数据交互，交互也就意味着一定会读写操作。只不过现在所有框架的使用都会隐藏这些底层的读取和写入操作，尤其是网络相关的知识，各种协议其实本质上就是对数据读写格式的一种约定。



---


#### 1. InputStream、OutputStream、Reader、Writer

IO API中，有四个顶级类，InputStream和OutputStream是字节流，读写的时候是以字节为单位，每次读取1字节，也就是8比特，可以读写任何文件，因为所有文件的本质都是由字节组成的。Reader和Writer则是字符流，也就是读取Java中char类型字符单位的数据，因此它也只能读写字符文件。

![1](https://github.com/nemolpsky/Note/raw/master/file/java_base/images/io_inputstream_uml.png)
![2](https://github.com/nemolpsky/Note/raw/master/file/java_base/images/io_outputstream_uml.png)
![3](https://github.com/nemolpsky/Note/raw/master/file/java_base/images/io_reader_uml.png)
![4](https://github.com/nemolpsky/Note/raw/master/file/java_base/images/io_writer_uml.png)


---

#### 2. 用途分类

除了按照上面最基础的读、写和字节、字符分类，还可以按照另一种方式来分类，节点流和处理流，节点流就是直接对数据进行读写操作，而处理流的内部则可以直接对其他的流进行处理，通常处理流都提供缓存池的功能，效率会更高。

![5](https://github.com/nemolpsky/Note/raw/master/file/java_base/images/io_mind_object.png)
![6](https://github.com/nemolpsky/Note/raw/master/file/java_base/images/io_mind_optype.png)

---


#### 3. 例子

最基本的使用文件字节流读取和写入。

```
    public static void copy1() throws Exception {
        // 创建个新文件
        File file = new File("D:\\download\\1.jpeg");
        try (
                // 打开一个输出流，写入到新文件
                FileOutputStream fileOutputStream = new FileOutputStream("D:\\download\\1_copy.jpeg");
                // 打开一个输入流，读取原始文件
                FileInputStream fileInputStream = new FileInputStream(file)
        ) {
            // 定义一个int变量来接收读取到的每个字节
            // 1个字节8比特，而比特是二进制格式，转换成十进制范围其实就是0-255
            // 这里循环读取，如果返回-1表示已经读取完了
            int c;
            while ((c = fileInputStream.read()) != -1) {
                // 打印0-255范围内的数字，结束打印1个-1
                System.out.println(c);
                // 读取到的字节通过输出流写入到新的文件中
                fileOutputStream.write(c);
            }
            System.out.println(c);
            // 整个过程执行完毕，就会有一张新的图片出来
        } catch (Exception e) {
            e.printStackTrace();
        }
    }
```

使用缓存字节流进行读写操作，使用比特数组作为缓存池，减少读写次数，效率更高。

```
    public static void copy2() throws Exception {
        // 创建个新文件
        File file = new File("D:\\download\\1.jpeg");
        try (
                // 打开一个缓存输出流，写入到新文件
                BufferedOutputStream bufferedOutputStream = new BufferedOutputStream(new FileOutputStream("D:\\download\\1_copy.jpeg"));
                // 打开一个缓存输入流，读取原始文件
                BufferedInputStream bufferedInputStream = new BufferedInputStream(new FileInputStream(file));
        ) {
            // 定义一个比特数组，每次读取1024字节，再写入
            byte[] bytes = new byte[1024];
            int c;
            while ((c = bufferedInputStream.read(bytes, 0, bytes.length)) != -1) {
                System.out.println(c);
                // 读取到的字节通过输出流写入到新的文件中
                bufferedOutputStream.write(bytes, 0, bytes.length);
            }
            System.out.println(c);
            // 整个过程执行完毕，就会有一张新的图片出来
        } catch (Exception e) {
            e.printStackTrace();
        }
    }
```

使用缓存字符流进行读写操作。每次读取一行，只适用于文本。

```
    public static void copy3() throws Exception {
        // 创建个新文件
        File file = new File("D:\\download\\test.txt");
        try (
                // 打开一个缓存输出流，写入到新文件
                BufferedWriter bufferedWriter = new BufferedWriter(new FileWriter("D:\\download\\copy_test.txt"));
                // 打开一个缓存输入流，读取原始文件
                BufferedReader bufferedReader = new BufferedReader(new FileReader(file));
        ) {
            // 定义一个字符串，每次读取一行
            String line;
            while ((line = bufferedReader.readLine()) != null) {
                System.out.println(line);
                // 读取到的字符串通过输出流写入到新的文件中
                bufferedWriter.write(line);
            }
            // 整个过程执行完毕，就会有一个新的文本
        } catch (Exception e) {
            e.printStackTrace();
        }
    }
```

使用压缩包读取流，读取压缩包中的内容，并且进行抽取。

```
    public static void copy4() throws Exception {
        // 创建解压路径
        String path = "D:/download/out";
        File directory = new File(path);
        directory.mkdir();
        // 压缩包路径
        String zipFileName = "D:/download/apache-jmeter-5.3.zip";
        try (// 创建压缩包读取流
             ZipInputStream zipInputStream = new ZipInputStream(new BufferedInputStream(new FileInputStream(zipFileName)))
        ) {
            // 读取的数据的数组
            byte[] buffer = new byte[2048];
            // 循环读取压缩包的条目内容
            ZipEntry entry;
            while ((entry = zipInputStream.getNextEntry()) != null) {
                // 拼接解压路径
                String innerPath = path + "/" + entry.getName();
                System.out.println("extract file:" + innerPath);

                // 创建文件夹和文件
                File file = new File(innerPath);
                if (!file.exists()) {
                    if (entry.isDirectory()) {
                        file.mkdir();
                    } else {
                        file.createNewFile();
                    }
                }

                // 只有文件才需要读取写入执行拷贝操作，文件夹忽略
                if (file.isFile()) {
                    try (// 使用缓存字节流读取每个条目
                         BufferedOutputStream bufferedOutputStream = new BufferedOutputStream(new FileOutputStream(file), buffer.length)) {
                        int len;
                        while ((len = zipInputStream.read(buffer)) > 0) {
                            bufferedOutputStream.write(buffer, 0, len);
                        }
                    }
                }
            }
        }
    }

```