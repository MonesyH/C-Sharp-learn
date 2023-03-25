1. 定义一个接口或抽象类，作为产品的基类。

2. 定义具体产品类，继承自基类。

3. 定义工厂类，包含一个方法，根据传入的参数创建不同的产品对象。

4. 在客户端代码中，通过工厂类创建产品对象，而不是直接实例化具体产品类。

以下是一个示例代码：

//定义产品接口 
```
interface IProduct { void Show(); }
```

//具体产品类1 
```
class Product1 : IProduct 
{ 
    public void Show() 
    { 
      Console.WriteLine("This is product 1."); 
    } 
}
```

//具体产品类2 
```
class Product2 : IProduct 
{ 
    public void Show() 
    { 
      Console.WriteLine("This is product 2."); 
    } 
}
```

//工厂类 
```
class Factory 
{ 
    private Dictionary<string, Type> _products = new Dictionary<string, Type>();

    public void RegisterProduct<T>(string type) where T : IProduct
    {
        _products.Add(type, typeof(T));
    }

    public IProduct CreateProduct(string type)
    {
        if (_products.ContainsKey(type))
        {
            Type productType = _products[type];
            return Activator.CreateInstance(productType) as IProduct;
        }
        else
        {
            throw new ArgumentException("Invalid type.");
        }
    }
}

```

//客户端代码 
```
Factory factory = new Factory(); 
factory.RegisterProduct<Product1>("1"); 
factory.RegisterProduct<Product2>("2"); 
IProduct product1 = factory.CreateProduct("1"); 
IProduct product2 = factory.CreateProduct("2"); 
product1.Show(); product2.Show();
```

输出结果为：
This is product 1. This is product 2.
