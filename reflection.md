# 一、C#和Java反射的区别
* C#的反射支持更多的特性，比如**泛型类型**的反射、**匿名类型**的反射、动态加载程序集等，而Java的反射则相对简单一些；
* 在C#中，通过反射可以获取到更多的元数据信息，比如方法参数名称、属性默认值等；
* Java中的反射是通过**Class**对象来获取信息，而C#中则是通过**Type**对象。
#  二、查看类型信息
需要引用以下：
using [System.Reflection](https://learn.microsoft.com/en-us/dotnet/api/system.reflection);
using [System.Type](https://learn.microsoft.com/en-us/dotnet/api/system.type);
假若我们要查看String的构造函数

```
Type t = typeof(System.String);
Console.WriteLine("Listing all the public constructors of the {0} type", t);
ConstructorInfo[] ci = t.GetConstructors(BindingFlags.Public | BindingFlags.Instance);  //一般复数的方法都是list
Console.WriteLine(ci[0]); //可以循环获取所有构造函数，这里只拿了第一个
```
## 补充“|”位或运算符：
用于比较两个二进制数的位，并在每个位上执行逻辑或运算。原理是对两个二进制数的每一位进行逻辑或运算，生成一个新的二进制数。当两个二进制数中至少有一个相应位上的值为 1时，结果中相应的位为 1.否则为 0。例如，下面是两个二进制数的位或运算的示例:
```
10101010
11001100
11101110
```
所以上面的BindingFlags.Public | BindingFlags.Instance就可以把Public和Instance组合起来

# 三、泛型的反射 
1. 获取泛型的Type类型
```
Type t = typeof(Dictionary<,>);
```
2. 使用[IsGenericType](https://learn.microsoft.com/en-us/dotnet/api/system.type.isgenerictype)属性确定类型是否为泛型，使用[IsGenericTypeDefinition](https://learn.microsoft.com/en-us/dotnet/api/system.type.isgenerictypedefinition)属性确定类型是否为泛型类型定义。
```
Console.WriteLine("   Is this a generic type? {0}",t.IsGenericType);
Console.WriteLine("   Is this a generic type definition? {0}",t.IsGenericTypeDefinition);
```
3. [使用GetGenericArguments](https://learn.microsoft.com/en-us/dotnet/api/system.type.getgenericarguments)方法获取包含泛型类型参数的数组。
```
Type[] typeParameters = t.GetGenericArguments();
```
4.获取泛型类型参数
```
Type genericType = typeof(List<>);
Type[] typeArgs = genericType.GetGenericArguments();
```
5. 创建实例
```
Type type = typeof(List<>).MakeGenericType(typeof(string));
object instance = Activator.CreateInstance(type);
```
6. 调用泛型的方法
```
MethodInfo method = typeof(List<>).GetMethod("Add");
MethodInfo genericMethod = method.MakeGenericMethod(typeof(string));
object instance = Activator.CreateInstance(typeof(List<string>));
genericMethod.Invoke(instance, new object[] { "example" });
```
# 四、反射获取Atrribute
参考文章：https://www.jianshu.com/p/0b076cf609b2 中的四
