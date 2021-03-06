# (四)Java高并发秒杀API之高并发优化
下载Redis
下载完后解压压缩包
进入解压后的文件夹里面 ,执行命令 make
然后再执行sudo make install
最后再启动REdis,启动命令为redis-server
执行命令'redis-cli -p 6379'查看运行情况
使用Java操作Redis
导入操作Redis的jedis的 jar包
 
        <dependency>
            <groupId>redis.clients</groupId>
            <artifactId>jedis</artifactId>
            <version>2.9.0</version>
        </dependency>
        
添加protostuff-core以及protostuff-runtime序列化jar包

        <dependency>
            <groupId>com.dyuproject.protostuff</groupId>
            <artifactId>protostuff-core</artifactId>
            <version>1.1.1</version>
        </dependency>
        <dependency>
            <groupId>com.dyuproject.protostuff</groupId>
            <artifactId>protostuff-runtime</artifactId>
            <version>1.1.1</version>
        </dependency>
        
在com.suny.dao下建包cache
然后建立类RedisDao


/**
* 操作Redis的dao类
*/

/**
 * @author liming
 * @date 2018/5/19   11:56
 */

public class RedisDao {

    private final Logger logger = LoggerFactory.getLogger(this.getClass());
    private final JedisPool jedisPool;
    private RuntimeSchema<Seckill> schema = RuntimeSchema.createFrom(Seckill.class);
    public RedisDao(String ip, int port) {
        jedisPool = new JedisPool(ip, port);
    }
    public Seckill getSeckill(long seckillId) {
        // redis操作业务逻辑
        try {
            Jedis jedis = jedisPool.getResource();
            String key = "seckill:" + seckillId;
            // 并没有实现内部序列化操作
            //get->byte[]字节数组->反序列化>Object(Seckill)
            // 采用自定义的方式序列化
            // 缓存获取到
            byte[] bytes = jedis.get(key.getBytes());
            jedis.close();
            if (bytes != null) {
                // 空对象
                Seckill seckill = schema.newMessage();
                ProtostuffIOUtil.mergeFrom(bytes, seckill, schema);
                // seckill被反序列化
                return seckill;
            }
        } catch (Exception e) {
            logger.error(e.getMessage(), e);
        }
        return null;
    }
    public String putSeckill(Seckill seckill) {
        //  set Object(Seckill) -> 序列化 -> byte[]
        try {
            Jedis jedis = jedisPool.getResource();
            String key = "seckill:" + seckill.getSeckillId();
            byte[] bytes = ProtostuffIOUtil.toByteArray(seckill, schema,
                    LinkedBuffer.allocate(LinkedBuffer.DEFAULT_BUFFER_SIZE));
            // 超时缓存
            int timeout = 60 * 60;
            String str = jedis.setex(key.getBytes(), timeout, bytes);
            jedis.close();
            return str;
        } catch (Exception e) {
            logger.error(e.getMessage(), e);
        }
        return null;
    }
}

下一步是在在applicationContext-dao.xml中注入redisDao

    <!--注入redisDao-->
    <bean id="redisDao" class="com.suny.dao.cache.RedisDao">
        <!--构造方法注入值-->
        <constructor-arg index="0" value="localhost"/>
        <constructor-arg index="1" value="6379"/>
    </bean>
    
改造exportSeckillUrl方法,一定要先注入redisDao

    @Autowired
    private RedisDao redisDao;


    @Override
    public Exposer exportSeckillUrl(long seckillId) {
        // 根据秒杀的ID去查询是否存在这个商品
        /* Seckill seckill = seckillMapper.queryById(seckillId);
        if (seckill == null) {
            logger.warn("查询不到这个秒杀产品的记录");
            return new Exposer(false, seckillId);
        }*/
        Seckill seckill = redisDao.getSeckill(seckillId);
        if (seckill == null) {
            // 访问数据库读取数据
            seckill = seckillMapper.queryById(seckillId);
            if (seckill == null) {
                return new Exposer(false, seckillId);
            } else {
                // 放入redis
                redisDao.putSeckill(seckill);
            }
        }
        // 判断是否还没到秒杀时间或者是过了秒杀时间
        LocalDateTime startTime = seckill.getStartTime();
        LocalDateTime endTime = seckill.getEndTime();
        LocalDateTime nowTime = LocalDateTime.now();
        //   开始时间大于现在的时候说明没有开始秒杀活动    秒杀活动结束时间小于现在的时间说明秒杀已经结束了
        if (nowTime.isAfter(startTime) && nowTime.isBefore(endTime)) {
            //秒杀开启,返回秒杀商品的id,用给接口加密的md5
            String md5 = getMd5(seckillId);
            return new Exposer(true, md5, seckillId);
        }
        return new Exposer(false, seckillId, nowTime, startTime, endTime);


    }
    
写存储过程,然后去Mysql控制台执行储存过程

    CREATE PROCEDURE `seckill`.`execute_seckill`(
      IN v_seckill_id BIGINT,
      IN v_phone BIGINT,
      IN v_kill_time TIMESTAMP,
      OUT r_result INT
    )
    BEGIN
        DECLARE insert_count INT DEFAULT 0;
        START TRANSACTION;
            INSERT IGNORE INTO success_killed (
            seckill_id,
            user_phone,
            create_time
        )
        
        VALUES
        (
          v_seckill_id,
          v_phone,
          v_kill_time
        );
      SELECT
        ROW_COUNT() INTO insert_count;
      IF (insert_count = 0)
      THEN ROLLBACK;
      SET r_result = - 1;
      ELSEIF (insert_count < 0)
      THEN ROLLBACK;
      SET r_result = - 2;
      `success_killed``execute_seckill`
      ELSE
      UPDATE
        seckill
      SET
        number = number - 1
      WHERE seckill_id = v_seckill_id
        AND end_time > v_kill_time
        AND start_time < v_kill_time
        AND number > 0;
      SELECT
        ROW_COUNT() INTO insert_count;
      IF (insert_count = 0)
      THEN ROLLBACK;
      SET r_result = 0;
      ELSEIF (insert_count < 0)
      THEN ROLLBACK;
      SET r_result = - 2;
      ELSE COMMIT;
      SET r_result = 1;
      END IF;
      END IF;
    END;
    $$
--  储存过程定义结束
 DELIMITER;
SET @r_result = -3;
--  执行储存过程
CALL execute_seckill(1003, 13502178891, now(), @r_result);
-- 获取结果
SELECT @r_result;
在SeckillMapper中编写killProduce()方法

    /**
     *  使用储存过程执行秒杀
     * @param paramMap
     */
    void killByProcedure(Map<String,Object> paramMap);
    
然后在SeckillMapper.xml中写sql语句

    <!--调用储存过程-->
    <select id="killByProcedure" statementType="CALLABLE">
        CALL execute_seckill(
                #{seckillId,jdbcType=BIGINT,mode=IN},
                #{phone,jdbcType=BIGINT,mode=IN},
                #{killTime,jdbcType=TIMESTAMP,mode=IN},
                #{result,jdbcType=INTEGER,mode=OUT}
        )
    </select>
    
下一步在SeckillService接口中中编写 killProduce()方法
 SeckillExecution executeSeckillProcedure(long seckillId,long userPhone,String md5);
导入commons-collections工具类

        <!--导入apache工具类-->
        <dependency>
            <groupId>commons-collections</groupId>
            <artifactId>commons-collections</artifactId>
            <version>3.2.2</version>
        </dependency>
        
然后SeckillServiceImpl实现killProduce()方法

    @Override
    public SeckillExecution executeSeckillProcedure(long seckillId, long userPhone, String md5) {
        if (md5 == null || !md5.equals(getMd5(seckillId))) {
            return new SeckillExecution(seckillId, SeckillStatEnum.DATE_REWRITE);
        }
        LocalDateTime killTime = LocalDateTime.now();
        Map<String, Object> map = new HashMap<>();
        map.put("seckillId", seckillId);
        map.put("phone", userPhone);
        map.put("killTime", killTime);
        map.put("result", null);
        // 执行储存过程,result被复制
        try {
            seckillMapper.killByProcedure(map);
            // 获取result
            int result = MapUtils.getInteger(map, "result", -2);
            if (result == 1) {
                SuccessKilled successKilled = successKilledMapper.queryByIdWithSeckill(seckillId, userPhone);
                return new SeckillExecution(seckillId, SeckillStatEnum.SUCCESS, successKilled);
            } else {
                return new SeckillExecution(seckillId, SeckillStatEnum.stateOf(result));
            }
        } catch (Exception e) {
            logger.error(e.getMessage(), e);
            return new SeckillExecution(seckillId, SeckillStatEnum.INNER_ERROR);
        }
    }
    
改造执行秒杀executeSeckill方法,减少一道虚拟机GC程序,优化性能

    @Transactional
    @Override
    public SeckillExecution executeSeckill(long seckillId, long userPhone, String md5) throws SeckillException {
        if (md5 == null || !md5.equals(getMd5(seckillId))) {
            logger.error("秒杀数据被篡改");
            throw new SeckillException("seckill data rewrite");
        }
        // 执行秒杀业务逻辑
        LocalDateTime nowTIme = LocalDateTime.now();
        try {
            // 记录购买行为
            int insertCount = successKilledMapper.insertSuccessKilled(seckillId, userPhone);
            if (insertCount <= 0) {
                // 重复秒杀
                throw new RepeatKillException("seckill repeated");
            } else {
                // 减库存 ,热点商品的竞争
                int reduceNumber = seckillMapper.reduceNumber(seckillId, nowTIme);
                if (reduceNumber <= 0) {
                    logger.warn("没有更新数据库记录,说明秒杀结束");
                    throw new SeckillCloseException("seckill is closed");
                } else {
                    // 秒杀成功了,返回那条插入成功秒杀的信息  进行commit
                    SuccessKilled successKilled = successKilledMapper.queryByIdWithSeckill(seckillId, userPhone);
                    return new SeckillExecution(seckillId, SeckillStatEnum.SUCCESS, successKilled);
                }
            }
        } catch (SeckillCloseException | RepeatKillException e1) {
            throw e1;
        }
    }
    
编写SeckillServiceImpl中的killProduce()方法的测试方法

    @Test
    public void executeSeckillProcedureTest() {
        long seckillId = 1001;
        long phone = 1368011101;
        Exposer exposer = seckillService.exportSeckillUrl(seckillId);
        if (exposer.isExposed()) {
            String md5 = exposer.getMd5();
            SeckillExecution execution = seckillService.executeSeckillProcedure(seckillId, phone, md5);
            System.out.println(execution.getStateInfo());
        }
    }
    
改造SeckillController中的execute方法调用,把一开始调用普通方法的改成调用储存过程的那个方法

     @RequestMapping(value = "/{seckillId}/{md5}/execution", method = RequestMethod.POST)
     @ResponseBody
     public SeckillResult<SeckillExecution> execute(@PathVariable("seckillId") long seckillId,
                                                    @PathVariable("md5") String md5,
                                                    @CookieValue(value = "userPhone", required = false) Long userPhone) {
         // 如果用户的手机号码为空的说明没有填写手机号码进行秒杀
         if (userPhone == null) {
             return new SeckillResult<>(false, "没有注册");
         }
         // 根据用户的手机号码,秒杀商品的id跟md5进行秒杀商品,没异常就是秒杀成功
         try {
            // 这里换成储存过程
            //  SeckillExecution execution = seckillService.executeSeckill(seckillId, userPhone, md5);
             SeckillExecution execution = seckillService.executeSeckillProcedure(seckillId, userPhone, md5);
             return new SeckillResult<>(true, execution);
         } catch (RepeatKillException e1) {
             // 重复秒杀
             SeckillExecution execution = new SeckillExecution(seckillId, SeckillStatEnum.REPEAT_KILL);
             return new SeckillResult<>(false, execution);
         } catch (SeckillCloseException e2) {
             // 秒杀关闭
             SeckillExecution execution = new SeckillExecution(seckillId, SeckillStatEnum.END);
             return new SeckillResult<>(false, execution);
         } catch (SeckillException e) {
             // 不能判断的异常
             SeckillExecution execution = new SeckillExecution(seckillId, SeckillStatEnum.INNER_ERROR);
             return new SeckillResult<>(false, execution);
         }
         // 如果有异常就是秒杀失败
     }

