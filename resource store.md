##### 一、收集各种资源网站

1. `https://www.cnblogs.com/xuwujing/p/10393111.html`      技术网站

2. `http://www.ityouknow.com/`    个人博客

3. `http://blog.didispace.com/`  程序员DD

4. `https://www.bysocket.com/`    个人技术博客

5. `http://doc.redisfans.com/index.html`     Redis命令参考

6. `https://freessl.cn/`     免费SSL

7. `https://github.com/yu199195/hmily`     高性能分布式事务tcc方案开源框架

8. `https://github.com/vipshop/vjtools`  唯品会开源的工具类使用

9. `https://mp.baomidou.com/guide/quick-start.html`  mybait-plus使用说明

10. `https://www.imooc.com/article/49814`  RabbitMQ消息可靠性投递解决方案 - 基于SpringBoot实现

11. `https://blog.csdn.net/sunlihuo/article/details/79700225?utm_source=copy`  redis+lua 实现分布式令牌桶，高并发限流

mysql test环境 10.13.31.139  homestead/secret
mysql dev 172.17.160.212:3306 homestead/secret


redis dev  redis.host=172.17.160.212
redis.port=7001
redis.timeout=10
redis.unit=SECONDS
redis.auth=123qwe

redis test  redis.host=10.13.31.136
redis.port=6380
redis.timeout=10
redis.unit=SECONDS
redis.auth=d61d5683-6357-4101-82cf-a09d7980a4b9


git同gitlab链接，免输入username和password
	ssh-keygen -t rsa -C "你的邮箱"


https://www.jianshu.com/p/4a275e779afa
https://juejin.im/post/5af02571f265da0b9e64fcfd#heading-7
http://jm.taobao.org/




启动Name Server
	start mqnamesrv.cmd	(window)
	nohup sh bin/mqnamesrv & tail -f ~/logs/rocketmqlogs/namesrv.log	(linux)

启动Broker
	start mqbroker.cmd -n 127.0.0.1:9876 autoCreateTopicEnable=true	 (window)
	nohup sh bin/mqbroker -n localhost:9876 & tail -f ~/logs/rocketmqlogs/broker.log	(linux)
	
Producer发送消息方式：
	同步：同步发送指消息发送方发出数据后会在收到接收方发回响应之后才发下一个数据包。一般用于重要通知消息，例如重要通知邮件、营销短信
	异步：异步发送指发送方发出数据后，不等接收方发回响应，接着发送下个数据包，一般用于可能链路耗时较长而对响应时间敏感的业务场景，例如用户视频上传后通知启动转码服务。
	单项：单向发送是指只负责发送消息而不等待服务器回应且没有回调函数触发，适用于某些耗时非常短但对可靠性要求并不高的场景，例如日志收集。
	
Producer Group
	这类 Producer 通常发送一类消息并且发送逻辑一致，所以将这些 Producer 分组在一起。从部署结构上看生产者通过 Producer Group 的名字来标记自己是一个集群。
	
Consumer
	拉取型消费者
	推送型消费者:首先要注册消费监听器，当监听器处触发后才开始消费消息。
	
Consumer Group
	
Broker 服务器
	Master和Slave
	单 Master 、多 Master 、多 Master 多 Slave（异步复制）、多 Master多 Slave（同步双写）
	
NameServer
	每个 Broker 在启动的时候会到 NameServer 注册


Message
	一条消息必须有一个主题（Topic），主题可以看做是你的信件要邮寄的地址。一条消息也可以拥有一个可选的标签（Tag）和额处的键值对，它们可以用于设置一个业务 key 并在 Broker 上查找此消息以便在开发期间查找问题
	
Topic
	
Tag

Message Queue
消息消费模式
	集群(Clustering) 和 广播(Broadcasting)
	一个消费者集群共同消费一个主题的多个队列，一个队列只会被一个消费者消费，如果某个消费者挂掉，分组内其它消费者会接替挂掉的消费者继续消费。
	而广播消费消息会发给消费者组中的每一个消费者进行消费。
	
消息顺序	Message Order
	顺序(Orderly) 和 并行(Concurrently)
	顺序消费表示消息消费的顺序同生产者为每个消息队列发送的顺序一致，所以如果正在处理全局顺序是强制性的场景，需要确保使用的主题只有一个消息队列。
	并行消费不再保证消息顺序，消费的最大并行数量受每个消费者客户端指定的线程池限制。
	
	
Producer最佳实践
	1、一个应用尽可能用一个 Topic，消息子类型用 tags 来标识，tags 可以由应用自由设置。只有发送消息设置了tags，消费方在订阅消息时，才可以利用 tags 在 broker 做消息过滤。
	2、每个消息在业务层面的唯一标识码，要设置到 keys 字段，方便将来定位消息丢失问题。由于是哈希索引，请务必保证 key 尽可能唯一，这样可以避免潜在的哈希冲突。
	3、消息发送成功或者失败，要打印消息日志，务必要打印 sendresult 和 key 字段。
	4、对于消息不可丢失应用，务必要有消息重发机制。例如：消息发送失败，存储到数据库，能有定时程序尝试重发或者人工触发重发。
	5、某些应用如果不关注消息是否发送成功，请直接使用sendOneWay方法发送消息。

Consumer最佳实践
	1、消费过程要做到幂等（即消费端去重）
	2、尽量使用批量方式消费方式，可以很大程度上提高消费吞吐量。
	3、优化每条消息消费过程
	
其他配置
	线上应该关闭autoCreateTopicEnable，即在配置文件中将其设置为false。
	RocketMQ在发送消息时，会首先获取路由信息。如果是新的消息，由于MQServer上面还没有创建对应的Topic，这个时候，如果上面的配置打开的话，会返回默认TOPIC的（RocketMQ会在每台broker上面创建名为TBW102的TOPIC）路由信息，然后Producer会选择一台Broker发送消息，选中的broker在存储消息时，发现消息的topic还没有创建，就会自动创建topic。
	后果就是：以后所有该TOPIC的消息，都将发送到这台broker上，达不到负载均衡的目的。
	所以基于目前RocketMQ的设计，建议关闭自动创建TOPIC的功能，然后根据消息量的大小，手动创建TOPIC。



package com.homestead.task.quartz.model;

import java.util.Date;

/**
 * @author:
 * @date 2016/9/30.
 */
public class ScheduleModel {

    private long id;

    private Date createTime = new Date();

    private Date updateTime;

    public long getId() {
        return id;
    }

    public void setId(long id) {
        this.id = id;
    }

    public Date getCreateTime() {
        return createTime;
    }

    public void setCreateTime(Date createTime) {
        this.createTime = createTime;
    }

    public Date getUpdateTime() {
        return updateTime;
    }

    public void setUpdateTime(Date updateTime) {
        this.updateTime = updateTime;
    }
}



package com.homestead.task.quartz.model;

import java.util.Properties;

/**
 * @author:
 * @date 2016/9/30.
 */
public class ScheduleQuartzModel extends ScheduleModel {

    private String className;

    private String cronExpression;

    private String triggerType;

    private String status;

    private Properties data;
    
    @Override
    public String toString() {
        return "ScheduleQuartzModel{" +
                "className='" + className + '\'' +
                ", cronExpression='" + cronExpression + '\'' +
                ", triggerType='" + triggerType + '\'' +
                ", status='" + status + '\'' +
                ", data='" + data + '\'' +
                '}';
    }

    public static final class QuartzScheduleModelBuilder {
        private String className;
        private String cronExpression;
        private String triggerType;
        private String status;
        private Properties data;

        public static QuartzScheduleModelBuilder newBuilder() {
            return new QuartzScheduleModelBuilder();
        }

        public QuartzScheduleModelBuilder withClassName(String className) {
            this.className = className;
            return this;
        }

        public QuartzScheduleModelBuilder withCronExpression(String cronExpression) {
            this.cronExpression = cronExpression;
            return this;
        }

        public QuartzScheduleModelBuilder withTriggerType(String triggerType) {
            this.triggerType = triggerType;
            return this;
        }

        public QuartzScheduleModelBuilder withStatus(String status) {
            this.status = status;
            return this;
        }

        public QuartzScheduleModelBuilder withData(Properties data) {
            this.data = data;
            return this;
        }

        public ScheduleQuartzModel build() {
            ScheduleQuartzModel scheduleQuartzModel = new ScheduleQuartzModel();
            scheduleQuartzModel.setClassName(this.className);
            scheduleQuartzModel.setStatus(this.status);
            scheduleQuartzModel.setCronExpression(this.cronExpression);
            scheduleQuartzModel.setTriggerType(this.triggerType);
            scheduleQuartzModel.setData(this.data);
            return scheduleQuartzModel;
        }
    }

    public String getClassName() {
        return className;
    }

    public void setClassName(String className) {
        this.className = className;
    }

    public String getCronExpression() {
        return cronExpression;
    }

    public void setCronExpression(String cronExpression) {
        this.cronExpression = cronExpression;
    }

    public String getTriggerType() {
        return triggerType;
    }

    public void setTriggerType(String triggerType) {
        this.triggerType = triggerType;
    }

    public String getStatus() {
        return status;
    }

    public void setStatus(String status) {
        this.status = status;
    }

    public Properties getData() {
        return data;
    }

    public void setData(Properties data) {
        this.data = data;
    }

}


package com.homestead.task.quartz.model;

import java.util.Objects;

/**
 * quatz对应的trigger任务类型
 *
 * @author:
 * @date 2016/9/30.
 */
public enum TriggerTypeEnum {
    SIMPLE_TRIGGER("simple_trigger"),
    CRON_TRIGGER("cron_trigger");

    private final String name;

    TriggerTypeEnum(String name) {
        this.name = name;
    }

    public String getName() {
        return this.name;
    }

    public static TriggerTypeEnum getTriggerTypeEnum(String name) {
        for (TriggerTypeEnum triggerTypeEnum : values()) {
            if (Objects.equals(triggerTypeEnum.getName(), name)) {
                return triggerTypeEnum;
            }
        }
        throw new IllegalArgumentException("triggerTypeEnum name " + name + " is not legal.");
    }
}


package com.homestead.task.quartz;

import static org.quartz.SimpleScheduleBuilder.simpleSchedule;

import java.util.ArrayList;
import java.util.Date;
import java.util.List;
import java.util.Objects;
import java.util.Properties;

import javax.annotation.PostConstruct;
import javax.annotation.PreDestroy;
import javax.annotation.Resource;

import org.quartz.CronScheduleBuilder;
import org.quartz.Job;
import org.quartz.JobBuilder;
import org.quartz.JobDetail;
import org.quartz.JobKey;
import org.quartz.Scheduler;
import org.quartz.SchedulerException;
import org.quartz.SchedulerFactory;
import org.quartz.Trigger;
import org.quartz.TriggerBuilder;
import org.quartz.TriggerKey;
import org.quartz.impl.StdSchedulerFactory;
import org.quartz.impl.triggers.CronTriggerImpl;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.jdbc.core.JdbcTemplate;
import org.springframework.stereotype.Component;
import org.springframework.util.CollectionUtils;

import com.homestead.common.util.DateHelper;
import com.homestead.task.quartz.model.ScheduleQuartzModel;
import com.homestead.task.quartz.model.TriggerTypeEnum;
import com.oppo.basic.heracles.client.core.store.KVStore;

/**
 * 定时任务管理器
 *
 * @author:
 */
@Component("scheduleQuartzManager")
public class ScheduleQuartzManager {

	private static final Logger logger = LoggerFactory.getLogger(ScheduleQuartzManager.class);

	private static final String DEFAULT_GROUP = "schedule_job";

	public static final String DATA_KEY = "data";

	@Resource(name = "writeJdbcTemplate")
	private JdbcTemplate mysqlJdbcTemplate;

	private Scheduler scheduler;

	private ScheduleQuartzManager() {

	}

	@Value("${job}")
	private String job;

	@PostConstruct
	public void start() throws Exception {
		logger.info("begin to get the config quartz list");

		Properties p = new Properties();
		p.setProperty("org.quartz.threadPool.threadCount", "30");

		SchedulerFactory schedulerFactory = new StdSchedulerFactory(p);
		scheduler = schedulerFactory.getScheduler();

		List<ScheduleQuartzModel> scheduleQuartzModelList = getScheduleQuartzModelList();
		if (!CollectionUtils.isEmpty(scheduleQuartzModelList)) {
			for (ScheduleQuartzModel scheduleQuartzModel : scheduleQuartzModelList) {
				if (null == scheduleQuartzModel) {
					continue;
				}
				if (!Objects.equals("valid", scheduleQuartzModel.getStatus())) {
					logger.info("job stopped {}", scheduleQuartzModel);
					continue;
				}
				JobDetail jobDetail = getJobDetail(scheduleQuartzModel);
				if (jobDetail == null) {// 不存在job 的class文件
					continue;
				}
				Trigger trigger = createTrigger(scheduleQuartzModel);
				if (null == trigger) {
					continue;
				}
				scheduler.scheduleJob(jobDetail, trigger);
			}
		}
		scheduler.start();
		this.reSchedule();
		logger.info("start the quartz success...");
	}

	private JobDetail getJobDetail(ScheduleQuartzModel scheduleQuartzModel) {
		Class<? extends Job> classNamInst = null;
		try {
			classNamInst = (Class<? extends Job>) Class.forName(scheduleQuartzModel.getClassName());
		} catch (Exception e) {
			logger.error("the class can not be newInstance, the class is :{}.", scheduleQuartzModel.getClassName(), e);
		}
		if (classNamInst == null) {
			return null;
		}
		String jobIdentity = scheduleQuartzModel.getClassName();
		JobDetail jobDetail = JobBuilder.newJob(classNamInst).withIdentity(jobIdentity, DEFAULT_GROUP).build();
		return jobDetail;
	}

	/**
	 * @param scheduleQuartzModel
	 *            任务的配置
	 * @returnr
	 */
	private Trigger createTrigger(ScheduleQuartzModel scheduleQuartzModel) {
		String className = scheduleQuartzModel.getClassName();
		String cronExpression = scheduleQuartzModel.getCronExpression();
		String triggerType = scheduleQuartzModel.getTriggerType();
		Properties data = scheduleQuartzModel.getData();
		logger.info("the className is :{}, the triggerType is :{}, the cronExpression is :{} .data{}", className, triggerType, cronExpression, data);
		Trigger trigger = null;
		TriggerTypeEnum triggerTypeEnum = TriggerTypeEnum.getTriggerTypeEnum(triggerType);
		String triggerIdentity = className;
		if (TriggerTypeEnum.SIMPLE_TRIGGER == triggerTypeEnum) {
			int cronExpressionInt;
			try {
				cronExpressionInt = Integer.parseInt(cronExpression);
			} catch (Exception e) {
				logger.error("parse the cron expression fail, the cornExpress can not transfer to int ,the value is :{}.", cronExpression);
				return null;
			}
			trigger = TriggerBuilder.newTrigger().withIdentity(triggerIdentity, DEFAULT_GROUP).startNow()
					.withSchedule(simpleSchedule().withIntervalInSeconds(cronExpressionInt))
					.withDescription(DateHelper.getDateString(scheduleQuartzModel.getUpdateTime())).build();
		} else if (TriggerTypeEnum.CRON_TRIGGER == triggerTypeEnum) {
			trigger = TriggerBuilder.newTrigger().withIdentity(triggerIdentity, DEFAULT_GROUP)
					.withSchedule(CronScheduleBuilder.cronSchedule(cronExpression))
					.withDescription(DateHelper.getDateString(scheduleQuartzModel.getUpdateTime())).build();
			((CronTriggerImpl) trigger).setStartTime(new Date());
		} else {
			logger.error(" the className is :{}, the trigger type is :{}, the trigger type is not support.", className, triggerType);
		}
		if (trigger != null && data != null && !data.isEmpty()) {
			trigger.getJobDataMap().put(DATA_KEY, data);
		}
		return trigger;
	}

	/**
	 * 停止任务
	 *
	 * @throws Exception
	 */
	@PreDestroy
	public void stop() {
		logger.info(" stop the scheduler.");
		try {
			if (null != scheduler) {
				scheduler.shutdown();
			}
		} catch (Exception e) {
			logger.error(" stop the scheduler failed , the fail message is :{}.", e.getMessage(), e);
		}
	}

	public List<ScheduleQuartzModel> getScheduleQuartzModelList() {
		List<ScheduleQuartzModel> list = new ArrayList<>();
		Properties o;
		for (String i : job.split("\n")) {
			o = KVStore.getProperty(i + ".properties");
			logger.info("{}", o);
			list.add(ScheduleQuartzModel.QuartzScheduleModelBuilder.newBuilder().withClassName(i).withCronExpression(o.getProperty("cron_expression"))
					.withStatus(o.getProperty("status", "invalid")).withTriggerType(o.getProperty("trigger_type")).withData(o).build());
		}
		return list;
	}

	// 监听
	public void reSchedule() {
		for (String i : job.split("\n")) {
			KVStore.Notify.getInstance().addKeyListener(i + ".properties", new KVStore.Notify.UpdateListener() {
				@Override
				public void handleEvent(Object newValue, Object oldValue) {
					synchronized (i.intern()) {
						Properties o = KVStore.getProperty(i + ".properties");
						try {
							ScheduleQuartzModel scheduleQuartzModel = ScheduleQuartzModel.QuartzScheduleModelBuilder.newBuilder().withClassName(i)
									.withCronExpression(o.getProperty("cron_expression")).withStatus(o.getProperty("status", "invalid"))
									.withTriggerType(o.getProperty("trigger_type")).withData(o).build();

							String className = scheduleQuartzModel.getClassName();
							Trigger trigger = scheduler.getTrigger(TriggerKey.triggerKey(className, DEFAULT_GROUP));
							// 存在会即修改
							if (trigger != null) {
								// 暂停
								scheduler.pauseJob(JobKey.jobKey(className, DEFAULT_GROUP));
								if (!Objects.equals("valid", scheduleQuartzModel.getStatus())) {
									logger.info("the className is :{}, delete", className);
									scheduler.unscheduleJob(trigger.getKey());
									scheduler.deleteJob(JobKey.jobKey(className, DEFAULT_GROUP));
								}
								trigger = createTrigger(scheduleQuartzModel);
								if (null == trigger) {
									return;
								}
								scheduler.rescheduleJob(trigger.getKey(), trigger);
								return;
							}

							if (!Objects.equals("valid", scheduleQuartzModel.getStatus())) {
								return;
							}

							JobDetail jobDetail = getJobDetail(scheduleQuartzModel);
							if (jobDetail == null) {// 不存在job 的class文件
								return;
							}
							trigger = createTrigger(scheduleQuartzModel);
							if (null == trigger) {
								return;
							}
							scheduler.scheduleJob(jobDetail, trigger);

						} catch (SchedulerException e) {
							e.printStackTrace();
						}
					}
				}
			});
		}
	}

}


package com.homestead.task.job.base;

import java.util.Properties;
import java.util.concurrent.Callable;
import java.util.concurrent.ThreadPoolExecutor;

import org.quartz.InterruptableJob;
import org.quartz.JobExecutionContext;
import org.quartz.JobExecutionException;
import org.quartz.Trigger;
import org.quartz.UnableToInterruptJobException;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.context.ApplicationContext;
import org.springframework.context.ApplicationContextAware;

import com.homestead.task.application.HeraclesConfiguration;
import com.homestead.task.quartz.ScheduleQuartzManager;
import com.lambdaworks.redis.cluster.api.sync.RedisClusterCommands;

public class BaseJob implements InterruptableJob, ApplicationContextAware {

	protected static final Logger LOGGER = LoggerFactory.getLogger(BaseJob.class);

	protected static ApplicationContext applicationContext;

	protected static HeraclesConfiguration configuration;

	protected static ThreadPoolExecutor globalThreadPoolExecutor;

	protected static RedisClusterCommands<String, String> redisClusterCommands;

	// 用于判断是否关闭线程池
	static public volatile boolean isClose = false;

	protected volatile boolean interrupt = false;

	protected Properties dbProperties;

	@SuppressWarnings("unchecked")
	public void setApplicationContext(ApplicationContext applicationContext) {
		BaseJob.applicationContext = applicationContext;
		BaseJob.configuration = (HeraclesConfiguration) applicationContext.getBean(HeraclesConfiguration.class);
		BaseJob.globalThreadPoolExecutor = (ThreadPoolExecutor) applicationContext.getBean("globalThreadPoolExecutor");
		BaseJob.redisClusterCommands = applicationContext.getBean("redisClusterCommands", RedisClusterCommands.class);
	}

	protected void wait(Object object, long sleep, long loop) {
		this.wait(object, sleep, loop, null);
	}

	protected void wait(long sleep, long loop, Callable<Object> ca) {
		this.wait(null, sleep, loop, ca);
	}

	protected void wait(Object object, long sleep, long loop, Callable<Object> ca) {
		int i = 0;
		while (object == null) {
			try {
				Thread.sleep(sleep);
				if (ca != null) {
					object = ca.call();
				}
				i++;
				if (i > loop)
					break;
			} catch (Exception e) {
				LOGGER.error(e.getMessage(), e);
			}
		}
	}

	@Override
	public void interrupt() throws UnableToInterruptJobException {
		interrupt = true;
	}

	@Override
	public void execute(JobExecutionContext context) throws JobExecutionException {
		// 取得job详情
		Trigger trigger = context.getTrigger();
		dbProperties = (Properties) trigger.getJobDataMap().get(ScheduleQuartzManager.DATA_KEY);
		if (dbProperties == null) {
			dbProperties = new Properties();
		}
	}

}



package com.homestead.task.job.base;

import javax.annotation.PostConstruct;

import org.springframework.stereotype.Component;

import com.homestead.task.quartz.ScheduleQuartzManager;

@Component
public class BeforeClose extends BaseJob {

	@PostConstruct
	public void beforeExit() {
		Runtime.getRuntime().addShutdownHook(new Thread() {
			@Override
			public void run() {
				LOGGER.info("收到kill消息，执行关闭操作");
				// 关闭订阅消费
				isClose = true;
				// 关闭线程池，等待线程池积压消息处理
				globalThreadPoolExecutor.shutdown();
				// // 判断线程池是否关闭
				while (!globalThreadPoolExecutor.isTerminated()) {
					try {
						Thread.sleep(200);
					} catch (InterruptedException e) {
					}
				}
				try {
					Thread.sleep(1000);
				} catch (InterruptedException e) {
				}

				ScheduleQuartzManager sq = (ScheduleQuartzManager) applicationContext.getBean("scheduleQuartzManager");
				sq.stop();

				LOGGER.info("线程池处理完毕");
			}
		});

	}
}


