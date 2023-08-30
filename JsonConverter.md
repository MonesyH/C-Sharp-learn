# [JsonConverter(typeof (StringEnumConverter))]

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
