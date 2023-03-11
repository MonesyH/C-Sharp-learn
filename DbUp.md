# 一. 何为成功的数据库管理
[Paul Stovell参考文章](https://paulstovell.com/database-deployment/)
1. 源代码控制
您的数据库不在源代码控制中？你不值得拥有。去使用 Excel。
2. 可测试性
我希望能够编写一个集成测试，备份旧状态，执行升级到当前状态，并验证数据没有损坏。
3. 持续集成
每次我签入时，我希望这些测试由我的构建服务器运行。我想要一个 CI 构建，它可以进行生产备份、恢复它，并每晚运行和测试任何升级。
4. 没有共享数据库
每个开发人员都应该能够在自己的机器上拥有数据库的副本。使用示例数据部署该数据库应该是一次单击。
5. Dogfooding 升级
如果 Susan 对数据库进行更改，Harry 应该能够在他自己的数据库上执行她的转换。如果他有与她不同的测试数据，他可能会发现她没有发现的错误。Harry 不应该只是毁掉他的数据库然后重新开始。
# 二、集成到项目 
[官方参考文章](https://dbup.readthedocs.io/en/latest/usage/)
##1. 引入DbUp相关包
```
Install-Package dbup-core
Install-Package dbup-mysql
Install-Package dbup-sqlserver
```
## 2. 新建一个类来进行配置
```
public void Run()
    {
        EnsureDatabase.For.MySqlDatabase(连接数据库的字符串);

        var upgradeEngine = DeployChanges.To.MySqlDatabase(连接数据库的字符串)
            .WithScriptsEmbeddedInAssembly(typeof(这个类).Assembly)
            .WithTransaction()  //开启事务
            .LogToAutodetectedLog()  //自动跟踪日志
            .LogToConsole()  //日志打印到控制台
            .Build();

        var result = upgradeEngine.PerformUpgrade(); //执行数据库升级

        if (!result.Successful)
            throw result.Error;
        {
            Console.ForegroundColor = ConsoleColor.Green;
            Console.WriteLine("Success!");
            Console.ResetColor();
        }
    }
```
### 补充：
* 入口点是`DeployChanges.To`
* 获取将要执行的脚本 `GetScriptsToExecute()`
* 获取已经执行的脚本 `GetExecutedScripts()`
* 检查是否需要升级 `IsUpgradeRequired()`
* 为任何新的迁移脚本创建版本记录而不执行它们 `MarkAsExecuted`
* 尝试连接到数据库 `TryConnect()`
* 执行数据库升级 `PerformUpgrade()`
* 日志脚本输出 `LogScriptOutput()`

# 三、指定数据库脚本
## 1. 把创建初始的数据库的脚本放到项目中
例子：
![注：Script0001_initial_tables.sql是创建数据库脚本](https://upload-images.jianshu.io/upload_images/20387877-d5a847a586d3bb73.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

##2 . 在项目文件中上面数据库脚本的位置
```
    <ItemGroup>
        <EmbeddedResource Include="Dbup\Scripts_2023\Script0001_initial_tables.sql" />
    </ItemGroup>
```
## 3. 在program中调用第二步创建类的Run方法
```
new 第二步的类.Run();
```
