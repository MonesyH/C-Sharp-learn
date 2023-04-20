# 一、背景
我需要用到[JsonProperty]来对某些字段指定别名（为了对接外部接口）进行序列化，到了swagger ui上之后，发现用到[JsonProperty]的字段都无法正常传递值，怀疑是请求时字段名不匹配导致接收不到。把原来的驼峰式命名换成[JsonProperty]定义的别名后，就能正常传参。

## 解决方案
### 1. 用[JsonPropertyName]来进行序列化（弃用）

结果：按照Swagger UI显示的示例传参能正常传递参数值，但是在进行序列化对接外部接口的时候，并不能按照[JsonPropertyName]里的参数来序列化。

### 2. 把所有参数都放在command里面，所有都不用[JsonProperty]或[JsonPropertyName]，接收到前端传参后再组装成一个有[JsonProperty]的DTO。（暂定）

后续： 需要组装的字段太多了，一个一个组装太冗长，暂时不这么去做。也有想过用Automapper映射过去，但实际上还是需要指定字段一个一个映射，同样冗长。

### 3. 从Swagger动手，写一个自定义的规则修改Swagger UI上传参的显示。（采用）

需要考虑的点： 
 * 这样改会不会动到全局的配置
* 应该只需要修改有[JsonProperty]的字段显示，而实际上只有command中的子类（因为只有子类可能会用到[JsonProperty]）需要修改
* 够不够generic

# 二、在SwashBuckle源码里面找找灵感
[GitHub源码地址](https://github.com/domaindrivendev/Swashbuckle.AspNetCore)
## 1. 实际上Swagger UI的数据是都是在https://localhost:xxx/swagger/v1/swagger.json里面拿到，然后渲染成Swagger UI
![swagger.json](https://upload-images.jianshu.io/upload_images/20387877-e29d107d51a765c6.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

## 2. 所以先去看看这个swagger.json是怎么生成的，定位到源码的SwaggerGenerator类
![SwaggerGenerator](https://upload-images.jianshu.io/upload_images/20387877-d029cd3e147ec653.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

能看到是有以下这四个来构成SwaggerGenerator：

```
private readonly IApiDescriptionGroupCollectionProvider _apiDescriptionsProvider;
private readonly ISchemaGenerator _schemaGenerator;
private readonly SwaggerGeneratorOptions _options;
private readonly IAuthenticationSchemeProvider _authenticationSchemeProvider;
```

这四个对象的作用如下：

1.  `IApiDescriptionGroupCollectionProvider`: 用于获取 API 的描述信息。在 [ASP.NET](http://asp.net/) Core 中，API 的描述信息被保存在 Swagger 中，这个对象能够提供所有 API 描述信息的集合。
据我观察（debugger的时候），它是获取  `washbuckle.AspNetCore.Annotations`中像[SwaggerRequestBody]、[SwaggerResponse]等自定义描述的信息

2.  `ISchemaGenerator`: 用于将 C# 类型转换为 Swagger 模型。Swagger 规范中使用 JSON Schema 描述 API 的输入和输出参数以及数据模型，这个对象会将 C# 类型映射到对应的 JSON Schema。

3.  `SwaggerGeneratorOptions`: 包含了 Swagger 生成器的选项配置。开发者可以通过这个对象来调整生成器的行为，例如配置 API 文档的标题、版本号等信息。

4.  `IAuthenticationSchemeProvider`: 用于获取身份验证方案（authentication scheme）。[ASP.NET](http://asp.net/) Core 提供了多种身份验证方式，例如基于 Cookie 的身份验证和基于 OAuth2 的身份验证。这个对象能够提供应用程序所定义的所有身份验证方案的集合，使得 Swagger 能够正确地显示需要授权访问的 API。

往后看一点：

```
public async Task<OpenApiDocument> GetSwaggerAsync(string documentName, string host = null, string basePath = null)
{
    var (applicableApiDescriptions, swaggerDoc, schemaRepository) = GetSwaggerDocumentWithoutFilters(documentName, host, basePath);

    swaggerDoc.Components.SecuritySchemes = await GetSecuritySchemes();

    // NOTE: Filter processing moved here so they may effect generated security schemes
    var filterContext = new DocumentFilterContext(applicableApiDescriptions, _schemaGenerator, schemaRepository);
    foreach (var filter in _options.DocumentFilters)
    {
        filter.Apply(swaggerDoc, filterContext);
    }

    swaggerDoc.Components.Schemas = new SortedDictionary<string, OpenApiSchema>(swaggerDoc.Components.Schemas, _options.SchemaComparer);

    return swaggerDoc;
}
```
这一段代码是灵感来源，主要是foreach那里会把所有filter的Apply都执行一次，说明到时候我也可以像他一样，做一个自定义的filter，然后交给它来循环执行。
PS：这个filter其实是实现IOperationFilter的filter差不多，都是需要实现接口的Apply方法。
![IOperationFilter](https://upload-images.jianshu.io/upload_images/20387877-3243e7332d13f51e.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

## 3. 那我重点要看的是 `ISchemaGenerator`里面GenerateSchema方法的源码
```
public OpenApiSchema GenerateSchema(
    Type modelType,
    SchemaRepository schemaRepository,
    MemberInfo memberInfo = null,
    ParameterInfo parameterInfo = null,
    ApiParameterRouteInfo routeInfo = null)
{
    if (memberInfo != null)
        return GenerateSchemaForMember(modelType, schemaRepository, memberInfo);

    if (parameterInfo != null)
        return GenerateSchemaForParameter(modelType, schemaRepository, parameterInfo, routeInfo);

    return GenerateSchemaForType(modelType, schemaRepository);
}
```
### （1）我要改的是MemberInfo，所以看`GenerateSchemaForMember`，这个方法太长，而主要看的是第一行的：
```
var dataContract = GetDataContractFor(modelType);
👇
private DataContract GetDataContractFor(Type modelType)
{
    var effectiveType = Nullable.GetUnderlyingType(modelType) ?? modelType;
    return _serializerDataContractResolver.GetDataContractForType(effectiveType);
}
```
![GetDataContractForType](https://upload-images.jianshu.io/upload_images/20387877-e1ea7b0195067d6e.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
这里就能看到他是怎么处理JsonProperty这个Attribute了
我就直接先跳到具体处理JsonProperty逻辑的源码里面了，想要详细看的话可以自己去看看过程。

```
public static class JsonPropertyExtensions
{
    public static bool TryGetMemberInfo(this JsonProperty jsonProperty, out MemberInfo memberInfo)
    {
        memberInfo = jsonProperty.DeclaringType?.GetMember(jsonProperty.UnderlyingName)
            .FirstOrDefault();

        return (memberInfo != null);
    }

    public static bool IsRequiredSpecified(this JsonProperty jsonProperty)
    {
        if (!jsonProperty.TryGetMemberInfo(out MemberInfo memberInfo))
            return false;

        if (memberInfo.GetCustomAttribute<JsonRequiredAttribute>() != null)
            return true;

        var jsonPropertyAttributeData = memberInfo.GetCustomAttributesData()
            .FirstOrDefault(attrData => attrData.AttributeType == typeof(JsonPropertyAttribute));

        return (jsonPropertyAttributeData != null) && jsonPropertyAttributeData.NamedArguments.Any(arg => arg.MemberName == "Required");
    }
}
```

这里也属于灵感来源吧，让我想通过反射[JsonProperty]这个Attribute来获取需要修改的字段。

### （2）以及`GenerateSchemaForMember`中的`GenerateConcreteSchema`
因为源码太长了我就大概说一下主要作用，有兴趣可以去看看：
主要是根据类型的不同分别调用 CreatePrimitiveSchema、CreateArraySchema、CreateDictionarySchema 或 CreateObjectSchema 函数来创建相应的 Schema。
而他比较有趣的一行是最后一行：
```
return returnAsReference
    ? GenerateReferencedSchema(dataContract, schemaRepository, schemaFactory)
    : schemaFactory();
```
来看看这个`GenerateReferencedSchema`方法
```
private OpenApiSchema GenerateReferencedSchema(
    DataContract dataContract,
    SchemaRepository schemaRepository,
    Func<OpenApiSchema> definitionFactory)
{
    if (schemaRepository.TryLookupByType(dataContract.UnderlyingType, out OpenApiSchema referenceSchema))
        return referenceSchema;

    var schemaId = _generatorOptions.SchemaIdSelector(dataContract.UnderlyingType);

    schemaRepository.RegisterType(dataContract.UnderlyingType, schemaId);

    var schema = definitionFactory();
    ApplyFilters(schema, dataContract.UnderlyingType, schemaRepository);

    return schemaRepository.AddDefinition(schemaId, schema);
}
```
而这个SchemaIdSelector就是看传参的时候会不会出现参数的类型是一个类，即有子类
```
private string DefaultSchemaIdSelector(Type modelType)
{
    if (!modelType.IsConstructedGenericType) return modelType.Name.Replace("[]", "Array");

    var prefix = modelType.GetGenericArguments()
        .Select(genericArg => DefaultSchemaIdSelector(genericArg))
        .Aggregate((previous, current) => previous + current);

    return prefix + modelType.Name.Split('`').First();
}
```
这里就能说明他的scheme是包含了所有的传参字段以及传参类型

# 三、实现
按照上面的灵感，整理一下思路：
* 选择任意一个filter（这里我先采用IOperationFilter，因为当时还不知道有IRequestBodyFilter）
* 根据对应的command（一个command对应一个请求），制定修改逻辑
* 修改逻辑：只command中的子类中有[JsonProperty]的字段
* 实现用command和子类构造自定义filter

自定义filter实现：
```
public class SwaggerCustomSchemeOperationFilter : IOperationFilter
{
    private readonly Type _mainParameter;
    private readonly List<Type> _subParameters;

    public SwaggerCustomSchemeOperationFilter(Type mainParameter, List<Type> subParameters)
    {
        _mainParameter = mainParameter;
        _subParameters = subParameters;
    }
    
    public void Apply(OpenApiOperation operation, OperationFilterContext context)
    {
        if (!_mainParameter.Name.Equals(operation.RequestBody?.Content?.FirstOrDefault().Value.Schema.Reference?.Id)) return;

        var references = context.SchemaRepository.Schemas
            .FirstOrDefault(x => x.Key == _mainParameter.Name).Value.Properties
            .Where(x => x.Value.Reference != null).ToList();

        if (references.IsNullOrEmpty()) return;
        
        foreach (var subParameter in _subParameters)
        {
            if (!references.Select(x => x.Value.Reference.Id).Contains(subParameter.Name)) continue;
            
            var schemaProperties = context.SchemaRepository.Schemas
                .Where(x => x.Key.Equals(subParameter.Name))
                .Select(x => x.Value.Properties).SingleOrDefault();
            
            if(schemaProperties == null) return;
            
            foreach (var property in subParameter.GetProperties())
            {
                var jsonPropertyAttribute = property.GetCustomAttribute<JsonPropertyAttribute>();
                                
                if (jsonPropertyAttribute?.PropertyName == null) continue;

                var needChangeProperty = schemaProperties.FirstOrDefault(x => string.Equals(x.Key, property.Name, comparisonType: StringComparison.OrdinalIgnoreCase));

                if (needChangeProperty.Key.IsNullOrEmpty()) continue;
                
                schemaProperties.Remove(needChangeProperty.Key);
                schemaProperties.Add(jsonPropertyAttribute.PropertyName, needChangeProperty.Value);
            }
        }
    }
}
```

然后在AddSwaggerGen中进行配置：

```
c.OperationFilter<SwaggerCustomSchemeOperationFilter>(
    typeof(xxxCommand), 
    new List<Type>
    {
        typeof(command中的子类), typeof(command中的子类)
    });
```

##  效果：

需要修改的子类：

![需要修改的子类](https://upload-images.jianshu.io/upload_images/20387877-252a723b4a403662.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

之前：

![之前](https://upload-images.jianshu.io/upload_images/20387877-5fb7ccb1096f2620.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

之后:

![之后](https://upload-images.jianshu.io/upload_images/20387877-c2072ac6e211db25.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
