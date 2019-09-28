## Exception体系：
所有的异常类都继承自Throwable类，它有两个子类，Error和Exception类。

1. Error

   Error是指程序无法处理的错误，比如OutOfMemoryError、ThreadDeath这些错误。当这些异常发生时，JVM会直接终止运行。

2. Exception

   Exception是指程序本身可以处理的异常，分运行时异常(RuntimeException，也称为非检查型异常)和非运行时异常(除RuntimeException之外所有的Exception，也称为检查型异常)，运行时异常都是自身代码所引起，而非运行时异常则是取决于外部因素，所以需要显式的进行检查。

   - RuntimeException，常见的RuntimeException有下列这些。
     - NullPointerException，空指针引用异常
     - ClassCastException，类型强制转换异常
     - IllegalArgumentException，非法参数异常
     - ArithmeticException，算术运算异常
     - IndexOutOfBoundsException，下标越界异常
     - NumberFormatException，数字格式异常
   - 非RuntimeException，除了RuntimeException之外的
     - IOException，IO流异常
     - ClassNotFoundException，找不到对应类异常

---

## 抛出异常：
代码中如果出现了异常可以选择使用```throws```关键字把这个异常抛给上层调用者，让上层调用者处理这个异常。

- 尽量把所有的可检查性异常抛出来，因为它不是仅仅取决于代码，可能还会受到很多环境因素影响。所以应该尽早的把这种异常抛给调用者，让调用者及时发现问题。
  
  - 
    ```
    class TestException
    {
      ...
      public Image loadlmage (String s) throws IOException
      {
        ...
      }
    }
    ```
- 尽量不要抛出```Error```类下的子类异常，因为这些异常不是代码所产生的问题，所有代码都可能会遇到这种情况，即使抛给使用者也没有任何意义。

- 对于那些非检查型异常也不应该抛出，因为那些都是我们代码产生的问题，我们应当尽量保证代码的健壮性来避免这些错误，还不是抛给使用者让他们来提醒我们。
  
  ```
  class TestException
  {
    ...
    // 数组越界的情况完全可以通过代码逻辑避免这种情况，不应该抛给使用者
    public Image loadlmage (int s) throws ArraylndexOutOfBoundsException
    {
      ...
    }
  }
  ```
---

## 规范的抛出异常：
抛出异常的时候最好返回异常的详细信息，方便定位问题，下面这个方法是在读取文件，而如果读取的长度少于期望的长度，就可以抛出一个错误，可以在```EOFException```里面设置读取的长度信息和文件的长度信息。

```
String readText (File file) throws EOFException
{
  ...
  while (...)
  {
    if (n<len){
      String message = "length:" + len + ",readLength:" + n;
      throw new EOFException();
    }
  }
  ...
}
```