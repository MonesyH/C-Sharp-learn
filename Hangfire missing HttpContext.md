# 一、背景
  我在把一系列任务丢进hangfire队列当中时发现有几个Job没有跑起来，然后我先去Hangfire的dashboard看看报错：
![父任务底下的子任务报错](https://upload-images.jianshu.io/upload_images/20387877-8a9b91e244e0220d.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
![报错信息](https://upload-images.jianshu.io/upload_images/20387877-92b5fe83c290b8b9.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

看到在执行到后面的任务时抛出: ` HttpContext is not available` 这个异常。

在代码中定位到在获取当前用户的时候调用到`IHttpContextAccessor`时，无法获取到http的上下文，导致报异常。

在Hangfire官方文档其实也有提到过这个问题：

https://docs.hangfire.io/en/latest/background-methods/using-ioc-containers.html

![官方文档](https://i.postimg.cc/RhmCmkKb/hangfire.png)

# 二、解决思路
## 实际需要解决的问题：在保存执行Job操作的用户时，无法获取到当前用户。

## 思路： 
* 思路一：在进入任务队列之前，把获取到当前用户（通过HttpContextAccessor获取的当前用户）复制到另一个单例接口，然后在实际任务逻辑中判断HttpContextAccessor是否拿得到，拿不到就用复制好的单例接口获取当前用户；（不采用❌）
* 思路二：在进入任务队列之前，把获取到当前用户（通过HttpContextAccessor获取的当前用户）赋值到一个全局的静态变量当中，然后在实际任务逻辑中判断HttpContextAccessor是否拿得到，拿不到就用赋值过的全局静态变量作为当前用户；（不采用❌）
* 思路三：定义一个内部成员以及加上一个切换器，当出现这些需要当前用户Id的Hangfire任务时，切换器切换到`Hangfire`的模式，这个`Hangfire`的模式就是在之前获取当前用户的逻辑中（HttpContextAccessor）切换成获取写死的内部成员。（采用✅）

思路一和二其实思想都差不多，也能解决这个问题，但相较于第三个思路来说，思路三更加**Generic**，而且没有动到主要的逻辑。

# 三、具体解决

## 1. 用DbUp往数据库中插入一条写死的内部用户数据

具体代码这里不展示，会涉及到代码逻辑，不过这个可以看看https://github.com/MonesyH/C-Sharp-learn/blob/main/DbUp%20Code-based%20Script.md

## 2. 定义一个切换器

```
public interface ICurrentUserSwitcher : ISingletonDependency
{
    CurrentUserSource CurrentUserSource { get; set; }
}

public class CurrentUserSwitcher : ICurrentUserSwitcher
{
    public CurrentUserSource CurrentUserSource { get; set; } = CurrentUserSource.Api;
}

public enum CurrentUserSource
{
    Api,
    Hangfire
}
```

## 3. 在获取当前用户的接口中注入切换器，按照切换器的状态选择获取当前用户的方式

```
public interface ICurrentUser
{
    Guid Id { get; }
}

public class CurrentUser : ICurrentUser
{
    private readonly IHttpContextAccessor _httpContextAccessor;
    private readonly ICurrentUserSwitcher _currentUserSwitcher;

    public CurrentUser(
        IHttpContextAccessor httpContextAccessor, 
        ICurrentUserSwitcher currentUserSwitcher)
    {
        _httpContextAccessor = httpContextAccessor;
        _currentUserSwitcher = currentUserSwitcher;
    }

    public Guid Id
    {
        get
        {
            return _currentUserSwitcher.CurrentUserSource switch
            {
                CurrentUserSource.Api => _httpContextAccessor.HttpContext == null 
                    ? throw new ApplicationException("HttpContext is not available") 
                    : //这里在HttpContext中提取之前登陆存的用户Id,
                CurrentUserSource.Hangfire => CurrentUsers.InternalUser.Id,
                _ => throw new NotSupportedException(nameof(_currentUserSwitcher.CurrentUserSource))
            };
        }
    }
}
```

## 4. 在加入任务之前切换`Hangfire`模式，任务结束之后切换成`Api`模式

### 加入任务之前切换`Hangfire`模式：
```
public class XxxJob
{
    private readonly ICurrentUserSwitcher _currentUserSwitcher;
    private readonly IXxxBackgroundJobClient _xxxBackgroundJobClient;

    public XxxJobr(ICurrentUserSwitcher currentUserSwitcher, IXxxBackgroundJobClient xxxBackgroundJobClient)
    {
        _currentUserSwitcher = currentUserSwitcher;
        _xxxBackgroundJobClient = xxxBackgroundJobClient;
    }

    public async Task Handle()
    {
        //加入队列之前切换成Hangfire的模式：
         _currentUserSwitcher.CurrentUserSource = CurrentUserSource.Hangfire;
            
        //任务加入队列：
        _xxxBackgroundJobClient.Enqueue<T>(x => x.XxxServiceAsync());
    }
}
```

### 任务结束之后切换成`Api`模式

```
public async Task XxxServiceAsync()
{
    // ... 加入任务队列操作，返回一个parentJobId
   // 执行完所有队列任务之后
     _XxxBackgroundJobClient.ContinueJobWith(parentJobId, () => SwitchCurrentUserSourceToApi());
}

public async Task SwitchCurrentUserSourceToApi()
{
    _currentUserSwitcher.CurrentUserSource = CurrentUserSource.Api;
    
    await Task.CompletedTask;
}
```
