# 一、运作过程
官方文档介绍：https://learn.microsoft.com/en-us/dotnet/csharp/asynchronous-programming/task-asynchronous-programming-model

直接上示例：

## 1、代码：
```
public async Task<int> GetUrlContentLengthAsync()
{
    var client = new HttpClient();

    Task<string> getStringTask =
        client.GetStringAsync("https://learn.microsoft.com/dotnet");

    DoIndependentWork();

    string contents = await getStringTask;

    return contents.Length;
}

void DoIndependentWork()
{
    Console.WriteLine("Working...");
}
```

## 2、实际上的运作过程
![过程](https://upload-images.jianshu.io/upload_images/20387877-ed0d0df338b25767.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

## 3、简析上图过程

* 首先`GetUrlContentLengthAsync`异步方法被调用
* 走到`GetStringAsync`异步方法时，会先暂停执行这个异步方法，然后把控制权交由其调用者`GetUrlContentLengthAsync`。
  并且承诺在在工作完成时生成实际的字符串值
* 执行完`DoIndependentWork `方法之后再把控制权给调用者`GetUrlContentLengthAsync`。
* 再下一步`string contents = await getStringTask;`,代表`GetUrlContentLengthAsync`正在等待`GetStringAsync`的结果，而调用方法正在等待`GetUrlContentLengthAsync`的结果
* 当`getStringTask`的出结果后，`GetUrlContentLengthAsync`也可以算出`content`的长度并返回

# 二、揭开Async/Await的面纱

假如有以下代码：

```
public async Task<ProjectVo> GetAsync(long projectId) 
{
    Project resultOfAwaiter1 = await projectRepo.GetAsync(projectId);
    List<Person> resultOfAwaiter2 = await personRepo.GetAsync(project.getMembers());
    ProjectVo result = ToProjectVo(resultOfAwaiter1, resultOfAwaiter2);
    return result;
}

```

反编译后的代码大概长这样：

```
[AsyncStateMachine(typeof (ProjectService.GetAsync_StateMachine))]
public Task<ProjectVo> GetAsync(long projectId)
{
    MainWindow.AwaitButtonClick_StateMachine stateMachine = new MainWindow.AwaitButtonClick_StateMachine();
    stateMachine.caller = this;
    stateMachine.projectId = projectId;
    stateMachine.builder = AsyncTaskMethodBuilder<ProjectVo>.Create();
    stateMachine.state = -1;
    // Start方法内部执行 -> stateMachine.MoveNext()
    stateMachine.builder.Start<ProjectService.GetAsync_StateMachine>(ref stateMachine);
    // 返回一个新的task（可能完成也可能未完成）
    return stateMachine.builder.Task;
}

[CompilerGenerated]
private sealed class GetAsync_StateMachine : IAsyncStateMachine
{
    public int state;
    public AsyncTaskMethodBuilder<ProjectVo> builder;
    public ProjectService caller;
    // 原函数的传入参数
    public long projectId;
    // 原函数的局部变量
    private Project resultOfAwaiter1;
    private List<Person> resultOfAwaiter2;
    private ProjectVo result;
    private TaskAwaiter<Project> awaiter1;
    private TaskAwaiter<List<Person>> awaiter2;

    void IAsyncStateMachine.MoveNext()
    {
        try
        {
            switch (this.state)
            {
                case 0:
                    this.state = -1;
                    break;
                case 1:
                    this.state = -1;
                    goto label_8;
                case -1:
                    // 开始第一个Task并获得awaiter，通过awaiter来观察Task是否完成。
                    this.awaiter1 = this.caller.projectRepo.GetAsync(this.projectId)
                        .GetAwaiter();
                    if (!this.awaiter1.IsCompleted)
                    {
                        this.state = 0;   
                        // 向未完成的Task中注册continuation action;
                        // continuation action会在Task完成时执行;
                        // 等同于awaiter1.onCompleted(() => this.MoveNext());
                        this.builder.AwaitUnsafeOnCompleted<TaskAwaiter<Project>, ProjectService.GetAsync_StateMachine>(ref this.awaiter1, ref this);
                        // return（即交出控制权给GetAsync的调用者）
                        return;
                    }
                    break;
            }
            // 第一个Task完成，获取结果
            this.resultOfAwaiter1 = this.awaiter1.GetResult();
            // 开始第二个Task
            this.awaiter2 = this.caller.personRepo
                .GetAsync(resultOfAwaiter1.getMembers())
                .GetAwaiter();
            if (!this.awaiter2.IsCompleted)
            {
                this.state = 1;
                // 向未完成的Task中注册continuation action
                this.builder.AwaitUnsafeOnCompleted<TaskAwaiter<List<Person>>, ProjectService.GetAsync_StateMachine>(ref this.awaiter2, ref this);
                // return
                return;
            }
label_8: // 标记，用于goto跳转
            // 第二个Task完成，获取结果
            this.resultOfAwaiter2 = this.awaiter2.GetResult();
            
            this.result = this.caller.ToProjectVo(this.resultOfAwaiter1, this.resultOfAwaiter2);
        }
        catch (Exception ex)
        {
            this.state = -2;
            this.builder.SetException(ex);
            return;
        }
        this.state = -2;
        // 将builder标记为completed；
        // 将未完成的task标记为completed；（这里的task指GetAsync的返回值）
        // set result并run continuation；
        this.builder.SetResult(this.result);
    }
}
```

* `stateMachine.builder.Start`方法内部是在调用`stateMachine.MoveNext`
* `stateMachine.builder.Task`返回 new task， task可能已完成也可能未完成，这取决于`stateMachine`是否达到了最终状态即-2

上述代码的状态机运作总结起来就是以下流程图：
![状态机流程](https://upload-images.jianshu.io/upload_images/20387877-c5d14894506cc966.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

由上图和代码可以得出：

* `AsyncStateMachine`内部状态的数量=n+2，n即async方法体内await出现的次数
    例如上面有两个Task，那 `AsyncStateMachine`内部状态的数量=4（0、1、-1、2）
    0、1就是这两个Task各自的等待状态，-1是初始状态、2是所有异步方法完成状态
* `AsyncStateMachine`的状态为非负整数时，它会暂停执行并交出控制权，只有当它的状态为`-1`时才会继续执行
* 如果足够幸运，只调用一次`MoveNext`就可以让`AsyncStateMachine`变成最终状态`-2`

# 三、那多重调用异步方法是怎么运作的呢？

参考以下例子：

```
public async Task OuterAsyncMethod()
{
    await Task.Delay(100);
    await InnerAsyncMethod();
    await Task.Delay(200);
}

public async Task InnerAsyncMethod()
{
    await Task.Delay(300);
}
```

在这个例子中，`OuterAsyncMethod`方法中调用了`InnerAsyncMethod`方法。
* 当执行到`await InnerAsyncMethod()`时，`OuterAsyncMethod`方法将进入等待状态，并将控制权交给`InnerAsyncMethod`的状态机。
* 当`InnerAsyncMethod`完成后，控制权将返回给`OuterAsyncMethod`的状态机，继续执行后续的异步操作。

这种执行顺序类似于递归调用，其中一个状态机等待另一个状态机完成后才能继续执行。这样可以确保多个异步方法按照预期的顺序执行，并正确地管理异步操作的状态转换和控制流程。
