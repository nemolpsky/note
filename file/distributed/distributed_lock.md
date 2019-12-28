## 分布式锁
在单台服务器的情况下解决并发问题可以直接使用```Java```自带的锁，比如```synchronized```、```Lock```等锁，或者是使用一些原子类，这些自带的同步机制都可以很好的解决单机下并发问题。但是在多台服务器上分布式的情况下，这些自带的同步机制就没办法解决并发问题了。

1. 数据库索引

   可以利用数据库自带的事务隔离性来当做锁用，创建一张```lock```表，在需要加锁的方法里面，每次执行业务逻辑代码前都先插入一条数据，判断如果插入成功就表示获得锁了，因为```method_name```字段是加了唯一索引，所以后面过来的方法都会插入失败，也就不会执行下面的逻辑，获得锁的请求执行完之后再把这条数据删除掉，后面的请求又可以继续来竞争这个锁了。

   ```
   CREATE TABLE `lock` (
     `id` int(11) NOT NULL AUTO_INCREMENT COMMENT '主键id',
     `method_name` varchar(64) COLLATE utf8mb4_bin DEFAULT NULL COMMENT '方法名',
     `create_time` timestamp NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP COMMENT '创建时间',
     PRIMARY KEY (`id`),
     UNIQUE KEY `method_name` (`method_name`)
   ) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_bin
   ```

   ```
   insert into lock(method_name) values ('method_name');

   delete from lock where method_name ='method_name';
   ```

   - 遇到的问题

     - 因为是完全依赖数据库的，所以如果数据库是单机配置的话，数据库挂了锁也就不可用了，不过单机数据库如果挂了，整个系统都挂了，也就无所谓锁能不能用了。
       
       配置多个数据库就可以解决这种问题了。
     - 没有配置失效时间，如果解锁的时候出问题，就永远都锁住了。
     
       创建一个定时任务，定期检查是否有锁长期被锁住。
     - 非阻塞的，后面来获取锁失败的请求都直接失败了，不会等待。
     
       使用合理的```while```循环来等待，一直重试获取锁的操作。
     - 非重入的，同一个线程在没有释放锁之前无法再次获得该锁，因为数据中数据已经存在了。

       增加字段记录请求的```ip```和线程等信息，如果判断是相同的就直接通过，相当于实现了重入。
     - 非公平锁，所有等待锁的线程凭运气去争夺锁。

       再创建张表记录等待的锁的时间，根据时间的先后去获取锁。


---

2. 数据库排他锁

   也可以换一种方式，利用```for update```锁住查询的那行数据的方式来作为锁，相比于上面那种插入的方式，因为利用了数据库排他锁的性质所以后面的请求获取不到锁都会默认阻塞在那里，而且即使是服务器宕机之后这个锁也会自动释放掉，但是只适用于单机服务器，而切没办法解决重入锁和公平锁的问题，局限性比较大。

   ```
   // 加锁方法
   public boolean lock() throws Exception{  
       // 关闭自动提交事务
       connection.setAutoCommit(false);
       try{  
           // for update会锁住该行数据，所以最开始进来的可以成功查到数据，直接返回true表示成功回去锁
           // 如果查不到则表示已经有其他的请求获取到锁了，进行等待重试
           // select * from methodLock where method_name=xxx for update;          
          }catch(Exception e){
           log.error("lock-error",e);
          }
       }
       return false;
   }

   // 释放锁方法
   public void unlock(){
       // 提交事务
       connection.commit(); 
   }
   ```


---

3. 基于Redis

   - SETNX

     基于```Redis```实现分布式锁的原理主要是使用```SETNX```这条命令，这条命令是指给一个```key```设置一个```value```，而如果这个```key```不存在则会设置成功，存在则会设置失败。而分布式锁的原理则是每次请求进来都设置一个使用```SETNX```命令设置一个```key```，```value```则是时间戳，如果成功就表示抢到锁了，没成功则表示没抢到锁，需要等待，而抢到锁的线程则在执行完之后删除这个```key```，还可以设置超时时间，避免锁一直被锁住。因为```Redis```是单线程的，所以这样实现分布式锁是可以保证并发问题的。

     ```
     redis> EXISTS test  // 为test的key不存在
     (integer) 0
     redis> SETNX test "aaa"  // test设置成功
     (integer) 1
     redis> SETNX test "bbb" // 为test的key已经存在，值不会变
     (integer) 0
     redis> GET test  // test的值还是原来的
     "aaa"
     ```

   - 并发问题

     虽然```SETNX```这个命令是不会有并发问题的，但是删除```key```的操作却可能导致并发问题，比如下列的步骤，就会导致一个线程把另外的线程的锁给释放掉，造成并发问题。

     - A获取到锁，A宕机，A的锁超时
     - B和C读取A的锁发现超时了
     - B抢先把A的超时的锁删掉了，执行删除命令删除```key```，又成功执行了```SETNX```命令，B获得了锁
     - C因为也检测到A的锁超时了，在上面B执行了一系列操作之后恰好也执行了删除命令，这样就把B的锁删除掉了，然后执行```SETNX```命令，C也获得了锁

     上述的并发问题主要是锁超时之后可能会同时被多个线程发现，然后执行删除操作，最后进来的线程会把前面的线程的锁释放掉，所以在执行删除操作前，又加了一层判断，```GETSET```是给对应的key设置值然后返回原来的旧值，所以拿这次获取到的旧值对比刚才检测A超时时获得到的超时时间对比，如果两个时间一样表示，中间没有别的线程操作，如果时间不一样，比如B已经做了操作，那B的超时时间自然和A的不一样，就判定无法获得锁，所以这样就保证了不会有并发问题。
      
     - A获取到锁，A宕机，A的锁超时
     - B和C读取A的锁发现超时了
     - B执行```GETSET```命令，对比获取到的时间戳是否是和A的超时时间一样，如果一样才执行删除命令，再执行```SETNX```命令B获得锁
     - C因为也检测到A的锁超时了，在上面B执行了一系列操作之后执行```GETSET```命令，此时对比时间发现返回的时间戳和A的超时时间不一样，不会获得锁

   - 流程图
     
     ![distributed_lock](https://github.com/nemolpsky/note/raw/master/file/distributed/images/distributed_lock.jpg)

   - 代码实现

     这是在网上看到的一个实现的很好的基于```Redis```的分布锁，还配合切面提供了注解配置。
     
     ```
     public class RedisLock {

        private static final Logger logger = LoggerFactory.getLogger(RedisLock.class);

        //key的过期时间，一天
        private static final int finalDefaultTTLwithKey = 24 * 3600;

        //锁默认超时时间，20秒
        private static final long defaultExpireTime = 20 * 1000;

        private static final boolean Success = true;

        @Resource( name = "redisTemplate")
        private RedisTemplate<String, String> redisTemplateForGeneralize;

        /**
         * 加锁,锁默认超时时间20秒
         * @param resource
         * @return
         */
        public boolean lock(String resource) {
            return this.lock(resource, defaultExpireTime);
        }

        /**
         * 加锁,同时设置锁超时时间
         * @param key 分布式锁的key
         * @param expireTime 单位是ms
         * @return
         */
        public boolean lock(String key, long expireTime) {

            logger.debug("redis lock debug, start. key:[{}], expireTime:[{}]",key,expireTime);
            long now = Instant.now().toEpochMilli();
            long lockExpireTime = now + expireTime;

            //setnx
            boolean executeResult = redisTemplateForGeneralize.opsForValue().setIfAbsent(key,String.valueOf(lockExpireTime));
            logger.debug("redis lock debug, setnx. key:[{}], expireTime:[{}], executeResult:[{}]", key, expireTime,executeResult);

            //取锁成功,为key设置expire
            if (executeResult == Success) {
                redisTemplateForGeneralize.expire(key,finalDefaultTTLwithKey, TimeUnit.SECONDS);
                return true;
            }
            //没有取到锁,继续流程
            else{
                // 使用自定义的重试方法来获取锁
                Object valueFromRedis = this.getKeyWithRetry(key, 3);
                // 避免获取锁失败,同时对方释放锁后,造成NPE
                if (valueFromRedis != null) {
                    //已存在的锁超时时间
                    long oldExpireTime = Long.parseLong((String)valueFromRedis);
                    logger.debug("redis lock debug, key already seted. key:[{}], oldExpireTime:[{}]",key,oldExpireTime);
                    //锁过期时间小于当前时间,锁已经超时,重新取锁
                    if (oldExpireTime <= now) {
                        logger.debug("redis lock debug, lock time expired. key:[{}], oldExpireTime:[{}], now:[{}]", key, oldExpireTime, now);
                        String valueFromRedis2 = redisTemplateForGeneralize.opsForValue().getAndSet(key, String.valueOf(lockExpireTime));
                        long currentExpireTime = Long.parseLong(valueFromRedis2);
                        //判断currentExpireTime与oldExpireTime是否相等
                        if(currentExpireTime == oldExpireTime){
                            //相等,则取锁成功
                            logger.debug("redis lock debug, getSet. key:[{}], currentExpireTime:[{}], oldExpireTime:[{}], lockExpireTime:[{}]", key, currentExpireTime, oldExpireTime, lockExpireTime);
                            redisTemplateForGeneralize.expire(key, finalDefaultTTLwithKey, TimeUnit.SECONDS);
                            return true;
                        }else{
                            //不相等,取锁失败
                            return false;
                        }
                    }
                }
                else {
                    logger.warn("redis lock,lock have been release. key:[{}]", key);
                    return false;
                }
            }
            return false;
        }

        /**
         *
         * @param key
         * @param retryTimes 重试次数
         * @return
         */
        private Object getKeyWithRetry(String key, int retryTimes) {
            int failTime = 0;
            while (failTime < retryTimes) {
                try {
                    return redisTemplateForGeneralize.opsForValue().get(key);
                } catch (Exception e) {
                    failTime++;
                    if (failTime >= retryTimes) {
                        throw e;
                    }
                }
            }
            return null;
        }

        /**
         * 解锁
         * @param key
         * @return
         */
        public boolean unlock(String key) {
            logger.debug("redis unlock debug, start. resource:[{}].",key);
            redisTemplateForGeneralize.delete(key);
            return Success;
        }
     }
     ```

     ```
     @Retention(RetentionPolicy.RUNTIME)
     @Target(ElementType.METHOD)
     public @interface RedisLockAnnoation {

     String keyPrefix() default "";

     /**
      * 要锁定的key中包含的属性
      */
     String[] keys() default {};

     /**
      * 是否阻塞锁；
      * 1. true：获取不到锁，阻塞一定时间；
      * 2. false：获取不到锁，立即返回
      */
     boolean isSpin() default true;

     /**
      * 超时时间
      */
     int expireTime() default 10000;

     /**
      * 等待时间
      */
     int waitTime() default 50;

     /**
      * 获取不到锁的等待时间
      */
     int retryTimes() default 20;
     }
     ```

     ```
     @Component
     @Aspect
     public class RedisLockAdvice {

         private static final Logger logger = LoggerFactory.getLogger(RedisLockAdvice.class);

         @Resource
         private RedisDistributionLock redisDistributionLock;

         @Around("@annotation(RedisLockAnnoation)")
         public Object processAround(ProceedingJoinPoint pjp) throws Throwable {
             //获取方法上的注解对象
             String methodName = pjp.getSignature().getName();
             Class<?> classTarget = pjp.getTarget().getClass();
             Class<?>[] par = ((MethodSignature) pjp.getSignature()).getParameterTypes();
             Method objMethod = classTarget.getMethod(methodName, par);
             RedisLockAnnoation redisLockAnnoation = objMethod.getDeclaredAnnotation(RedisLockAnnoation.class);

             //拼装分布式锁的key
             String[] keys = redisLockAnnoation.keys();
             Object[] args = pjp.getArgs();
             Object arg = args[0];
             StringBuilder temp = new StringBuilder();
             temp.append(redisLockAnnoation.keyPrefix());
             for (String key : keys) {
                 String getMethod = "get" + StringUtils.capitalize(key);
                 temp.append(MethodUtils.invokeExactMethod(arg, getMethod)).append("_");
             }
             String redisKey = StringUtils.removeEnd(temp.toString(), "_");

             //执行分布式锁的逻辑
             if (redisLockAnnoation.isSpin()) {
                 //阻塞锁
                 int lockRetryTime = 0;
                 try {
                     while (!redisDistributionLock.lock(redisKey, redisLockAnnoation.expireTime())) {
                         if (lockRetryTime++ > redisLockAnnoation.retryTimes()) {
                             logger.error("lock exception. key:{}, lockRetryTime:{}", redisKey, lockRetryTime);
                             throw ExceptionUtil.geneException(CommonExceptionEnum.SYSTEM_ERROR);
                         }
                         ThreadUtil.holdXms(redisLockAnnoation.waitTime());
                     }
                     return pjp.proceed();
                 } finally {
                     redisDistributionLock.unlock(redisKey);
                 }
             } else {
                 //非阻塞锁
                 try {
                     if (!redisDistributionLock.lock(redisKey)) {
                         logger.error("lock exception. key:{}", redisKey);
                         throw ExceptionUtil.geneException(CommonExceptionEnum.SYSTEM_ERROR);
                     }
                     return pjp.proceed();
                 } finally {
                     redisDistributionLock.unlock(redisKey);
                 }
             }
         }
     }
     ```
   - Redislock算法

     因为使用```Redis```集群的话，有可能造成A线程在A节点上```SETNX```值成功，但是还没有同步到B节点，而刚好B线程进入到B节点执行```SETNX```，这个时候就会A和B同时拿到锁，所以官方提供了```Redislock```算法，可以保证集群下并发安全，下列就是这个算法的步骤。

     - 得到当前的时间，单位为毫秒
     - 尝试顺序地在每个实例上申请锁，需要使用相同的```key```和```value```，需要合理设置```client```与```master```节点沟通的超时时间，避免时间过长浪费时间。
     - 当```client```在大于等于3个```master```上成功申请到锁的时候，它会计算申请锁消耗了多少时间，这部分消耗的时间采用获得锁的当下时间减去第一步获得的时间戳得到，如果锁的```持续时长(lock validity time)```比流逝的时间多的话，那么就获取到锁了。
     - 锁真正的```持续时长(lock validity time)```应该是```持续时长(lock validity time)``` - ```申请锁期间流逝的时间```
     - 如果```client```申请锁失败，那么就会在少部分申请成功锁的```master```节点上执行释放锁的操作，重置状态

---

4. 基于Zookeeper

   基于```Zookeeper```的特性，使用临时有序节点是完全可以避免并发问题的，而且因为是创建的临时节点，客户端宕机之后也不会让锁永远锁住。而且在删除节点的时候也不会有不当操作导致的并发问题，此外还有例如```curator```这样的开源项目，专门封装出了```Zookeeper```分布式锁实现，简单易用。

   - 创建一个临时的有序节点
   - 对比整个有序节点的列表，如果自己创建的有序节点是最小的就表示可以拿到锁
   - 执行完业务逻辑后删除有序节点


---
  
## 总结

1. 数据库

   简单易操作，但是考虑的细节可能会比较多，而且可能会对数据库性能造成影响。

2. Redis

   比较流行的方式，相比其他的实现方式，需要自己实现锁，集群下实现更加复杂，但是性能较好。

3. Zookeeper

   简单易懂，有现成的封装框架，但是需要了解```Zookeeper```整体的原理，才能对锁的实现和原理有一个比较清晰的理解。