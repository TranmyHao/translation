## LightInject [官网文档](https://www.lightinject.net/)-中文译文
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

LightInject 通过使用```System.Reflection.Emit```或编译后的表达式树的形式使用动态代码编译。当从容器中请求一个服务的时候，生成服务实例的代码会被生成和编译，然后，LightInject会生成一个指向编译后的代码的委托，用于支持后续对该服务的请求查询。因此，生成服务实例的代码只会在首次请求的时候会被编译一次。这些委托保存在AVL树中，保证了在给定服务类型的条件下查找对应服务委托的最优性能。事实上，拉开容器性能差距的正是容器如何处理从数据结构中查找这些委托。大部分高性能的容器时候的代码都非常类似，但是用来存储委托的方式可能稍有不同。

LightInject实现了无锁的服务查找。意思是在服务实例生成和编译后，请求获取服务的过程不会加锁。事实上，LightInject唯一使用锁的地方就是在生成给定服务代码的时候。所以，当存在大量并发的对不同服务的首次请求时，确实会潜存锁竞争的问题。

为了解决这个潜在的性能问题，LightInject提供了一个用在程序开始时手动编译的API。这个API将会立刻编译所有注册过的服务。

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
像```Foo```这样被直接从容器中请求的服务被称为```根服务```

瞧瞧生成```Foo```实例的IL代码：

``` IL
    newobj Void .ctor() //Bar
    newobj Void .ctor(LightInject.SampleLibrary.IBar) //Foo
```

这段代码显示：一个```Bar```的实例被生成并被压入栈中，之后```Foo```的实例被生成。这就是指向```Foo```的委托对应的代码。

现在我们知道了：LightInject并不一定会为一个给定的服务生成委托代码，因此，粗暴的调用```container.Compile()```可能会导致很多委托代码被生成但却从未被调用。通常情况下这也并不是什么大问题，但当我们有上万个服务，就不得不考虑到这种情况。

出于各种理由，LightInject不会对根服务进行验证。

替代方案是在编译服务前使用断言：
``` csharp
    container.Compiler(sr => sr.ServiceType == typeof(Foo));
```
---

> Open Generics()略

---
> 生命周期

除非额外指明，LightInject默认把所有对象当做暂时的（不会复用）。

``` csharp
    container.Register<IFoo,Foo>();
    var firstInstance = container.GetInstance<IFoo>();
    var secondInstance = container.GetInstance<IFoo>();
    Assert.AreNotSame(firstInstance, secondInstance);
```

> PerScopeLifetime(每作用域生命周期)

保证给定服务在一个作用域下仅有一个实例。容器会负责调用所有在作用域内生成的disposable对象的```Dispose```方法。

``` csharp
    container.Register<IFoo,Foo>(new PerScopeLifetime());
    using(container.BeginScope())
    {

        var firstInstance = container.GetInstance<IFoo>();
        var secondInstance = container.GetInstance<IFoo>();
        Assert.AreSame(firstInstance, secondInstance);
    }
```

_注：如果一个服务在作用域中注册，但是却在作用域外被请求，则会抛出```InvalidOperationException```异常_

> PerContainerLifetime(每容器生命周期)

保证给定服务在一个容器中只会有一个实例。容器会在自己被释放的时候负责调用所有disposable对象的```Dispose```方法。

``` csharp
    using(container = new ServiceContainer())
    {
        container.Register<IFoo,Foo>(new PerContainerLifetime());   
        var firstInstance = container.GetInstance<IFoo>();
        var secondInstance = container.GetInstance<IFoo>();
        Assert.AreSame(firstInstance, secondInstance);
    }
```

> PerRequestLifeTime(每请求生命周期)

为每次请求生成一个实例，并且在作用域结束的时候调用```Dispose```方法。这种类型的生命周期只在当工厂类实现了IDisposable接口时使用。

``` csharp
    container.Register<IFoo,Foo>(new PerRequestLifeTime());
    using(container.BeginScope())
    {       
        var firstInstance = container.GetInstance<IFoo>();
        var secondInstance = container.GetInstance<IFoo>();
        Assert.AreNotSame(firstInstance, secondInstance);
    }
```

_注：如果一个服务在作用域中注册，但是却在作用域外被请求，则会抛出```InvalidOperationException```异常_

> 自定义生命周期

通过实现 ```ILifetime``` 接口，可以创建自定义的生命周期。

``` csharp
    internal interface ILifetime
    {
        object GetInstance(Func<object> instanceFactory, Scope currentScope);        
    }
```

下面的例子展示了如何让实现一个保证每线程一个实例的自定义生命周期：

``` csharp
    public class PerThreadLifetime : ILifetime
    {
        ThreadLocal<object> instances = new ThreadLocal<object>();

        public object GetInstance(Func<object> instanceFactory, Scope currentScope)
        {
            if (instances.value == null)
            {
                instances.value = instanceFactory();
            }
            return instances.value;
        }
    }
```
但是如果服务是一个```disposable```的服务呢？情况会变得稍微复杂点：

``` csharp
    public class PerThreadLifetime : ILifetime
    {
        ThreadLocal<object> instances = new ThreadLocal<object>();

        public object GetInstance(Func<object> instanceFactory, Scope currentScope)
        {           
            if (instances.value == null)
            {               
                object instance = instanceFactory();                
                IDisposable disposable = instance as IDisposable;               
                if (disposable != null)
                {
                    if (currentScope == null)
                    {
                        throw new InvalidOperationException("Attempt to create an disposable object 
                                                            without a current scope.")
                    }
                    currentScope.TrackInstance(disposable);
                }

                instances.value = instance;
            }
            return instance.value;
        }
    }
```

_重要！：一个生命周期对象只用用于控制一个服务，决不能用与注册多个服务时共享_

错误姿势：
``` csharp
    ILifetime lifetime = new PerContainerLifeTime();
    container.Register<IFoo,Foo>(lifetime);
    container.Register<IBar,Bar>(lifetime);
```
正确姿势：
``` csharp
    container.Register<IFoo,Foo>(new PerContainerLifeTime());
    container.Register<IBar,Bar>(new PerContainerLifeTime());
```

一个生命周期对象能够被跨线程访问，这也是我们在开发一个新的生命周期实现的时候必须考虑的事。

---

> Async 和 Await

默认情况下，作用域被各自线程单独管理，这意味着当容器搜索当前作用域的时候，实际上搜索的是与当前线程绑定的作用域。

当引入 async/await模式的时候，则会存在请求一个绑定在别的线程上的服务实例的情况。

让我们通过示例来解释这种情况：

首先我们新建一个返回```Task<IBar>```的接口。
``` csharp
    public interface IAsuncFoo
    {
        Task<IBar> GetBar();
    }
```

接下来我们这样实现这个接口：让```IBar```的实例在别的线程被请求。

``` csharp
    public class AsyncFoo : IAsyncFoo
    {
        private readonly Lazy<IBar> lazyBar;

        public AsyncFoo(Lazy<IBar> lazyBar)
        {
            this.lazyBar = lazyBar;
        }

        public async Task<IBar> GetBar()
        {
            await Task.Delay(10);
            return lazyBar.Value; <--This code is executed on another thread (continuation).
        }
    }
```

接着我们使用PerScopeLifetime的方式注册依赖（```IBar```），这会使得实例被绑定在当前作用域。

``` csharp
    var container = new ServiceContainer();
    container.Register<IBar, Bar>(new PerScopeLifetime());
    container.Register<IAsyncFoo, AsyncFoo>();

    using (container.BeginScope())
    {
        var instance = container.GetInstance<IAsyncFoo>();
        ExceptionAssert.Throws<AggregateException>(() => instance.GetBar().Wait());                
    }
```
抛出的异常信息如下：
```
Attempt to create a scoped instance without a current scope.
```

原因上面已经解释过了：当前作用域被绑定在生成他的线程里，而接下来运行的代码，本质上却是在别的线程中请求获取服务的实例。

为了解决这个问题，LightInject现在支持跨越逻辑```调用上下文(CallContext)```。
``` csharp
    var container = new ServiceContainer();
    container.ScopeManagerProvider = new PerLogicalCallContextScopeManagerProvider();
    container.Register<IBar, Bar>(new PerScopeLifetime());
    container.Register<IAsyncFoo, AsyncFoo>();

    using (container.BeginScope())
    {
        var instance = container.GetInstance<IAsyncFoo>();
        var bar = instance.GetBar().Result;
        Assert.IsInstanceOfType(bar, typeof(IBar));
    }
```
_注意：```PerLogicalCallContextScopeManagerProvider```只支持在.Net 4.5以下运行。想要了解更多相关信息，可以查看Stephen Cleary的[文章](http://blog.stephencleary.com/2013/04/implicit-async-context-asynclocal.html)_

---

> 依赖（Dependencies）

构造函数注入
``` csharp
    public interface IFoo {}        
    public interface IBar {}

    public class Foo : IFoo
    {
        public Foo(IBar bar) 
        {
            Bar = bar;
        }

        public IBar Bar { get; private set; } 
    }

    public class Bar : IBar {}
```

隐式服务注册：
注册服务的时候不指定任何关于如何解析构造函数依赖的实现类型的信息的。
``` csharp
    container.Register<IFoo, Foo>();
    container.Register<IBar, Bar>();
    var foo = (Foo)container.GetInstance<IFoo>();
    Assert.IsInstanceOfType(foo.Bar, typeof(Bar));
```
_注意：当实现类型(Foo)拥有多个构造函数的时候，LightInject会选择参数最多的构造函数_

为了更好的控制构造函数依赖的注入，我们可以提供一个产生给定构造函数依赖的工厂。
``` csharp
    container.RegisterConstructorDependency<IBar>((factory, parameterInfo） => new Bar());
```
这等价于告诉容器每当遇见一个```IBar```的构造函数依赖的时候注入一个新的```Bar```实例。

显示服务注册：
在注册服务的时候提供了明确的信息：如何生成服务实例亿级如何解析构造函数依赖。

``` csharp
    container.Register<IBar, Bar>();
    container.Register<IFoo>(factory => new Foo(factory.GetInstance<IBar>));
    var foo = (Foo)container.GetInstance<IFoo>();
    Assert.IsNotNull(foo.Bar);
```
---

>参数：

当我们想在解析服务的时候提供一个或多个值的时候，可以使用参数。
``` csharp
    public class Foo : IFoo
    {
        public Foo(int value)
        {
            Value = value;
        }

        public int Value { get; private set; }
    }
```
``` csharp
    container.Register<int, IFoo>((arg, factory) => new Foo(arg));
    var foo = (Foo)container.GetInstance<int, IFoo>(42);
    Assert.AreEqual(42,foo.Value);
```
也可以同时提供值和依赖
``` csharp
    public class Foo : IFoo
    {
        public Foo(int value, IBar bar)
        {
            Value = value;
        }

        public int Value { get; private set; }
        public IBar Bar { get; private set; }
    }
```
``` csharp
    container.Register<IBar, Bar>();
    container.Register<int, IFoo>((factory, value) => new Foo(value, factory.GetInstance<IBar>()));
    var foo = (Foo)container.GetInstance<int, IFoo>(42);
    Assert.AreEqual(42, foo.Value);
    Assert.IsNotNull(foo.Bar);
```
---
 
> 属性注入

``` csharp
    public interface IFoo {}

    public interface IBar {}

    public class Foo : IFoo
    {
        public IBar Bar { get; set; }
    }

    public class Bar : IBar {}
```
隐式服务注册：
注册服务的时候没有明确指明如何解析一个属性依赖。
``` csharp
    container.Register<IFoo, Foo>();
    container.Register<IBar, Bar>();
    var foo = (Foo)container.GetInstance<IFoo>();
    Assert.IsNotNull(foo.bar);
```

_注意： LightInject把所有可读/写的属性作为依赖，但是在实现上却采取宽松策略，这意味着如果无法注入一个属性依赖将不会抛出任何异常_

为了更好的控制属性依赖注入，可以提供一个产生被注入依赖实例的工厂。
``` csharp
    container.RegisterPropertyDependency<IBar>((factory, propertyInfo) => new Bar());
```

这种写法告诉容器每当遇上一个```IBar```属性的时候都注入一个新的```Bar```实例。

显示服务注册：
注册服务的时候提供明确指令：如何创建服务实例以及如何解析属性依赖。
``` csharp
    container.Register<IBar, Bar>();
    container.Register<IFoo>(factory => new Foo() {Bar = factory.GetInstance<IBar>()}) 
    var foo = (Foo)container.GetInstance<IFoo>();
    Assert.IsNotNull(foo.bar);
```
 在已存在的实例上注入属性：
 有时候我们并不会在服务实例生成的时候进行控制，所以LightInject可以对已经存在的实例进行注入。

 ``` csharp
    container.Register<IBar, Bar>();
    var foo = new Foo();
    container.InjectProperties(foo);
    Assert.IsNotNull(foo);
 ```

 > 禁用属性注入

 属性注入默认是起用的，禁用方法如下：
 ``` csharp
    var container = new ServiceContainer(new ContainerOptions { EnablePropertyInjection = false });
 ```
 _注意：事实上强烈建议关掉属性注入，除非真的强烈需要这个特性。仅仅是出于向后兼容的原因，才没有默认禁用属性注入。_

---

> 初始化程序

使用```Initialize```方法可以执行服务实例初始化处理。

``` csharp
    container.Register<IFoo, FooWithPropertyDependency>();
    container.Initialize(registration => registration.ServiceType == typeof(IFoo), 
        (factory, instance) => ((FooWithPropertyDependency)instance).Bar = new Bar());
    var foo = (FooWithProperyDependency)container.GetInstance<IFoo>();
    Assert.IsInstanceOfType(foo.Bar, typeof(Bar));
```
---

> 程序集扫描

LightInject能够在给定程序集中查找类型来注册服务。
``` csharp
    container.RegisterAssembly(typeof(IFoo).Assembly)
```
为了过滤出要注入进容器的服务，可以使用断言来判断服务类型和实现类型。
``` csharp
    container.RegisterAssembly(typeof(IFoo).Assembly, (serviceType, implementingType) => serviceType.NameSpace == "SomeNamespace");
```
也可以继续搜索模板来扫描一组程序集文件。
``` csharp
    container.RegisterAssembly("SomeAssemblyName*.dll");  
```
当扫描程序集的时候，LightInject会使用实现类型的名字作为默认的服务名称。改该行为可以通过指定一个基于服务类型和实现类型提供名字的函数委托来改变。

``` csharp
    container.RegisterAssembly(
        typeof(IFoo).Assembly, 
        () => new PerContainerLifetime(), 
        (serviceType, implementingType) => serviceType.NameSpace == "SomeNamespace",
        (serviceType, implementingType) => "Provide custom service name here");
```

也可以实现```IServiceNameProvider```接口来全局的改变所有注册服务的命名。

``` csharp
    public class CustomServiceNameProvider : IServiceNameProvider
    {
        public string GetServiceName(Type serviceType, Type implementingType)
        {
            return "Provide custom service name here";  
        }
    }
```
只需要在开始扫描程序集之前改变容器的属性即可：
``` csharp
    container.ServiceNameProvider = new CustomServiceNameProvider();
```

---

> 聚合根（Composition Root）

当LightInject扫描程序集的时候，它会寻找一个```ICompositionRoot```接口的实现.
``` csharp
    public class SampleCompositionRoot : ICompositionRoot
    {               
        public void Compose(IServiceRegistry serviceRegistry)
        {     
            serviceRegistry.Register(typeof(IFoo),typeof(Foo));
        }
    }
```
如果找到一个或多个该接口的实现，这些实现都会被执行。

_注意：任何包含在目标程序集中但是没有在聚合根中注册的其它服务，都不会被注册_

程序一般不会只有一个聚合根，因为这意味着需要引用所有其它程序集。拥有多个聚合根可以让服务以组的方式更自然的组合在一起。另一个在聚合根中注册服务的好处是：注册的代码能更容易的在自动化测试中重用。

---

> 懒聚合根（Lazy Composition Roots）

LightInject具备按需注册服务的能力。对于一个含有超多服务的大型应用，在一开是注册所有的服务可能并不是最好的方式，因为大量的程序集加载会使得启动时间大大增加。

如果请求一个未注册的服务，LightInject会去含有该服务的程序集中搜索。

---

> 聚合根特性（CompositionRootAttribute）

当程序集被扫描的时候，LightInject会寻找实现了```ICompositionRoot```接口的实现。对于含有大量的程序集而程序集有包含大量类型的程序来说，这将是消耗巨大的操作。而```CompositionRootAttribute```正是一个程序集级别的特性，它能够轻松的帮助LightInject定位聚合根。
``` csharp
[assembly: CompositionRootType(typeof(SampleCompositionRoot))]
```

> 注册源（RegisterFrom）

允许显示地运行一个聚合根：
``` csharp
    container.RegisterFrom<SampleCompositionRoot>();
```
---

> 泛型（Generics）

``` csharp
    public interface IFoo<T> {};
    public class Foo<T> : IFoo<T> {};
```
容器会基于服务请求生成闭合的泛型类型(closed generic type)。
``` csharp
    container.Register(typeof(IFoo<>), typeof(Foo<>));
    var instance = container.GetInstance(typeof(IFoo<int>));
    Assert.IsInstanceOfType(instance, typeof(Foo<int>));
```

> 约束（Constraints）

LightInject 实施通用约束。（然后没有更多描述了。。。）

---

> ```Lazy<T>```

LightInject可以将服务解析为```Lazy<T>```的一个实例。这样当我们想要延迟解析服务直到真的需要的时候。
``` csharp
    public interface IFoo {}
    public class Foo : IFoo {}
```
``` csharp
    container.Register<IFoo, Foo>();
    var lazyFoo = container.GetInstance<Lazy<IFoo>>();
    Assert.IsNotNull(lazyFoo.Value);
```

---

> 函数工厂（Function Factories）

_未完待续..._