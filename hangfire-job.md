# 一、hangfire概述

## hangfire如何实现
Hangfire 的定时任务并不是使用 Timer 类来实现的。Hangfire 是一个用于在 .NET 应用程序中执行后台作业和定时任务的库，其核心原理是基于持久化存储（如 SQL Server、Redis 等）和轮询（Polling）实现的。

具体来说，Hangfire 会将需要执行的后台作业和定时任务存储到持久化存储中，然后使用轮询来检查这些任务是否需要执行。这种方式与传统的定时器实现方式有所不同，因为它能够确保在应用程序重启或崩溃后，Hangfire 仍然能够继续执行尚未完成的任务。

当然，Hangfire 库中也有与定时器相关的 API，比如 Cron 表达式等，可以用于配置定时任务的执行时间。但是，这些 API 只是为了方便用户配置定时任务的执行时间，实际上它们并不是 Hangfire 库的核心实现。

## hangfire的轮询
Hangfire 的轮询实现是比较复杂的，涉及到多个线程、队列、定时器等多个组件。下面是一个大致的流程，介绍了 Hangfire 源码中是如何实现轮询的：

1. Hangfire 启动时，会创建一个 BackgroundJobServer 实例，并调用其 StartAsync 方法。

2. StartAsync 方法会创建一个 BackgroundProcessingServer 实例，并启动一个轮询线程和一组工作线程.
```
public Task StartAsync(CancellationToken cancellationToken)
{
    {
        InitializeProcessingServer();                
    }

    return Task.CompletedTask;
}

private void InitializeProcessingServer()
{
    _processingServer = _factory != null && _performer != null && _stateChanger != null
        ? new BackgroundJobServer(_options, _storage, _additionalProcesses, null, null, _factory, _performer,
            _stateChanger)
        : new BackgroundJobServer(_options, _storage, _additionalProcesses);
}
```
3. 轮询逻辑会定时查询存储中的任务信息，检查哪些任务需要执行。查询的间隔默认为 15 秒，可以通过配置文件进行调整。
```
public void ConfigureServices(IServiceCollection services)
{
    services.Configure<DashboardOptions>(options =>
    {
        options.QueuePollInterval = TimeSpan.FromSeconds(30);
    });
    services.AddHangfireServer();
}

```
4. 如果发现有需要执行的任务，轮询逻辑会将任务信息加入到一个队列中。

5. 另外一个线程池中的线程会从队列中取出任务信息，并执行任务。这些线程被称为 Worker 线程。

6. Worker 线程会根据任务类型，创建相应的实例，并执行任务。任务执行的过程中，Worker 线程会不断地向存储中写入任务的执行状态信息。

7. 如果任务执行成功，Worker 线程会从队列中删除该任务。如果任务执行失败，Hangfire 会根据配置文件中的重试策略，尝试重新执行该任务。

8. Hangfire 的轮询逻辑和 Worker 线程会一直运行，直到应用程序停止或被关闭。

# 二、 核心类
## 1. Job类和Background类
* Job类表示一个可执行的工作单元，其中包含工作单元的类型、方法名称和参数等信息。在Hangfire中，Job类是通过一个Job类别来表示的，其中包含一个可执行的函数和其参数。
```
public class Job
{
    public string Id { get; set; }
    public Type Type { get; set; }
    public string Method { get; set; }
    public object[] Args { get; set; }
    public IDictionary<string, string> StateData { get; set; }
    public DateTimeOffset CreatedAt { get; set; }
    public TimeSpan ExpireIn { get; set; }
}
```
BackgroundJob类是对Job类的一个包装，它提供了对Job类的更高级别的封装，以方便使用和管理。
其中提供各式各样的Enqueue和Schedule方法，还包含了Job类的ID、状态、进度和执行结果等信息。
```
public static string Enqueue([NotNull, InstantHandle] Expression<Action> methodCall)
{
    return ClientFactory().Enqueue(methodCall);
}

public static string Enqueue([NotNull] string queue, [NotNull, InstantHandle] Expression<Action> methodCall)
{
    return ClientFactory().Enqueue(queue, methodCall);
}

public static string Enqueue([NotNull, InstantHandle] Expression<Func<Task>> methodCall)
{
    return ClientFactory().Enqueue(methodCall);
}

public static string Enqueue([NotNull] string queue, [NotNull, InstantHandle] Expression<Func<Task>> methodCall)
{
    return ClientFactory().Enqueue(queue, methodCall);
}

public static string Enqueue<T>([NotNull, InstantHandle] Expression<Action<T>> methodCall)
{
    return ClientFactory().Enqueue(methodCall);
}

...

public static string Schedule(
    [NotNull, InstantHandle] Expression<Func<Task>> methodCall,
    TimeSpan delay)
{
    return ClientFactory().Schedule(methodCall, delay);
}

public static string Schedule(
    [NotNull] string queue,
    [NotNull, InstantHandle] Expression<Func<Task>> methodCall,
    TimeSpan delay)
{
    return ClientFactory().Schedule(queue, methodCall, delay);
}

public static string Schedule(
    [NotNull, InstantHandle] Expression<Action> methodCall, 
    DateTimeOffset enqueueAt)
{
    return ClientFactory().Schedule(methodCall, enqueueAt);
}

public static string Schedule(
    [NotNull] string queue,
    [NotNull, InstantHandle] Expression<Func<Task>> methodCall,
    DateTimeOffset enqueueAt)
{
    return ClientFactory().Schedule(queue, methodCall, enqueueAt);
}

public static string Schedule<T>(
    [NotNull, InstantHandle] Expression<Action<T>> methodCall,
    TimeSpan delay)
{
    return ClientFactory().Schedule(methodCall, delay);
}
```
## 2. JobStorage类
JobStorage类是Hangfire中存储后台作业信息的核心组件。它提供了一种机制来存储和检索作业的信息，包括作业的状态、进度和执行结果等。

JobStorage类是一个抽象类，定义了一组接口来访问后台作业信息的存储和检索。具体的存储和检索机制可以由不同的存储提供程序来实现。
```
public abstract IMonitoringApi GetMonitoringApi();

public abstract IStorageConnection GetConnection();
```

## 3. Server类
Server类是Hangfire中执行后台调度的核心组件。它不断地从JobStorage中读取作业信息，并在调度时间到达时执行作业。Server类提供了一个可扩展的机制，使得可以使用不同的作业处理器来执行不同类型的作业。

Server类的实现基于一个无限循环，不断地从JobStorage中获取作业并执行。每个作业的执行过程是在一个独立的线程中进行的，以避免阻塞Server的主循环。

# 三 、拓展：Lazy<T>延迟初始化
在BackgroundJob中第一行看到
```
private static readonly Lazy<IBackgroundJobClient> CachedClient 
            = new Lazy<IBackgroundJobClient>(() => new BackgroundJobClient(), LazyThreadSafetyMode.PublicationOnly); 
```
所以顺便看了。

对象的惰性初始化意味着它的创建被推迟到第一次使用时。惰性初始化主要用于提高性能、避免无用计算和减少程序内存需求。这些是最常见的场景：

*   当您有一个创建成本很高的对象时，程序可能不会使用它。例如，假设您在内存中有一个`Customer`对象，该`Orders`对象的属性包含大量`Order`对象，要初始化这些对象需要数据库连接。如果用户从不要求显示订单或在计算中使用数据，则没有理由使用系统内存或计算周期来创建它。通过使用`Lazy<Orders>`声明`Orders`延迟初始化的对象，可以避免在不使用该对象时浪费系统资源。

*   当您有一个创建起来代价高昂的对象时，您希望将其创建推迟到其他代价高昂的操作完成之后。例如，假设您的程序在启动时加载了多个对象实例，但只有其中一些是立即需要的。您可以通过将不需要的对象的初始化推迟到创建所需的对象之后来提高程序的启动性能。

虽然您可以编写自己的代码来执行惰性初始化，但我们建议您改用[Lazy<T>](https://learn.microsoft.com/en-us/dotnet/api/system.lazy-1)。[Lazy<T>](https://learn.microsoft.com/en-us/dotnet/api/system.lazy-1)及其相关类型还支持线程安全并提供一致的异常传播策略。

[官方文档(1)](https://learn.microsoft.com/en-us/dotnet/framework/performance/lazy-initialization)
[官方文档(2)](https://learn.microsoft.com/en-us/dotnet/api/system.lazy-1?view=net-5.0)

默认情况下，[Lazy<T>](https://learn.microsoft.com/en-us/dotnet/api/system.lazy-1)对象是线程安全的。也就是说，如果构造函数没有指定线程安全的种类，则它创建的[Lazy<T>](https://learn.microsoft.com/en-us/dotnet/api/system.lazy-1)对象是线程安全的。在多线程场景中，第一个访问线程安全[Lazy<T>对象的](https://learn.microsoft.com/en-us/dotnet/api/system.lazy-1)[Value](https://learn.microsoft.com/en-us/dotnet/api/system.lazy-1.value)属性的线程会为所有线程上的所有后续访问初始化它，并且所有线程共享相同的数据。因此，哪个线程初始化对象并不重要，竞争条件是良性的。
```
Lazy<int> number = new Lazy<int>(() => Thread.CurrentThread.ManagedThreadId);

Thread t1 = new Thread(() => Console.WriteLine("number on t1 = {0} ThreadID = {1}",
                                        number.Value, Thread.CurrentThread.ManagedThreadId));
t1.Start();

Thread t2 = new Thread(() => Console.WriteLine("number on t2 = {0} ThreadID = {1}",
                                        number.Value, Thread.CurrentThread.ManagedThreadId));
t2.Start();

Thread t3 = new Thread(() => Console.WriteLine("number on t3 = {0} ThreadID = {1}", number.Value,
                                        Thread.CurrentThread.ManagedThreadId));
t3.Start();

t1.Join();
t2.Join();
t3.Join();
```
![](https://upload-images.jianshu.io/upload_images/20387877-0a959cf4be2b934f.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

* 当然可以使用ThreadLocal来使得每个线程上的有单独数据

```
ThreadLocal<int> threadLocalNumber = new ThreadLocal<int>(() => Thread.CurrentThread.ManagedThreadId);
Thread t4 = new Thread(() => Console.WriteLine("threadLocalNumber on t4 = {0} ThreadID = {1}",
            threadLocalNumber.Value, Thread.CurrentThread.ManagedThreadId));
t4.Start();

Thread t5 = new Thread(() => Console.WriteLine("threadLocalNumber on t5 = {0} ThreadID = {1}",
            threadLocalNumber.Value, Thread.CurrentThread.ManagedThreadId));
t5.Start();

Thread t6 = new Thread(() => Console.WriteLine("threadLocalNumber on t6 = {0} ThreadID = {1}",
            threadLocalNumber.Value, Thread.CurrentThread.ManagedThreadId));
t6.Start();
```
![ThreadLocal结果](https://upload-images.jianshu.io/upload_images/20387877-82b84e0f7ef95437.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
