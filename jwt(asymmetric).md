# 一、前言
上一篇实现了对称加密：https://github.com/MonesyH/C-learn/blob/main/jwt(symmetry).md

这一篇实现非对称加密

# 二、生成公私钥
```
[Route("rsa"), HttpPost]
public IActionResult GenerateRsaKeyParies([FromServices] IWebHostEnvironment env)
{
    RSAParameters privateKey, publicKey;
    
    using (var rsa = new RSACryptoServiceProvider(2048))
    {
        try
        {
            privateKey = rsa.ExportParameters(true);
            publicKey = rsa.ExportParameters(false);
        }
        finally
        {
            rsa.PersistKeyInCsp = false;
        }
    }

    var dir = Path.Combine(env.ContentRootPath, "Rsa");
    if (!Directory.Exists(dir))
    {
        Directory.CreateDirectory(dir);
    }

    System.IO.File.WriteAllText(Path.Combine(dir, "key.private.json"), JsonConvert.SerializeObject(privateKey));
    System.IO.File.WriteAllText(Path.Combine(dir, "key.public.json"), JsonConvert.SerializeObject(publicKey));

    return Ok();
}
```

# 三、修改
## 1. Startup.cs或者自定义Extension文件中的AddSingleton方法

```
services.AddSingleton(sp => new SigningCredentials("你的密钥", SecurityAlgorithms.HmacSha256Signature));
```
上面改成👇
```
public IWebHostEnvironment env { get; set; }

var rsaSecurityPrivateKeyString = File.ReadAllText(Path.Combine(env.ContentRootPath, "Rsa", "key.private.json"));
var rsaSecurityPublicKeyString = File.ReadAllText(Path.Combine(env.ContentRootPath, "Rsa", "key.public.json"));
RsaSecurityKey rsaSecurityPrivateKey = new(JsonConvert.DeserializeObject<RSAParameters>(rsaSecurityPrivateKeyString));
RsaSecurityKey rsaSecurityPublicKey = new(JsonConvert.DeserializeObject<RSAParameters>(rsaSecurityPublicKeyString));
services.AddSingleton(sp => new SigningCredentials(rsaSecurityPrivateKey, SecurityAlgorithms.RsaSha256Signature));
```
## 2. Startup.cs或者自定义Extension文件中的AddJwtBearer方法
```
IssuerSigningKey = new SymmetricSecurityKey(Encoding.UTF8.GetBytes(secret))
```
上面改成👇
```
IssuerSigningKey = rsaSecurityPublicKey
```
## 3. appsettings.json中存放的钥匙
之前是自定义的一串字符串，现在替换成生成的私钥字符串（生成的目录是在根目录的rsa文件夹中）

## 4. 生成token的方法（我的是GenerateJwtToken）
```
private string GenerateJwtToken(List<Claim> claims)
{
    var tokenHandler = new JwtSecurityTokenHandler();
    var secret = Encoding.UTF8.GetBytes(_secret.Value);  //选择你的字符编码
    var tokenDescriptor = new SecurityTokenDescriptor
    {
        Subject = new ClaimsIdentity(claims),
        Expires = DateTime.UtcNow.AddDays(7),
        SigningCredentials = new SigningCredentials(new SymmetricSecurityKey("你的密钥"), SecurityAlgorithms.HmacSha256Signature)
    };
    var token = tokenHandler.CreateToken(tokenDescriptor);
    
    return tokenHandler.WriteToken(token);
}
```
上面改成👇
```
private string GenerateJwtToken(List<Claim> claims)
{
    var tokenHandler = new JwtSecurityTokenHandler();
    RsaSecurityKey rsaSecurityPrivateKey = new(JsonConvert.DeserializeObject<RSAParameters>("生成的公钥字符串"));
    
    var tokenDescriptor = new SecurityTokenDescriptor
    {
        Subject = new ClaimsIdentity(claims),
        Expires = DateTime.UtcNow.AddDays(7),
        SigningCredentials = new SigningCredentials(rsaSecurityPrivateKey, SecurityAlgorithms.RsaSha256Signature)
    };
    var token = tokenHandler.CreateToken(tokenDescriptor);
    
    return tokenHandler.WriteToken(token);
}
```

# 三、测试
## 1. 生成公私钥
![生成的钥](https://upload-images.jianshu.io/upload_images/20387877-b9600d2739fcc216.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

## 2. 登陆获取token
![登陆](https://upload-images.jianshu.io/upload_images/20387877-acac33b3429c745f.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

## 3. 访问需要权限的接口
![请求接口](https://upload-images.jianshu.io/upload_images/20387877-e04bf02aabf256d7.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
