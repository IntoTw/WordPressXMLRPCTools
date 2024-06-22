---
title: '分布式系统：xxl-job改造spring-cloud'
date: 2020-10-14T00:00:00+08:00
lastmod: 2020-10-14T00:00:00+08:00
outdatedInfoWarning: true
featuredImage: "https://images.intotw.cn/blog/2023/11/40abddd196be7e9cb79b83534d4983a4.webp"
featuredImagePreview: "https://images.intotw.cn/blog/2023/11/40abddd196be7e9cb79b83534d4983a4.webp"
tags: ["xxl-job",spring-cloud","分布式"]
categories: ["技术"]
lightgallery: true
---



修改后的源码仓库地址:[GitHub](https://github.com/IntoTw/xxl-job-cloud). ：


## 改造原因
1. 原有的xxl-job使用自己实现的http协议进行注册以及调度等，与目前框架中本身的注册中心格格不入，会影响健康检查、日志处理、问题排查。
2. 技术栈统一。避免执行器内包含两套注册逻辑。
3. 提高分布式健壮性，原有的服务注册以及发现等功能较弱，且与实际应用可用与否完全无关，经常存在xxl-job线程出问题，但主服务正常，或主服务出问题，但xxl-job线程正常。
4. 灰度扩展，目前系统灰度使用eureka定制实现，为执行器支持灰度，必须进行改造。

## 主要改造思路
### 调度中心
调度中心侧获取服务时，将原有的基于数据库的地址list，修改为动态从eureka中心获取服务的地址列表，两者通过xxl-job-admin配置的执行器app-name，与执行器的spring.application.name（即注册到eureka的服务标识）关联。
```java
//com.xxl.job.admin.core.trigger.XxlJobTrigger
XxlJobGroup group = XxlJobAdminConfig.getAdminConfig().getXxlJobGroupDao().load(jobInfo.getJobGroup());
//@edit 如果是自动获取地址的话，则使用
if (group.getAddressType() == 0) {
    group.setAddressList(SpringAdminContext.getEurekaAddressList(group.getAppname()));
}
```
```java
//这种静态和spring容器不分的写法，还是有点别扭的
@Component("springAdminContext")
public class SpringAdminContext {
    @Autowired
    DiscoveryClient discoveryClient;

    private static SpringAdminContext springAdminContext;
    @PostConstruct
    public void initialize() {
        springAdminContext = this;
        springAdminContext.discoveryClient=this.discoveryClient;
    }
    public static String getEurekaAddressList(String appName){
        //may be springContext not init
        if(springAdminContext !=null){
            DiscoveryClient discoveryClient = SpringAdminContext.springAdminContext.discoveryClient;
            List<ServiceInstance> instances = discoveryClient.getInstances(appName);
            StringBuilder addressBuilder = new StringBuilder();
            for (int i = 0; i < instances.size(); i++) {
                addressBuilder.append(instances.get(i).getUri().toString());
                if(i!=instances.size()) {
                    addressBuilder.append(",");
                }
            }
            return addressBuilder.toString();
        }else {
            return "";
        }

    }
}
```
调度中心侧调度服务时，增加灰度策略，在获取到eureka的instanceList后，从instance的meta原数据中取出灰度标识，进行灰度调度。代码结合1中的列表获取，具体灰度实现与百度到的eureka灰度相同，略。
### 调度中心 执行器侧
通过修改core包，将原有的注册线程删除，并删除embedServer的实现，修改为springMVC。
```java
//修改后的代替embedServer的处理类
@Controller
public class XxlJobHandlerController {
    private static final Logger logger = LoggerFactory.getLogger(XxlJobHandlerController.class);

    private ExecutorBiz executorBiz;
    @Value("${xxl.job.accessToken:}")
    private String accessToken;
    @Autowired
    ThreadPoolExecutor bizThreadPool;
    @PostConstruct
    public void start() {
        executorBiz = new ExecutorBizImpl();
    }

    @PostMapping("/job/{method}")
    @ResponseBody
    public ReturnT jobHandle(HttpServletRequest httpServletRequest, HttpServletResponse httpServletResponse, @PathVariable("method") String methodName) {
        return doHandlerReq(httpServletRequest,httpServletResponse,"/"+methodName);
    }
    // ---------------------- registry ----------------------

    protected ReturnT doHandlerReq(HttpServletRequest httpServletRequest,HttpServletResponse httpServletResponse,String method) {
        try {
            //read request
            //@edit 这里请求都模拟的原有处理方式，包括header这些
            int contentLength = httpServletRequest.getContentLength();
            byte[] reqBody=new byte[contentLength];
            httpServletRequest.getInputStream().read(reqBody,0,contentLength);
            String requestData=new String(reqBody, StandardCharsets.UTF_8);
            String uri = method;
            String accessTokenReq = httpServletRequest.getHeader(XxlJobRemotingUtil.XXL_JOB_ACCESS_TOKEN);
            //@edit 这里原有是netty纯异步，但是到这边http就不太合适了，这么写虽然看起来吞吐会下降，但是一般web容器现在底层也支持nio了，应该关系不大。
            FutureTask<ReturnT> stringFutureTask=new FutureTask<ReturnT>(() -> process(uri, requestData, accessTokenReq));
            // invoke
            bizThreadPool.execute(stringFutureTask);
            ReturnT returnT = stringFutureTask.get();
            httpServletResponse.setHeader(HttpHeaders.CONTENT_TYPE, "text/html;charset=UTF-8");


            return returnT;
        } catch (Exception e) {
            return new ReturnT<String>(ReturnT.FAIL_CODE, "invalid request, uri-mapping empty.");
        }
    }

    private ReturnT process(String uri, String requestData, String accessTokenReq) {

        if (uri == null || uri.trim().length() == 0) {
            return new ReturnT<String>(ReturnT.FAIL_CODE, "invalid request, uri-mapping empty.");
        }
        if (accessToken != null
                && accessToken.trim().length() > 0
                && !accessToken.equals(accessTokenReq)) {
            return new ReturnT<String>(ReturnT.FAIL_CODE, "The access token is wrong.");
        }

        // services mapping
        try {
            if ("/beat".equals(uri)) {
                return executorBiz.beat();
            } else if ("/idleBeat".equals(uri)) {
                IdleBeatParam idleBeatParam = GsonTool.fromJson(requestData, IdleBeatParam.class);
                return executorBiz.idleBeat(idleBeatParam);
            } else if ("/run".equals(uri)) {
                TriggerParam triggerParam = GsonTool.fromJson(requestData, TriggerParam.class);
                return executorBiz.run(triggerParam);
            } else if ("/kill".equals(uri)) {
                KillParam killParam = GsonTool.fromJson(requestData, KillParam.class);
                return executorBiz.kill(killParam);
            } else if ("/log".equals(uri)) {
                LogParam logParam = GsonTool.fromJson(requestData, LogParam.class);
                return executorBiz.log(logParam);
            } else {
                return new ReturnT<String>(ReturnT.FAIL_CODE, "invalid request, uri-mapping(" + uri + ") not found.");
            }
        } catch (Exception e) {
            logger.error(e.getMessage(), e);
            return new ReturnT<String>(ReturnT.FAIL_CODE, "request error:" + ThrowableUtil.toString(e));
        }
    }
}
```
修改处理callback的逻辑，使得任务执行结果可以回调到admin
```java
//com.xxl.job.core.executor
//@edit 这个方法主要用于执行器在获取调度中心列表时调用，目的是执行器获取调度中心列表进行callback回调通知执行结果。
    //如果callback失败或者这里出问题，那么会导致在管理台看到的执行结果永远没有。
    //管理台的调度结果和执行结果是分开的，调度结果依赖单次http请求，执行结果依赖callback
    // @see TriggerCallbackThread
    public static List<AdminBiz> getAdminBizList(){
        List<AdminBiz> adminBizs=new ArrayList<>();
        String adminAppName="xxl-job-admin-cloud";
        List<String> addressList = Arrays.asList(SpringContext.getEurekaAddressList(adminAppName).split(","));
        addressList.forEach(e ->{
            adminBizs.add(new AdminBizClient(e.concat("/").concat("xxl-job-admin"),accessToken));
        });
        return adminBizs;
    }
```


## 总结
其实还有一些细节的修改包括eureka的配置，原有注册代码等的调整，就不好一一列出来了，具体可以拉下来项目搜索@edit，主要修改的地方我都加了这个。
