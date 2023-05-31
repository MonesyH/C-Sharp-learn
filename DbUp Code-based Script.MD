[官方文档](https://dbup.readthedocs.io/en/latest/usage/#code-based-scripts)
# 一、场景
1. 动态生成脚本内容：使用 Code-based Scripts，可以在运行时动态生成 SQL 脚本的内容。这对于根据特定条件或逻辑生成脚本非常有用，例如根据数据或配置信息生成初始数据、动态创建表结构等。

2. 编写复杂的迁移逻辑：Code-based Scripts 允许您编写复杂的迁移逻辑，使用 C# 语言的完整功能和编程模型。这使得可以在脚本中执行更复杂的操作，例如数据转换、数据迁移、表重命名、索引重建等。

3. 代码重用和封装：通过 Code-based Scripts，可以将常用的数据库操作逻辑封装成可重用的脚本类。这样可以提高代码的可维护性和复用性，减少重复的代码编写。

4. 与应用程序逻辑集成：Code-based Scripts 允许将数据库迁移与应用程序的其他逻辑集成在一起。可以在迁移过程中执行应用程序特定的操作，如调用业务逻辑、更新配置文件、创建/修改其他对象等。

5. 更好的类型安全和编译时检查：通过使用 C# 代码编写脚本，可以在编译时捕获一些错误和问题。这提供了更好的类型安全性和编译时检查，减少运行时错误的可能性。

# 二、具体使用

1. 创建表：

```
public class CreateTables : IScript
{
    public override string ProvideScript(Func<IDbCommand> commandFactory)
    {
        return @"
            CREATE TABLE Customers (
                Id INT PRIMARY KEY,
                Name VARCHAR(100) NOT NULL,
                Email VARCHAR(100) NOT NULL
            );
            
            CREATE TABLE Orders (
                Id INT PRIMARY KEY,
                CustomerId INT NOT NULL,
                OrderDate DATETIME NOT NULL,
                FOREIGN KEY (CustomerId) REFERENCES Customers(Id)
            );
        ";
    }
}
```

2. 插入数据：

```
public class InsertInitialData : IScript
{
    public override string ProvideScript(Func<IDbCommand> commandFactory)
    {
        return @"
            INSERT INTO Customers (Id, Name, Email) VALUES (1, 'John Doe', 'john@example.com');
            INSERT INTO Customers (Id, Name, Email) VALUES (2, 'Jane Smith', 'jane@example.com');
        ";
    }
}
```

3. 修改表结构:
 
```
public class AlterTable : IScript
{
    public override string ProvideScript(Func<IDbCommand> commandFactory)
    {
        return @"
            ALTER TABLE Customers ADD COLUMN Address VARCHAR(200);
        ";
    }
}
```
4. 在DbUp中扫描这些文件：重点是WithScriptsAndCodeEmbeddedInAssembly这一个Api

```
var upgradeEngine = DeployChanges.To.MySqlDatabase(_connectionString)
    .WithScriptsAndCodeEmbeddedInAssembly(typeof(DbUpRunner).Assembly)
    .WithTransaction()
    .LogToAutodetectedLog()
    .LogToConsole()
    .Build();

var result = upgradeEngine.PerformUpgrade();

if (!result.Successful)
    throw result.Error;
```

# 三、`WithScriptsAndCodeEmbeddedInAssembly`是怎么进行扫描的
1. `WithScriptsAndCodeEmbeddedInAssembly` 在源码中会返回一个以`EmbeddedScriptAndCodeProvider`为参数的`WithScripts`方法

2. 实际上`EmbeddedScriptAndCodeProvider`就是去扫描所有脚本和代码的类，他继承了`IScriptProvider`，实现`IScriptProvider`中的`GetScripts`方法。

```
public IEnumerable<SqlScript> GetScripts(IConnectionManager connectionManager)
{
    var sqlScripts = embeddedScriptProvider
        .GetScripts(connectionManager)
        .Concat(ScriptsFromScriptClasses(connectionManager))
        .OrderBy(x => x.Name)
        .Where(x => filter(x.Name))
        .ToList();

    return sqlScripts;
}
```
这里的`GetScripts`是去拿.sql为后缀的脚本，而C#代码使是用后面的`.Concat(ScriptsFromScriptClasses(connectionManager))`来获取

```
IEnumerable<SqlScript> ScriptsFromScriptClasses(IConnectionManager connectionManager)
{
    var script = typeof(IScript);

    return connectionManager.ExecuteCommandsWithManagedConnection(dbCommandFactory => assembly.GetTypes()
        .Where(type => { return script.IsAssignableFrom(type) && type.IsClass && !type.IsAbstract; })
        .Select(s => (SqlScript)new LazySqlScript(s.FullName + ".cs", sqlScriptOptions, () => ((IScript)Activator.CreateInstance(s)).ProvideScript(dbCommandFactory)))
        .ToList());
}
```

这段代码就能把实现了`IScript`接口的类扫描出来，获取这些类的实例，并通过其 `ProvideScript `方法提供对应的 SQL 脚本内容，并返回一个包含所有脚本的集合。
