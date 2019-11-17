## 字符串拼接

1. 拼接String

   反编译之后会发现是会创建一个StringBuilder来追加字符，这里使用的是jad，如果使用其他的反编译出来的是String.valueOf(text1) + text2。

   ```
   	public static String add(String text1, String text2) {
		return text1 + text2;
	}
   ```

   ```
    public static String add(String text1, String text2)
    {
        return (new StringBuilder(String.valueOf(text1))).append(text2).toString();
    }
   ```

2. concat()方法

   底层原理是把新建一个更大的数组把原来的字符串copy进去

   ```
	public static String concat(String text1, String text2) {
		return text1.concat(text2);
	}
   ```

   ```
    public String concat(String str) {
        int otherLen = str.length();
        if (otherLen == 0) {
            return this;
        }
        int len = value.length;
        char buf[] = Arrays.copyOf(value, len + otherLen);
        str.getChars(buf, len);
        return new String(buf, true);
    }
   ```

3. StringBuilder

   内部也是一个字符数组，将追加的字符放入数组中，如果数组长度不够的话还会扩大数组，然后copy过去。

   ```
	public static String builder(String text1, String text2) {
		StringBuilder builder = new StringBuilder();
		return builder.append(text1).append(text2).toString();
	}
   ```

   ```
    @Override
    public StringBuilder append(String str) {
        super.append(str);
        return this;
    }

    public AbstractStringBuilder append(String str) {
        if (str == null)
            return appendNull();
        int len = str.length();
        ensureCapacityInternal(count + len);
        str.getChars(0, len, value, count);
        count += len;
        return this;
    }   
   ```

4. StringBuffer

   也是和StringBuilder调用的同一个方法，只不过加了synchronized关键字保证线程安全。

   ```
	public static String buffer(String text1, String text2) {
		StringBuffer buffer = new StringBuffer();
		return buffer.append(text1).append(text2).toString();
	}
   ```

   ```
    @Override
    public synchronized StringBuffer append(String str) {
        toStringCache = null;
        super.append(str);
        return this;
    }

   ```

---


## 效率对比

1. 测试代码

   ```
	public static void main(String[] args) {

		String text1 = "abc";
		String text2 = "abc";
		String.valueOf(text1);

		Long start = System.currentTimeMillis();
		for (int i = 0; i < 10000000; i++) {
			add(text1, text2);
		}
		Long end = System.currentTimeMillis();
		calTime(start, end, "add");

		Long start1 = System.currentTimeMillis();
		for (int i = 0; i < 10000000; i++) {
			concat(text1, text2);
		}
		Long end1 = System.currentTimeMillis();
		calTime(start1, end1, "concat");

		Long start2 = System.currentTimeMillis();
		for (int i = 0; i < 10000000; i++) {
			builder(text1, text2);
		}
		Long end2 = System.currentTimeMillis();
		calTime(start2, end2, "builder");

		Long start3 = System.currentTimeMillis();
		for (int i = 0; i < 10000000; i++) {
			buffer(text1, text2);
		}
		Long end3 = System.currentTimeMillis();
		calTime(start3, end3, "buffer");

	}

	public static String add(String text1, String text2) {
		return text1 + text2;
	}

	public static String concat(String text1, String text2) {
		return text1.concat(text2);
	}

	public static String builder(String text1, String text2) {
		StringBuilder builder = new StringBuilder();
		return builder.append(text1).append(text2).toString();
	}

	public static String buffer(String text1, String text2) {
		StringBuffer buffer = new StringBuffer();
		return buffer.append(text1).append(text2).toString();
	}

	public static void calTime(Long start, Long end, String method) {
		System.out.println(method + "-" + (end - start));
	}

   ```

2. 结果

   ```
   add-448
   concat-310
   builder-117
   buffer-121
   ```

3. 结论

   可以看到大致的结果是```StringBuilder < StringBuffer < concat < String```，直接拼接的话，每次循环都要新建一个StringBuilder对象，concat则是每次对copy数组，而StringBuilder和StringBuffer因为是单线程情况下测试，差别不大，所以单线程情况下尽量使用StringBuilder，多线程情况下则使用StringBuffer来保证线程安全问题。