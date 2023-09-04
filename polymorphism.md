# 一、背景

在接第三方的拨打电话接口时，基本就按照第三方的文档去把示例代码拉下来，跑一遍，跑通之后就当完工了，一个服务一个对应的第三方接口：

![原版设计](https://upload-images.jianshu.io/upload_images/20387877-df54841013b1526f.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

👆上面这种写法当然能用，但是从长远来看，这种通信类的接口有可能不止有这个`Twilio`，后面有可能就换成百度的、阿里的、谷歌的或者其他第三方，就意味着我需要像上面那样分散、垂直的方式去设计这些接口，虽然也能用，但是不漂亮，所以不能接受。

# 二、简要设计

## 1、总体我们需要一个`Communication`的文件夹存放这些有关通信的服务
## 2、我们对每个服务设计一个最上层的`Provider`，例如：`PhoneCallProvider`（打电话服务）
## 3、每个Provider都带有一个特定标识，用于辨认是哪个第三方提供的服务
## 4、同一个第三方的服务接口整合到一个`Service`
## 5、不同第三方的服务去实现各个`Provider`

# 三、 详细设计

直接上设计图：

![具体设计图](https://upload-images.jianshu.io/upload_images/20387877-6574e1e3dd23c271.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

稍微解释一下：

* `PhoneCallProvider`、`SmsProvider`或者说其他通信类服务都统一放到`Communication`包中管理
* 这些`Provider`在被调用的时候会根据`CommunicationProviderSwitcher`来决定选择哪一个第三方来提供相关服务
* 假如说选择的第三方是`Twilio`，底下会有它各种服务实现各种`Provider`
* 具体的实际逻辑由`TwilioService`接入的api提供

# 四、核心代码

层级结构：
![层级](https://upload-images.jianshu.io/upload_images/20387877-bb36836ce45e4604.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

* `PhoneCallProvider`:
```
public interface IPhoneCallProvider : IScopedDependency
{
    public PhoneCallProvider PhoneCallProvider { get; }
    
    Task<PhoneCallResponseDto> PhoneCallAsync(
        string to, string body, object from, string recordingTrack = null, bool? isRecord = false, CancellationToken cancellationToken = default);
}
```

* `CommunicationProviderSwitcher`:

```
public interface ICommunicationProviderSwitcher : IScopedDependency
{    
    IPhoneCallProvider PhoneCallProvider(PhoneCallProvider? phoneCallProvider = null, bool? random = true);
}

public class CommunicationProviderSwitcher : ICommunicationProviderSwitcher
{
    private readonly IEnumerable<IPhoneCallProvider> _phoneCallProviders;
    private readonly PhoneCallProvidersSetting _phoneCallProvidersSetting;

    public CommunicationProviderSwitcher(
        IEnumerable<IPhoneCallProvider> phoneCallProviders,
        PhoneCallProvidersSetting phoneCallProvidersSetting)
    {
        _phoneCallProviders = phoneCallProviders;
        _phoneCallProvidersSetting = phoneCallProvidersSetting;
    }

    public IPhoneCallProvider PhoneCallProvider(PhoneCallProvider? phoneCallProvider = null, bool? random = true)
    {
        if (phoneCallProvider.HasValue) return _phoneCallProviders.FirstOrDefault(x => x.PhoneCallProvider == phoneCallProvider);
        
        var phoneCallProviders = _phoneCallProvidersSetting.Value.ToList();

        phoneCallProvider = phoneCallProviders.Count switch
        {
            > 1 when random.HasValue && random.Value => phoneCallProviders[new Random().Next(0, phoneCallProviders.Count - 1)],
            < 1 => throw new MissingThirdPartyProviderException(),
            _ => phoneCallProviders.FirstOrDefault()
        };

        var provider = _phoneCallProviders.FirstOrDefault(x => x.PhoneCallProvider == phoneCallProvider);
        
        Log.Information("PhoneCallProvider: {PhoneCallProvider}", provider?.PhoneCallProvider);

        return provider;
    }
}
```

* `TwilioService`:

```
public interface ITwilioService : IScopedDependency
{
    Task<TwilioPhoneCallResponseDto> PhoneCallAsync(
        string to, string resourceUrl, TwilioNumberGroup group, string recordingTrack = null, bool? isRecord = false, CancellationToken cancellationToken = default);
}

public async Task<TwilioPhoneCallResponseDto> PhoneCallAsync(
    string to, string resourceUrl, TwilioNumberGroup group, string recordingTrack = null, bool? isRecord = false, CancellationToken cancellationToken = default)
{
    var from = await GetTwilioNumberByGroupAsync(group, cancellationToken).ConfigureAwait(false);
    
    TwilioClient.Init(_twilioSettings.AccountSid, _twilioSettings.AuthToken);
    
    var callResource = await CallResource.CreateAsync(
        record: isRecord,
        recordingTrack: recordingTrack,
        url: new Uri(resourceUrl),
        to: new PhoneNumber(to),
        from: new PhoneNumber(from)
    ).ConfigureAwait(false);
    
    return _mapper.Map<TwilioPhoneCallResponseDto>(callResource);
}

public async Task<string> GetTwilioNumberByGroupAsync(TwilioNumberGroup group, CancellationToken cancellationToken)
{
    var numbers = await _twilioDataProvider
        .GetActiveTwilioNumbersByGroupAsync(group.ToString(), cancellationToken).ConfigureAwait(false);
    
    var number = numbers.FirstOrDefault()?.Phone;

    if (numbers.Count > 1)
    {
        var random = new Random();

        number = numbers[random.Next(0, numbers.Count - 1)].Phone;
    }

    return number;
}
```

* `TwilioPhoneCallProvider`:

```
public class TwilioPhoneCallProvider : IPhoneCallProvider
{
    private readonly IMapper _mapper;
    private readonly ITwilioService _twilioService;

    public TwilioPhoneCallProvider(IMapper mapper, ITwilioService twilioService)
    {
        _mapper = mapper;
        _twilioService = twilioService;
    }

    public async Task<PhoneCallResponseDto> PhoneCallAsync(
        string to, string body, object group, string recordingTrack = null, bool? isRecord = false, CancellationToken cancellationToken = default)
    {
        if (!Enum.TryParse<TwilioNumberGroup>(group.ToString(), out var numberGroup)) return null;
        
        var response = await _twilioService
            .PhoneCallAsync(to, body, numberGroup, recordingTrack, isRecord, cancellationToken).ConfigureAwait(false);
            
        return _mapper.Map<PhoneCallResponseDto>(response);
    }
}
```
