# 一、 单元测试、集成测试与TDD、BDD的区别
## 1. 单元测试 vs 集成测试：
* 单元测试是针对代码中的最小测试单位，通常是单个函数或方法进行测试，它的目标是检测代码中的错误和缺陷。单元测试通常由开发人员编写，它可以快速准确地识别代码中的问题，并有助于维护代码的可靠性和可维护性。

* 集成测试则是对多个单元测试组成的模块或组件进行测试，以确保它们能够协同工作。集成测试通常由测试人员执行，它有助于检测不同模块之间的集成问题，并确保整个系统的稳定性。

## 2. TDD vs BDD：
* TDD（测试驱动开发）是一种开发方法，它要求在编写代码之前先编写测试用例。这些测试用例被用来指导代码编写的过程，并且在编写代码后用来验证代码的正确性。TDD可以确保代码具有高度的可测试性，并且有助于减少代码中的缺陷。

* BDD（行为驱动开发）是一种开发方法，它将测试用例视为行为规范。BDD的目标是确保开发人员和测试人员能够共同理解应用程序的行为，从而减少不必要的沟通障碍。BDD测试用例通常是从用户故事或需求规范中派生出来的，并且通常以自然语言编写。

# 二、 XUnit
[官方文档(rider版)](https://xunit.net/docs/getting-started/netfx/jetbrains-rider)

* 和别的框架比较不同的地方：不用[Test]而是使用[Fact]，如他文档中解释道**“事实是永远正确的测试。他们测试不变的条件。”**而[Theory]**理论是仅适用于一组特定数据的测试。**

简单示例： 
```
[Theory]
[InlineData(2, 2, 4)]
[InlineData(3, 3, 6)]
[InlineData(2, 2, 5)]
public void MyTheory(int x, int y, int sum)
{
   Assert.Equal(sum, Calculator.Add(x, y));
}
```
运行结果： 
![result](https://upload-images.jianshu.io/upload_images/20387877-ab40c38a72260ef8.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

# 三、Shoudly
[GitHub官方地址](https://github.com/shouldly/shouldly)

* 旧的断言方式：

```
Assert.That(contestant.Points, Is.EqualTo(1337));
```
报错信息：
```
Expected 1337 but was 0
```
* Shoudly的断言方式:

```
contestant.Points.ShouldBe(1337);
```
 报错信息：

```
contestant.Points should be 1337 but was 0
```

# 四、需要集成到项目中的步骤
## 1. 把整个测试流程都放在一个生命周期当中

IAsyncLifetime和IDisposable

它们的作用如下：
* IAsyncLifetime：IAsyncLifetime接口包含两个异步方法，用于管理对象的生命周期。这两个方法分别是InitializeAsync和DisposeAsync。InitializeAsync方法在对象创建后立即调用，用于初始化对象的状态。DisposeAsync方法在对象销毁前调用，用于清理对象占用的资源。使用IAsyncLifetime接口可以确保对象在创建和销毁时都能够正确地初始化和清理。

* IDisposable：IDisposable接口包含一个Dispose方法，用于释放对象占用的资源。Dispose方法在对象销毁前调用，用于清理对象占用的资源。使用IDisposable接口可以确保对象在销毁时能够正确地释放占用的资源，例如文件句柄、数据库连接、网络连接等。

因此要同时实现IAsyncLifetime, IDisposable以及InitializeAsync、DisposeAsync和Dispose方法
```
public class TestBase : IAsyncLifetime, IDisposable
{
    public async Task InitializeAsync()
    {
    }

    public void Dispose()
    {
    }

    public Task DisposeAsync()
    {
        return Task.CompletedTask;
    }
}
```
## 2. 根据原来的appsettings.json文件在根目录重新生成一份json文件，并修改其数据库的库名（测试时用不带数据仅有结构的新库）
```
private IConfiguration RegisterConfiguration(ContainerBuilder containerBuilder)
{
    var targetJson = $"appsettings{_testTopic}.json";

    File.Copy("appsettings.json", targetJson, true);

    dynamic jsonObj = JsonConvert.DeserializeObject(File.ReadAllText(targetJson));

    jsonObj["ConnectionStrings"]["你的连接字符串"] =
        jsonObj["ConnectionStrings"]["你的连接字符串"].ToString()
            .Replace("Database=正式的数据库名", $"Database={测试的数据库名}");

    File.WriteAllText(targetJson, JsonConvert.SerializeObject(jsonObj));

    var configuration = new ConfigurationBuilder().AddJsonFile(targetJson).Build();
    containerBuilder.RegisterInstance(configuration).AsImplementedInterfaces();

    return configuration;
}
```
## 3. 重新把需要注册的配置搬到测试solution中

这里用的是Autofac的Module的注册方式
```
private void RegisterBaseContainer(ContainerBuilder containerBuilder, IConfiguration configuration)
{
   containerBuilder.RegisterModule(new 之前定义的注册Module());
}
```
## 4. 开启生命周期后，判断是否需要根据sql文件跑DbUp
```
 xxx.BeginLifetimeScope();

 private static readonly ConcurrentDictionary<string, bool> ShouldRunDbUpDatabases = new();
 private void RunDbUpIfRequired()
 {
     if (!ShouldRunDbUpDatabases.GetValueOrDefault("测试的数据库名", true))
          return;

     new DbUpRunner("测试数据库的连接字符串").Run();
 
     ShouldRunDbUpDatabases[_databaseName] = false;
}
```
## 5.  编写一个工具类，给不同情况（接口类型不同、返回类型不同）定制请求接口的方法

示例： 
```
protected async Task<R> Run<T, R>(Func<T, Task<R>> action, Action<ContainerBuilder> extraRegistration = null)
{
    var dependency = extraRegistration != null
         ? _scope.BeginLifetimeScope(extraRegistration).Resolve<T>()
         : _scope.BeginLifetimeScope().Resolve<T>();
    return await action(dependency);
}
    
protected async Task<R> RunWithUnitOfWork<T, R>(Func<T, Task<R>> action, Action<ContainerBuilder> extraRegistration = null)
{
    var scope = extraRegistration != null
        ? _scope.BeginLifetimeScope(extraRegistration)
        : _scope.BeginLifetimeScope();

    var dependency = scope.Resolve<T>();
    var unitOfWork = scope.Resolve<IUnitOfWork>();

    var result = await action(dependency);
    await unitOfWork.SaveChangesAsync();

    return result;
}
```

一般来说，T是需要调用到的接口，R是结果类型。
## 6. 如果需要用到当前用户信息，可以提供一个默认的CurrentUser

```
public class TestCurrentUser : ICurrentUser
{
    public int Id { get; } = 1;

    public string UserName { get; set; } = "TEST_USER";
}
```
并在注册的地方加上
```
 containerBuilder.RegisterInstance(new TestCurrentUser()).As<ICurrentUser>();
```
## 7. 在生命周期结束的时候把数据库的测试数据删掉
```
private void ClearDatabaseRecord()
{
    try
    {
        var connection = new MySqlConnection("连接测试数据库的字符串");

        var deleteStatements = new List<string>();

        connection.Open();

        using var reader = new MySqlCommand(
                $"SELECT table_name FROM INFORMATION_SCHEMA.tables WHERE table_schema = '测试数据库名';",
                connection)
            .ExecuteReader();

        deleteStatements.Add($"SET SQL_SAFE_UPDATES = 0");
        while (reader.Read())
        {
            var table = reader.GetString(0);

            if (!_tableRecordsDeletionExcludeList.Contains(table))
            {
                deleteStatements.Add($"DELETE FROM `{table}`");
            }
        }

        deleteStatements.Add($"SET SQL_SAFE_UPDATES = 1");

        reader.Close();

        var strDeleteStatements = string.Join(";", deleteStatements) + ";";

        new MySqlCommand(strDeleteStatements, connection).ExecuteNonQuery();

        connection.Close();
    }
    catch (Exception ex)
    {
        Console.WriteLine($"Error cleaning up data, {_testTopic}, {ex}");
    }
}
```
上面代码的意思：

连接到数据库，查询该数据库中所有的表名，然后删除除了在排除列表中的表之外的所有表中的记录。
其中，SQL_SAFE_UPDATES被设置为0以允许不安全的更新操作，以便执行删除操作。最后，SQL_SAFE_UPDATES被设置为1以确保安全更新操作。

# 不太相关的拓展

在.NET框架中，STA和MTA是指针对COM组件进行多线程编程时使用的不同线程模型。

* STA（Single-Threaded Apartment）是一种单线程模型，它要求在一个COM组件中所有的方法调用都在同一个线程中完成，因此COM组件只能在一个线程中被访问。在STA模型下，如果多个线程需要同时访问同一个COM组件，那么这些线程必须使用互斥机制来协调访问，从而保证线程安全。

* MTA（Multithreaded Apartment）是一种多线程模型，它允许多个线程同时访问同一个COM组件，不需要使用互斥机制来协调访问。在MTA模型下，COM组件中的每个对象都可以在不同的线程中被访问，从而提高了并发性能。

在.NET框架中，通过使用ComVisible属性来设置STA或MTA模型。默认情况下，所有的.NET组件都是MTA模型，如果需要使用STA模型，则需要将ComVisible属性设置为true，并使用Thread类的SetApartmentState方法将线程设置为STA模式。
