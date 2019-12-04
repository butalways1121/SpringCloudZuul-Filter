# SpringCloudZuul-Filter
SpringCloudZuul-Filter
---
&emsp;&emsp;**上一篇博客介绍了[SpringCloud Zuul路由网关的基本使用](https://butalways1121.gitee.io/2019/11/28/SpringCloud%E4%B9%8B%E8%B7%AF%E7%94%B1%E7%BD%91%E5%85%B3Zuul%E5%9F%BA%E7%A1%80%E4%BD%BF%E7%94%A8/)，本文的内容是SpringCloud（基于SpringBoot 2.X，Finchley.SR2版）中的路由网关过滤器Filter和异常处理的简单使用教程，戳[这里](https://github.com/butalways1121/SpringCloudZuul-Filter)下载源码。**
<!--more-->
## SpringCloud Zuul Filter
### Zuul的核心
&emsp;&emsp;Filter是Zuul的核心，用来实现对外服务的控制，能够在HTTP请求和响应的路由过程中执行一系列操作。Filter的生命周期有4个，分别是“PRE”、“ROUTING”、“POST”、“ERROR”，整个生命周期可以用下图来表示：
![](https://raw.githubusercontent.com/butalways1121/img-Blog/master/105.png)
* PRE：这种过滤器在请求被路由之前调用。我们可利用这种过滤器实现身份验证、在集群中选择请求的微服务、记录调试信息等；
* ROUTING：这种过滤器将请求路由到微服务。这种过滤器用于构建发送给微服务的请求，并使用Apache HttpClient或Netfilx Ribbon请求微服务；
* POST：这种过滤器在路由到微服务以后执行。这种过滤器可用来为响应添加标准的HTTP Header、收集统计信息和指标、将响应从微服务发送给客户端等；
* ERROR：在其他阶段发生错误时执行该过滤器。


### Zuul中默认实现的Filter
| 类型 | 顺序 | 过滤器 | 功能 |
|:-------:|:-------:|:-------:|:-------:|
| pre | -3 | ServletDetectionFilter | 标记处理Servlet的类型 |
| pre | -2 | Servlet30WrapperFilter | 包装HttpServletRequest请求 |
| pre |1 | FormBodyWrapperFilter | 包装请求体 |
| route | 1 | DebugFilter | 标记调试标志 |
| route | 5 | PreDecorationFilter | 处理请求上下文供后续使用 |
| route | 10 | RibbonRoutingFilter | serviceId请求转发 |
| route | 100 | SimpleHostRoutingFilter | url请求转发 |
| route | 100 | SendForwardFilter | forward请求转发 |
| post | 0 | SendErrorFilter | 处理有错误的请求响应 |
| post | 1000 | SendResponseFilter | 处理正常的请求响应 |


## SpringCloud Zuul Filter示例
&emsp;&emsp;由于在上一篇中我们已经完成了Zuul路由网关的基本功能实现，所以这里直接在之前的项目基础上修改即可。
### 1.注册中心
application.properties文件中的`spring.application.name`配置为`springcloud-zuul-filter-eureka`即可。
### 2.服务端
application.properties文件中的`spring.application.name`配置为`springcloud-zuul-filter-gateway`，接着，添加配置`zuul.SendErrorFilter.error.disable=true`，表示禁用SendErrorFilter过滤器，否则无法使用自定义的异常过滤器的。

> 这里在说一下禁用过滤器的规则：组件实现的过滤器，满足执行条件时都是会执行的，若我们想禁用某个过滤器时，可以在配置文件中配置，配置规则：`zuul.<SimpleClassName>.<filterType>.disable=true`，其中，`SimpleClassName`为类名，`filterType`过滤器类型.


然后自定义一个Filter过滤器，在服务网关中自定义过滤器只需要继承`ZuulFilter`抽象类，然后实现其定义的四个抽象函数就可对请求进行拦截与过滤，代码示例如下，注意这里需要使用`Bean`注解使该过滤类生效：
```bash
@Component
public class MyZuulFilter extends ZuulFilter{

	//shouldFilter()方法表示是否执行该过滤器，也可以说是该过滤器的一个开关
	@Override
	public boolean shouldFilter() {
		//此方法可以根据请求的url进行判断是否需要拦截
		return true;
	}

	//run()方法是过滤器的具体逻辑。在该函数中，我们可以实现自定义的过滤逻辑，来确定是否要拦截当前的请求，不对其进行后续的路由，或是在请求路由返回结果之后，对处理结果做一些加工等
	//获取请求的url，如果该url携带了token，我们就让他通过，否则直接拦截
	@Override
	public Object run() throws ZuulException {
		//获取请求的上下文类 注意是：com.netflix.zuul.context包下的
		RequestContext ctx = RequestContext.getCurrentContext();
		HttpServletRequest request = ctx.getRequest();
		ctx.addZuulResponseHeader("Content-type", "text/json;charset=UTF-8");
		ctx.getResponse().setCharacterEncoding("UTF-8");
		System.out.println("请求地址:"+request.getRequestURI());
		String token = request.getParameter("token");
		String msg="请求成功!";
		if(token==null) {
			//使其不进行转发
		   ctx.setSendZuulResponse(false);
		   msg="请求失败！原因是token为空！";
		   ctx.setResponseBody(msg);
		   ctx.setResponseStatusCode(HttpStatus.UNAUTHORIZED.value());
		   //或者添加一个额外参数也可以 传递参数可以使用
//		   ctx.set("checkAuth",false);
		}
		System.out.println(msg);
		return msg;
	}

	//设置为前置过滤器，在请求被路由之前调用
	@Override
	public String filterType() {
		//前置过滤器
		return FilterConstants.PRE_TYPE;
	}

	@Override
	public int filterOrder() {
		//执行顺序  0 靠前执行 在spring cloud zuul提供的pre过滤器之后执行，默认的是小于0的。	
		//除了参数校验类的过滤器 一般上直接放在 PreDecoration前
		//即：PRE_DECORATION_FILTER_ORDER - 1;
		//常量类都在：org.springframework.cloud.netflix.zuul.filters.support.FilterConstants 下
		return 0;
	}
	
	//将该过滤类使用Bean注解使其生效
	@Bean
	public MyZuulFilter zuulFilter() {
	    return new MyZuulFilter();
	}

}
```


接着，再定义一个异常处理类，以便在使用Zuul发生异常时调用来对异常进行处理。同样，自定义异常处理也只需继承`SendErrorFilter`类并实现相应的方法即可，示例代码如下：
```bash
@Component
public class MyErrorFilter extends SendErrorFilter{

	@Override
	public Object run() {
		String msg="请求失败!";
		//重写 run方法		
		try{
			RequestContext ctx = RequestContext.getCurrentContext();
			//直接复用异常处理类
			ExceptionHolder exception = findZuulException(ctx.getThrowable());
			System.out.println("错误信息:"+exception.getErrorCause());
			msg+="error:"+exception.getErrorCause();
			System.out.println("msg:"+msg);

			HttpServletResponse response = ctx.getResponse();
			//设置编码
			response.setCharacterEncoding("UTF-8");
			response.getOutputStream().write(msg.getBytes());           	 
		}catch (Exception ex) {
			ex.printStackTrace();
			ReflectionUtils.rethrowRuntimeException(ex);
		}
		return msg;
	}
	
	/**
	 *  自定义异常过滤器
	 */
	@Bean
	//@ConditionalOnProperty(name="zuul.SendErrorFilter.error.disable")
	public MyErrorFilter errorFilter() {
	    return new MyErrorFilter();
	}
	

}
```


最后，再定义一个异常回退类用于某个服务无法访问时进行回退处理。这里的定义需要实现`FallbackProvider`接口中的方法，其中`getRoute()方法是指定需要回退服务的名称`，`fallbackResponse()方法是提供基于执行失败原因并进行回退响应`。另外，这里的 `private static final String SERVER_NAME="springcloud-zuul-filter-server2"`指定了当服务springcloud-zuul-filter-server2无法调用时才执行该异常回退处理：
```bash
@Component
public class MyFallback implements FallbackProvider {
    private static  final  String SERVER_NAME="springcloud-zuul-filter-server2";

    @Override
    public String getRoute() {
        return SERVER_NAME;
    }

    @Override
    public ClientHttpResponse fallbackResponse(String route, Throwable cause) {

        //标记不同的异常为不同的http状态值
        if (cause instanceof HystrixTimeoutException) {
            return response(HttpStatus.GATEWAY_TIMEOUT);
        } else {
            //可继续添加自定义异常类
            return response(HttpStatus.INTERNAL_SERVER_ERROR);
        }
    }
    //处理
    private ClientHttpResponse response(final HttpStatus status) {
        String msg="该"+SERVER_NAME+"服务暂时不可用!";
        return new ClientHttpResponse() {
            @Override
            public HttpStatus getStatusCode() throws IOException {
                return status;
            }

            @Override
            public int getRawStatusCode() throws IOException {
                return status.value();
            }

            @Override
            public String getStatusText() throws IOException {
                return status.getReasonPhrase();
            }

            @Override
            public void close() {
            }

            @Override
            public InputStream getBody() throws IOException {
                return new ByteArrayInputStream(msg.getBytes());
            }

            @Override
            public HttpHeaders getHeaders() {
                HttpHeaders headers = new HttpHeaders();
                headers.setContentType(MediaType.APPLICATION_JSON);
                return headers;
            }
        };
    }

    @Bean
    public MyFallback eurekaClientFallback() {
        return new MyFallback();
    }   
}
```
### 3.客户端
两个客户端服务无需大的修改，将application.properties文件中的服务名分别修改为`springcloud-zuul-filter-server1`和`springcloud-zuul-filter-server2`即可。
### 4.测试
完成项目修改之后，依次启动springcloud-zuul-filter-eureka、springcloud-zuul-filter-gateway、springcloud-zuul-filter-server1和springcloud-zuul-filter-server2这四个项目。其中8006是注册中心`springcloud-zuul-filter-eureka`的端口，9009是服务端`springcloud-zuul-filter-gateway`的端口，9010是第一个客户端`springcloud-zuul-filter-server1`的端口，9011是第二个客户端`springcloud-zuul-filter-server2`的端口。

**注：路由网关的默认规则是通过`http://ZUUL_HOST:ZUUL_PORT/微服务实例名(serverId)/**来转发值serviceId对应的微服务。例如，本项目中如果在浏览器输入：`http://localhost:9009/springcloud-zuul-filter-server1/hello`， 它就会跳转访问到：`http://localhost:9010/hello/`这个地址`。**
#### （1）自定义过滤器功能测试
首先，在浏览器中输入：`http://localhost:9009/springcloud-zuul-filter-server1/hello?name=butalways`，返回：
```
请求失败！原因是token为空!
```
再加上`token`之后进行访问，输入`http://localhost:9009/springcloud-zuul-filter-server1/hello/butalways?token=123`，返回：
```
butalways,Hello World!
```
以上结果说明完成了自定义过滤器的处理。
#### （2）自定义异常类处理功能测试
停止服务`springcloud-zuul-filter-server1`，在浏览器中输入：`http://localhost:9009/springcloud-zuul-filter-server1/hello/butalways?token=123`，返回：
```
请求失败!error:TIMEOUT请求失败!error:TIMEOUT
```
以上结果说明完成了自定义异常的处理。
#### （3）自定义异常回退处理功能测试
接着停止服务`springcloud-zuul-filter-server2`，在浏览器中输入：`http://localhost:9009/springcloud-zuul-filter-server2/hi?name=butalwaystoken=123`，返回：
```
该springcloud-zuul-filter-server2服务暂时不可用!
```
以上结果说明实现了自定义异常回退处理。

**注：自定义的异常处理和异常回退处理最好不要针对同一个服务同时使用，如果同时使用，会优先进行自定义异常回退处理的处理。**
