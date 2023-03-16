# 一、和Middleware关系
## 1. 联系
实际上asp.net提供的filter底层都是使用中间件来实现的
例如AuthenticationFilter权限过滤器：
```
namespace Microsoft.AspNetCore.Authentication;

public class AuthenticationMiddleware
{
    private readonly RequestDelegate _next;

    public AuthenticationMiddleware(RequestDelegate next, IAuthenticationSchemeProvider schemes)
    {
        ArgumentNullException.ThrowIfNull(next);
        ArgumentNullException.ThrowIfNull(schemes);

        _next = next;
        Schemes = schemes;
    }

    public IAuthenticationSchemeProvider Schemes { get; set; }

    public async Task Invoke(HttpContext context)
    {
        context.Features.Set<IAuthenticationFeature>(new AuthenticationFeature
        {
            OriginalPath = context.Request.Path,
            OriginalPathBase = context.Request.PathBase
        });

        var handlers = context.RequestServices.GetRequiredService<IAuthenticationHandlerProvider>();
        foreach (var scheme in await Schemes.GetRequestHandlerSchemesAsync())
        {
            var handler = await handlers.GetHandlerAsync(context, scheme.Name) as IAuthenticationRequestHandler;
            if (handler != null && await handler.HandleRequestAsync())
            {
                return;
            }
        }

        var defaultAuthenticate = await Schemes.GetDefaultAuthenticateSchemeAsync();
        if (defaultAuthenticate != null)
        {
            var result = await context.AuthenticateAsync(defaultAuthenticate.Name);
            if (result?.Principal != null)
            {
                context.User = result.Principal;
            }
            if (result?.Succeeded ?? false)
            {
                var authFeatures = new AuthenticationFeatures(result);
                context.Features.Set<IHttpAuthenticationFeature>(authFeatures);
                context.Features.Set<IAuthenticateResultFeature>(authFeatures);
            }
        }

        await _next(context);
    }
}

```
## 2. 区别
1. 中间件是基于委托实现的，它可以形成一个递归的处理流程，将HttpContext对象传递给下一个中间件，从而实现全局处理。而Filter是基于特性实现的，只能对指定的Action或Controller进行过滤处理，无法对全局进行处理。

2. 中间件可以在请求处理过程的任何阶段执行操作，如请求处理前、后或异常处理时执行一些操作，从而实现全局处理。而Filter只能在Action或Controller执行前、后或异常处理时执行操作，无法对请求处理过程的其他阶段进行操作。

3. 中间件可以修改HttpContext对象，如修改请求头、响应头等信息，从而实现全局处理。而Filter不能修改HttpContext对象，只能读取和操作请求和响应，无法实现全局处理。
## 3. 使用场景
1. 中间件适用于处理一系列操作，并将HttpContext对象传递给下一个中间件，如身份认证、授权、请求响应缓存等。

2. Filter适用于在请求处理过程中执行一些操作，如请求验证、身份验证、日志记录等。

3. 中间件和Filter都可以用于请求处理前、后或异常处理时执行一些操作，如请求计时、异常处理、响应压缩等。
# 二、工作原理
## Asp.Net可分两条管道，一条是中间件的管道，另一条就是Filter的管道
![流程](https://upload-images.jianshu.io/upload_images/20387877-5d3127d7494fa3c5.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

## Filter 类型
*   [授权筛选器AuthorizationFilters](https://learn.microsoft.com/zh-cn/aspnet/core/mvc/controllers/filters?view=aspnetcore-6.0#authorization-filters)：

    *   首先运行。
    *   确定用户是否获得请求授权。
    *   如果请求未获授权，可以让管道短路。

*   [资源筛选器ResourceFilters](https://learn.microsoft.com/zh-cn/aspnet/core/mvc/controllers/filters?view=aspnetcore-6.0#resource-filters)：

    *   授权后运行。
    *   [OnResourceExecuting](https://learn.microsoft.com/zh-cn/dotnet/api/microsoft.aspnetcore.mvc.filters.iresourcefilter.onresourceexecuting) 在筛选器管道的其余阶段之前运行代码。 例如，`OnResourceExecuting` 在模型绑定之前运行代码。
    *   [OnResourceExecuted](https://learn.microsoft.com/zh-cn/dotnet/api/microsoft.aspnetcore.mvc.filters.iresourcefilter.onresourceexecuted) 在管道的其余阶段完成之后运行代码。

*   [操作筛选器ActionFilters](https://learn.microsoft.com/zh-cn/aspnet/core/mvc/controllers/filters?view=aspnetcore-6.0#action-filters)：

    *   在调用操作方法之前和之后立即运行。
    *   可以更改传递到操作中的参数。
    *   可以更改从操作返回的结果。
    *   不可在 Razor Pages 中使用。

*   [异常筛选器ExceptionFilters](https://learn.microsoft.com/zh-cn/aspnet/core/mvc/controllers/filters?view=aspnetcore-6.0#exception-filters)在向响应正文写入任何内容之前，对未经处理的异常应用全局策略。

*   [结果筛选器ResultFilters](https://learn.microsoft.com/zh-cn/aspnet/core/mvc/controllers/filters?view=aspnetcore-6.0#result-filters)：

    *   在执行操作结果之前和之后立即运行。
    *   仅当操作方法成功执行时才会运行。
    *   对于必须围绕视图或格式化程序的执行的逻辑，会很有用。

![filter管道顺序](https://upload-images.jianshu.io/upload_images/20387877-42f9e3917d3f81a0.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

# 三、注册方式
## 1.  Action 注册方式
Action 注册方式是局部注册方式，针对控制器中的某个方法上标注特性的方式进行注册，代码如下：
```
 [AuthonizationFilter()]
 public IActionResult Index()
 {
     return View();
 }
```
## 2. Controller 注册方式
对整个Controller都进行注册，代码如下：
```
 [AuthonizationFilter()]
 public class FirstController : Controller
  {
        private ILogger<FirstController> _logger;

        public FirstController(ILogger<FirstController> logger)
        {
            _logger = logger;
        }
 }
```
## 3. 全局注册方式
比如要全局处理系统中的异常，或者收集操作日志等，需要全局注册一个ExceptionFilter 来实现，就不需要每一个Controller 中进行代码注册，方便快捷。代码如下：
```
 public void ConfigureServices(IServiceCollection services)
  {
            //全局注册异常过滤器
            services.AddControllersWithViews(option=> {
                option.Filters.Add<ExecptionFilter>();
            });

            services.AddSingleton<ISingletonService, SingletonService>();
}
```

## 4. TypeFilter 和 ServiceFilter 注册方式
上面的五大过滤器中事例代码中其中有一个过滤器的代码比较特，再来回顾ExceptionFilter过滤器的实现代码：
```
    public class ExecptionFilter : Attribute, IExceptionFilter
    {
        private ILogger<ExecptionFilter> _logger;
        //构造注入日志组件
        public ExecptionFilter(ILogger<ExecptionFilter> logger)
        {
            _logger = logger;
        }

        public void OnException(ExceptionContext context)
        {
            //日志收集
            _logger.LogError(context.Exception, context?.Exception?.Message??"异常");
        }
    }
```
从上面的代码中可以发现 ExceptionFilter 过滤器实现中存在日志服务的构造函数的注入，也就是说该过滤器依赖于其他的日志服务，但是日志服务都是通过DI 注入进来的；再来回顾下上面Action 注册方式或者Controller 注册方式 也即Attribute 特性标注注册方式，本身基础的特性是不支持构造函数的，是在运行时注册进来的，那要解决这种本身需要对服务依赖的过滤器需要使用 TypeFilter 或者ServiceFilter 方式进行过滤器的标注注册。

### TypeFilter 和ServiceFilter 的区别

1. ServiceFilter和TypeFilter都实现了IFilterFactory

2. ServiceFilter需要对自定义的Filter进行注册，TypeFilter不需要

3. ServiceFilter的Filter生命周期源自于您如何注册，而TypeFilter每次都会创建一个新的实例

TypeFilter 使用方式

代码如下：
```
[TypeFilter(typeof(ExecptionFilter))]
public IActionFilter Index2()
{
      return View();
}
```
通过上面的代码可以发现AuthonizationFilter 是默认的构造器，但是如果过滤器中构造函数中存在参数，需要注入服务，比如上面的ExceptionFilter 代码，就不能使用这种方式进行注册，需要使用服务特性的方式，我们可以选择使用 代码如下：
```
[TypeFilter(typeof(ExecptionFilter))]
public IActionFilter Index2()
{
     return View();
}
```
ServiceFilter 使用方式

控制器中的代码如下：
```
[ServiceFilter(typeof(ExecptionFilter))]
public IActionFilter Index2()
{
    return View();
}
```
注册服务的代码如下：
```
public void ConfigureServices(IServiceCollection services)
{
       Console.WriteLine("ConfigureServices");
       services.AddControllersWithViews();

       //services.AddControllersWithViews(option=> {
       //    option.Filters.Add<ExecptionFilter>();
       //});
        
        //注册过滤器服务，使用ServiceFilter 方式必须要注册 否则会报没有注册该服务的相关异常
        services.AddSingleton<ExecptionFilter>();
}
```
