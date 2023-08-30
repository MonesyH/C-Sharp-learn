# 一、背景

今天对接twilio的打电话接口的时候想把他拨出之后的结果换成自己的dto，然后其中看到他有个字段是用的这个类型：

```
[JsonConverter(typeof(StringEnumConverter))]
public sealed class SourceEnum : StringEnum
{
  public static readonly RecordingResource.SourceEnum Dialverb = new RecordingResource.SourceEnum("DialVerb");
  public static readonly RecordingResource.SourceEnum Conference = new RecordingResource.SourceEnum(nameof (Conference));
  public static readonly RecordingResource.SourceEnum Outboundapi = new RecordingResource.SourceEnum("OutboundAPI");
  public static readonly RecordingResource.SourceEnum Trunking = new RecordingResource.SourceEnum(nameof (Trunking));
  public static readonly RecordingResource.SourceEnum Recordverb = new RecordingResource.SourceEnum("RecordVerb");
  public static readonly RecordingResource.SourceEnum Startcallrecordingapi = new RecordingResource.SourceEnum("StartCallRecordingAPI");
  public static readonly RecordingResource.SourceEnum Startconferencerecordingapi = new RecordingResource.SourceEnum("StartConferenceRecordingAPI");

  private SourceEnum(string value)
    : base(value)
  {
  }

  public SourceEnum()
  {
  }

  public static implicit operator RecordingResource.SourceEnum(string value) => new RecordingResource.SourceEnum(value);
}
```

其中我没见过的是：

*  `[JsonConverter(typeof (StringEnumConverter))]`
* `sealed`
* `implicit operator`

# 二、[JsonConverter(typeof (StringEnumConverter))]

假如有以下枚举：

```
[JsonConverter(typeof(StringEnumConverter))]
public enum Colors
{
    Red,
    Green,
    Blue
}
```

不使用 StringEnumConverter 的情况下，序列化 Colors.Red 会得到 0。但是，如果使用了 StringEnumConverter，则序列化 Colors.Red 会得到 "Red"。

除了上面的使用方法，还可以：

```
var json = JsonConvert.SerializeObject(color, new StringEnumConverter());
var deserializedColor = JsonConvert.DeserializeObject<Colors>(json, new StringEnumConverter());
```

假如我没有这个工具，我要自己手写一个：

```
public static string EnumToString<TEnum>(TEnum enumValue) where TEnum : struct
{
    return enumValue.ToString();
}

public static TEnum StringToEnum<TEnum>(string stringValue) where TEnum : struct
{
    if (Enum.TryParse(stringValue, out TEnum result))
    {
        return result;
    }

    throw new ArgumentException($"Cannot convert \"{stringValue}\" to {typeof(TEnum).Name}.");
}

//使用
// Serialize: Enum -> String
var serialized = EnumToString(color);

// Deserialize: String -> Enum
var deserialized = StringToEnum<Colors>(serialized);
```


# 三、sealed

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

# 四、implicit operator

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
