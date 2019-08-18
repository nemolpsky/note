# 第三章
---

## ThreadLocalRandom类原理
Random类在高并发情况下会有很大的性能消耗，所以新增了ThreadLocalRandom类来代替它。

## Random类
- Random使用
  ```
  public static void main(String[] args) {
      // 初始化对象
	  Random random = new Random();
      // 随机输出0-99的随机值
	  System.out.println(random.nextInt(100));
  }
  ```
- 源码分析
  可以看到```Random```类中```nextInt()```方法去生成随机数，调用了```next()```方法，这里面会使用旧的种子来生成新的种子，种子变量```seed```是由```final```修饰的，所以在多线程的情况下都是共用一个种子变量，所以有可能线程A拿到旧种子还没生成新种子，线程B也拿到相同的旧种子，这个时候A和B生成的新种子就是一样的了。所以```Random```类使用了原子类```AtomicLong```中的```compareAndSet()```方法来生成新的种子，这个方法是CAS操作，所以只会有一个线程成功，但是这样虽然解决了问题却会造成性能消耗，当很多个线程并发执行的时候只有一个线程可以成功修改数据，而其他的线程都会进行自旋重试。
  ```
  public int nextInt(int bound) {
      if (bound <= 0)
          throw new IllegalArgumentException(BadBound);

      // 1.获取新的种子来生成随机数 
      int r = next(31);
      int m = bound - 1;
      if ((bound & m) == 0)  // i.e., bound is a power of 2
          r = (int)((bound * (long)r) >> 31);
      else {
          for (int u = r;
               u - (r = u % bound) + m < 0;
               u = next(31))
              ;
      }
      return r;
  }

  protected int next(int bits) {
    long oldseed, nextseed;

    //2.获取旧的种子
    AtomicLong seed = this.seed;
    do {
        oldseed = seed.get();
        nextseed = (oldseed * multiplier + addend) & mask;
      //3.使用cas更新，也就是乐观锁
    } while (!seed.compareAndSet(oldseed, nextseed));
    return (int)(nextseed >>> (48 - bits));
  }

  //种子变量值是原子类，final修饰
  private final AtomicLong seed;
  ```
## ThreadLocalRandom类
- ThreadLocalRandom使用
  ```
  public static void main(String[] args) {
	  ThreadLocalRandom random = ThreadLocalRandom.current();
	  System.out.println(random.nextInt(100));
  }
  ```
- 源码解析
  - 可以看到```ThreadLocalRandom```类初始化并不是```new```出来的，而是调用了静态方法```current()```判断如果获取值为0则进行初始化调用了```localInit()```方法，这个方法里则是初始化一个随机种子，也就是说所有的线程其实都是获取同一个```ThreadLocalRandom```实例。
  ```
  public static ThreadLocalRandom current() {
      // 1.判断是否需要初始化
      if (UNSAFE.getInt(Thread.currentThread(), PROBE) == 0)
          localInit();
      return instance;
  }

  static final void localInit() {
      // 2.初始化一个种子
      int p = probeGenerator.addAndGet(PROBE_INCREMENT);
      int probe = (p == 0) ? 1 : p; // skip 0
      long seed = mix64(seeder.getAndAdd(SEEDER_INCREMENT));
      Thread t = Thread.currentThread();
      UNSAFE.putLong(t, SEED, seed);
      UNSAFE.putInt(t, PROBE, probe);
  }
  ```
  - 而如果调用```nextInt()```方法则会调用```nextSeed()```方法来生成新种子，可以看到```nextSeed()```里使用了```UNSAF```类，可以理解为有和```ThreadLocal```类一样的作用，将每个线程的变量单独存放，所以多线程下每个子线程都是拿自己的旧种子来生成新种子，不存在并发问题，也不存在性能消耗。
  ```
  public int nextInt(int bound) {
    if (bound <= 0)
      throw new IllegalArgumentException(BadBound);
      //　拿旧种子生成新种子
      int r = mix32(nextSeed());
      int m = bound - 1;
      if ((bound & m) == 0) // power of two
          r &= m;
      else { // reject over-represented candidates
          for (int u = r >>> 1;
               u + m - (r = u % bound) < 0;
               u = mix32(nextSeed()) >>> 1)
              ;
      }
      return r;
  }

  final long nextSeed() {
      Thread t; long r; // read and update per-thread seed
      // 获取当前线程的旧种子生成新种子
      UNSAFE.putLong(t = Thread.currentThread(), SEED,
                     r = UNSAFE.getLong(t, SEED) + GAMMA);
      return r;
  }
  ```
  