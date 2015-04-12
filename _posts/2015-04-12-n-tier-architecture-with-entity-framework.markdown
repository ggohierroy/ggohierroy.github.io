---
layout: post
title:  "N-Tier architecture with Web API and Entity Framework"
date:   2015-04-12 16:45:00
categories: entity-framework
---
Web API and Entity Framework are two wonderful tools that allow you to create back-ends for your websites very quickly and effortlessly. Using a model and the Entity Framework context, it is possible to automatically generate controllers with all the CRUDs. Unfortunately, this model has its limits when you need to add any sort of complex business logic that might depend on other entities for example. It quickly becomes apparent that in order to efficiently reuse code, you will need to add a business layer to your application. In this post, I will demonstrate the solution I came up with to integrate this business layer into my application.

In a tiered application, you will usually find 3 tiers:

1. The web service tier
2. The business logic tier
3. The data access tier

In our case, **the data access tier will be Entity Framework**, and it totally makes sense since this tier is a way to abstract your database, which is exactly what Entity Framework already does.

Many ideas found in this post were stolen from [this article][asp-net-article] on the offical ASP.NET website which describes how to implement the Repository and Unit of Work patterns.

## The Context

The first thing I do when I start an Entity Framework project is to **disable lazy loading**. Lazy loading saves a minimal amount of lines of code and can result in serious performance issues. The obvious example is the case in which you access a navigation property inside a loop. If you did not eagerly load this property, then there will be a trip to the database at every iteration. I prefer doing eager loading when at all possible.

When I disable lazy loading, I usually also **disable proxy creation**. Proxy creation is necessary for lazy loading, and also makes sure that the context automatically stays in sync when you modify an entity. When you turn it off, the changes will be synced when `DetectChanges` is called on the context. I haven't seen any evidence that one method or the other is better performance-wise, and since we will not be using lazy loading, then I choose to turn off proxy creation as well.

This logic will usually be in the constructor of your context:
 
{% highlight C# %}
public Context()
    : base("Default")
{
    Configuration.ProxyCreationEnabled = false;
    Configuration.LazyLoadingEnabled = false;
}
{% endhighlight %}

## The Model

The models are both model and DTO. The properties which are DTO specific are marked as `[NotMapped]`. The calls made by the controller will always return a DTO version of the model, where only the necessary data is returned (all the navigation properties have a `null` value), and the `[NotMapped]` properties are filled. This simplifies the code greatly since you don't have to maintain 2 classes whenever your model changes.

{% highlight C# %}
public class Client : IObject
{
    public Client()
    {
        // Initialize default values
        Name = "Foo";
    }

    [Key]
    public int Id { get; set; }

    public string Name { get; set; }

    public string Value { get; set; }

    public int? ForeignObjectId { get; set; }

    [ForeignKey("ForeignObjectId")]
    public ForeignObject ForeignObject { get; set; }

    public string CreatedBy { get; set; }
    public DateTime CreatedDate { get; set; }
    public string UpdatedBy { get; set; }
    public DateTime UpdatedDate { get; set; }
    public string DeleteBy { get; set; }
    public DateTime? DeletedDate { get; set; }
    public bool Deleted { get; set; }

    [NotMapped]
    public string ForeignObjectDescription { get; set; }
}
{% endhighlight %}

The `IObject` interface simply defines the properties which are shared by all your models. In your case that might simply be the primary key. In my case, it looks something like this:

{% highlight C# %}
public interface IObject
{
    int Id { get; set; }
    string CreatedBy { get; set; }
    DateTime CreatedDate { get; set; }
    string UpdatedBy { get; set; }
    DateTime UpdatedDate { get; set; }
    string DeleteBy { get; set; }
    DateTime? DeletedDate { get; set; }
    bool Deleted { get; set; }
}
{% endhighlight %}

Having an interface like this will allow me to create a generic business object base class later on.

## The Web Service Tier

Since we'll be moving most of the logic to the business tier, there will be very little left in the the web service tier. However, the one thing will see in the web service tier is the usage of the `UnitOfWork` object.

### Unit of Work pattern

As previously mentioned, the idea for this came from [this article][asp-net-article]. The pattern allows me to have a **single Entity Framework context** per HTTP request. This context will be shared because the `UnitOfWork` class will instantiate it and also pass it to the business tier classes when they are accessed.

In this controller, you can see that the `UnitOfWork` object is instantiated at the same time as the controller. This also creates the context. You then have access to your business tier from the `UnitOfWork` object (e.g. `_unitOfWork.Clients`).

{% highlight C# %}
public class ClientController : ApiController
{
    private readonly UnitOfWork _unitOfWork = new UnitOfWork();

    public Client Insert(Client client)
    {
        _unitOfWork.Clients.Insert(client);
        _unitOfWork.Save();
        return client;
    }
}
{% endhighlight %}

Here is what the `UnitOfWork` class might look like. The main difference from the article is that instead of passing the context to the `Clients` object, I pass the `UnitOfWork` object (itself). This makes it easier to make calls to other business tier objects from within such objects.

{% highlight C# %}
public class UnitOfWork : IDisposable
{
    public Context Context { get; private set; }

    public UnitOfWork()
    {
        Context = new Context();
    }

    private Clients _clients;

    public Clients Clients
    {
        get { return _clients ?? (_clients = new Clients(this)); }
    }

    public void Save()
    {
        Context.SaveChanges();
    }

    // IDisposable implementation
    private bool _disposed;

    protected virtual void Dispose(bool disposing)
    {
        if (!_disposed)
        {
            if (disposing)
            {
                Context.Dispose();
            }
        }
        _disposed = true;
    }

    public void Dispose()
    {
        Dispose(true);
        GC.SuppressFinalize(this);
    }
}
{% endhighlight %}

### Controller Sample

{% highlight C# %}
public class ClientController : ApiController
{
    private readonly UnitOfWork _unitOfWork = new UnitOfWork();

    public Client Create()
    {
        return _unitOfWork.Clients.Create();
    }

    public List<Client> Search(ClientFindCriteria findCriteria)
    {
        return _unitOfWork.Clients.SearchDto(findCriteria);
    }

    public Client Read(int id)
    {
        return _unitOfWork.Clients.ReadDto(id);
    }

    public Client Insert(Client client)
    {
        _unitOfWork.Clients.Insert(client);
        _unitOfWork.Save();
        return client;
    }

    public Client Update(Client client)
    {
        _unitOfWork.Clients.Update(client);
        _unitOfWork.Save();
        return client;
    }

    public Client Delete(Client client)
    {
        _unitOfWork.Clients.Delete(client);
        _unitOfWork.Save();
        return client;
    }

    protected override void Dispose(bool disposing)
    {
        _unitOfWork.Dispose();
        base.Dispose(disposing);
    }
}
{% endhighlight %}

## The Business Logic Tier

Here I'm going to simply show you the business object generic base class and then explain how it all works.

{% highlight C# %}
public class BusinessObjectBase<TEntity, TSearchCriteria>
    where TEntity : class, IObject, new()
    where TSearchCriteria : class
{
    protected UnitOfWork UnitOfWork { get; private set; }
    public virtual Expression<Func<TEntity, TEntity>> ToDto { get { return e => e; } }
    public virtual Expression<Func<TEntity, TEntity>> ToSearchDto { get { return e => e; } }

    public BusinessObjectBase(UnitOfWork unitOfWork)
    {
        UnitOfWork = unitOfWork;
    }

    public virtual TEntity Create()
    {
        return new TEntity();
    }

    public virtual IQueryable<TEntity> Search(TSearchCriteria searchCriteria = null, bool asNoTracking = false, params Expression<Func<TEntity, object>> [] includes)
    {
        var entities = UnitOfWork.Context.Set<TEntity>(asNoTracking);

        foreach (var include in includes)
        {
            entities = entities.Include(include);
        }

        if(searchCriteria != null)
            entities = Filter(entities, searchCriteria);

        entities = entities.Where(e => !e.Deleted);

        return entities;
    }

    public List<TEntity> SearchDto(TSearchCriteria searchCriteria = null)
    {
        var entities = (IQueryable<TEntity>)UnitOfWork.Context.Set<TEntity>();

        if(searchCriteria != null)
            entities = Filter(entities, searchCriteria);

        entities = entities.Where(e => !e.Deleted);

        var results = entities.Select(ToSearchDto).ToList();

        return results;
    }

    public TEntity Read(int id, bool asNoTracking = false, params Expression<Func<TEntity, object>>[] includes)
    {
        var toReturn = UnitOfWork.Context.Set<TEntity>(asNoTracking);

        foreach (var include in includes)
        {
            toReturn = toReturn.Include(include);
        }
            
        return toReturn.FirstOrDefault(e => e.Id == id && !e.Deleted);
    }

    public virtual TEntity ReadDto(int id)
    {
        var dto = UnitOfWork.Context
            .Set<TEntity>()
            .Where(e => e.Id == id && !e.Deleted)
            .Select(ToDto)
            .FirstOrDefault();

        return dto;
    }

    public virtual void Insert(TEntity toInsert)
    {
        var user = User.GetUsername();
        var date = DateTime.Now;
        toInsert.CreatedBy = user;
        toInsert.CreatedDate = date;
        toInsert.UpdatedBy = user;
        toInsert.UpdatedDate = date;

        UnitOfWork.Context.Set<TEntity>().Add(toInsert);
    }

    public virtual void Update(TEntity toUpdate)
    {
        var user = User.GetUsername();
        var date = DateTime.Now;
        toUpdate.UpdatedBy = user;
        toUpdate.UpdatedDate = date;

        var entry = UnitOfWork.Context.Entry(toUpdate);
        if (entry.State == EntityState.Detached)
            entry.State = EntityState.Modified;
    }

    public virtual void Delete(TEntity toDelete)
    {
        var user = User.GetUsername();
        var date = DateTime.Now;
        toDelete.DeleteBy = user;
        toDelete.DeletedDate = date;
        toDelete.Deleted = true;

        var entry = UnitOfWork.Context.Entry(toDelete);
        if (entry.State == EntityState.Detached)
            entry.State = EntityState.Modified;
    }

    public virtual IQueryable<TEntity> Filter(IQueryable<TEntity> entities, TSearchCriteria findCriteria)
    {
        return entities;
    }
}
{% endhighlight %}

### Class definition

{% highlight C# %}
public class BusinessObjectBase<TEntity, TSearchCriteria>
    where TEntity : class, IObject, new()
    where TSearchCriteria : class
{% endhighlight %}

Two types need to be specified when deriving from the `BusinessOjbectBase`:

1. `TEntity`: The main entity type, i.e., the model.
2. `TSearchCriteria`: The object which contains the criteria used to filter during a search.

### The Constructor

{% highlight C# %}
public BusinessObjectBase(UnitOfWork unitOfWork)
{
    UnitOfWork = unitOfWork;
}
{% endhighlight %}

This is fairly straightforward. As previously mentioned, the constructor takes a `UnitOfWork`, which will alow the business object to access the context as well as other business objects.

### The Create Operation

This is a method that can be called if you need to initialize an entity server-side. You will be able to specify default values in the entity's constructor, or you can override the method if a more complex initialization is required, for instance if you need to make database calls.

{% highlight C# %}
public virtual TEntity Create()
{
    return new TEntity();
}
{% endhighlight %}

### The Search Operation

{% highlight C# %}
public virtual IQueryable<TEntity> Search(TSearchCriteria searchCriteria = null, bool asNoTracking = false, params Expression<Func<TEntity, object>> [] includes)
{
    var entities = UnitOfWork.Context.Set<TEntity>(asNoTracking);

    foreach (var include in includes)
    {
        entities = entities.Include(include);
    }

    if(searchCriteria != null)
        entities = Filter(entities, searchCriteria);

    entities = entities.Where(e => !e.Deleted);

    return entities;
}

public List<TEntity> SearchDto(TSearchCriteria searchCriteria = null)
{
    var entities = (IQueryable<TEntity>)UnitOfWork.Context.Set<TEntity>();

    if(searchCriteria != null)
        entities = Filter(entities, searchCriteria);

    entities = entities.Where(e => !e.Deleted);

    var results = entities.Select(ToSearchDto).ToList();

    return results;
}

public virtual IQueryable<TEntity> Filter(IQueryable<TEntity> entities, TSearchCriteria findCriteria)
{
    return entities;
}

public virtual Expression<Func<TEntity, TEntity>> ToSearchDto { get { return e => e; } }
{% endhighlight %}

Both methods can have search criteria specified in order to filter the results. This will be done with the use of the `Filter` virtual method, which simply returns the whole set by default. You will need to override it to do any sort of real filtering.

The first method, which returns a set of entities, can be called from the back-end. The query can be done with `AsNoTracking()` specified, which can be useful at times. You can also specify lambda expressions which will be used to `Include()` related entities.

The second method is specifically for the controller to call, and returns a set of DTOs. The `ToSearchDTO` expression needs to be overriden to map the DTO specific properties.

### The Read Operation

{% highlight C# %}
public TEntity Read(int id, bool asNoTracking = false, params Expression<Func<TEntity, object>>[] includes)
{
    var toReturn = UnitOfWork.Context.Set<TEntity>(asNoTracking);

    foreach (var include in includes)
    {
        toReturn = toReturn.Include(include);
    }
        
    return toReturn.FirstOrDefault(e => e.Id == id && !e.Deleted);
}

public virtual TEntity ReadDto(int id)
{
    var dto = UnitOfWork.Context
        .Set<TEntity>()
        .Where(e => e.Id == id && !e.Deleted)
        .Select(ToDto)
        .FirstOrDefault();

    return dto;
}

public virtual Expression<Func<TEntity, TEntity>> ToDto { get { return e => e; } }
{% endhighlight %}

This is very similar to the search methods.

### The Insert Operation

{% highlight C# %}
public virtual void Insert(TEntity toInsert)
{
    var user = User.GetUsername();
    var date = DateTime.Now;
    toInsert.CreatedBy = user;
    toInsert.CreatedDate = date;
    toInsert.UpdatedBy = user;
    toInsert.UpdatedDate = date;

    UnitOfWork.Context.Set<TEntity>().Add(toInsert);
}
{% endhighlight %}

This simply adds the object to the set.

### The Update Operation

{% highlight C# %}
public virtual void Update(TEntity toUpdate)
{
    var user = User.GetUsername();
    var date = DateTime.Now;
    toUpdate.UpdatedBy = user;
    toUpdate.UpdatedDate = date;

    var entry = UnitOfWork.Context.Entry(toUpdate);
    if (entry.State == EntityState.Detached)
        entry.State = EntityState.Modified;
}
{% endhighlight %}

This method needs to attach the object to the context if it isn't already. If the object was received by the controller, it will be detached, and we need to manually attach it like this. If the object was read in the back-end for instance, it will already be attached and tracked by the context, so nothing needs to be done.

### The Delete Operation

{% highlight C# %}
public virtual void Delete(TEntity toDelete)
{
    var user = User.GetUsername();
    var date = DateTime.Now;
    toDelete.DeleteBy = user;
    toDelete.DeletedDate = date;
    toDelete.Deleted = true;

    var entry = UnitOfWork.Context.Entry(toDelete);
    if (entry.State == EntityState.Detached)
        entry.State = EntityState.Modified;
}
{% endhighlight %}

Here, we also need to attach the object if necessary.

### Business Object Sample

{% highlight C# %}
public class Clients : BusinessObjectBase<Client, ClientFindCriteria>
{
    public Clients(UnitOfWork unitOfWork)
        : base(unitOfWork)
    {
    }

    public override IQueryable<Client> Filter(IQueryable<Client> entities, ClientFindCriteria findCriteria)
    {
        if (!string.IsNullOrEmpty(findCriteria.Keyword))
            entities = entities.Where(c => c.Name.Contains(findCriteria.Keyword));

        return entities;
    }

    public override Expression<Func<Client, Client>> ToDto { get { return _toDto; } }

    private readonly Expression<Func<Client, Client>> _toDto = c =>
        new Client
        {
            Id = c.Id,
            Name = c.Name,
            Value = c.Value,
            ForeignObjectId = c.ForeignObjectId,
            ForeignObjectDescription = c.ForeignObject.Name,
            CreatedBy = c.CreatedBy,
            CreatedDate = c.CreatedDate,
            UpdatedBy = c.UpdatedBy,
            UpdatedDate = c.UpdatedDate,
            DeleteBy = c.DeleteBy,
            DeletedDate = c.DeletedDate,
            Deleted = c.Deleted
        };
}
{% endhighlight %}

You can see that with very little code, you can now read, insert, update, and delete clients.

## Conclusion

So there it is, a complete N-Tier architecture with Web API and Entity Framework. Send me your feedback to my e-mail address, I would love to hear from you.

[asp-net-article]: http://www.asp.net/mvc/overview/older-versions/getting-started-with-ef-5-using-mvc-4/implementing-the-repository-and-unit-of-work-patterns-in-an-asp-net-mvc-application