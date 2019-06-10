/*
 * Copyright (C) 2018, All rights Reserved, Designed By www.xiniaoyun.com
 * @author: 孙昊
 * @date: 2018-10-15 13:33
 * @Copyright: 2018 www.xiniaoyun.com Inc. All rights reserved.
 */
package com.oppo.csc.hrms.common.util;

import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.stereotype.Component;

/**
 * 改造自Twitter主键生成器snowflake.<p/>
 * Twitter_Snowflake<br>
 * SnowFlake的结构如下(每部分用-分开):<br>
 * <code>0 - 0000000000 0000000000 0000000000 0000000000 0 - 00000 - 00000 - 000000000000</code>
 * <ol>
 * <li>1位标识，由于long基本类型在Java中是带符号的，最高位是符号位，正数是0，负数是1，所以id一般是正数，最高位是0</li>
 * <li>41位时间截(毫秒级)，注意，41位时间截不是存储当前时间的时间截，而是存储时间截的差值（当前时间截 - 开始时间截)
 * 得到的值），这里的的开始时间截，一般是我们的id生成器开始使用的时间，由我们程序来指定的（如下下面程序IdWorker类的startTime属性）。41位的时间截，可以使用69年，年T = (1L << 41) / (1000L * 60 * 60 * 24 * 365) = 69</li>
 * <li>10位的数据机器位，可以部署在1024个节点，包括5位datacenterId和5位workerId</li>
 * <li>12位序列，毫秒内的计数，12位的计数顺序号支持每个节点每毫秒(同一机器，同一时间截)产生4096个ID序号</li>
 * </ol>
 * 加起来刚好64位，为一个Long型。<br>
 * SnowFlake的优点是，整体上按照时间自增排序，并且整个分布式系统内不会产生ID碰撞(由数据中心ID和机器ID作区分)，并且效率较高，经测试，SnowFlake每秒能够产生26万ID左右
 *
 * @author <a href="mailto:sunhao.java@gmail.com">sunhao(sunhao.java@gmail.com)</a>
 */
@Component
public class Snowflake {
    /**
     * 开始时间截 (2015-01-01)
     */
    private static final long BEGIN_TIME_MILLIS = 1420041600000L;

    /**
     * 机器id所占的位数
     */
    private static final long WORKER_ID_BITS = 5L;

    /**
     * 数据标识id所占的位数
     */
    private static final long DATA_CENTER_ID_BITS = 5L;

    /**
     * 支持的最大机器id，结果是31 (这个移位算法可以很快的计算出几位二进制数所能表示的最大十进制数)
     */
    private static final long MAX_WORKER_ID = ~(-1L << WORKER_ID_BITS);

    /**
     * 支持的最大数据标识id，结果是31
     */
    private static final long MAX_DATA_CENTER_ID = ~(-1L << DATA_CENTER_ID_BITS);

    /**
     * 序列在id中占的位数
     */
    private static final long SEQUENCE_BITS = 12L;

    /**
     * 机器ID向左移12位
     */
    private static final long WORKER_ID_SHIFT = SEQUENCE_BITS;

    /**
     * 数据标识id向左移17位(12+5)
     */
    private static final long DATA_CENTER_ID_SHIFT = SEQUENCE_BITS + WORKER_ID_BITS;

    /**
     * 时间截向左移22位(5+5+12)
     */
    private static final long TIMESTAMP_LEFT_SHIFT = SEQUENCE_BITS + WORKER_ID_BITS + DATA_CENTER_ID_BITS;

    /**
     * 生成序列的掩码，这里为4095 (0b111111111111=0xfff=4095)
     */
    private static final long SEQUENCE_MASK = ~(-1L << SEQUENCE_BITS);

    /**
     * 工作机器ID(0~31)
     */
    private static long workerId;

    /**
     * 数据中心ID(0~31)
     */
    private static long dataCenterId;

    /**
     * 毫秒内序列(0~4095)
     */
    private static long sequence = 0L;

    /**
     * 上次生成ID的时间截
     */
    private static long lastTimestamp = -1L;

    /**
     * 构造函数
     *
     * @param workerId     工作ID (0~31)
     * @param dataCenterId 数据中心ID (0~31)
     */
    @Autowired
    public Snowflake(@Value("${snowflake.worker-id:1}") long workerId, @Value("${snowflake.data-center-id:1}") long dataCenterId) {
        if (workerId > MAX_WORKER_ID || workerId < 0) {
            throw new IllegalArgumentException(String.format("worker Id can't be greater than %d or less than 0", MAX_WORKER_ID));
        }
        if (dataCenterId > MAX_DATA_CENTER_ID || dataCenterId < 0) {
            throw new IllegalArgumentException(String.format("dataCenter Id can't be greater than %d or less than 0", MAX_DATA_CENTER_ID));
        }
        Snowflake.workerId = workerId;
        Snowflake.dataCenterId = dataCenterId;
    }

    /**
     * 获得下一个ID (该方法是线程安全的)
     *
     * @return SnowflakeId
     */
    public static synchronized long nextId() {
        long timestamp = timeGen();

        //如果当前时间小于上一次ID生成的时间戳，说明系统时钟回退过这个时候应当抛出异常
        if (timestamp < lastTimestamp) {
            throw new RuntimeException(
                    String.format("Clock moved backwards.  Refusing to generate id for %d milliseconds", lastTimestamp - timestamp));
        }

        //如果是同一时间生成的，则进行毫秒内序列
        if (lastTimestamp == timestamp) {
            sequence = (sequence + 1) & SEQUENCE_MASK;
            //毫秒内序列溢出
            if (sequence == 0) {
                //阻塞到下一个毫秒,获得新的时间戳
                timestamp = tilNextMillis(lastTimestamp);
            }
        }
        //时间戳改变，毫秒内序列重置
        else {
            sequence = 0L;
        }

        //上次生成ID的时间截
        lastTimestamp = timestamp;

        //移位并通过或运算拼到一起组成64位的ID
        return ((timestamp - BEGIN_TIME_MILLIS) << TIMESTAMP_LEFT_SHIFT)
                | (dataCenterId << DATA_CENTER_ID_SHIFT)
                | (workerId << WORKER_ID_SHIFT)
                | sequence;
    }

    /**
     * 阻塞到下一个毫秒，直到获得新的时间戳
     *
     * @param lastTimestamp 上次生成ID的时间截
     * @return 当前时间戳
     */
    private static long tilNextMillis(long lastTimestamp) {
        long timestamp = timeGen();
        while (timestamp <= lastTimestamp) {
            timestamp = timeGen();
        }
        return timestamp;
    }

    /**
     * 返回以毫秒为单位的当前时间
     *
     * @return 当前时间(毫秒)
     */
    private static long timeGen() {
        return System.currentTimeMillis();
    }
}


/*
 * Copyright (C) 2018, All rights Reserved
 * @author: 杨宁
 * @Copyright: 2019 All rights reserved.
 * 注意：本内容仅限于南京微欧科技有限公司内部传阅，禁止外泄以及用于其他的商业目的
 */
package com.oppo.csc.hrms.common.util;

import com.github.dozermapper.core.DozerBeanMapperBuilder;
import com.github.dozermapper.core.Mapper;
import com.vip.vjtools.vjkit.collection.ArrayUtil;

import java.util.ArrayList;
import java.util.List;

/**
 * 自定义一个BeanMapper用来做各种O的转换.
 *
 * @author YangNing
 */
public class CscBeanMapper {

    private final static Mapper MAPPER;

    static {
        MAPPER = DozerBeanMapperBuilder.buildDefault();
    }

    /**
     * 简单的复制出新类型对象.
     *
     * @param source           源数据
     * @param destinationClass 目标类型
     * @param <S>              源类型
     * @param <D>              目标类型
     * @return 目标类型对象
     */
    public static <S, D> D map(S source, Class<D> destinationClass) {
        if (null == source) {
            return null;
        }

        return MAPPER.map(source, destinationClass);
    }

    /**
     * 简单的复制出新对象ArrayList
     *
     * @param sourceList       源数据List
     * @param destinationClass 目标List中的类型
     * @param <S>              源类型
     * @param <D>              目标类型
     * @return 包裹着目标类型的List
     */
    public static <S, D> List<D> mapList(Iterable<S> sourceList, Class<D> destinationClass) {
        if (null == sourceList) {
            return null;
        }

        List<D> destinationList = new ArrayList<>();
        for (S source : sourceList) {
            if (source != null) {
                destinationList.add(MAPPER.map(source, destinationClass));
            }
        }
        return destinationList;
    }

    /**
     * 简单复制出新对象数组
     *
     * @param sourceArray      源数据Array
     * @param destinationClass 目标Array中的类型
     * @param <S>              源类型
     * @param <D>              目标类型
     * @return 包裹着目标类型的Array
     */
    public static <S, D> D[] mapArray(final S[] sourceArray, final Class<D> destinationClass) {
        if (null == sourceArray) {
            return null;
        }

        D[] destinationArray = ArrayUtil.newArray(destinationClass, sourceArray.length);

        int i = 0;
        for (S source : sourceArray) {
            if (source != null) {
                destinationArray[i] = MAPPER.map(sourceArray[i], destinationClass);
                i++;
            }
        }

        return destinationArray;
    }

    /**
     * 基于Dozer将对象A的值拷贝到对象B中.
     *
     * @param source            源对象
     * @param destinationObject 目标对象
     */
    public static void copy(Object source, Object destinationObject) {
        MAPPER.map(source, destinationObject);
    }
}


package com.oppo.csc.hrms.component.logback.aspect;

import com.alibaba.fastjson.JSON;
import net.logstash.logback.encoder.org.apache.commons.lang.StringUtils;
import org.aspectj.lang.ProceedingJoinPoint;
import org.aspectj.lang.annotation.Around;
import org.aspectj.lang.annotation.Aspect;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.core.annotation.Order;
import org.springframework.stereotype.Component;
import org.springframework.util.StopWatch;
import org.springframework.web.context.request.RequestContextHolder;
import org.springframework.web.context.request.ServletRequestAttributes;

import javax.servlet.http.HttpServletRequest;
import java.util.ArrayList;
import java.util.Arrays;
import java.util.Enumeration;
import java.util.List;

/**
 * 统一日志切面,打印入参
 *
 * @author zhangtao
 */
@Aspect
@Component
@Order(1)
public class LogAspect {
    private static final Logger logger = LoggerFactory.getLogger(LogAspect.class);
    private static final String COMMA = ",";

    @Around("@within(org.springframework.web.bind.annotation.RestController)")
    public Object deBefore(ProceedingJoinPoint joinPoint) throws Throwable {
        before(joinPoint);
        final StopWatch stopWatch = new StopWatch();
        stopWatch.start();
        Object object = joinPoint.proceed();
        stopWatch.stop();

        after(object, stopWatch.getTotalTimeMillis());

        return object;
    }

    private void before(ProceedingJoinPoint joinPoint) {
        ServletRequestAttributes attributes = (ServletRequestAttributes) RequestContextHolder.getRequestAttributes();
        if (attributes == null) {
            return;
        }

        HttpServletRequest request = attributes.getRequest();
        Enumeration<String> names = request.getHeaderNames();

        List<String> header = new ArrayList<>();
        while (names.hasMoreElements()) {
            String name = names.nextElement();
            header.add(name + " = " + request.getHeader(name));
        }

        if (logger.isInfoEnabled()) {
            String message = "\n【CSC】请求相关信息：\n【请求头信息】->【{}】,\n【请求方法】->【{}】,\n【请求参数】->【{}】";
            logger.info(message, StringUtils.join(header, COMMA), joinPoint.getSignature(), Arrays.toString(joinPoint.getArgs()));
        }
    }

    private void after(Object object, long totalTimeMillis) {
        if (logger.isInfoEnabled()) {
            String message = "\n【CSC】执行情况：\n执行时间为：【{}毫秒】\n返回值为：【{}】";
            logger.info(message, totalTimeMillis, JSON.toJSONString(object));
        }
    }
}

