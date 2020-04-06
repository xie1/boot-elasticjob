### SpringBoot使用Elastic-Job-lite，实现动态创建定时任务，任务持久化
Elastic-Job是当当开源的一个分布式调度解决方案，由两个相互独立的子项目Elastic-Job-Lite和Elastic-Job-Cloud组成。

Elastic-Job-Lite定位为轻量级无中心化解决方案，使用jar包的形式提供分布式任务的协调服务；Elastic-Job-Cloud采用自研Mesos Framework的解决方案，额外提供资源治理、应用分发以及进程隔离等功能。

这里以Elastic-Job-lite为例，跟SpringBoot进行整合，当当的官方文档中并没有对SpringBoot集成作说明，所有的配置都是基于文档中的xml的配置修改出来的。

### 一、起步
准备好一个SpringBoot的项目，pom.xml中引入Elastic-job，mysql，jpa等依赖

```
<dependencies>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter</artifactId>
        </dependency>

        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-test</artifactId>
            <scope>test</scope>
        </dependency>

        <dependency>
            <groupId>com.dangdang</groupId>
            <artifactId>elastic-job-lite-spring</artifactId>
            <version>2.1.5</version>
        </dependency>

        <dependency>
            <groupId>org.projectlombok</groupId>
            <artifactId>lombok</artifactId>
        </dependency>


        <dependency>
            <groupId>mysql</groupId>
            <artifactId>mysql-connector-java</artifactId>
        </dependency>

        <dependency>
            <groupId>com.zaxxer</groupId>
            <artifactId>HikariCP</artifactId>
        </dependency>
    </dependencies>
```

### 二、配置
使用yaml进行相关属性的配置，主要配置的是数据库连接池，jpa

```
regCenter:
  serverList: 127.0.0.1:2181
  namespace: elastic-job-lite-springboot

spring:
  datasource:
    url: jdbc:mysql://localhost:3306/test?characterEncoding=utf-8&verifyServerCertificate=false&useSSL=false&requireSSL=false
    driver-class-name: com.mysql.jdbc.Driver
    username: xie
    password: xie123456
    type: com.zaxxer.hikari.HikariDataSource
   #  自动创建更新验证数据库结构
  jpa:
    hibernate:
        ddl-auto: update
    show-sql: true
    database: mysql


simpleJob:
  cron: 0/5 * * * * ?
  shardingTotalCount: 3
  shardingItemParameters: 0=Beijing,1=Shanghai,2=Guangzhou

dataflowJob:
  cron: 0/5 * * * * ?
  shardingTotalCount: 3
  shardingItemParameters: 0=Beijing,1=Shanghai,2=Guangzhou
 ```

 elastic-job相关的配置使用java配置实现，代替官方文档的xml配置
```
@Configuration
@ConditionalOnExpression("'${regCenter.serverList}'.length() > 0")
public class RegistryCenterConfig {
    
    @Bean(initMethod = "init")
    public ZookeeperRegistryCenter regCenter(@Value("${regCenter.serverList}") final String serverList, @Value("${regCenter.namespace}") final String namespace) {
        return new ZookeeperRegistryCenter(new ZookeeperConfiguration(serverList, namespace));
    }
}


@Configuration
public class SimpleJobConfig {
    
    @Resource
    private ZookeeperRegistryCenter regCenter;
    
    @Resource
    private JobEventConfiguration jobEventConfiguration;
    
    @Bean
    public SimpleJob simpleJob() {
        return new SpringSimpleJob();
    }
    
    @Bean(initMethod = "init")
    public JobScheduler simpleJobScheduler(final SimpleJob simpleJob, @Value("${simpleJob.cron}") final String cron, @Value("${simpleJob.shardingTotalCount}") final int shardingTotalCount,
                                           @Value("${simpleJob.shardingItemParameters}") final String shardingItemParameters) {
        return new SpringJobScheduler(simpleJob, regCenter, getLiteJobConfiguration(simpleJob.getClass(), cron, shardingTotalCount, shardingItemParameters), jobEventConfiguration);
    }
    
    private LiteJobConfiguration getLiteJobConfiguration(final Class<? extends SimpleJob> jobClass, final String cron, final int shardingTotalCount, final String shardingItemParameters) {
        return LiteJobConfiguration.newBuilder(new SimpleJobConfiguration(JobCoreConfiguration.newBuilder(
                jobClass.getName(), cron, shardingTotalCount).shardingItemParameters(shardingItemParameters).build(), jobClass.getCanonicalName())).overwrite(true).build();
    }
}
```
所有相关的配置到这里就已经OK了，接下来开始具体的编码实现
### 三、定时任务实现
先实现一个自己的任务类，需要实现elastic-job提供的SimpleJob接口，实现它的execute(ShardingContext shardingContext)方法
```
public class SpringSimpleJob implements SimpleJob {
    
    @Resource
    private FooRepository fooRepository;
    
    @Override
    public void execute(final ShardingContext shardingContext) {
        System.out.println(String.format("Item: %s | Time: %s | Thread: %s | %s",
                shardingContext.getShardingItem(), new SimpleDateFormat("HH:mm:ss").format(new Date()), Thread.currentThread().getId(), "SIMPLE"));
        List<Foo> data = fooRepository.findTodoData(shardingContext.getShardingParameter(), 10);
        for (Foo each : data) {
            fooRepository.setCompleted(each.getId());
        }
    }
}
```
接下来实现一个分布式的任务监听器，如果任务有分片，分布式监听器会在总的任务开始前执行一次，结束时执行一次。监听器在之前的ElasticJobConfig已经注册到了Spring容器之中。
```
public class ElasticJobListener extends AbstractDistributeOnceElasticJobListener {
    @Resource
    private TaskRepository taskRepository;

    public ElasticJobListener(long startedTimeoutMilliseconds, long completedTimeoutMilliseconds) {
        super(startedTimeoutMilliseconds, completedTimeoutMilliseconds);
    }

    @Override
    public void doBeforeJobExecutedAtLastStarted(ShardingContexts shardingContexts) {
    }

    @Override
    public void doAfterJobExecutedAtLastCompleted(ShardingContexts shardingContexts) {
        //任务执行完成后更新状态为已执行
        JobTask jobTask = taskRepository.findOne(Long.valueOf(shardingContexts.getJobParameter()));
        jobTask.setStatus(1);
        taskRepository.save(jobTask);
    }
}
```
实现一个ElasticJobHandler，用于向Elastic-job中添加指定的作业配置，作业配置分为3级，分别是JobCoreConfiguration，JobTypeConfiguration和LiteJobConfiguration。LiteJobConfiguration使用JobTypeConfiguration，JobTypeConfiguration使用JobCoreConfiguration，层层嵌套。
```
@Component
public class ElasticJobHandler {
    @Resource
    private ZookeeperRegistryCenter registryCenter;
    @Resource
    private JobEventConfiguration jobEventConfiguration;
    @Resource
    private ElasticJobListener elasticJobListener;

    /**
     * @param jobName
     * @param jobClass
     * @param shardingTotalCount
     * @param cron
     * @param id                 数据ID
     * @return
     */
    private static LiteJobConfiguration.Builder simpleJobConfigBuilder(String jobName,
                                                                       Class<? extends SimpleJob> jobClass,
                                                                       int shardingTotalCount,
                                                                       String cron,
                                                                       String id) {
        return LiteJobConfiguration.newBuilder(new SimpleJobConfiguration(
                JobCoreConfiguration.newBuilder(jobName, cron, shardingTotalCount).jobParameter(id).build(), jobClass.getCanonicalName()));
    }

    /**
     * 添加一个定时任务
     *
     * @param jobName            任务名
     * @param cron               表达式
     * @param shardingTotalCount 分片数
     */
    public void addJob(String jobName, String cron, Integer shardingTotalCount, String id) {
        LiteJobConfiguration jobConfig = simpleJobConfigBuilder(jobName, MyElasticJob.class, shardingTotalCount, cron, id)
                .overwrite(true).build();

        new SpringJobScheduler(new MyElasticJob(), registryCenter, jobConfig, jobEventConfiguration, elasticJobListener).init();
    }
}
```




最后，就可以开始验证整个流程了，代码如下
```
@SpringBootApplication
public class ElasticJobApplication implements CommandLineRunner {
    @Resource
    private ElasticJobService elasticJobService;

    public static void main(String[] args) {
        SpringApplication.run(ElasticJobApplication.class, args);
    }

    @Override
    public void run(String... strings) throws Exception {
        elasticJobService.scanAddJob();
    }
}
```
可以看到，在启动过程中，多个任务被加入到了Elastic-job中，并且一小段时间之后，任务一次执行，执行成功之后，因为我们配置了监听器，会打印数据库的更新SQL，当任务执行完成，再查看数据库，发现状态也更改成功。数据库中同时也会多出两张表JOB_EXECUTION_LOG，JOB_STATUS_TRACE_LOG，这是我们之前配置的JobEventConfiguration，通过数据源持久化了作业配置的相关数据，这两张表的数据可以供Elastic-job提供的运维平台使用，具体请查看官方文档。







