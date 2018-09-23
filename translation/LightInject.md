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
---

> 编译

LightInject 通过使用```System.Reflection.Emit```或编译后的表达式树的形式使用动态代码编译。当从容器中请求一个服务的时候，生成服务实例的代码会被生成和编译，然后，LightInject会生成一个指向编译后的代码的委托，用于支持后续对该服务的请求查询。因此，生成服务实例的代码只会在首次请求的时候会被编译一次。这些委托保存在AVL树中，保证了在给定服务类型的条件下查找对应服务委托的最优性能。事实上，拉开容器性能差距的正是容器如何处理从数据结构中查找这些委托。大部分高性能的容器时候的代码都非常类似，但是用来存储委托的方式可能稍有不同。

LightInject实现了无锁的服务查找。意思是在在服务实例生成和编译后，请求获取服务的过程不会加锁。事实上，LightInject唯一使用锁的地方就是在生成给定服务代码的时候。所以，当存在大量并发的对不同服务的首次请求时，确实会潜存锁竞争的问题。

为了解决这个潜在的性能问题，LightInject提供了一个用在程序开始时手动编译的API。这个API将会立刻编译所有注册过的服务。

```csharp
    container.Compile();
```

值得提醒的是：并不是所有的服务都会有生成自己对应的委托。
考虑如下服务：
``` csharp
    public class Foo
    {
        public Foo(Bar bar)
        {
            Bar = bar;
        }
    }
```
注册和解析过程如下：
``` csharp
    container.Register<Foo>();
    container.Register<Bar>();
    var foo = container.GetInstance<Foo>();
``` 
在本例中，LightInject只会为```Foo```生成委托，因为只有它被直接请求。生成```Bar```实例的代码被内嵌在了生成```Foo```实例的代码中，因此只有一个指向生成```Foo```实例的代码的委托被生成。
像```Foo```这样被直接从容器中请求的服务被称为```根服务```

瞧瞧生成```Foo```实例的IL代码：

``` IL
    newobj Void .ctor() //Bar
    newobj Void .ctor(LightInject.SampleLibrary.IBar) //Foo
```

这段代码显示：一个```Bar```的实例被生成并被压入栈中，之后```Foo```的实例被生成。这就是指向```Foo```的委托对应的代码。

现在我们知道了：LightInject并不一定会为一个给定的服务生成委托代码，因此，粗暴的调用```container.Compile()```可能会导致很多委托代码被生成但却从未被调用。通常情况下这也并不是什么大问题，但当我们有上万个服务，就不得不考虑到这种情况。

出于各种理由，LightInject不会对根服务进行验证。

替代方案是在编译服务前使用断言：
``` csharp
    container.Compiler(sr => sr.ServiceType == typeof(Foo));
```
---

> Open Generics()略

---
> 生命周期


