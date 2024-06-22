---
title: '单元测试：单元测试中的mock'
date: 2021-01-20
lastmod: 2021-01-20
outdatedInfoWarning: true
featuredImage: "https://images.intotw.cn/blog/2023/11/40abddd196be7e9cb79b83534d4983a4.webp"
featuredImagePreview: "https://images.intotw.cn/blog/2023/11/40abddd196be7e9cb79b83534d4983a4.webp"
tags: ["spring","junit"]
categories: ["技术"]
lightgallery: true
---

&#160; &#160; &#160; &#160;公司要求提升单元测试的质量，提高代码的分支覆盖率和行覆盖率，安排我研究单元测试，指定方案分享并在开发部普及开。整理完资料后，同步一下到博客。

## 单元测试中的mock的目的

&#160; &#160; &#160; &#160;mock的主要目的是让单元测试**Write Once, Run Everywhere**,即编写一次后，可以在任意时刻任意环境运行，无需依赖数据库网络等。

## Mock工具介绍

&#160; &#160; &#160; &#160;Mock工具经过调研，基本上是表格下面的这么个情况:

| mockserver方案    | 开源 | 支持随机参数 | 支持请求延时模拟 | 支持参数上下文 | 仓库分组 | 接口管理 | 仪表盘 | 日志 | 支持管理台配置 | 支持编程 |
| ----------------- | ---- | ------------ | ---------------- | -------------- | -------- | -------- | ------ | ---- | -------------- | -------- |
| rap2，easy-mock等 | 是   | 是           | 否               | 否             | 是       | 是       | 否     | 否   | 是             | 是       |
| wiremock          | 是   | 否           | 是               | 否             | 否       | 否       | 否     | 是   | 否             | 是       |
| mock-server       | 是   | 是           | 是               | 是             | 是       | 否       | 否     | 是   | 否             | 是       |
| postman           | 否   | 否           | 否               | 否             | 否       | 否       | 否     | 否   | 否             | 否       |


&#160; &#160; &#160; &#160;简要介绍下各个的特点和为什么没选：
1. rap2和easy-mock等，都是基于node开发的，和我们开发部的主力语言Java相性一般，后续改造难度大，并且不支持请求超时的配置和上下文的配置，优点是使用操作简单，pass。
2. wiremock，和rap2差不多，就是多个支持延时请求，不过是英文的，pass
3. mock-server，基于java语言的，底层是netty，编程自由，比较适合java技术栈的团队。
4. postman，虽然有mock功能，但是只能针对某个请求的返回固定mock，并且每次启动mock的端口和url完全随机，无法接受，pass。

&#160; &#160; &#160; &#160;我们最后选的是mockito和mock-server，mockito因为是java的mock工具包，所以并不在上面的表格里。

## mockito

### 相关介绍：

&#160; &#160; &#160; &#160;这个包是spring官方也推荐的mock依赖，在spring-boot-starter-test中默认就会自动包含。
&#160; &#160; &#160; &#160;这个包提供的相关类，主要功能就是对某个对象进行mock，通过其提供的特殊的语法，对某个对象的返回以及行为做mock。
### 应用场景：
&#160; &#160; &#160; &#160;单元测试时，如果依赖其他系统的RPC调用（比如feign或dubbo），可以针对相关RPC的调用对象进行直接mock，直接返回成功、超时、异常，减少依赖。
&#160; &#160; &#160; &#160;在对系统内部的某些工具类或者数据库层进行单元测试时，可以模拟一些异常情况，比如数据库超时、框架层抛出某些很难复现的特定异常返回，可以通过直接mock实现来达到效果。
&#160; &#160; &#160; &#160;mockito除了mock外也支持spy，mock与spy的区别是，mock产生的是一个空对象，对mock对象未做配置的方法调用均返回null或异常。spy产生的是一个代理对象，对那些做了配置的方法按照配置的预期返回，未做配置的方法直接会调用原方法。

### 使用方式（spring）：
1. maven中引入：
```xml
<dependency>
    <groupId>org.mockito</groupId>
    <artifactId>mockito-all</artifactId>
    <version>1.9.5</version>
    <scope>test</scope>
</dependency>
```
2. 在测试类中进行如下定义：
```java
//需要mock的服务，一般是RPC，也可以是工具类，总而言之是一个对象
@Mock
TestRpc testRpc;
@Autowired
TestService testService;

//在@Before中对其进行初始化
@Before
public void initMocks() throws Exception {
	//1.1 初始化的api，在这一步执行后，testRpc被初始化为一个mock对象
    MockitoAnnotations.initMocks(this);
    //1.2 使用mock对象替换spring中的bean：这里是将后面要用到的testService中的testRpc这个rpc对象，
    //替换为上面@Mock为我们创建的mock对象，然后我们就可以对这个对象进行mock了，这里的替换是spring容器级别的替换
    //注意，理论上对RPC的service进行mock即可，即替换调用RPC的那个bean中的rpc对象。
    ReflectionTestUtils.setField(AopTargetUtils.getTarget(orderPayFacade), "testRpc", testRpc);
    //1.3 定义mock返回：对新的mock对象进行定义，当后续请求这个rpc的该方法时，会直接return一个空的成功对象
    final ResultRpc<TestVO> testVo = new ResultRpc<>();
    when(testRpc.getAccountByBindCardId("101010")).thenReturn(testVo);
}
```
或者
```java
//另一种初始化方式，更加简单快捷
//这里是另一种写法，设置一个默认的answer，不用每个方法都设置一次返回,也可以继续进行上面那种方式的when配置
final TestRpc testRpcMock = mock(TestRpc.class, new Answer<TestRes>() {
    @Override
    public TestRes answer(InvocationOnMock invocationOnMock) throws Throwable {
        final TestRes testRes = new TestRes();
        testRes.setConfigId(0L);
        testRes.setCityId(86);
        testRes.setServiceId("01");
        testRes.setSysJoinType(0);
        testRes.setMerchantId("320212018002");
        testRes.setMerchantCode("");
        return testRes;
    }
});
ReflectionTestUtils.setField(AopTargetUtils.getTarget(testService), "testRpc", testRpcMock);
```
3. 然后直接正常执行测试即可。

#### 使用方式（spring-boot及以上）：
&#160; &#160; &#160; &#160;前面说了spring-boot-starter较高版本（2.0以上）的test中默认会包括该依赖，所以直接使用就行，更方便的是无需使用反射工具替换spring上下文的bean，使用@MockBean注解标识bean即可。


## mock-server
### 相关资料：
官方文档 <https://www.mock-server.com/>

### 应用场景：

&#160; &#160; &#160; &#160;当进行单元测试时，如果我们需要进行http请求级别的模拟以及mock，那么我们就可以使用mockserver
&#160; &#160; &#160; &#160;当然mockito也可以通过直接mock那些http请求的类来达到相似效果，不过使用mock-server，我们可以更逼真的模拟http的环境，以提前发现那些只有在使用网络下才会出现的问题。
&#160; &#160; &#160; &#160;既可以集成在maven的test生命周期里，也可以直接单独启动做一个server。

### 使用方式：
1. maven中引入：
```xml
<dependency>
    <groupId>org.mock-server</groupId>
    <artifactId>mockserver-netty</artifactId>
    <version>5.11.1</version>
</dependency>
```
2. 在测试类中进行如下定义：
```java
private final int mockPort = 19999;
private ClientAndServer mockServer;
//在@Before中对其进行初始化
@Before
public void initMocks() throws Exception {
	//1.1 初始化的api：启动mockserver
	mockServer = startClientAndServer(mockPort);
	//1.2 配置mockServer
	mockServer
        .when(
                request()
                        .withMethod("POST")
                        .withPath("/test/pay_v1/trade/pay")
                        .withContentType(MediaType.APPLICATION_JSON)
        )
        .respond(
               new TestResponseCallBack()
        );
}
public static class TestResponseCallBack implements ExpectationResponseCallback{
    private final Gson gson=new Gson();
    @Override
    public HttpResponse handle(HttpRequest httpRequest) throws Exception {
        log.info("------------{}",httpRequest);
        if (httpRequest.getMethod().getValue().equals("POST")) {
            //校验签名
            boolean verify = doVerifySign(httpRequest);
            if (!verify){
                return response()
                        .withStatusCode(OK_200.code())
                        .withBody(gson.toJson(CommonResult.failure(CommonErrors.SIGNATURE_VERIFY_FAIL)));
            }
            //构造返回
            return createResponse(httpRequest);
        } else {
            return notFoundResponse();
        }
    }
    private HttpResponse createResponse(HttpRequest httpRequest) throws Exception {
        final HttpRequest httpRequest1 = httpRequest;
        final String req = new String(httpRequest.getBodyAsRawBytes());
        String respBody="";
        final JSONObject jsonObject= JSON.parseObject(req);
        //比如对参数做一些校验
        Assert.assertNotNull(jsonObject.getString("user_id"));
        //构造返回，可以根据请求的内容构造，这里随便写个返回，
        final String user_id = jsonObject.getString("user_id");
        respBody="{\"success\": true,\"errcode\": \"0000\",\"errmsg\": \"成功\",\"result\": {\"user_id\": \"123456\",\"reserved\":"+user_id+"\"\"}}";
        //这里如果必要的话，也可以触发一个延时的回调
        new Thread(new Runnable() {
            @Override
            public void run() {
                LockSupport.parkNanos(1000000000L*2);
                final String notify_url = jsonObject.getString("notify_url");
                HttpHeaders headers = new HttpHeaders();
                headers.setContentType(org.springframework.http.MediaType.APPLICATION_JSON);
                headers.add("Accept", MediaType.APPLICATION_JSON.toString());
                JSONObject param = new JSONObject();
                param.put("username", "123");
                HttpEntity<String> formEntity = new HttpEntity<String>(param.toJSONString(), headers);
                String result = new RestTemplate().postForObject(notify_url, formEntity, String.class);
                log.info("发送回调：{}",param.toJSONString());
            }
        }).start();
        return response()
                .withStatusCode(OK_200.code())
                .withBody(respBody);
    }
    private boolean doVerifySign(HttpRequest httpRequest) throws Exception {
        String signature = httpRequest.getFirstHeader(RequestHeader.Signature);
        String message = new String(httpRequest.getBodyAsRawBytes(), StandardCharsets.UTF_8);
        String md5HexMessage = DigestUtils.md5Hex(message.getBytes(StandardCharsets.UTF_8));
        return RSAUtils.doCheck(md5HexMessage, signature, privateKey, StandardCharsets.UTF_8.displayName());
    }
}

```
3. 然后直接正常执行测试即可。

## cobertura-maven-plugin
&#160; &#160; &#160; &#160;前面的2个mock工具，结合cobertura-maven-plugin，可以瞬间跑起一个带代码覆盖率的测试。

### 使用方式：
1. maven
```xml
<plugin>
    <groupId>org.codehaus.mojo</groupId>
    <artifactId>cobertura-maven-plugin</artifactId>
    <version>2.7</version>
</plugin>
```
2. 执行测试：mvn clean cobertura:cobertura -f pom.xml
3. 到target/site下打开index文件查看结果:
![](https://images.intotw.cn/blog/2023/09/fe1b63362f3df3afd5db09387071191b.jpeg)


### 总结
&#160; &#160; &#160; &#160;本文简单介绍了3个工具的使用，主要是提供了一个可行的方案去推进单元测试，具体3个工具的详细使用细节以及进阶，可以自行查找资料。
