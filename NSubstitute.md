# 一、 什么是NSubstitute
在单元测试中，通常需要模拟一些依赖项以便能够隔离测试的代码，以确保代码在不同的情况下的行为符合预期。NSubstitute通过提供一个简洁的语法和强大的功能来帮助我们创建模拟对象。

它允许创建和配置假的（mock）对象，以便在测试代码中模拟其他对象的行为。使用NSubstitute可以编写更容易维护和可测试的代码。

使用NSubstitute可以轻松地模拟接口、虚方法和抽象类，可以设置模拟对象的返回值、抛出异常、记录调用和验证调用等。它还支持对模拟对象的成员进行调用顺序的检查、模拟对象的任意数量的返回值等高级功能。

[具体请查阅官方文档](https://nsubstitute.github.io/help/getting-started/)

简易示例：
```
//模拟一个计算器
var calculator = Substitute.For<ICalculatorService>();
//模拟其Add方法最终返回5
calculator.Add(1, 2).Returns(5);
calculator.Add(1, 2).ShouldBe(3); //提示报错，应该等于5

//模拟其返回值为DEC
calculator.Mode.Returns("DEC");
calculator.Mode.ShouldBe("DEC"); //测试成功

//判断有收到Add(1,2)和没有收到Add(5,7)的Call
calculator.Add(1, 2);
calculator.Received().Add(1, 2);
calculator.DidNotReceive().Add(5, 7);

//替代更多的行为
calculator
    .Add(Arg.Any<int>(), Arg.Any<int>())
    .Returns(x => (int)x[0] + (int)x[1]);
calculator.Add(5, 10).ShouldBe(15); 
```
# 二、NSubstitute是如何运作的
从[官方文档](https://nsubstitute.github.io/help/how-nsub-works/)中我们可以看到他用到的是[Castle DynamicProxy 库](https://github.com/castleproject/Core)来实现动态代理去mock一个实例。

我先来介绍一下什么是代理模式
## 1. 代理模式
有一天，小明要结婚了，他有两种方式来完成结婚这件事。

* 第一种就是小明亲力亲为，包揽婚礼所有事情。
第二种就是找婚庆公司来帮忙筹办婚礼，小明只需要人来就行。这里的婚庆公司就是代理，婚礼的筹办过程就是代理模式的应用。
在这个例子里，一共有3个角色：

* 小明：真实角色，因为是小明要结婚。
婚庆公司：代理角色，帮小明处理婚礼事宜。
结婚（接口）：抽象角色，这个是小明和婚庆公司都要做的事情，只是两者具体做的内容不一样

## 2. 动态代理和静态代理的区别

* 静态：婚庆公司只帮小明一个人搞婚礼

* 动态：婚庆公司可以根据不同人处理不同婚礼

## 3. 动态代理和静态代理的实现
* 静态：
```
class StaticProxy
{
    public interface ISubject
    {
        void Request(string message);
    }

    public class RealSubject : ISubject
    {
        public void Request(string message)
        {
            Console.WriteLine($"RealSubject: {message}");
        }
    }


    public class Proxy : ISubject
    {
        private readonly RealSubject _realSubject;

        public Proxy(RealSubject realSubject)
        {
            _realSubject = realSubject;
        }

        public void Request(string message)
        {
            if (CheckAccess())
            {
                _realSubject.Request(message);
                LogRequest(message);
            }
        }

        private bool CheckAccess()
        {
            Console.WriteLine("Proxy: Checking access prior to sending a real request.");
            return true;
        }

        private void LogRequest(string message)
        {
            Console.WriteLine($"Proxy: Logging request: {message}");
        }
    }

    public static void main()
    {
        var realSubject = new RealSubject();
        var proxy = new Proxy(realSubject);

        proxy.Request("Hello World");

    }
}
```

* 动态：
```
using System.Reflection;

class DynamicProxy
{
    // 定义一个接口
    public interface IService
    {
        void Request();
    }

    // 实现该接口的服务类
    public class Service : IService
    {
        public void Request()
        {
            Console.WriteLine("Service is handling the request.");
        }
    }   

    // 定义一个动态代理类
    public class DynamicProxy : DispatchProxy
    {
        private IService _service;

        protected override object Invoke(MethodInfo targetMethod, object[] args)
        {
            Console.WriteLine("Dynamic proxy is handling the request.");
            return targetMethod.Invoke(_service, args);
        }

        public static T Create<T>(IService service) where T : class
        {
            object proxy = Create<T, DynamicProxy>();
            ((DynamicProxy)proxy)._service = service;
            return (T)proxy;
        }
    }

    // 使用动态代理类
    static void Main(string[] args)
    {
        IService service = new Service();
        IService proxy = DynamicProxy.Create<IService>(service);
        proxy.Request();
    }
}
```
