# implicit operator 关键字

[官网地址](https://learn.microsoft.com/en-us/dotnet/csharp/language-reference/operators/user-defined-conversion-operators)

分为显式转换（explicit operator）和隐式转换（implicit operator）

实际上都是语法糖，那我用最简朴的方式去实现大概是长这个样：

* `explicit operator`:

```
public class IntegerWrapper
{
    private int _value;

    public IntegerWrapper(int value)
    {
        _value = value;
    }

    public static int ToInt(IntegerWrapper wrapper)
    {
        return wrapper._value;
    }

    public static IntegerWrapper FromInt(int value)
    {
        return new IntegerWrapper(value);
    }
}

// 使用方法：
IntegerWrapper wrapper = new IntegerWrapper(10);
int number = IntegerWrapper.ToInt(wrapper);
```

* `implicit operator`:

```
public class StringWrapper
{
    private string _value;
    public StringWrapper(string value) => _value = value;

    public string ToStr() => _value;
    public static StringWrapper FromStr(string s) => new StringWrapper(s);
}

// 使用方法：
StringWrapper wrapper = new StringWrapper("test");
string number = wrapper;
```

C# 提供的语法糖写法：

* `explicit operator`:

```
public class IntegerWrapper
{
    private int _value;

    public IntegerWrapper(int value)
    {
        _value = value;
    }

    public static explicit operator int(IntegerWrapper wrapper)
    {
        return wrapper._value;
    }

    public static explicit operator IntegerWrapper(int value)
    {
        return new IntegerWrapper(value);
    }
}

// 使用方法：
IntegerWrapper wrapper = new IntegerWrapper(10);
int number = (int)wrapper;
```

* `implicit operator`:

```
public class StringWrapper
{
    private string _value;
    public StringWrapper(string value) => _value = value;

    public static implicit operator string(StringWrapper wrapper) => wrapper._value;
    public static implicit operator StringWrapper(string s) => new StringWrapper(s);
}

// 使用方法：
StringWrapper wrapper = new StringWrapper("test");
string number = wrapper;
```
