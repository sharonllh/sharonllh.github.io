---
layout: post
title: .NET Core依赖注入中的Scoped Dependency是什么？
date: 2024-12-20 +0800
tags: [.NET, DI]
categories: [.NET]
---

在.NET Core自带的依赖注入框架中，依赖的生命周期有三种：
- Singleton
- Scoped
- Transient

Singleton和Transient都比较好理解，Scoped的概念却比较含糊。本文对Scoped Dependency进行深入探究。

## Scoped Dependency含义

下面是官方文档[.NET dependency injection](https://learn.microsoft.com/en-us/dotnet/core/extensions/dependency-injection#scoped)的解释：

```
For web applications, a scoped lifetime indicates that services are created once per client request (connection). Register scoped services with AddScoped.

In apps that process requests, scoped services are disposed at the end of the request.
```

可以看到，官方文档并没有给出一般化的解释，而是从Web App的角度去描述的——Scoped Dependency是request级别的，对每个request只创建一次，在request完成后被dispose。

这个解释不能说不对，但是不够清晰。所谓的Scoped Dependency，其实就是在一个scope中只创建一次的依赖，在scope被dispose时，创建的依赖也会被dispose。这里的scope可以是一个request，也可以是任何自定的范围。

具体地，每个scope是一个`IServiceScope`对象，其中包含一个`IServiceProvider`属性，可以用来解析依赖，源码如下：

```cs
/// <summary>
/// The <see cref="System.IDisposable.Dispose"/> method ends the scope lifetime. Once Dispose
/// is called, any scoped services that have been resolved from
/// <see cref="Microsoft.Extensions.DependencyInjection.IServiceScope.ServiceProvider"/> will be
/// disposed.
/// </summary>
public interface IServiceScope : IDisposable
{
    /// <summary>
    /// The <see cref="System.IServiceProvider"/> used to resolve dependencies from the scope.
    /// </summary>
    IServiceProvider ServiceProvider { get; }
}
```

在.NET Core Web API中，会在每个request开始时创建一个scope，并在request结束时dispose这个scope。可以通过`HttpContext.RequestServices`来获取这个scope的Service Provider。

下面的例子很好地展示了Scoped Dependency的含义，以及三种依赖的区别：

```cs
// Dependency.cs
public class SingletonDependency : Dependency
{
}

public class ScopedDependency : Dependency
{
}

public class TransientDependency : Dependency
{
}

public class Dependency
{
    private readonly Guid id;

    public Dependency()
    {
        this.id = Guid.NewGuid();
    }

    public override string? ToString()
    {
        return $"{this.GetType().Name}: {this.id}";
    }
}

// Program.cs
public class Program
{
    public static void Main(string[] args)
    {
        var builder = WebApplication.CreateBuilder(args);

        // Add services to the container.

        builder.Services.AddControllers();

        builder.Services.AddSingleton<SingletonDependency>();
        builder.Services.AddScoped<ScopedDependency>();
        builder.Services.AddTransient<TransientDependency>();

        var app = builder.Build();

        // Configure the HTTP request pipeline.

        app.UseHttpsRedirection();

        app.UseAuthorization();

        app.MapControllers();

        app.Run();
    }
}

// TestsController.cs
[Route("api/[controller]")]
[ApiController]
public class TestsController : ControllerBase
{
    private readonly SingletonDependency singletonDependency;
    private readonly ScopedDependency scopedDependency;
    private readonly TransientDependency transientDependency;

    public TestsController(SingletonDependency singletonDependency, ScopedDependency scopedDependency, TransientDependency transientDependency)
    {
        this.singletonDependency = singletonDependency;
        this.scopedDependency = scopedDependency;
        this.transientDependency = transientDependency;
    }

    [HttpGet]
    public ActionResult Get()
    {
        Console.WriteLine("\n---Dependencies injected in constructor---");
        Console.WriteLine(this.singletonDependency);
        Console.WriteLine(this.scopedDependency);
        Console.WriteLine(this.transientDependency);

        var singleton1 = this.HttpContext.RequestServices.GetService<SingletonDependency>();
        var scoped1 = this.HttpContext.RequestServices.GetService<ScopedDependency>();
        var transient1 = this.HttpContext.RequestServices.GetService<TransientDependency>();

        Console.WriteLine("\n---Dependencies resolved from RequestServices---");
        Console.WriteLine(singleton1);
        Console.WriteLine(scoped1);
        Console.WriteLine(transient1);

        using (var scope = this.HttpContext.RequestServices.CreateScope())
        {
            var singleton2 = scope.ServiceProvider.GetService<SingletonDependency>();
            var scoped2 = scope.ServiceProvider.GetService<ScopedDependency>();
            var transient2 = scope.ServiceProvider.GetService<TransientDependency>();

            Console.WriteLine("\n---Dependencies resolved from a scope---");
            Console.WriteLine(singleton2);
            Console.WriteLine(scoped2);
            Console.WriteLine(transient2);
        }

        using (var scope = this.HttpContext.RequestServices.CreateScope())
        {
            var singleton3 = scope.ServiceProvider.GetService<SingletonDependency>();
            var scoped3 = scope.ServiceProvider.GetService<ScopedDependency>();
            var transient3 = scope.ServiceProvider.GetService<TransientDependency>();

            Console.WriteLine("\n---Dependencies resolved from a new scope---");
            Console.WriteLine(singleton3);
            Console.WriteLine(scoped3);
            Console.WriteLine(transient3);
        }

        var singleton4 = this.HttpContext.RequestServices.GetService<SingletonDependency>();
        var scoped4 = this.HttpContext.RequestServices.GetService<ScopedDependency>();
        var transient4 = this.HttpContext.RequestServices.GetService<TransientDependency>();

        Console.WriteLine("\n---Dependencies resolved from RequestServices again---");
        Console.WriteLine(singleton4);
        Console.WriteLine(scoped4);
        Console.WriteLine(transient4);

        this.HttpContext.Features.Set<IServiceProvidersFeature>(null);
        var singleton5 = this.HttpContext.RequestServices.GetService<SingletonDependency>();
        var scoped5 = this.HttpContext.RequestServices.GetService<ScopedDependency>();
        var transient5 = this.HttpContext.RequestServices.GetService<TransientDependency>();

        Console.WriteLine("\n---Dependencies resolved from a new RequestServices---");
        Console.WriteLine(singleton5);
        Console.WriteLine(scoped5);
        Console.WriteLine(transient5);

        return new OkObjectResult("Success");
    }
}
```

在浏览器中调用`http://localhost:XXX/api/tests`，输出如下：

```
---Dependencies injected in constructor---
SingletonDependency: c7ca604d-4f7a-4101-bb8e-dd129429d6d1
ScopedDependency: 19ea6e60-e33d-4ae1-b57c-b10cead6a6a4
TransientDependency: 91471f22-7573-4198-a035-466e95577415

---Dependencies resolved from RequestServices---
SingletonDependency: c7ca604d-4f7a-4101-bb8e-dd129429d6d1
ScopedDependency: 19ea6e60-e33d-4ae1-b57c-b10cead6a6a4
TransientDependency: 5b7309b1-0593-46f0-968a-4c652b1973b7

---Dependencies resolved from a scope---
SingletonDependency: c7ca604d-4f7a-4101-bb8e-dd129429d6d1
ScopedDependency: 51f9f85b-f7d3-4a7f-a01c-85ecc95bd668
TransientDependency: 238c8533-3bd9-46c3-8756-29af49e0097d

---Dependencies resolved from a new scope---
SingletonDependency: c7ca604d-4f7a-4101-bb8e-dd129429d6d1
ScopedDependency: c01cd3a4-9374-4c00-978c-faa5bcf898e5
TransientDependency: 026e0433-a06e-40ec-805c-37896390d492

---Dependencies resolved from RequestServices again---
SingletonDependency: c7ca604d-4f7a-4101-bb8e-dd129429d6d1
ScopedDependency: 19ea6e60-e33d-4ae1-b57c-b10cead6a6a4
TransientDependency: bfbcc1dd-8e31-4842-9149-bd73c3052620

---Dependencies resolved from a new RequestServices---
SingletonDependency: c7ca604d-4f7a-4101-bb8e-dd129429d6d1
ScopedDependency: e4ddad1c-5e59-4e81-9849-89074221258b
TransientDependency: 1d968e26-0cfd-44f7-8fb3-8888d8eed3e8
```

再次调用，输出如下：

```
---Dependencies injected in constructor---
SingletonDependency: c7ca604d-4f7a-4101-bb8e-dd129429d6d1
ScopedDependency: d9c06190-da0e-4297-a1be-d96db6718bc5
TransientDependency: c4d0bd16-00bd-41c4-93c4-e3a3acdf9a02

---Dependencies resolved from RequestServices---
SingletonDependency: c7ca604d-4f7a-4101-bb8e-dd129429d6d1
ScopedDependency: d9c06190-da0e-4297-a1be-d96db6718bc5
TransientDependency: 836ba1d1-da91-4b63-8e22-4917e6ddb64c

---Dependencies resolved from a scope---
SingletonDependency: c7ca604d-4f7a-4101-bb8e-dd129429d6d1
ScopedDependency: 2797c148-0a7b-4c4f-90b4-c5efe12e35ee
TransientDependency: c48642b7-a40a-4131-8bdd-4bdb92831c76

---Dependencies resolved from a new scope---
SingletonDependency: c7ca604d-4f7a-4101-bb8e-dd129429d6d1
ScopedDependency: 08e22f26-9a5a-47fa-8aa9-5bbbb94ddba0
TransientDependency: 22310b3a-6e73-4052-b467-d81a69cedcda

---Dependencies resolved from RequestServices again---
SingletonDependency: c7ca604d-4f7a-4101-bb8e-dd129429d6d1
ScopedDependency: d9c06190-da0e-4297-a1be-d96db6718bc5
TransientDependency: 5c28a617-59d1-4fde-bed5-2f5c6aadb7eb

---Dependencies resolved from a new RequestServices---
SingletonDependency: c7ca604d-4f7a-4101-bb8e-dd129429d6d1
ScopedDependency: 06a035f1-e626-4a4a-96d3-5f45809f20d0
TransientDependency: 42825c8e-0248-4e06-92ce-8ee7c453d899
```

可以看到：
- 每次解析Singleton，得到的都是同一个对象。
- 每次解析Transient，得到的都是不同的对象。
- 对于同一个request，
    - 通过构造器注入和`RequestServices`去解析Scoped Dependency，得到的是同一个对象。
    - 创建一个scope去解析Scoped Dependency，会得到新的对象。
    - 通过设置`IServiceProvidersFeature`，可以得到一个新的`RequestServices`，从而解析出新的Scoped Dependency。

## RequestServices源码解析

通过`HttpContext.RequestService`可以获得request scoped的Service Provider，这是怎么实现的呢？源码主要涉及3个类。

### DefaultHttpContext

在`DefaultHttpContext`中，通过底层的`IServiceProvidersFeature`来获取`RequestServices`。获取这个feature并不是每次都会从feature collection读，会有一层cache，封装在`_features`中。

```cs
public sealed class DefaultHttpContext : HttpContext
{
    private static readonly Func<DefaultHttpContext, IServiceProvidersFeature> _newServiceProvidersFeature = context => new RequestServicesFeature(context, context.ServiceScopeFactory);

    private FeatureReferences<FeatureInterfaces> _features;

    public DefaultHttpContext(IFeatureCollection features)
    {
        _features.Initalize(features);
        _request = new DefaultHttpRequest(this);
        _response = new DefaultHttpResponse(this);
    }

    private IServiceProvidersFeature ServiceProvidersFeature =>
        _features.Fetch(ref _features.Cache.ServiceProviders, this, _newServiceProvidersFeature)!;

    public override IServiceProvider RequestServices
    {
        get { return ServiceProvidersFeature.RequestServices; }
        set { ServiceProvidersFeature.RequestServices = value; }
    }

    struct FeatureInterfaces
    {
        public IItemsFeature? Items;
        public IServiceProvidersFeature? ServiceProviders;
        public IHttpAuthenticationFeature? Authentication;
        public IHttpRequestLifetimeFeature? Lifetime;
        public ISessionFeature? Session;
        public IHttpRequestIdentifierFeature? RequestIdentifier;
    }
}
```

### FeatureReferences

通过`FeatureReferences`获取一个feature，会先读cache，再从feature collection中读，最后根据传入的`factory`来创建。

当feature collection有变化时，`Revision`会改变，此时会清掉所有feature的cache。

```cs
public struct FeatureReferences<TCache>
{
    public void Initalize(IFeatureCollection collection)
    {
        Revision = collection.Revision;
        Collection = collection;
    }

    public IFeatureCollection Collection { get; private set; }

    public int Revision { get; private set; }

    // cache is a public field because the code calling Fetch must
    // be able to pass ref values that "dot through" the TCache struct memory,
    // if it was a Property then that getter would return a copy of the memory
    // preventing the use of "ref"
    public TCache? Cache;

    // Careful with modifications to the Fetch method; it is carefully constructed for inlining
    // See: https://github.com/aspnet/HttpAbstractions/pull/704
    [MethodImpl(MethodImplOptions.AggressiveInlining)]
    public TFeature? Fetch<TFeature, TState>(
        ref TFeature? cached,
        TState state,
        Func<TState, TFeature?> factory) where TFeature : class?
    {
        var flush = false;
        var revision = Collection?.Revision ?? ContextDisposed();
        if (Revision != revision)
        {
            // Clear cached value to force call to UpdateCached
            cached = null!;
            // Collection changed, clear whole feature cache
            flush = true;
        }

        return cached ?? UpdateCached(ref cached!, state, factory, revision, flush);
    }

    // Update and cache clearing logic, when the fast-path in Fetch isn't applicable
    private TFeature? UpdateCached<TFeature, TState>(ref TFeature? cached, TState state, Func<TState, TFeature?> factory, int revision, bool flush) where TFeature : class?
    {
        if (flush)
        {
            // Collection detected as changed, clear cache
            Cache = default;
        }

        cached = Collection.Get<TFeature>();
        if (cached == null)
        {
            // Item not in collection, create it with factory
            cached = factory(state);
            // Add item to IFeatureCollection
            Collection.Set(cached);
            // Revision changed by .Set, update revision to new value
            Revision = Collection.Revision;
        }
        else if (flush)
        {
            // Cache was cleared, but item retrieved from current Collection for version
            // so use passed in revision rather than making another virtual call
            Revision = revision;
        }

        return cached;
    }
}
```

### RequestServicesFeature

`RequestServicesFeature`是`IServiceProvidersFeature`的实现类。

第一次调用`RequestServices`时，会创建一个scope，然后返回其Service Provider。之后再调用，返回的都是第一次创建的Service Provider。

当`RequestServicesFeature`被dispose时，会dispose创建出来的scope，从而dispose所有的Scoped Dependency。

有一个隐藏的小细节，在创建scope之前，调用了`_context.Response.RegisterForDisposeAsync(this)`，这行代码为`RequestServicesFeature`注册了dispose，当response完成时会自动调用`RequestServicesFeature`的dispose方法。

```cs
public class RequestServicesFeature : IServiceProvidersFeature, IDisposable, IAsyncDisposable
{
    private readonly IServiceScopeFactory? _scopeFactory;
    private IServiceProvider? _requestServices;
    private IServiceScope? _scope;
    private bool _requestServicesSet;
    private readonly HttpContext _context;

    public RequestServicesFeature(HttpContext context, IServiceScopeFactory? scopeFactory)
    {
        _context = context;
        _scopeFactory = scopeFactory;
    }

    public IServiceProvider RequestServices
    {
        get
        {
            if (!_requestServicesSet && _scopeFactory != null)
            {
                _context.Response.RegisterForDisposeAsync(this);
                _scope = _scopeFactory.CreateScope();
                _requestServices = _scope.ServiceProvider;
                _requestServicesSet = true;
            }
            return _requestServices!;
        }

        set
        {
            _requestServices = value;
            _requestServicesSet = true;
        }
    }

    public ValueTask DisposeAsync()
    {
        switch (_scope)
        {
            case IAsyncDisposable asyncDisposable:
                var vt = asyncDisposable.DisposeAsync();
                if (!vt.IsCompletedSuccessfully)
                {
                    return Awaited(this, vt);
                }
                // If its a IValueTaskSource backed ValueTask,
                // inform it its result has been read so it can reset
                vt.GetAwaiter().GetResult();
                break;
            case IDisposable disposable:
                disposable.Dispose();
                break;
        }

        _scope = null;
        _requestServices = null;

        return default;

        static async ValueTask Awaited(RequestServicesFeature servicesFeature, ValueTask vt)
        {
            await vt;
            servicesFeature._scope = null;
            servicesFeature._requestServices = null;
        }
    }

    public void Dispose()
    {
        DisposeAsync().AsTask().GetAwaiter().GetResult();
    }
}
```