# 一、使用场景

当你需要循环添加子队列的时候就可以用`ContinueJobWith`+`Aggregate`。

例如有以下场景：

在把执行完创建群聊的任务之后，把定时往群里发信息的功能也加入到队列当中，特别注意的是：有很多个时间段都需要发通知，所以就需要加入多个定时发通知的任务队列。

这个场景的伪代码：

```
var parentId = Enqueue(创建群聊);

parentId = ContinueJobWith(parentId, 立即往群发送一条信息);

foreach(var time in 需要发送的时间段)
{
    parentId = ContinueJobWith(parentId, 在time的时候往群发送一条信息);
}
```

# 二、`ContinueJobWith`和`Aggregate`的用法

## 1. `ContinueJobWith` ：用于连接后续作业

![官方](https://upload-images.jianshu.io/upload_images/20387877-c19059756d8db9c9.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

从源码上来看和`Enqueue`、`Schedule`都是差不多：

* `Enqueue`：

```
public static string Enqueue<T>(
    [NotNull] this IBackgroundJobClient client,
    [NotNull] string queue,
    [NotNull, InstantHandle] Expression<Action<T>> methodCall)
{
    if (client == null) throw new ArgumentNullException(nameof(client));
    return client.Create(queue, methodCall, new EnqueuedState());
}
```

* `Schedule`：

```
public static string Schedule<T>(
    [NotNull] this IBackgroundJobClient client,
    [NotNull] string queue,
    [NotNull, InstantHandle] Expression<Action<T>> methodCall,
    DateTimeOffset enqueueAt)
{
    if (client == null) throw new ArgumentNullException(nameof(client));
    return client.Create(queue, methodCall, new ScheduledState(enqueueAt.UtcDateTime));
}
```

* `ContinueJobWith`：

```
public static string ContinueJobWith<T>(
    [NotNull] this IBackgroundJobClient client,
    [NotNull] string parentId,
    [NotNull] string queue,
    [NotNull, InstantHandle] Expression<Action<T>> methodCall,
    [CanBeNull] IState nextState = null,
    JobContinuationOptions options = JobContinuationOptions.OnlyOnSucceededState)
{
    if (client == null) throw new ArgumentNullException(nameof(client));

    var state = new AwaitingState(parentId, nextState ?? new EnqueuedState(), options);
    return client.Create(queue, methodCall, state);
}
```

实际上都调用了`client.Create`的接口来创建队列，但在第三个的状态参数有所不同，他们都继承于`IState`。

* `Enqueue` ：`EnqueuedState`（已入队状态）当一个作业被添加到任务队列中等待执行时，它的状态将被标记为`EnqueuedState`。这表示作业已经准备好被执行，但还没有被分配给可用的后台工作线程。
* `Schedule ` ：`ScheduledState `（已计划状态）如果一个作业被设置了延迟执行时间，它的状态将会从`EnqueuedState`变为`ScheduledState`。这表示作业已经被计划在未来的特定时间执行，但还没有到达执行时间。
* `ContinueJobWith ` ：`AwaitingState`（等待状态）当一个作业处于`ScheduledState`，即已计划状态，但尚未到达执行时间时，它的状态将被标记为`AwaitingState`。这表示作业正在等待执行时间的到来。

## 2.  Aggregate 聚合函数
[官方文档](https://learn.microsoft.com/en-us/dotnet/api/system.linq.enumerable.aggregate?view=net-6.0)

我个人是把他当作一个累加器来使用的

官方的例子：

1. 找到名字最长的

```
string[] fruits = { "apple", "mango", "orange", "passionfruit", "grape" };

string longestName = fruits.Aggregate("banana",
     (longest, next) => next.Length > longest.Length ? next : longest, fruit => fruit.ToUpper());

Console.WriteLine("The fruit with the longest name is {0}.", longestName);
```

![名字最长的运行结果](https://upload-images.jianshu.io/upload_images/20387877-571a03e1042059ce.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

2. 反转单词顺序

```
string sentence = "the quick brown fox jumps over the lazy dog";

string[] words = sentence.Split(' ');

string reversed = words.Aggregate((workingSentence, next) => next + " " + workingSentence);

Console.WriteLine(reversed);
```
![反转单词顺序](https://upload-images.jianshu.io/upload_images/20387877-769817e706e6ab1a.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

# 三、ContinueJobWith和Aggregate的配合

一开始的伪代码：

```
var parentId = Enqueue(创建群聊);

parentId = ContinueJobWith(parentId, 立即往群发送一条信息);

foreach(var time in 需要发送的时间段)
{
    parentId = ContinueJobWith(parentId, 在time的时候往群发送一条信息);
}
```

改写为：

```
var parentId = Enqueue(创建群聊);

parentId = ContinueJobWith(parentId, 立即往群发送一条信息);

需要发送的时间段.Aggregate(parentId, (current, time) => 
{
    return ContinueJobWith(current, 在time的时候往群发送一条信息);
})
```


