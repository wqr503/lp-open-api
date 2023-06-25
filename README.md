## 脑洞的由来
场景：服务间调用通过统一的Feign接口实现多服务间Api协同

## 功能需求
1. 能通过服务名调用，服务名可动态修改
2. 能通过url调用，url地址可动态修改
3. 由于接口被多服务继承，需保证方法体尽量少变动
4. 传递参数有对应的字段和备注

## 问题
1. @RequestHeader 注解对应单个Header配置，header的改变会导致方法体的变动
2. @RequestParam 注解对应单个Get请求中param配置，get请求参数的变动会导致方法体的变动
3. @RequestHeader 支持Map，但是Map对参数提示不友好

### Git地址
https://gitee.com/wqrzsy/lp-demo/tree/master/lp-open-api

## 更多demo请关注
[springboot demo实战项目](https://www.jianshu.com/c/6e4240dceb27)
[java 脑洞](https://www.jianshu.com/c/45db2a1a841c)
[java 面试宝典](https://www.jianshu.com/c/2899dc0daa81)
[开源工具](https://www.jianshu.com/c/180e879891d2)

## 功能实现
整体项目结构
![image.png](https://upload-images.jianshu.io/upload_images/17317532-4ed6634dd7c9211f.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


可以看到用到的类很少，下面列举下关键类

RequestParam 通过调用initMap方法实现将bean参数装入Map
```
public interface RequestParam extends Map<String, Object> {

    default void initMap() {
        initMap(true, Util.UTF_8);
    }

    default void initMap(boolean encodeValue) {
        initMap(encodeValue, Util.UTF_8);
    }

    void initMap(boolean encodeValue, Charset charset);

}
```
ParameterExpander 通过Expander触发调用RequestParam的initMap方法
```
/**
 * param 处理器
 */
public class ParameterExpander implements Param.Expander {

    private boolean encode = false;

    private Charset charset = Util.UTF_8;

    public static ParameterExpander build(boolean encode, Charset charset) {
        ParameterExpander expander = new ParameterExpander();
        expander.encode = encode;
        expander.charset = charset;
        return expander;
    }

    @Override
    public String expand(Object value) {
        if(BlankAide.isBlank(value)) {
            return null;
        }
        if(value instanceof RequestParam) {
            RequestParam requestParam = (RequestParam) value;
            requestParam.initMap(encode, charset);
            return "requestParam.initMap()";
        }
        return value.toString();
    }

}

```
RequestHeaderParamParameterProcessor对RequestHeaderParameterProcessor增强，通过Expander触发RequestParam的initMap方法，把对象属性注入Map中，然后通过headerMapIndex触发Map转param
```
/**
 * RequestHeaderParam注解处理器
 * 通过Expander触发RequestParam的initMap方法，把对象属性注入Map中，然后通过headerMapIndex触发Map转param
 */
public class RequestHeaderParamParameterProcessor extends RequestHeaderParameterProcessor {

    private boolean encode = false;

    private Charset charset = Util.UTF_8;

    public static RequestHeaderParamParameterProcessor build(boolean encode, Charset charset) {
        RequestHeaderParamParameterProcessor requestHeaderParamParameterProcessor = new RequestHeaderParamParameterProcessor();
        requestHeaderParamParameterProcessor.encode = encode;
        requestHeaderParamParameterProcessor.charset = charset;
        return requestHeaderParamParameterProcessor;
    }

    @Override
    public boolean processArgument(AnnotatedParameterContext context, Annotation annotation, Method method) {
        int parameterIndex = context.getParameterIndex();
        Class<?> parameterType = method.getParameterTypes()[parameterIndex];
        MethodMetadata data = context.getMethodMetadata();
        if (RequestParam.class.isAssignableFrom(parameterType)) {
            context.setParameterName("s");
            Map<Integer, Param.Expander> indexToExpander= data.indexToExpander();
            indexToExpander.put(parameterIndex, ParameterExpander.build(encode, charset));
            data.indexToEncoded().put(parameterIndex, Boolean.FALSE);
            MethodMetadata metadata = context.getMethodMetadata();
            if (metadata.headerMapIndex() == null) {
                metadata.headerMapIndex(parameterIndex);
            }
            return true;
        }
        return super.processArgument(context, annotation, method);
    }
}
```
## 实际运用
demo项目地址: https://gitee.com/wqrzsy/lp-demo/tree/master/lp-open-api-demo
1. 首先开启功能的注解 @EnableOpenApiClients
```
@SpringBootApplication
@EnableOpenApiClients
public class TestApplication {

    public static void main(String[] args) {
        ConfigurableApplicationContext run = SpringApplication.run(TestApplication.class);
    }

}

```

2. api定义
```
public interface TestApi {

    @GetMapping(value = "/test", produces = {MediaType.APPLICATION_JSON_UTF8_VALUE})
    String test(@RequestParam GetRequestParam param, @RequestHeader HeaderParam header);

}
```
3. FeignClient

```
/**
 * value对应服务名称
 * url 对应地址，如果url不为空，则优先url，否则走服务名调用
 */
@FeignClient(value = "${text.name:TestApiClient}", url = "${text.url:}")
public interface TestApiClient extends TestApi {

}

```

4. 实际使用

```
@Service
public class TestService {

    @Autowired
    private TestApiClient testApiClient;

    @PostConstruct
    public void init() {
        GetRequestParam getRequestParam = new GetRequestParam();
        getRequestParam.setName("我是HelloWorld");
        getRequestParam.setAge(10);
        HeaderParam headerParam = new HeaderParam();
        headerParam.setHeader_key_2("我是key2");
        headerParam.setHeader_key_1("我是Key1");
        String test = testApiClient.test(getRequestParam, headerParam);
        System.out.println("testApiClient返回的结果：" + test);
    }

}

```

5. 然后我们来看下测试用的Controller
```
@RestController
public class TestController {

    @GetMapping("/test")
    public String test(String name, String age, HttpServletRequest request) {
        System.out.println("获取参数name:" + name);
        System.out.println("获取参数age:" +age);
        System.out.println("获取header的header_key_1:" + request.getHeader("header_key_1"));
        System.out.println("获取header的header_key_2:" + request.getHeader("header_key_2"));
        return "success";
    }

}
```
6. Controller接收到结果
   ![image.png](https://upload-images.jianshu.io/upload_images/17317532-39822f204df01b16.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


## demo项目导入
参考： https://www.jianshu.com/p/cd0275a2f5fb


如果这篇文章对你有帮助请给个star
![image.png](https://upload-images.jianshu.io/upload_images/17317532-6b5aacdb5f954d92.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)




