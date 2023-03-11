# 1. 跟java的注解一样吗？
## 相同：
* 都是将代码和资源打包成一个单独的文件，以便于分发和部署应用程序.
* 都可以包含依赖项和配置文件。
* 都可以被用于打包应用程序、库和插件
* 都具有各自的一个“清单”manifest文件来描述其assmebly或jar包的内容和依赖项
## 不同：
* C#的“assembly"是一个可执行的二进制文件，而Java的“jar包"是一个ZIP格式的文件
* C#中的“assembly"是.NET程序集的容器，而Java中的“jar包"是Java类文件、资源文件和元数据文件的容器。
* C#的“assembly”可以包含**多个**.NET程序集，而Java的“jar包"只包含**一个**Java应用程序库或插件。
* C#的“assembly"可以包含可执行代码和库，而Java的"jar包"只包含库和类文件。

总的来说，C#的“assembly"和Java的"iar包"都有相似的作用，但是它们的实现方式和使用方法略有不同。

# 2. 组装内容
通常，静态程序集可以由四个元素组成:
1. [assembly manifest](https://learn.microsoft.com/en-us/dotnet/standard/assembly/manifest),程序集清单，其中包含：
* 程序集的标识（它的名称和版本）。
* 一个文件表，描述构成程序集的所有其他文件，例如您创建的.exe或.dll文件所依赖的其他程序集、位图文件或自述文件。
* 程序集引用列表，它是所有外部依赖项的列表，例如.dll或其他文件。程序集引用包含对全局对象和私有对象的引用。全局对象可用于所有其他应用程序。
2. 类型元数据
3. 实现类型的 Microsoft 中间语言(MSIL) 代码。它是由编译器从一个或多个源代码文件生成的。
4. 一组资源。

# 3. 添加对程序集的引用
.NET 类库中的大多数程序集都是自动引用的。如果未自动引用系统程序集，可通过以下方式之一添加引用：

*   对于 .NET 和 .NET Core，添加对包含程序集的 NuGet 包的引用。在 Visual Studio 中使用 NuGet 包管理器，或将程序集的[<PackageReference>元素添加到](https://learn.microsoft.com/en-us/dotnet/core/tools/dependencies#the-packagereference-element)*.csproj*或*.vbproj*项目。
*   对于 .NET Framework，使用Visual Studio 中的** “添加引用”对话框或 ** [C#](https://learn.microsoft.com/en-us/dotnet/csharp/language-reference/compiler-options/inputs#references)或[Visual Basic](https://learn.microsoft.com/en-us/dotnet/visual-basic/reference/command-line-compiler/reference)`-reference`编译器的命令行选项添加对程序集的引用。

# 4. 强命名程序集和签名工具
** 不要依赖强名称来确保安全。它们仅提供唯一身份 **
有两种签名方式：
1. 使用强名称，使用强名称对程序集进行签名会将公钥加密添加到包含程序集清单的文件中。强名称签名有助于验证名称的唯一性，防止名称欺骗，并在解析引用时为调用者提供一些身份
2. 使用 Signool.exe(签名工具)。关联信任级别与强名称。
两个方式可以同时使用，也可以单独使用其中一个。

# 5. 版本控制
程序集的特定版本和依赖程序集的版本记录在程序集的manifest清单中。运行时的默认版本策路是应用程序仅使用构建和测试时使用的版本运行，除非在配置文件(应用程序配置文件、发布者策略文件和计算机的管理员配置文件)中被显式版本策略覆盖。
运行时执行几个步骤来解析程序集绑定请求:
1. 检查原始程序集引用以确定要绑定的程序集的版本
2. 检查所有适用的配置文件以应用版本策略
3. 根据原始程序集引用和配置文件中指定的任何重定向确定正确的程序集，并确定应绑定到调用程序集的版本。
4. 检查全局程序集缓存、配置文件中指定的代码库，然后使用运行时如何定位程序集中解释的探测规则检查应用程序的目录和子目录。
