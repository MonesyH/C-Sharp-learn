# sealed关键字

[官网地址](https://learn.microsoft.com/zh-cn/dotnet/csharp/language-reference/keywords/sealed)

在下面的示例中，`Z `继承自` Y`，但 `Z` 无法替代在 `X` 中声明并在 `Y `中密封的虚函数` F`：
```
class X
{
    protected virtual void F() { Console.WriteLine("X.F"); }
    protected virtual void F2() { Console.WriteLine("X.F2"); }
}

class Y : X
{
    sealed protected override void F() { Console.WriteLine("Y.F"); }
    protected override void F2() { Console.WriteLine("Y.F2"); }
}

class Z : Y
{
    protected override void F2() { Console.WriteLine("Z.F2"); }
}
```

需要注意的点：

* 将 [abstract](https://learn.microsoft.com/zh-cn/dotnet/csharp/language-reference/keywords/abstract) 修饰符与密封类结合使用是错误的，因为抽象类必须由提供抽象方法或属性的实现的类来继承。

* 应用于方法或属性时，`sealed` 修饰符必须始终与 [override](https://learn.microsoft.com/zh-cn/dotnet/csharp/language-reference/keywords/override) 结合使用。

若要确定是否密封类、方法或属性，通常应考虑以下两点：

* 派生类通过可以自定义类而可能获得的潜在好处。

* 派生类可能采用使它们无法再正常工作或按预期工作的方式来修改类的可能性。
