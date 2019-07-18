Edge Service是ServiceComb提供的JAVA网关服务开发框架。Edge Service作为整个微服务系统对外的接口，向最终用户提供服务，接入RESTful请求，转发给内部微服务。Edge Service以开发框架的形式提供，开发者可以非常简单的搭建一个Edge Service服务，通过简单的配置就可以定义路由转发规则。同时Edge Service支持扩展，服务映射、请求解析、加密解密、鉴权等逻辑都可以通过扩展实现。 
  
Edge Service本身也是一个微服务，需遵守ServiceComb微服务开发的规则。其本身可以部署为多实例，前端使用负载均衡装置进行负载分发；也可以部署为主备，直接接入用户请求。开发者可以根据Edge Service承载的逻辑和业务访问量、组网情况来规划。  
  
本文将演示如何通过Edge Service作为网关服务对后端的微服务进行请求转发，场景如下：首先通过Web页面注册一个账号，然后使用该账号登录，其中：  
  
|Name | Academy | score |
|---- |----|----|
|Harry Potter | Gryffindor| 90 |
|Hermione Granger | Gryffindor | 100 |
|Draco Malfoy | Slytherin | 90|
  
操作|外部接口|内部接口 
- | :-: | -: 
账号注册|POST: /rest/crm/user|POST: /user/v1/ 
账号登录|POST: /rest/crm/auth/login|POST: /auth/v1/login 
  
 ![](https://github.com/servicestage-demo/edge/blob/master/edge.jpg)
  
Edge Service开发  
Maven Setting相关配置：  
1.profiles中增加如下配置。  
```
<profile>
    <id>MyProfile</id>   //id自定义
    <repositories>
        <repository>
            <id>HuaweiCloudSDK</id>
            <url>https://mirrors.huaweicloud.com/repository/maven/huaweicloudsdk/</url>
            <releases>
                <enabled>true</enabled>
            </releases>
            <snapshots>
                <enabled>false</enabled>
            </snapshots>
        </repository>
    </repositories>
</profile>
```  
2.在mirrors节点中增加： 
```
<mirror>
    <id>huaweicloud</id>
    <mirrorOf>*,!HuaweiCloudSDK</mirrorOf>
    <url>https://mirrors.huaweicloud.com/repository/maven/</url>
</mirror>  
```
3.新增activeProfiles配置
```
<activeProfiles>
    <activeProfile>MyProfile</activeProfile>   //跟1中的MyProfile保持一致
</activeProfiles>
```
操作步骤：  
1.基于ServiceComb框架创建一个微服务工程（可以直接拷贝这里的）
只需要一个启动类，不用其他任何编码：
```
import org.apache.servicecomb.foundation.common.utils.BeanUtils;
import org.apache.servicecomb.foundation.common.utils.Log4jUtils;

public class ScmedgeserviceApplication {
	public static void main(String[] args) throws Exception{
	    Log4jUtils.init();
	    BeanUtils.init();
	}
}
  ```
2.修改POM  
a)添加华为云微服务的依赖管理配置：
```
<dependencyManagement>
    <dependencies>
      <dependency>
        <groupId>com.huawei.paas.cse</groupId>
        <artifactId>cse-dependency</artifactId>
        <version>2.3.62</version>
        <type>pom</type>
        <scope>import</scope>
      </dependency>
    </dependencies>
</dependencyManagement> 
```
b)添加EdgeService的核心库，以及华为云微服务的核心依赖库  
```
<dependencies>
    <dependency>
      <groupId>org.apache.servicecomb</groupId>
      <artifactId>edge-core</artifactId>
    </dependency>
    <dependency>
      <groupId>org.apache.servicecomb</groupId>
      <artifactId>metrics-core</artifactId>
    </dependency>
    <dependency>
      <groupId>com.huawei.paas.cse</groupId>
      <artifactId>cse-solution-service-engine</artifactId>
    </dependency>
  </dependencies>
  ```
Edge Service配置  
Edge Service的配置也在microservice.yaml中完成，本文只聚焦Edge相关配置说明：  
1.配置路由转发规则：  
```
cse:
  http:
    dispatcher:
      edge:
        url:
          enabled: true
          mappings:
            crm-user:
              prefixSegmentCount: 2
              path: "/rest/crm/user/v1/.*"
              microserviceName: user
              versionRule: 0.0.1-2.0.0
            crm-auth:
              prefixSegmentCount: 2
              path: "/rest/crm/auth/login/v1/.*"
              microserviceName: auth
              versionRule: 0.0.1-2.0.0
```
crm-user配置项表示的含义是将请求路径为/rest/crm/user/.*的请求，转发到crm-user这个微服务，并且只转发到版本号为0.0.1-2.0.0的实例（不含2.0.0）。转发的时候URL为/user/v1/.*。path使用的是JDK的正则表达式，可以查看Pattern类的说明。prefixSegmentCount表示前缀的URL Segment数量，前缀不包含在转发的URL路径中。有三种形式的versionRule可以指定。2.0.0-3.0.0表示版本范围，含2.0.0，但不含3.0.0；2.0.0+表示大于2.0.0的版本，含2.0.0；2.0.0表示只转发到2.0.0版本。2，2.0等价于2.0.0。  
2.跨域配置  
跨域资源共享(CORS, Cross-Origin Resource Sharing)允许Web服务器进行跨域访问控制，使浏览器可以更安全地进行跨域数据传输。当用户需要从浏览器上跨域发送REST请求时就有可能要用到CORS机制，接收跨域请求的微服务需要开启CORS支持。
```
cors:
    enabled: true
    origin: "http://*.*.yourcompany.com"
    allowCredentials: true
    allowedHeader: "appid,content-type"
    allowedMethod: GET,POST,PUT,DELETE
    maxAge: 3600  
```
具体属性的含义，请参考：
https://docs.servicecomb.io/java-chassis/zh_CN/general-development/CORS.html
  
 
 
Edge Service常见问题  
  
>1.部署后业务调不通？    
  
>>按如下方式排查microservice.yaml文件相关配置：  
  
>>>a)Edge Service和被转发微服务的APPLICATION_ID配置是否不一致  
  
>>>b)Edge Service和被转发微服务的service_description.environment配置是否不一致  
  
>>>c)转发规则里的path、microserviceName、versionRule是否配置正确  
  
>>按如下方式在ServiceStage上排查：  
  
>>>d)Edge Service和被转发微服务是否注册到同一个微服务引擎  
  
>>>e)Edge Service和被转发微服务是否部署且注册成功  
  
>>>f)Edge Service的监听端口是否和其他微服务冲突  
  
>2.跨域不生效？    
  
>>a)Origin配置的表达式是否正确  
  
>>b)Origin表达式不支持通过逗号分隔配置多个地址  
  
>>c)header里是否有自定义字段，需要在allowedHeader里配置，如上例中的appid 
  
>>d)相关接口的操作是否在allowedMethod上配置正确  
