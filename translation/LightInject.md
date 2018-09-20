## LightInject 官网文档-中文译文
#### _<center>By 谭明豪</center>_
---
>安装

LightInject 通过NuGet提供两种发布版本。

二进制
``` bash
    PM> Install-Package LightInject
```

这种方式直接向目标工程添加一个指向LightInject.dll的引用。

源码

``` bash
    PM> Install-Package LightInject.Source
```

这种方式会向当前工程添加一个源码文件(LightInject.cs)。

---
> 创建容器(Container)

``` Csharp
    var container = new LightInject.ServiceContainer();
```
LightInject中的容器(Container)实现了IDisposable接口，使用完毕后应当正确销毁。当然，在using语句这样的限制域中使用也是非常合适的。

---
>默认服务(Default service)

```csharp
    public interface IFoo{}

    public class Foo : IFoo{}

    container.Register<IFoo, Foo>();

    var instance = container.GetInstance<IFoo>();

    Assert.IsInstanceOfType(instance, typeof(Foo));
```
---
> 命名服务(Named service)
```csharp
    public class Foo : IFoo{}

    public class AnotherFoo : IFoo{}

    container.Register<IFoo, Foo>();

    container.Register<IFoo, AnotherFoo>("AnotherFoo");

    var instance = container.GetInstance<IFoo>("AnotherFoo");

    Assert.IsInstanceOfType(instance, typeof(AnotherFoo));
```
如果只注册了一个命名服务，LightInject能够将该命名服务当做默认服务解析。
```csharp
container.Register<IFoo, AnotherFoo>("AnotherFoo");
var instance = container.GetInstance<IFoo>();
Assert.IsInstanceOfType(instance, typeof(AnotherFoo));
```
---
> 未解析的的服务

通过调用```RegisterFallback```方法，LightInject能够解析未曾注册过的服务。
```csharp
    var container = new ServiceContainer();
    container.RegisterFallback((type, s) => true, request => new Foo());
    var foo = container.GetInstance<IFoo>();
```
