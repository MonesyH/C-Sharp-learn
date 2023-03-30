原来的样子：
```
public enum MyEnum
{
    Value1,

    Value2,

    Value3
}

var enumDescriptions =
    new Dictionary<string, MyEnum>()
    {
        {"description1", MyEnum.Value1},
        {"description2", MyEnum.Value2},
        {"description3", MyEnum.Value3},
        {"description4", MyEnum.Value4},
    };
```

使用Attribute：
```
public enum MyEnum
{
    [Description("This is the first value")]
    Value1,
    [Description("This is the second value")]
    Value2,
    [Description("This is the third value")]
    Value3
}

var enumDescriptions = Enum.GetValues(typeof(MyEnum))
    .Cast<MyEnum>()
    .ToDictionary(
        e => e,
        e =>
        {
            FieldInfo field = e.GetType().GetField(e.ToString());
            DescriptionAttribute[] attributes = (DescriptionAttribute[])field.GetCustomAttributes(typeof(DescriptionAttribute), false);
            return attributes.Length > 0 ? attributes[0].Description : e.ToString();
        }
    );
```
