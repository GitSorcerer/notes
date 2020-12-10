# Scheduled定时任务

### 相关的jar包

```properties
  <!-- quartz定时任务 -->
    <dependency>
      <groupId>org.quartz-scheduler</groupId>
      <artifactId>quartz</artifactId>
      <version>2.3.0</version>
    </dependency>

    <dependency>
      <groupId>org.quartz-scheduler</groupId>
      <artifactId>quartz-jobs</artifactId>
      <version>2.3.0</version>
    </dependency>
```

### 配置文件

#### xml文件

```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
	   xsi:schemaLocation="http://www.springframework.org/schema/beans
	   http://www.springframework.org/schema/beans/spring-beans-3.2.xsd"
	default-lazy-init="false">

	<!-- 调度器 -->
    <bean name="scheduler" class="org.springframework.scheduling.quartz.SchedulerFactoryBean"> 
        <!-- 配置spring上下文 -->
        <property name="applicationContextSchedulerContextKey" value="applicationContext" />
        <!-- quartz属性配置 -->
        <property name="configLocation" value="classpath:quartz.properties" />
    </bean>

    <!-- 加载定时任务 -->
    <bean id="quartzJob" class="com.zccs.assets_management.service.SysJobService" init-method="initJob" />

    <!-- 为任务配置springID,以便实现注入 某些业务需要用到spring的依赖注入 -->
    <!--<bean id="testJob" class="com.zccs.assets_management.scheduled.task.AuthValidJob"></bean>-->
</beans>
```

#### quartz.properties文件

```properties
#线程池设置
org.quartz.threadPool.threadCount=60
org.quartz.threadPool.threadPriority=5

#内存中JobStore的设置，当服务器恢复时会丢失
org.quartz.jobStore.class=org.quartz.simpl.RAMJobStore
```

#### config.properties文件

```properties
#配置定时任务的分组  多个则用，分隔
TASK_GROUP=01
#输出定时任务信息
#JOB_OUTPUT_INFO=1
```

#### web.xml文件

```xml
 <!-- 关联到spring主配置文件的路径地址 classpath根目录 -->
  <context-param>
    <param-name>contextConfigLocation</param-name>
    <param-value>classpath:spring/applicationContext.xml;classpath:/quartz.xml</param-value>
  </context-param>
```

### 相关代码

#### 并发执行

```java
/**
 * @Description: JobDetail的concurrent设为true，支持多个job并发运行
 * @Author gaoh
 * @Date 2019/8/13
 */
public class JobFactoryConcurrent implements Job {

	/**
	 * 定时任务最终就是根据cron时间表达式 不断的调用 execute方法  然后通过反射执行具体的方法
	 * @param context
	 * @throws JobExecutionException
	 */
	@Override
	public void execute(JobExecutionContext context) throws JobExecutionException {
		SysJobModel task = (SysJobModel) context.getMergedJobDataMap().get(SysJobModel.class.getSimpleName());
		BaseUtils.invokJobMethod(task);
	}
}
```

#### 非并发

```java
/**
 * @Description: JobDetail的concurrent设为false，禁止并发执行多个相同定义的JobDetail，标签@DisallowConcurrentExecution控制禁止并发
 * @Author gaoh
 * @Date 2019/8/13
 * @see DisallowConcurrentExecution   设置不允许并发
 */
@DisallowConcurrentExecution
public class JobFactory implements Job {

	@Override
	public void execute(JobExecutionContext context) throws JobExecutionException {
		SysJobModel task = (SysJobModel) context.getMergedJobDataMap().get(SysJobModel.class.getSimpleName());
		BaseUtils.invokJobMethod(task);
	}
}
```

#### service主业务层

```java
/**
 * 
 * @Description: 定时任务业务层
 * @author gaoh
 * @date 20190813
 *
 */
@SuppressWarnings({ "rawtypes", "unchecked" })
public class SysJobService {

    @Resource
	private SqlSessionFactory sqlSessionFactory;

    @Resource
    private SysJobMapper sysJobMapper;

    @Resource
    private SchedulerFactoryBean schedulerFactoryBean; // 在quartz-model.xml配置注入

    private static Scheduler scheduler = null;

    public static String[] TASK_GROUP; //定时任务分组

    @Resource
    private PictureDao pictureDao;

    /**
     * 查询所有
     * @return
     */
    public List<SysJobModel> selectList() {
        return sysJobMapper.selectAll();
    }
    /**
     * 单条记录查询
     * @param model
     * @return
     */
    public SysJobModel selectOne(SysJobModel model) {
        return sysJobMapper.selectOne(model);
    }

    /**
     * 新增
     *
     * @param request
     * @throws Exception
     */
    @Transactional(rollbackFor = {Exception.class})
    public void add(HttpServletRequest request) throws Exception {
        String userName = SecurityUtils.getSubject().getSession().getAttribute(SysParameter.USER_NAME).toString();
        String taskJob = request.getParameter("taskJob");
        SysJobModel model = new SysJobModel();
        BaseUtils.serializeArray2Model(model, taskJob);

        if (!validCron(model.getCron().trim())) {
            throw new BusinessException("不是有效的cron表达式!");
        }

//        model.setId(CryptographyUtils.getUUID16());
        model.setCron(model.getCron().trim());
        model.setClassName(model.getClassName().trim());
        model.setMethod("execute");
        model.setIsEnable("0");

        model.setCreateBy(userName);
        model.setCreateDate(new Date());
        model.setUpdateBy(userName);
        model.setUpdateDate(new Date());

        sysJobMapper.insertSelective(model);
    }

    /**
     * 修改
     *
     * @param request
     * @throws Exception
     */
    @Transactional(rollbackFor = {Exception.class})
    public void update(HttpServletRequest request) throws Exception {
        String userName = SecurityUtils.getSubject().getSession().getAttribute(SysParameter.USER_NAME).toString();
        String taskJob = request.getParameter("taskJob");
        SysJobModel model = new SysJobModel();
        BaseUtils.serializeArray2Model(model, taskJob);

        if (!validCron(model.getCron().trim())) {
            throw new BusinessException("不是有效的cron表达式!");
        }

        model.setCron(model.getCron().trim());
        model.setClassName(model.getClassName().trim());
        model.setUpdateBy(userName);
        model.setUpdateDate(new Date());

        sysJobMapper.updateByPrimaryKeySelective(model);

    }


    /**
     * 删除
     * @param request
     * @throws Exception
     */
    @Transactional(rollbackFor = { Exception.class })
    public void delete(HttpServletRequest request) throws Exception {
        String ids = request.getParameter("ids");

        if (StringUtils.isNotBlank(ids)) {
            SqlSession sqlSession = null;
            try {
                sqlSession = sqlSessionFactory.openSession(ExecutorType.BATCH);
                // 必须用sqlSession获得mapper
                SysJobMapper mapper = sqlSession.getMapper(SysJobMapper.class);
                String[] idsTemp = ids.split("~");
                for (int i = 0; i < idsTemp.length; i++) {
                    SysJobModel model = new SysJobModel();
                    model.setId(Integer.parseInt(idsTemp[i]));
                    mapper.deleteByPrimaryKey(model);
                    // 批量提交 每50条提交一次
                    if ((i + 1) % 50 == 0) {
                        sqlSession.flushStatements();
                    }
                }
                sqlSession.flushStatements();
            }catch (Exception e){
                throw new BusinessException(e.getMessage());
            }finally {
                if(sqlSession != null){
                    sqlSession.close();
                }
            }
        } else {
            throw new BusinessException("请选择需要删除的数据！");
        }
    }

    /**
     * 启用定时任务
     * @param ids
     * @throws Exception
     */
    @Transactional(rollbackFor = { Exception.class })
    public void startJob(String ids) throws Exception {
        // 只有配置了job_group的服务才能进行修改时间表达式操作
        if (TASK_GROUP == null || TASK_GROUP.length == 0) {
            throw new BusinessException("系统未配置定时任务分组[TASK_GROUP]，没有操作权限!");
        }
        if (scheduler == null) {
            throw new BusinessException("数据库模式的定时任务初始化失败，请检查定时任务配置！");
        }

//        String ids = request.getParameter("ids");
        String[] idsTemp = ids.split("~");
        for (int i = 0; i < idsTemp.length; i++) {
            SysJobModel model = new SysJobModel();
            model.setId(Integer.parseInt(idsTemp[i]));
            model = sysJobMapper.selectByPrimaryKey(model);
            if(model == null){
                throw new BusinessException("定时任务[ID："+idsTemp[i]+"]不存在，请检查!");
            }
            if("0".equals(model.getIsEnable())) {
                // 开启定时任务
                // 反射检查定时任务类是否存在
                try{
                    Class.forName(model.getClassName());
                }catch(Exception e){
                    throw new BusinessException("定时任务["+model.getClassName()+"]不存在，请检查!");
                }
                operJob(model, 1);
                BaseUtils.loggerDebug(model.getName() + "已成功启动！");
            }else{
                continue;
            }

            model.setIsEnable("1");
            sysJobMapper.updateByPrimaryKeySelective(model);
        }
    }

    /**
     * 停用定时任务
     * @param request
     * @throws Exception
     */
    @Transactional(rollbackFor = { Exception.class })
    public void stopJob(HttpServletRequest request) throws Exception {
        // 只有配置了job_group的服务才能进行修改时间表达式操作
        if (TASK_GROUP == null || TASK_GROUP.length == 0) {
            throw new BusinessException("系统未配置定时任务分组[TASK_GROUP]，没有操作权限!");
        }
        if (scheduler == null) {
            throw new BusinessException("数据库模式的定时任务初始化失败，请检查定时任务配置！");
        }

        String ids = request.getParameter("ids");
        String[] idsTemp = ids.split("~");
        for (int i = 0; i < idsTemp.length; i++) {
            SysJobModel model = new SysJobModel();
            model.setId(Integer.parseInt(idsTemp[i]));
            model = sysJobMapper.selectByPrimaryKey(model);
            if(model == null){
                throw new BusinessException("定时任务(ID："+idsTemp[i]+")不存在，请检查!");
            }
            if("1".equals(model.getIsEnable())) {
                // 停用定时任务
                operJob(model, 0);
                BaseUtils.loggerDebug(model.getName() + "以停用！");
            }else{
                continue;
            }

            model.setIsEnable("0");
            sysJobMapper.updateByPrimaryKeySelective(model);
        }
    }

    /**
     * 更新时间规则
     *
     * @param jobModel
     * @throws Exception
     */
    @Transactional(rollbackFor = {Exception.class})
    public void editRuleExecute(SysJobModel jobModel) throws Exception {
      /*  String id = request.getParameter("id");
        String cron = request.getParameter("cron").trim();*/
        Integer id = jobModel.getId();
        String cron = jobModel.getCron();

        if (id==null) {
            throw new BusinessException("定时任务ID不能为空!");
        }
        if (StringUtils.isBlank(cron)) {
            throw new BusinessException("时间规则不能为空!");
        } else {
            if (!validCron(cron)) {
                throw new BusinessException("不是有效的cron表达式!");
            }
        }
        SysJobModel model = new SysJobModel();

        model.setId(id);
        model = sysJobMapper.selectByPrimaryKey(model);

        if (model == null) {
            throw new BusinessException("未找到该定时任务!");
        }
        // 只有配置了job_group的服务才能进行修改时间表达式操作
        if (TASK_GROUP == null || TASK_GROUP.length == 0) {
            throw new BusinessException("系统未配置定时任务分组[TASK_GROUP]，没有操作权限!");
        }
        if (scheduler == null) {
            throw new BusinessException("数据库模式的定时任务初始化失败，请检查定时任务配置！");
        }

        model.setCron(cron);
        sysJobMapper.updateByPrimaryKeySelective(model);

        if ("1".equals(model.getIsEnable())) { // 只有正在启用的定时任务才重新生成trigger
            TriggerKey triggerKey = TriggerKey.triggerKey(model.getName(), model.getGroupNo());
            CronTrigger trigger = (CronTrigger) scheduler.getTrigger(triggerKey);
            if (trigger == null) {
                return;
            }

            // 重置时间
            CronScheduleBuilder scheduleBuilder = CronScheduleBuilder.cronSchedule(model.getCron());
            trigger = trigger.getTriggerBuilder().withIdentity(triggerKey).withSchedule(scheduleBuilder).build();

            scheduler.rescheduleJob(triggerKey, trigger);
        }
    }

    /**
     * 检查定时任务名称是否重复
     * @param request
     * @return
     * @throws Exception
     * @throws Exception
     */
    public void sysJobCheckName(HttpServletRequest request) throws Exception {
        SysJobModel model = new SysJobModel();

        if (StringUtils.isBlank(request.getParameter("name"))) {
            return;
        } else {
                 model.setName(BaseUtils.encodeUTF8(request.getParameter("name")));
        }

        if (sysJobMapper.selectCount(model) > 0) {
            throw new BusinessException("名称已存在");
        }
    }

    /**
     * 检查定时任务调用类是否重复
     * @param request
     * @return
     * @throws Exception
     * @throws Exception
     */
    public void sysJobCheckClassName(HttpServletRequest request) throws Exception {
        SysJobModel model = new SysJobModel();

        if (StringUtils.isBlank(request.getParameter("className"))) {
            return;
        } else {
            model.setClassName(BaseUtils.encodeUTF8(request.getParameter("className")));
        }
        if (sysJobMapper.selectCount(model) > 0) {
            throw new BusinessException("调用类已存在");
        }
    }

    /**
     * 项目启动时调用的初始化job方法
     * 要在quartz.xml中配置
     */
    public void initJob() throws Exception {

        //加载配置文件中的分组信息
        String taskConfig = "";
        Object task_group = BaseUtils.getConfig().get("TASK_GROUP");
        if (task_group != null) {
             taskConfig = task_group.toString();
        }

        TASK_GROUP = StringUtils.isNotBlank(taskConfig) ? taskConfig.split(",") : new String[0];

        //获取分组任务信息数据，如果存在分组，再找分组里对应开启的任务
        if(TASK_GROUP == null || TASK_GROUP.length == 0 ){
            BaseUtils.loggerDebug("系统未配置定时任务分组[TASK_GROUP]...");
            return;
        }
        //初始化scheduler
        scheduler = schedulerFactoryBean.getScheduler();
        if (scheduler == null) {
            BaseUtils.loggerDebug("数据库模式的定时任务初始化失败，请检查定时任务配置！");
            return;
        }

        List<SysJobModel> jobList = new ArrayList();
        for(int i = 0; i < TASK_GROUP.length; i++){
            Example ex = new Example(SysJobModel.class);
            Criteria criteria = ex.createCriteria();
            criteria.andEqualTo("groupNo", TASK_GROUP[i]);
            List<SysJobModel> listJob = sysJobMapper.selectByExample(ex);
            jobList.addAll(listJob);
        }

        BaseUtils.loggerDebug("系统当前任务组中定时任务总数："+jobList.size()+"");

        for(int j = 0; j < jobList.size(); j++){
            SysJobModel model = jobList.get(j);
            if ("0".equals(model.getIsEnable())) {
                continue;
            }
            operJob(model, 1);
        }
    }

    /**
     * 操作job
     * @param model
     * @param sign 0停用 1启用
     * @throws SchedulerException
     */
    public void operJob(SysJobModel model, int sign) throws SchedulerException {
        if (model == null) {
            return;
        }
        //获取分组任务信息数据，如果存在分组，再找分组里对应开启的任务
        if(TASK_GROUP == null || TASK_GROUP.length == 0 ){
            throw new BusinessException("系统未配置定时任务分组[TASK_GROUP]...");
        }
        //初始化scheduler
        scheduler = schedulerFactoryBean.getScheduler();
        if (scheduler == null) {
            throw new BusinessException("数据库模式的定时任务初始化失败，请检查定时任务配置！");
        }

        //停用
        if(sign == 0){
            JobKey jobKey = JobKey.jobKey(model.getName(), model.getGroupNo());
            scheduler.deleteJob(jobKey);
        }else if(sign == 1){//启用
            // 往TriggerKey中插入name与Group,TriggerKey继承Key类,key类中有name与group两个字段,group默认为default
            TriggerKey triggerKey = TriggerKey.triggerKey(model.getName(), model.getGroupNo());
            // 创建trigger对象
            CronTrigger trigger = (CronTrigger) scheduler.getTrigger(triggerKey);

            //若trigger对象不存在，则创建一个
            if (trigger == null) {
                //判断job并发还是等待
                Class clazz = "0".equals(model.getIsConcurrent()) ? JobFactory.class : JobFactoryConcurrent.class;
                JobDetail jobDetail = JobBuilder.newJob(clazz).withIdentity(model.getName(), model.getGroupNo()).build();
                /**
                 * @see  JobFactoryConcurrent
                 * @see  JobFactory
                 * 将任务信息 存到任务详情里面  供反射调用
                 */
                jobDetail.getJobDataMap().put(SysJobModel.class.getSimpleName(), model);
                CronScheduleBuilder scheduleBuilder = CronScheduleBuilder.cronSchedule(model.getCron());
                trigger = TriggerBuilder.newTrigger().withIdentity(model.getName(), model.getGroupNo()).withSchedule(scheduleBuilder).startNow().build();

                scheduler.scheduleJob(jobDetail, trigger);
            } else {
                //Trigger已存在，那么更新相应的定时设置
                CronScheduleBuilder scheduleBuilder = CronScheduleBuilder.cronSchedule(model.getCron());
                //按新的cronExpression表达式重新构建trigger
                trigger = trigger.getTriggerBuilder().withIdentity(triggerKey).withSchedule(scheduleBuilder).build();

                //按新的trigger重新设置job执行
                scheduler.rescheduleJob(triggerKey, trigger);
            }
        }
    }

    /**
     * 校验cron表达式合法性
     * @param cron
     * @return
     */
    private static Boolean validCron(String cron){
        if(StringUtils.isBlank(cron)){
            return false;
        }
        String regMiao = "([0-9]{1,2}|[0-9]{1,2}\\-[0-9]{1,2}|\\*|[0-9]{1,2}\\/[0-9]{1,2}|[0-9]{1,2}\\,[0-9]{1,2})";
        String regFen = "\\s([0-9]{1,2}|[0-9]{1,2}\\-[0-9]{1,2}|\\*|[0-9]{1,2}\\/[0-9]{1,2}|[0-9]{1,2}\\,[0-9]{1,2})";
        String regShi = "\\s([0-9]{1,2}|[0-9]{1,2}\\-[0-9]{1,2}|\\*|[0-9]{1,2}\\/[0-9]{1,2}|[0-9]{1,2}\\,[0-9]{1,2})";
        String regRi = "\\s([0-9]{1,2}|[0-9]{1,2}\\-[0-9]{1,2}|\\*|[0-9]{1,2}\\/[0-9]{1,2}|[0-9]{1,2}\\,[0-9]{1,2}|\\?|L|W|C)";
        String regYue = "\\s([0-9]{1,2}|[0-9]{1,2}\\-[0-9]{1,2}|\\*|[0-9]{1,2}\\/[0-9]{1,2}|[0-9]{1,2}\\,[0-9]{1,2}|JAN|FEB|MAR|APR|MAY|JUN|JUL|AUG|SEP|OCT|NOV|DEC)";
        String regZhou = "\\s([1-7]{1}|[1-7]{1}\\-[1-7]{1}|[1-7]{1}\\#[1-7]{1}|\\*|[1-7]{1}\\/[1-7]{1}|[1-7]{1}\\,[1-7]{1}|MON|TUES|WED|THUR|FRI|SAT|SUN|\\?|L|C)";
        String regNian = "(\\s([0-9]{4}|[0-9]{4}\\-[0-9]{4}|\\*|[0-9]{4}\\/[0-9]{4}|[0-9]{4}\\,[0-9]{4})){0,1}";
        String regEx = regMiao + regFen + regShi + regRi + regYue + regZhou + regNian;
        //String regEx = "(((^([0-9]|[0-5][0-9])(\\,|\\-|\\/){1}([0-9]|[0-5][0-9]) )|^([0-9]|[0-5][0-9]) |^(\\* ))((([0-9]|[0-5][0-9])(\\,|\\-|\\/){1}([0-9]|[0-5][0-9]) )|([0-9]|[0-5][0-9]) |(\\* ))((([0-9]|[01][0-9]|2[0-3])(\\,|\\-|\\/){1}([0-9]|[01][0-9]|2[0-3]) )|([0-9]|[01][0-9]|2[0-3]) |(\\* ))((([0-9]|[0-2][0-9]|3[01])(\\,|\\-|\\/){1}([0-9]|[0-2][0-9]|3[01]) )|(([0-9]|[0-2][0-9]|3[01]) )|(\\? )|(\\* )|(([1-9]|[0-2][0-9]|3[01])L )|([1-7]W )|(LW )|([1-7]\\#[1-4] ))((([1-9]|0[1-9]|1[0-2])(\\,|\\-|\\/){1}([1-9]|0[1-9]|1[0-2]) )|([1-9]|0[1-9]|1[0-2]) |(\\* ))(([1-7](\\,|\\-|\\/){1}[1-7])|([1-7])|(\\?)|(\\*)|(([1-7]L)|([1-7]\\#[1-4]))))|(((^([0-9]|[0-5][0-9])(\\,|\\-|\\/){1}([0-9]|[0-5][0-9]) )|^([0-9]|[0-5][0-9]) |^(\\* ))((([0-9]|[0-5][0-9])(\\,|\\-|\\/){1}([0-9]|[0-5][0-9]) )|([0-9]|[0-5][0-9]) |(\\* ))((([0-9]|[01][0-9]|2[0-3])(\\,|\\-|\\/){1}([0-9]|[01][0-9]|2[0-3]) )|([0-9]|[01][0-9]|2[0-3]) |(\\* ))((([0-9]|[0-2][0-9]|3[01])(\\,|\\-|\\/){1}([0-9]|[0-2][0-9]|3[01]) )|(([0-9]|[0-2][0-9]|3[01]) )|(\\? )|(\\* )|(([1-9]|[0-2][0-9]|3[01])L )|([1-7]W )|(LW )|([1-7]\\#[1-4] ))((([1-9]|0[1-9]|1[0-2])(\\,|\\-|\\/){1}([1-9]|0[1-9]|1[0-2]) )|([1-9]|0[1-9]|1[0-2]) |(\\* ))(([1-7](\\,|\\-|\\/){1}[1-7] )|([1-7] )|(\\? )|(\\* )|(([1-7]L )|([1-7]\\#[1-4]) ))((19[789][0-9]|20[0-9][0-9])\\-(19[789][0-9]|20[0-9][0-9])))";
        Pattern pattern = Pattern.compile(regEx);
        Matcher matcher = pattern.matcher(cron);
        if(matcher.matches()){
            return true;
        }else{
            return false;
        }
    }


    /**
     * 获取定时任务详细信息
     * @param request
     * @return
     * @throws Exception
     */
    public Map<String, Object> sysJobGetDetail(HttpServletRequest request) throws Exception {
        Map<String, Object> resultMap = new HashMap<String, Object>();
        String id = request.getParameter("id");
        if (StringUtils.isBlank(id)) {
            throw new BusinessException("定时任务ID不能为空，请检查！");
        }
        if (scheduler == null) {
            throw new BusinessException("数据库模式的定时任务初始化失败，请检查定时任务配置！");
        }

        GroupMatcher<JobKey> matcher = GroupMatcher.anyJobGroup();
        Set<JobKey> jobKeys = scheduler.getJobKeys(matcher);

        // 根据jobId查询jobMdoel
        SysJobModel jobModel = new SysJobModel();
        jobModel.setId(Integer.parseInt(id));
        jobModel = sysJobMapper.selectByPrimaryKey(jobModel);
        if(jobModel == null){
            throw new BusinessException("未找到ID为["+id+"]的数据，请检查！");
        }

//        List<SysJobModel> jobList = new ArrayList<SysJobModel>();
        for (JobKey jobKey : jobKeys) {
            List<? extends Trigger> triggers = scheduler.getTriggersOfJob(jobKey);
            for (Trigger trigger : triggers) {
                if(jobModel.getName().equals(jobKey.getName()) && jobModel.getGroupNo().equals(jobKey.getGroup())){
                    //                System.out.println("定时任务名称："+jobKey.getName());
                    //                System.out.println("定时任务分组："+jobKey.getGroup());
                    //                System.out.println("触发器key（分组+名称）："+trigger.getKey());
                    //                Trigger.TriggerState triggerState = scheduler.getTriggerState(trigger.getKey());
                    //                String status = triggerState.name();
                    //                System.out.println("定时任务状态："+status);

                    if (trigger instanceof CronTrigger) {
                        CronTrigger cronTrigger = (CronTrigger) trigger;
//                        String cronExpression = cronTrigger.getCronExpression();
                        //                    System.out.println("cron表达式："+cronExpression);
                        //                    System.out.println("本次执行时间："+DateUtils.date2Str(cronTrigger.getStartTime(), null));
                        //                    System.out.println("下次执行时间："+DateUtils.date2Str(cronTrigger.getNextFireTime(), null));
                        // 下次执行时间传递至前台页面
                        resultMap.put("data", DateFormatUtils.date2Str(cronTrigger.getNextFireTime(), null));
                    }
                }else{
                    continue;
                }
            }
        }
        return resultMap;
    }
}
```

#### BaseUtils类

```java
/**
 * @Author: GH
 * @Date: 2019/6/30 10:34
 * @Version 1.0
 * <p>
 * 通用方法
 */
public class BaseUtils {
    // slf4j-Logger
    private static final Logger logger = LoggerFactory.getLogger(BaseUtils.class);
    /**
     * 将序列化的表单转化为model
     *
     * @param model
     * @param data
     */
    private static final String DATECLASS = "java.util.Date";
    /**
     * URL UTF-8 转码
     *
     * @param str
     * @return
     */
    private static final String UNDEFINED = "undefined";

    /**
     * 日志输出(普通)
     *
     * @param str
     */
    public static void loggerDebug(String str) {
        logger.debug(str);
    }

    /**
     * 日志输出(占位符)
     *
     * @param str
     * @param obj
     */
    public static void loggerDebug(String str, Object[] obj) {
        logger.debug(str, obj);
    }

    /**
     * 警告日志输出(普通)
     *
     * @param str
     */
    public static void loggerWarn(String str) {
        logger.warn(str);
    }

    /**
     * 警告日志输出(占位符)
     *
     * @param str
     */
    public static void loggerWarn(String str, Object[] obj) {
        logger.warn(str, obj);
    }

    /**
     * 异常日志输出
     *
     * @param e
     * @return
     */
    public static String loggerError(Throwable e) {
        String msg = "";
        if (e instanceof NullPointerException) {
            msg = e.toString();
        } else {
            msg = e.getMessage();
        }
        logger.error(msg, e);
        return msg;
    }

    public static void serializeArray2Model(Object model, String data) {
        Map dataMap = jsonArrStr2Map(data);
        try {
            BeanUtilsBean.getInstance().getConvertUtils().register(false, false, 0); // 解决Bigdecimal为null报错问题
            //ConvertUtils.register(new DateLocaleConverter(), Date.class); // Date注册的转化器
            //2019-04-08 17:27:56
            ConvertUtils.register(new Converter() {
                @Override
                public Object convert(Class arg0, Object arg1) {
                    if (!(DATECLASS.equals(arg0.getName()))) {
                        return null;
                    }
                    BaseUtils.loggerDebug("注册字符串转换date类型转换器");
                    if (arg1 == null) {
                        return null;
                    }
                    // 毫秒数处理
                    if (arg1 instanceof Long && (((Long) arg1).toString()).length() == 13) {
                        // 强制转毫秒数
                        Date timeMillis = new Date();
                        try {
                            timeMillis.setTime((Long) arg1);
                            return timeMillis;
                        } catch (Exception e4) {
                            BaseUtils.loggerDebug("非时间戳字符串或暂未支持的日期字符串类型：{}", new Object[]{arg1});
                            return arg1;
                        }
                    }
                    if (!(arg1 instanceof String)) {
                        return arg1;
                    } else {
                        String str = (String) arg1;
                        if (StringUtils.isBlank(str)) {
                            return null;
                        }
                        // 开始转换日期
                        SimpleDateFormat sd = new SimpleDateFormat("yyyy-MM-dd HH:mm:ss");
                        try {
                            return sd.parse(str);
                        } catch (ParseException e) {
                            SimpleDateFormat sd1 = new SimpleDateFormat("yyyy/MM/dd HH:mm:ss");
                            try {
                                return sd1.parse(str);
                            } catch (ParseException e1) {
                                SimpleDateFormat sd2 = new SimpleDateFormat("yyyy-MM-dd");
                                try {
                                    return sd2.parse(str);
                                } catch (ParseException e2) {
                                    SimpleDateFormat sd3 = new SimpleDateFormat("yyyy/MM/dd");
                                    try {
                                        return sd3.parse(str);
                                    } catch (ParseException e3) {
                                        BaseUtils.loggerDebug("非日期字符串或暂未支持的日期字符串类型：{}", new Object[]{str});
                                        return arg1;
                                    }
                                }
                            }
                        }
                    }
                }
            }, Date.class); // Date注册的转化器（扩展）

            org.apache.commons.beanutils.BeanUtils.populate(model, dataMap);
        } catch (Exception e) {
            loggerError(e);
            throw new BusinessException("serializeArray转换model失败，" + e.getMessage());
        }
    }

    /**
     * 将request参数转化为model
     *
     * @param request
     * @return
     */
    public static void request2Model(Object model, HttpServletRequest request) { // 返回值为随意的类型传入数为request和该随意类型
        try {
            Enumeration en = request.getParameterNames(); //获得参数的一个列举
            BeanUtilsBean.getInstance().getConvertUtils().register(false, false, 0);// 解决Bigdecimal为null报错问题
            //ConvertUtils.register(new DateLocaleConverter(), Date.class); // Date注册的转化器

            ConvertUtils.register(new Converter() {
                @Override
                public Object convert(Class arg0, Object arg1) {
                    if (!DATECLASS.equals(arg0.getName())) {
                        return null;
                    }
                    BaseUtils.loggerDebug("注册字符串转换date类型转换器");
                    if (arg1 == null) {
                        return null;
                    }
                    // 毫秒数处理
                    if (arg1 instanceof Long && (((Long) arg1).toString()).length() == 13) {
                        // 强制转毫秒数
                        Date timeMillis = new Date();
                        try {
                            timeMillis.setTime((Long) arg1);
                            return timeMillis;
                        } catch (Exception e4) {
                            BaseUtils.loggerDebug("非时间戳字符串或暂未支持的日期字符串类型：{}", new Object[]{arg1});
                            return arg1;
                        }
                    }
                    if (!(arg1 instanceof String)) {
                        return arg1;
                    } else {
                        String str = (String) arg1;
                        if (StringUtils.isBlank(str)) {
                            return null;
                        }
                        // 开始转换日期
                        SimpleDateFormat sd = new SimpleDateFormat("yyyy-MM-dd HH:mm:ss");
                        try {
                            return sd.parse(str);
                        } catch (ParseException e) {
                            SimpleDateFormat sd1 = new SimpleDateFormat("yyyy/MM/dd HH:mm:ss");
                            try {
                                return sd1.parse(str);
                            } catch (ParseException e1) {
                                SimpleDateFormat sd2 = new SimpleDateFormat("yyyy-MM-dd");
                                try {
                                    return sd2.parse(str);
                                } catch (ParseException e2) {
                                    SimpleDateFormat sd3 = new SimpleDateFormat("yyyy/MM/dd");
                                    try {
                                        return sd3.parse(str);
                                    } catch (ParseException e3) {
                                        SimpleDateFormat sd4 = new SimpleDateFormat("yyyy");
                                        try {
                                            return sd4.parse(str);
                                        } catch (ParseException e4) {
                                            BaseUtils.loggerDebug("非日期字符串或暂未支持的日期字符串类型：{}", new Object[]{str});
                                            return arg1;
                                        }
                                    }
                                }
                            }
                        }
                    }
                }
            }, Date.class); // Date注册的转化器（扩展）

            //遍历列举来获取所有的参数
            while (en.hasMoreElements()) {
                String name = (String) en.nextElement();
                String value = BaseUtils.encodeUTF8(request.getParameter(name));
                org.apache.commons.beanutils.BeanUtils.setProperty(model, name, value.trim());
            }
        } catch (Exception e) {
            BaseUtils.loggerError(e);
            throw new BusinessException("request转换model失败，" + e.getMessage());
        }
    }

    /**
     * 将JSONArray字符串转成map(支持JSON对象，数组只支持name： value：形式)，前台jquery serializeArray()方法专用
     *
     * @param data
     * @return
     */
    public static Map<String, Object> jsonArrStr2Map(String data) {
        Map<String, Object> resultMap = new HashMap<String, Object>();
        // 获得你字符串所属的对象
        Object json = new JSONTokener(data).nextValue();
        if (json instanceof net.sf.json.JSONObject) {
            JSONObject jsonObj = JSONObject.parseObject(data);
            resultMap = JSONObject.parseObject(jsonObj.toJSONString(), new TypeReference<Map<String, Object>>() {
            });
        } else if (json instanceof net.sf.json.JSONArray) {
            JSONArray jsonArray = JSONObject.parseArray(data);

            for (Object o : jsonArray) {
                JSONObject jsonObj = (JSONObject) o;
                resultMap.put(jsonObj.getString("name"), jsonObj.getString("value"));
            }
        } else {
            throw new BusinessException("无效的JSON字符串！");
        }
        return resultMap;
    }

    public static String encodeUTF8(String str) {
        if (StringUtils.isBlank(str)) {
            return "";
        }
        if (UNDEFINED.equals(str)) {
            return "";
        }
        if (str.equals(new String(str.getBytes(StandardCharsets.ISO_8859_1), StandardCharsets.ISO_8859_1))) {
            return new String(str.getBytes(StandardCharsets.ISO_8859_1), StandardCharsets.UTF_8).trim();
        } else {
            return str;
        }
    }


    /**
     * 获取 文件上传本地磁盘路径  图片
     *
     * @return
     */
    public static String getLocalPath(String localName) {
//        String localPath = "C:ZZCS" + File.separator + "~usr" + File.separator + "local" + File.separator + "ZCCS";
        String localPath = "D:" + File.separator + "assets_management" + File.separator + "upload" + File.separator + localName + "~usr" + File.separator + "local" + File.separator + "ZCCS";
        if (StringUtils.isBlank(localPath)) {
            throw new BusinessException("未找到文件上传本地磁盘路径配置！");
        }
        String[] localPathArr = localPath.split("~");
        String os = System.getProperty("os.name");
        if (os.toLowerCase().startsWith("win")) {
            return localPathArr[0];
        } else {
            if (localPathArr.length < 2) {
                throw new BusinessException("未找到文件上传本地磁盘Linux路径配置！");
            }
            return localPathArr[1];
        }
    }

    /**
     * 获取 文件上传本地磁盘路径 资产档案
     * @return
     */
//    public static String getAssetFilePath(String localPath) {
////        String localPath = "D:"+ File.separator +"assets_management"+ File.separator +"upload"+ File.separator +"AssetFiles"+ "~usr" + File.separator + "local" + File.separator + "ZCCS";
//        if(StringUtils.isBlank(localPath)){
//            throw new BusinessException("未找到文件上传本地磁盘路径配置！");
//        }
//        String[] localPathArr = localPath.split("~");
//        String os = System.getProperty("os.name");
//        if (os.toLowerCase().startsWith("win")) {
//            return localPathArr[0];
//        } else {
//            if(localPathArr.length < 2){
//                throw new BusinessException("未找到文件上传本地磁盘Linux路径配置！");
//            }
//            return localPathArr[1];
//        }
//    }

    /**
     * 删除文件
     *
     * @param path
     * @throws Exception
     */
    public static void deleteFile(String path) {
        if (StringUtils.isNotBlank(path)) {
            File file = new File(path);
            if (file.isFile() && file.exists()) {
                file.delete();
            }
        }
    }

    /**
     * 删除目录下所有文件及文件夹
     *
     * @param path
     */
    public static void deleteDirectory(String path) {
        if (StringUtils.isNotBlank(path)) {
            File file = new File(path);
            //1級文件刪除
            if (!file.isDirectory()) {
                file.delete();
            } else if (file.isDirectory()) {
                //2級文件列表
                String[] filelist = file.list();
                //获取新的二級路徑
                for (int j = 0; j < filelist.length; j++) {
                    File filessFile = new File(path + "\\" + filelist[j]);
                    if (!filessFile.isDirectory()) {
                        filessFile.delete();
                    } else if (filessFile.isDirectory()) {
                        //递归調用
                        deleteDirectory(path + "\\" + filelist[j]);
                    }
                }
                file.delete();
            }
        }
    }


    /**
     * 求两个日期的时间差
     *
     * @param startTime
     * @param endTime
     * @return 相差的天数
     */
    public static int getDistanceTime(Date startTime, Date endTime) {
        int days = 0;
        long time1 = startTime.getTime();
        long time2 = endTime.getTime();

        long diff;
        if (time1 < time2) {
            diff = time2 - time1;
        } else {
            diff = time1 - time2;
        }
        days = (int) (diff / (24 * 60 * 60 * 1000));
        return days;
    }

    /**
     * 返回日期前    年/月/天 的日期
     *
     * @param date  日期
     * @param num  前多少 年/月/天  的日期
     * @param param 年/月/天
     * @return
     */
    public static Date getDate(Date date, Integer num, String param) {
        Calendar calendar = Calendar.getInstance();//日历对象
        calendar.setTime(date);//设置当前日期
        if ("DATE".equalsIgnoreCase(param)) {//日
            calendar.add(Calendar.DATE, -num);
        } else if ("MONTH".equalsIgnoreCase(param)) {//月
            calendar.add(Calendar.MONTH, -num);
        } else if ("YEAR".equalsIgnoreCase(param)) {//年
            calendar.add(Calendar.YEAR, -num);
        }
        return calendar.getTime();
    }

    /**
     * 获取 文件上传本地磁盘路径  文件
     *
     * @return
     */
    public static String getFilePath() {
        String localPath = "D:" + File.separator + "assets_management" + File.separator + "upload" + File.separator + "contract" + "~usr" + File.separator + "local" + File.separator + "ZCCS";
        if (StringUtils.isBlank(localPath)) {
            throw new BusinessException("未找到文件上传本地磁盘路径配置！");
        }
        String[] localPathArr = localPath.split("~");
        String os = System.getProperty("os.name");
        if (os.toLowerCase().startsWith("win")) {
            return localPathArr[0];
        } else {
            if (localPathArr.length < 2) {
                throw new BusinessException("未找到文件上传本地磁盘Linux路径配置！");
            }
            return localPathArr[1];
        }
    }

    /**
     * 获取客户端浏览器类型、编码下载文件名
     *
     * @param request
     * @param fileName
     * @return String
     */
    public static String encodeFileName(HttpServletRequest request, String fileName) {
        String userAgent = request.getHeader("User-Agent");
        String rtn = "";
        try {
            String new_filename = URLEncoder.encode(fileName, "UTF8");
            // 如果没有UA，则默认使用IE的方式进行编码，因为毕竟IE还是占多数的
            rtn = "filename=\"" + new_filename + "\"";
            if (userAgent != null) {
                userAgent = userAgent.toLowerCase();
                // IE浏览器，只能采用URLEncoder编码
                if (userAgent.indexOf("msie") != -1) {
                    rtn = "filename=\"" + new_filename + "\"";
                }
                // Opera浏览器只能采用filename*
                else if (userAgent.indexOf("opera") != -1) {
                    rtn = "filename*=UTF-8''" + new_filename;
                }
                // Safari浏览器，只能采用ISO编码的中文输出
                else if (userAgent.indexOf("safari") != -1) {
                    rtn = "filename=\"" + new String(fileName.getBytes("UTF-8"), "ISO8859-1") + "\"";
                }
                // Chrome浏览器，只能采用MimeUtility编码或ISO编码的中文输出
                else if (userAgent.indexOf("applewebkit") != -1) {
                    new_filename = MimeUtility.encodeText(fileName, "UTF8", "B");
                    rtn = "filename=\"" + new_filename + "\"";
                }
                // FireFox浏览器，可以使用MimeUtility或filename*或ISO编码的中文输出
                else if (userAgent.indexOf("mozilla") != -1) {
                    rtn = "filename*=UTF-8''" + new_filename;
                }
            }
        } catch (UnsupportedEncodingException e) {
            e.printStackTrace();
        }
        return rtn;
    }

    /**
     * 计算时间差，以小时为单位。如：2018-08-08 和 2018-08-07 相差24h
     *
     * @param startTime
     * @param endTime
     * @return
     */
    public double getDistanceTime2(Date startTime, Date endTime) {
        double hour;
        double time1 = startTime.getTime();
        double time2 = endTime.getTime();

        double diff;
        if (time1 < time2) {
            diff = time2 - time1;
        } else {
            diff = time1 - time2;
        }
        hour = (diff / (60 * 60 * 1000));
        return hour;
    }


    /**
     * 解析IP地址
     * @return
     */
    public static String analysisIP(HttpServletRequest request) {
        String IP = "";
        if (request.getHeader("x-forwarded-for") == null) {
            IP = request.getRemoteAddr();
        } else {
            IP = request.getHeader("x-forwarded-for");
        }
        if ("0:0:0:0:0:0:0:1".equals(IP)) {
            IP = "127.0.0.1";
        }
        return IP;
    }

    /**
     * 反射执行job中的方法（springId(即spring配置的bean id)优先考虑）
     * @param job
     */
    public static void invokJobMethod(SysJobModel job) {
        Object object = null;
        Class clazz = null;

        if (StringUtils.isNotBlank(job.getSpringId())) {
            object = SpringBeanFactoryUtils.getApplicationContext().getBean(job.getSpringId());
        } else if (StringUtils.isNotBlank(job.getClassName())) {
            try {
                clazz = Class.forName(job.getClassName());
                object = clazz.newInstance();
            } catch (Exception e) {
                BaseUtils.loggerError(e);
            }
        }

        if (object == null) {
            BaseUtils.loggerDebug("定时任务[" + job.getName() + "]启动失败，请检查配置！");
            return;
        }

        clazz = object.getClass();
        Method method = null;
        try {
            //通过反射得到execute方法
            method = clazz.getDeclaredMethod("execute");
        } catch (NoSuchMethodException e) {
            BaseUtils.loggerDebug("定时任务["+job.getName()+"]中没有找到[execute()]方法，请检查！");
        } catch (SecurityException e) {
            BaseUtils.loggerError(e);
        }

        if (method != null) {
            try {
                method.invoke(object);
                // 判断是否输出定时任务执行信息
                if ("1".equals(getConfig().get("JOB_OUTPUT_INFO"))) {
                    BaseUtils.loggerDebug("定时任务[" + job.getName() + "]启动成功！");
                }
            } catch (Exception e) {
                BaseUtils.loggerError(e);
            }
        }
    }


    private static Map<String, Object> config;

    /**
     * 加载config.properties 文件
     * @return
     */
    public static Map<String, Object> getConfig() {
        if (config == null) {
            try {
                config = readProperties(SysParameter.PARAM_PATH);
                if(config != null){
                    BaseUtils.loggerDebug("加载[config.properties]成功！");
                }else{
                    throw new BusinessException("加载[config.properties]失败！");
                }
            } catch (Exception e) {
                BaseUtils.loggerError(e);
            }
        }
        return config;
    }



    /**
     * 读取并解析配置文件
     * @param fileName classpath下文件名
     * @return
     * @throws Exception
     */
    public static Map<String, Object> readProperties(String fileName) throws Exception {
        Map<String, Object> reslutMap = new HashMap<String, Object>();
        FileInputStream in = null;

        String absolutePath = BaseUtils.getAbsoluteClasspath() + File.separator + fileName.trim();
        in = new FileInputStream(absolutePath);
        Properties properties = new Properties(); //实例化
        properties.load(in); //从filePath得到键值对

        Enumeration<?> enmObject = properties.keys(); //得到所有的主键信息（这里的主键信息主要是简化的主键，也是信息的关键）

        while (enmObject.hasMoreElements()) { //对每一个主键信息进行检索处理，跟传入的返回值信息是否有相同的地方（如果有相同的地方，取出主键信息的属性，传回给返回信息）
            String curKey = ((String) enmObject.nextElement()).trim();
            if(curKey.contains("#")){ // 带#号的key为注释内容，自动忽略
                continue;
            }
            String curMessage = new String(properties.getProperty(curKey).getBytes("ISO-8859-1"), "UTF-8").trim();
            reslutMap.put(BaseUtils.encodeUTF8(curKey), BaseUtils.encodeUTF8(curMessage));
        }

        in.close();

        return reslutMap;
    }

    /**
     * 获取classpath绝对路径
     * @return
     */
    public static String getAbsoluteClasspath() {
        String absolutePath = Thread.currentThread().getContextClassLoader().getResource("").toString(); //tomcat绝对路径
        absolutePath = absolutePath.replaceAll("file:/", "");
        // https://www.cnblogs.com/zhengxl5566/p/10783422.html
        absolutePath = absolutePath.replaceAll("%20", " ");
        absolutePath = File.separator + absolutePath.trim();
        BaseUtils.loggerDebug("classpath:" + absolutePath);
        return absolutePath;
    }

}
```

#### SpringBeanFactoryUtils类

```java

/**
 * 普通类调用Spring注解方式的Service层bean
 */
@Component
public class SpringBeanFactoryUtils implements ApplicationContextAware {

	private static ApplicationContext context;

	/**
	 * 可以把ApplicationContext对象inject到当前类中作为一个静态成员变量
	 * @param applicationContext ApplicationContext 对象.
	 * @throws BeansException
	 */
	@Override
    public void setApplicationContext(ApplicationContext applicationContext) throws BeansException {
		context = applicationContext;
	}

	/**
	 * 获取ApplicationContext
	 * @return
	 */
	public static ApplicationContext getApplicationContext() {
		return context;
	}

	/**
	 * 一个便利的方法，能够快速得到一个bean
	 * @param beanName
	 * @return
	 */
	public static Object getBeanByName(String beanName) {
		return context.getBean(beanName);
	}

	/**
	 * 一个便利的方法，根据能够快速得到一个bean
	 * @param t
	 * @param <T>
	 * @return
	 */
    public static <T> Object getBeanByClass(Class<T> t) {
        return context.getBean(t);
    }
    
}
```

### 执行任务的类

#### job

```java
/**
 * @author gaoh
 * @version 1.0
 * @date 2019/8/14 14:38
 * <p>
 * 电费的执行计划
 */
@Component
public class ElectricPayJob {

    private FeePayCostDao feePayCostDao;

    public void init() {
        feePayCostDao = SpringBeanFactoryUtils.getApplicationContext().getBean(FeePayCostDao.class);
    }

    /**
     * 任务的执行方法
     */
    @Transactional(rollbackFor = {Exception.class})
    public void execute() {
        Example example = new Example(FeePayCostModel.class);
        Example.Criteria criteria = example.createCriteria();
        //电费
        criteria.andEqualTo("payType", "1");
        //未缴费
        criteria.andEqualTo("payStatus", "0");
        //查询水费信息
        List<FeePayCostModel> list = feePayCostDao.selectByExample(example);
        BaseUtils.loggerDebug("水费信息:");
        for (FeePayCostModel feePayCost : list) {
            BaseUtils.loggerDebug(JSON.toJSONString(feePayCost));
        }
        //查询电费的策略  例如 缴费的前二十天开始预警等等

    }
}
```

### 实体类

#### sys_job

```java
private static final long serialVersionUID = 1L;
@Id
private Integer id;
private String name;//任务名称
private String groupNo;//任务分组
private String status;//任务状态
private String cron;//规则表达式
private String springId;//调用类SpringID
private String className;//调用类
private String method;//调用类方法
private String isConcurrent;//是否并发，默认否
private String isEnable;//是否启用
private Date startTime;//本次运行时间
private Date nextFireTime;//下次运行时间
private String isSys;//是否是系统级(1是 其它否)
private String memo;//备注
private String createBy;//创建人
private Date createDate;//创建时间
private String updateBy;//修改人
private Date updateDate;//修改时间
```

