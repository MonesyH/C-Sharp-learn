# 一、中间件的前身：HttpHandler和module
![以前的工作方式](https://upload-images.jianshu.io/upload_images/20387877-b1060e43b5accf19.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

* **module**：负责对传入的 HTTP 请求进行预处理，并将请求传递给下一个处理程序。访问模块可以执行各种操作，例如身份验证、授权、缓存、日志记录等等。ASP.NET 中的许多模块都是访问模块，例如 FormsAuthenticationModule、WindowsAuthenticationModule、OutputCacheModule、UrlRoutingModule 等等。

* **HttpHandler**：负责处理传入的 HTTP 请求，并生成响应。处理程序可以执行各种操作，例如读取和写入数据、调用服务、生成视图等等。ASP.NET 中的许多处理程序都是控制器和视图，例如 MvcHandler、WebApiHandler、RazorView、WebFormsView 等等。

# 二、如今HttpHandler和module摇身一变，同时变成中间件
![如今的工作方式](https://upload-images.jianshu.io/upload_images/20387877-3c317a763744ef82.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

## 主要变化：

1. 实现IModule和IHttpHandler的代码迁移到中间件上
```
namespace MyApp.Modules
{
    public class MyModule : IHttpModule
    {
        public void Dispose()
        {
        }

        public void Init(HttpApplication application)
        {
            application.BeginRequest += (new EventHandler(this.Application_BeginRequest));
            application.EndRequest += (new EventHandler(this.Application_EndRequest));
        }

        private void Application_BeginRequest(Object source, EventArgs e)
        {
            HttpContext context = ((HttpApplication)source).Context;

            // Do something with context near the beginning of request processing.
        }

        private void Application_EndRequest(Object source, EventArgs e)
        {
            HttpContext context = ((HttpApplication)source).Context;

            // Do something with context near the end of request processing.
        }
    }
}
```
[换成👇](#)
```
namespace MyApp.Middleware
{
    public class MyMiddleware
    {
        private readonly RequestDelegate _next;

        public MyMiddleware(RequestDelegate next)
        {
            _next = next;
        }

        public async Task Invoke(HttpContext context)
        {
            // Do something with context near the beginning of request processing.

            await _next.Invoke(context);

            // Clean up.
        }
    }

    public static class MyMiddlewareExtensions
    {
        public static IApplicationBuilder UseMyMiddleware(this IApplicationBuilder builder)
        {
            return builder.UseMiddleware<MyMiddleware>();
        }
    }
}
```
2. IModule和IHttpHandler的配置改成把中间件放进管道

```
<?xml version="1.0" encoding="utf-8"?>
<!--ASP.NET 4 web.config-->
<configuration>
  <system.webServer>
    <modules>
      <add name="MyModule" type="MyApp.Modules.MyModule"/>
    </modules>
  </system.webServer>
</configuration>
```
[换成👇](#)
```
 app.UseMyMiddleware();
```

# 三、中间件管道
## 1. 中间件管道
通过用委托的invoke实现图中递归链的效果
![中间件管道](https://upload-images.jianshu.io/upload_images/20387877-caefd428766bda1a.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


## 2. 管道顺序
![管道顺序](https://upload-images.jianshu.io/upload_images/20387877-969dc6c73464f683.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
推荐顺序：
```
    app.UseHttpsRedirection();
    app.UseStaticFiles();
    // app.UseCookiePolicy();

    app.UseRouting();
    // app.UseRequestLocalization();
    // app.UseCors();

    app.UseAuthentication();
    app.UseAuthorization();
    // app.UseSession();
    // app.UseResponseCompression();
    // app.UseResponseCaching();

    app.UseEndpoints(endpoints =>
        {
            endpoints.MapControllers();
        });
```

