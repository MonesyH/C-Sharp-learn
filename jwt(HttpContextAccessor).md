# 一、 为什么需要HttpContextAccessor?
我们在需要授权的控制器中打上[Authorize]授权标签，就可以在ControllerBase的User属性获取到基于声明的权限标识(ClaimsPrincipal)。遗憾的是这只是针对Controller层面，很多场景下我们是需要在Service层乃至数据层获直接使用用户信息，这种情况我们就使用不了User了。

[官方文档](https://learn.microsoft.com/en-us/dotnet/api/microsoft.aspnetcore.http.httpcontextaccessor?source=recommendations&view=aspnetcore-5.0)

HttpContextAccessor源码：
```
public class HttpContextAccessor : IHttpContextAccessor
{
    private static readonly AsyncLocal<HttpContextHolder> _httpContextCurrent = new AsyncLocal<HttpContextHolder>();

    /// <inheritdoc/>
    public HttpContext? HttpContext
    {
        get
        {
            return _httpContextCurrent.Value?.Context;
        }
        set
        {
            var holder = _httpContextCurrent.Value;
            if (holder != null)
            {
                // Clear current HttpContext trapped in the AsyncLocals, as its done.
                holder.Context = null;
            }

            if (value != null)
            {
                // Use an object indirection to hold the HttpContext in the AsyncLocal,
                // so it can be cleared in all ExecutionContexts when its cleared.
                _httpContextCurrent.Value = new HttpContextHolder { Context = value };
            }
        }
    }

    private sealed class HttpContextHolder
    {
        public HttpContext? Context;
    }
}
```
这段代码用于在异步上下文中获取和设置当前的HTTP请求的上下文。具体来说，它使用了AsyncLocal<T>类，该类提供了一种线程本地存储（Thread Local Storage）的方法，它可以在异步调用栈中跟踪上下文信息，而不会受到线程切换的影响。

在类HttpContextAccessor中，HttpContext属性定义了一个可读可写的访问器。当读取该属性时，它返回HttpContextHolder中的Context属性。
当设置该属性时，它首先从_httpContextCurrent.Value获取HttpContextHolder实例，然后将其Context属性设置为null。
接着，如果传入的value不为null，则创建一个新的HttpContextHolder对象，并将其Context属性设置为传入的value。最
后，将新创建的HttpContextHolder对象保存到_httpContextCurrent.Value中。

这样，当请求处理程序需要访问当前HTTP上下文时，可以使用HttpContextAccessor.HttpContext属性来获取。这个属性的值是线程本地的，每个线程都有自己的值，不会受到其他线程的干扰。同时，由于它使用了AsyncLocal<T>，所以它也能正确地处理异步代码中的上下文切换。
# 二、解决办法
Aspnet Core是通过注入HttpContext的访问器对象IHttpContextAccessor来获取当前的HttpContext。
1. 在Startup的ConfigureServices方法中注册IHttpContextAccessor
```
public void ConfigureServices(IServiceCollection services)
{
    //...
    services.AddHttpContextAccessor();
}
```
2. 编写CurrentUser来记录当前登陆用户的信息
```
public interface ICurrentUser
{
    public int UserId { get; }
}

public class CurrentUser : ICurrentUser
{
    private readonly IHttpContextAccessor _httpContextAccessor;

    public CurrentUser(IHttpContextAccessor httpContextAccessor)
    {
        _httpContextAccessor = httpContextAccessor;
    }
    
    int ICurrentUser.UserId => int.Parse(_httpContextAccessor.HttpContext!.User.FindFirst(ClaimTypes.NameIdentifier)!.Value);
}
```
3. 在startup.cs或自定义extension中注册CurrentUser
```
services.AddScoped<ICurrentUser, CurrentUser>();
```
