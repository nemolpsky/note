## ArrayList
- addAll(Collection<? extends E> c)，合并两个集合，并集
- addAll(int index, Collection<? extends E> c)，合并两个集合，在指定位置添加
- clone()，克隆一个新的list
- removeAll(Collection<?> c)，当前集合移除指定集合中的元素，差集
- removeRange(int fromIndex, int toIndex)，移除指定范围的元素
- retainAll(Collection<?> c)，保留两个集合都有的元素，交集
- toArray()，返回一个包含所有集合元素的数组

---

## Collections
- max(Collection<? extends T> coll)，求最大值
- min(Collection<? extends T> coll)，求最小值
- reverse(List<?> list)，翻转顺序
- sort(List<T> list)，自然排序

---

## Arrays
- copyOf()，复制数组的元素返回一个新的数组
- copyOfRange(int[] original, int from, int to)，指定复制的起始和结束位置
- toString()，打印数组
- sort(int[] a)，排序
- sort(int[] a, int fromIndex, int toIndex)，在指定范围内排序
- equals()，比较两个数组是否相等

--- 

## ObjectUtils
- isCheckedException(Throwable ex)，判断是否是可检查异常
- isArray(@Nullable Object obj)，判断是否是数组
- isEmpty(@Nullable Object obj)，判空，支持Optional、String、数组、集合和数组
- unwrapOptional(@Nullable Object obj)，获取Optional中的值
- containsElement(@Nullable Object[] array, Object element)，判断数组中是否包含了指定元素
- addObjectToArray(@Nullable A[] array, @Nullable O obj)，往数组中增加元素
- arrayEquals(Object o1, Object o2)，比较数组是否相等
