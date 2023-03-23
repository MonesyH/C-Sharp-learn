# 一、为什么需要Extension

在传统的面向对象编程中，如果要为一个类添加新的方法，通常需要继承该类并创建一个子类。但是，这种方法存在一些问题，例如：

* 如果要为多个类添加相同的方法，就需要创建多个子类，代码量会很大。
* 如果要为 .NET Framework 或其他库中的类型添加方法，则不能直接修改这些类型的源代码。

而扩展方法使您能够向现有类型“添加”方法，而无需创建新的派生类型、重新编译或以其他方式修改原始类型。

最常见的扩展方法是 LINQ 标准查询运算符，它们将查询功能添加到现有的[System.Collections.IEnumerable](https://learn.microsoft.com/en-us/dotnet/api/system.collections.ienumerable)和[System.Collections.Generic.IEnumerable<T>](https://learn.microsoft.com/en-us/dotnet/api/system.collections.generic.ienumerable-1)类型。

# 二、 如何自定义一个Extension

string例子：
```
public static class MyExtensions
{
     public static int WordCount(this string str)
     {
         return str.Split(new char[] { ' ', '.', '?' },
                    StringSplitOptions.RemoveEmptyEntries).Length;
     }
}

public void static main()
{
    string s = "Hello Extension Methods";
    int i = s.WordCount();
}
```

同样也可以像调用静态方法那样实现：
```
string s = "Hello Extension Methods";
int i = MyExtensions.WordCount(s);
```
