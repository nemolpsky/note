## HashSet底层原理

```HashSet```是最常用的```Set```集合之一，可以保证元素的唯一性。

---

1. 底层原理

   ```HashSet```底层完全就是在```HashMap```的基础上包了一层，只不过存储的时候```value```是默认存储了一个```Object```的静态常量，取的时候也是只返回```key```，所以看起来就像```List```一样。


2. 构造方法

   可以看到四个构造方法都是初始化一个```HashMap```，初始化的容量和装填因子也是直接用的```HashMap```的默认配置。

   ```
    private transient HashMap<E,Object> map;

    public HashSet() {
        map = new HashMap<>();
    }

    public HashSet(int initialCapacity) {
        map = new HashMap<>(initialCapacity);
    }

    public HashSet(int initialCapacity, float loadFactor) {
        map = new HashMap<>(initialCapacity, loadFactor);
    }

    public HashSet(Collection<? extends E> c) {
        map = new HashMap<>(Math.max((int) (c.size()/.75f) + 1, 16));
        addAll(c);
    }
   ```

3. add()/remove()/contains()方法

   可以看到这三个方法都是直接调用的```HashMap```的实现。

   ```
    public boolean add(E e) {
        return map.put(e, PRESENT)==null;
    }

    public boolean remove(Object o) {
        return map.remove(o)==PRESENT;
    }

    public boolean contains(Object o) {
        return map.containsKey(o);
    }

   ```

4. 保证唯一性

   ```HashSet```是调用的```HashMap```的```put()```方法，而```put()```方法中有这么一行逻辑，如果```哈希值```和```key```都一样，就会直接拿新值覆盖旧值，而```HashSet```就是利用这个特性来保证唯一性。

   ```
    if (p.hash == hash && ((k = p.key) == key || (key != null && key.equals(k))))
        e = p;
   ```

   所以在存放对象的时候需要重写```hashCode()```和```equals()```方法，因为就是用这两个方法来判断唯一性的，否则就会出现下面这样的情况，创建两个属性一样的对象，放入```HashSet```中会发现重复了，那是因为创建两个对象肯定哈希值是不一样的，所以需要自己重写```hashCode()```和```equals()```。

   ```
   public class TestVo {

      int id;
      String name;

      public TestVo(int id, String name) {
          this.id = id;
          this.name = name;
      }

      @Override
      public String toString() {
          return  "TestVo{" +
                  "id=" + id +
                  ", name='" + name + '\'' +
                  '}';
      }
   }

   public static void main(String[] args){
      HashSet set = new HashSet();

      TestVo testVo1 = new TestVo(1,"a");
      TestVo testVo2 = new TestVo(1,"a");

      set.add(testVo1);
      set.add(testVo2);

      Iterator<TestVo> iterator = set.iterator();
      while (iterator.hasNext()){
          System.out.println(iterator.next());
      }
   }

   // 输出结果
   TestVo{id=1, name='a'}
   TestVo{id=1, name='a'}

   ```

   重写```hashCode()```和```equals()```方法，都基于```id```来判断，这样重复``````id的就不会重复存入了，所以存入对象的时候是可以自己设置按什么规则来去重的。

   ```
   @Override
      public int hashCode() {
          return id;
      }

      @Override
      public boolean equals(Object obj) {
          return obj.equals(id);
      }
   }
   // 输出结果
   TestVo{id=1, name='a'}
   ```

---

## 总结
- ```HashSet```底层原理完全就是包装了一下```HashMap```
- ```HashSet```的唯一性保证是依赖与```hashCode()```和```equals()```两个方法，所以存入对象的时候一定要自己重写这两个方法来设置去重的规则。
- 因为底层是```HashMap```，而存储的又是```key```，所以没有```get()```方法来直接获取，只能遍历获取。