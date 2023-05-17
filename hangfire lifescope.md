# 一、背景
在使用hangfire的Enqueue接口把任务放到后台执行时发现并没有执行到实际任务的内容，而且没有报错。
大概是这样写的:

```
//_xxxService是在一开始用Autofac注入，并在构造函数中构造过
BackgroundJob.Enqueue(() => _xxxService.xxxMethod());
```

调用之后，hangfire的dashboard上面也有显示该延迟Job，但并没执行实际任务。

# 二、解决方法

根据同事的意见改变写法：

```
BackgroundJob.Enqueue<IxxxService>(x => x.xxxMethod());
```

按照上面的写法就能正常执行实际任务。

## 1. 主要原因： 生命周期问题
在调用Hangfire把任务放到后台的接口，像Enqueue、RecurringJob.AddOrUpdate等接口，Hangfire会自己启用一个生命周期来执行这些接口中实际任务的逻辑。
而在一开始获取Autofac容器中`IxxxService`的实例进不了Hangfire自己创建的生命周期导致没有执行到实际方法。

## 2. 详细流程
hangfire在执行接口中实际任务时：

* 首先用的是依赖`BackgroundProcessingServer`的`BackgroundProcessDispatcherBuilder`中的`Create`方法
*  其中的`ExecuteProcess`会调用`Worker类`的`Execute`方法
* `Execute`会先把Job的状态变成处理状态中，然后执行`PerformJob`方法中`IBackgroundJobPerformer`的`Perform`方法来调用实际任务
* `Perform`中首先用 `JobActivator`来启用一个生命周期，然后确定这个Job的方法是不是静态的，确认不是后要实例化一个，调用`InvokeMethod`来执行Job，即实际任务

 ```
public object Perform(PerformContext context)
{
    using (var scope = _activator.BeginScope(context))
    {
        object instance = null;

        if (context.BackgroundJob.Job == null)
        {
            throw new InvalidOperationException("Can't perform a background job with a null job.");
        }
        //context.BackgroundJob.Job.Method就是需要执行的方法
        if (!context.BackgroundJob.Job.Method.IsStatic)
        {
            instance = scope.Resolve(context.BackgroundJob.Job.Type);

            if (instance == null)
            {
                throw new InvalidOperationException(
                    $"JobActivator returned NULL instance of the '{context.BackgroundJob.Job.Type}' type.");
            }
        }

        var arguments = SubstituteArguments(context);
        //把方法的实例拿去执行
        var result = InvokeMethod(context, instance, arguments);

        return result;
    }
}
```
* 在Core版本中的`InvokeMethod`会通过反射根据Job是否要异步执行来用不同的方式来执行任务（hangfire还有一个.net framework的版本，framework的版本就不会给你判断用不用异步）

```
private object InvokeMethod(PerformContext context, object instance, object[] arguments)
{
    var methodInfo = context.BackgroundJob.Job.Method;
    var tuple = Tuple.Create(methodInfo, instance, arguments);
    var returnType = methodInfo.ReturnType;
    //判断这个方法需不需要异步执行
    if (returnType.IsTaskLike(out var getTaskFunc))
    {
        if (_taskScheduler != null)
        {
            return InvokeOnTaskScheduler(context, tuple, getTaskFunc);
        }

        return InvokeOnTaskPump(context, tuple, getTaskFunc);
    }

    return InvokeSynchronously(tuple);
}
```

## 3. 另一种模仿hangfire的解决方案
```
using (var scope = container.BeginLifetimeScope())
{
    // 解析 IxxxService 实例
    var xxxService = scope.Resolve<IxxxService>();

    // Enqueue 后台任务
    BackgroundJob.Enqueue(() => xxxService.xxxMethod());
}
```

