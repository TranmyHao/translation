## LightInject 官网文档-中文译文
#### _<center>By 谭明豪</center>_
---
>安装

LightInject 通过NuGet提供两种发布版本。

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
```
``` csharp
    container.Register<IFoo, Foo>();

    var instance = container.GetInstance<IFoo>();

    Assert.IsInstanceOfType(instance, typeof(Foo));
```
---
> 命名服务(Named service)
```csharp
    public class Foo : IFoo{}

    public class AnotherFoo : IFoo{}
```
```csharp

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

通过调用```RegisterFallback```方法，LightInject能够解析未曾注册过的服务。
```csharp
    var container = new ServiceContainer();

    container.RegisterFallback((type, s) => true, request => new Foo());

    var foo = container.GetInstance<IFoo>();
```

```RegisterFallback```方法中的第一个参数用来表示服务是否可以“延迟解析”，第二个参数则是被请求服务的一个实例，目的是用来提供被请求服务的类型和服务名称。

---
> IEnumerable<T>
当注册的多个服务拥有相同的服务类型(Service Type)，LightInject能够将这些服务视为一个IEnumerable<T>。
``` csharp
    public class Foo : IFoo {}
    public calss AnotherFoo : IFoo {}
```
``` csharp
    container.Register<IFoo, Foo>();

    container.Register<IFoo, AnotherFoo>("AnotherFoo");

    var instances = container.GetInstance<IEnumerable<IFoo>>()

    Assert.AreEqual(2, instances.Count());
```
也可以使用```GetAllInstances```方法代替。
``` csharp
    var instance = container.GetAllInstances<IFoo>();

    Assert.AreEqual(2, instance.Count());
```
另外，LightInject支持下列```IEnumerable<T>```子类型。
+ Array
+ ICollection<T>
+ IList<T>
+ IReadOnlyCollection<T> (Net 4.5 and Windows Runtime)
+ IReadOnlyList<T> (Net 4.5 and Windows Runtime)

LightInject默认将解析所有与请求元素类型相匹配的服务。

``` csharp
    container.Register<Foo>();
    
    container.Register<DerivedFoo>();
    
    var instances = container.GetAllInstances<Foo>();
    
    Assert.AreEqual(2, instances.Count());
```
这种行为可以通过修改容器的```EnableVariance```选项修改。
``` csharp
    var container = new ServiceContainer(new ContainerOptions { EnableVariance = false });
    
    container.Register<Foo>();
    
    container.Register<DerivedFoo>();
    
    var instances = container.GetAllInstances<Foo>();
    
    Assert.AreEqual(1, instances.Count());
```
---
> 服务顺序

有时了解多个服务在容器中的顺序对应用程序非常重要，LightInject通过按照服务名称来处理服务的顺序问题。
```csharp
    container.Register<IFoo, Foo1>("A");
    container.Register<IFoo, Foo2>("B");
    container.Register<IFoo, Foo3>("C");

    var instances = container.GetAllInstances<IFoo>().ToArray();
    
    Assert.IsType<Foo1>(instances[0]);
    Assert.IsType<Foo2>(instances[1]);   
    Assert.IsType<Foo3>(instances[2]);
```
在给定服务类型的时候，我们也可以通过使用```RegisterOrdered```方法来按顺序注册多个服务的实现。

``` csharp
    var container = CreateContainer();
    
    container.RegisterOrdered(typeof(IFoo), new[] {typeof(Foo1), typeof(Foo2), typeof(Foo3)},
        type => new PerContainerLifetime());

    var instances = container.GetAllInstances<IFoo>().ToArray();

    Assert.IsType<Foo1>(instances[0]);
    Assert.IsType<Foo2>(instances[1]);
    Assert.IsType<Foo3>(instances[2]);
```

```RegisterOrdered```方法给每个实现起了一个可用的服务名称。默认的服务名称形如```001``` ```002```等，也可以通过传入一个格式化函数来改变这个默认的规定。

``` csharp
    container.RegisterOrdered(typeof(IFoo<>), new[] { typeof(Foo1<>), typeof(Foo2<>), typeof(Foo3<>) },
    type => new PerContainerLifetime(), i => $"A{i.ToString().PadLeft(3,'0')}");

    var services = container.AvailableServices.Where(sr => sr.ServiceType == typeof(IFoo<>))
        .OrderBy(sr => sr.ServiceName).ToArray();

    Assert.Equal("A001", services[0].ServiceName);
    Assert.Equal("A002", services[1].ServiceName);
    Assert.Equal("A003", services[2].ServiceName);
```
---

> 常量

可以直接注册一些值作为常量。（总感觉这个功能怪怪的）

``` csharp
    container.RegisterInstance<string>("SomeValue");
    var value = container.GetInstance<string>();
    Assert.AreEqual("SomeValue, value);
```