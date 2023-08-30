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

https://github.com/MonesyH/C-Sharp-learn/blob/main/JsonConverter.md

# 三、sealed

https://github.com/MonesyH/C-Sharp-learn/blob/main/sealed.md

# 四、implicit operator

https://github.com/MonesyH/C-Sharp-learn/blob/main/implicit%20operator.md
