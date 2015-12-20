![peasy](https://www.dropbox.com/s/2yajr2x9yevvzbm/peasy3.png?dl=0&raw=1)

# Peasy.DataProxy.EF6

Peasy.DataProxy.EF6 provides the [EF6DataProxyBase](https://github.com/peasy/Peasy.DataProxy.EF6/blob/master/Peasy.DataProxy.EF6/EF6DataProxyBase.cs) class.  EF6DataProxyBase is an abstract class that implements [IDataProxy](https://github.com/ahanusa/Peasy.NET/wiki/Data-Proxy), and can be used to very quickly and easily provide a data proxy that communicates with a database using Entity Framework 6.0.

###Where can I get it?

First, install NuGet. Then create a project for your Entity Framework data proxy implementations to live.  Finally, install Peasy.DataProxy.EF6 from the package manager console:

``` PM> Install-Package Peasy.DataProxy.EF6 ```

You can also download and add the Peasy.DataProxy.EF6 project to your solution and set references where applicable

### Creating a concrete EF6 data proxy

To create an EF6 repository, you must inherit from [EF6DataProxyBase](https://github.com/peasy/Peasy.DataProxy.EF6/blob/master/Peasy.DataProxy.EF6/EF6DataProxyBase.cs).  There is one contractual obligation to fullfill.

1.) Override [GetDbContext](https://github.com/peasy/Peasy.DataProxy.EF6/blob/master/Peasy.DataProxy.EF6/EF6DataProxyBase.cs#L25) - this method should return your custom [DbContext](https://msdn.microsoft.com/en-us/library/system.data.entity.dbcontext(v=vs.113).aspx) that all methods in the proxy will consume.

Here is a sample implementation

```c#
// Your DbContext ...
public class OrdersDotComContext : DbContext
{
    public DbSet<Category> Categories { get; set; }
    public DbSet<Customer> Customers { get; set; }
}

public class CustomersDataProxy : EF6DataProxyBase<Customer, Customer, long>
{
    protected override DbContext GetDbContext()
    {
        return new OrdersDotComContext();
    }
}

```

In this example, we create an EF6 customers data proxy.  The first thing to note is that we supply ```Customer``` as our ```DTO``` and ```TEntity``` generic constraints, respectively.  By contractual agreement, these types must implement [```IDomainObject<T>```](https://github.com/peasy/Peasy.NET/blob/master/Peasy.Core/IDomainObject.cs).  The ```long``` specifies the key type that will be used for all of our arguments to the [IDataProxy](https://github.com/peasy/Peasy.NET/wiki/Data-Proxy) methods.

As part of our contractual obligation, we override ```GetDbContext``` to provide the context in which the data proxy will consume to work with Entity Framework.

Note that in this example, we use the Customer class that will become arguments and return types for our data proxy methods.  We could also specify 2 different class types as well, which might be the case if you need to supply EF with a different class, which will be the case if you use a [database first](https://msdn.microsoft.com/en-us/data/jj206878.aspx) approach. 

An implementation might look like this:

```c#
public class CustomersDataProxy : EF6DataProxyBase<Customer, tCustomer, long>
{
    protected override DbContext GetDbContext()
    {
        return new OrdersDotComContext();
    }
}
```

In this example, we provide ```Customer``` and ```tCustomer``` as the first 2 generic constraints, where Customer is your [DTO]() and tCustomer reprensents an Entity Framework class generated by database first.  It should be noted that Customer and tCustomer must implement [```IDomainObject<T>```](https://github.com/peasy/Peasy.NET/blob/master/Peasy.Core/IDomainObject.cs).

In the case that you use a database first approach, you will want to create partial classes for your entities that implement IDomainObject<T> in a separate class file(s) so they aren't clobbered when regenerating them after structural database changes.

EF6DataProxyBase relies on [Automapper](https://github.com/AutoMapper/AutoMapper) to map DTOs to entities and as a result, if you supply generic constraints of differing types for the DTO and TEntity constraints, you must [provide mappings](https://github.com/AutoMapper/AutoMapper/wiki/Getting-started) in your application to account for this.  If you like, you can always use an alternate mapping tool of your [choice](https://github.com/peasy/Peasy.DataProxy.EF6#mapping-logic).

By simply inheriting from EF6DataProxyBase and overriding GetDbContext, you have a full-blown peasy data proxy that performs CRUD operations against a database table using Entity Framework 6.0.

### Execution hooks

Each EF6 [data proxy]() method gives you an opporunity to perform logic before and after execution of each method via the OnBeforeXYZ and OnAfterXYZ hooks.  These are particularly useful for initializing DTO data, logging, or writing custom concurrency handling logic.

Here is an example of providing concurrency logic for an InventoryItemRepository

```c#
    protected override void OnBeforeUpdateExecuted(DbContext context, InventoryItem entity)
    {
        var existing = context.Set<InventoryItem>()
                              .FirstOrDefault(e => e.ID.Equals(entity.ID) && e.Version == entity.Version);
        if (existing == null)
            throw new ConcurrencyException($"{entity.GetType().Name} with id {entity.ID.ToString()} was already changed");
	
        entity.IncrementVersionByOne();
    }
```

In this example, we override OnBeforeUpdateExecuted and ensure that an inventory item with an ID and specific version number exist, either throwing a Concurrency Exception in the scenario where it has changed, or providing our custom version incrementing logic.  A full implementation of this class can be found [here](https://github.com/peasy/Samples/blob/master/Orders.com.DAL.EF/InventoryItemRepository.cs)

### Mapping Logic

By default, EF6DataProxyBase is configured to use [Automapper](https://github.com/AutoMapper/AutoMapper) to map DTOs to Entities.  However, if you use a different mapping tool by creating a facade (wrapper) class that implements [Peasy.DataProxy.EF6.IMapper](https://github.com/peasy/Peasy.DataProxy.EF6/blob/master/Peasy.DataProxy.EF6/IMapper.cs) and supplying it to the overloaded constructor of EF6DataProxyBase.

Here is a sample implementation of a custom mapper using [Mapster](https://github.com/eswann/Mapster)

```c#
public class MapsterHelper : IMapper
{
    public TDestination Map<TSource, TDestination>(TSource source)
    {
        return TypeAdapter.Adapt<TDestination>(source);
    }
	
    public TDestination Map<TSource, TDestination>(TSource source, TDestination destination)
    {
        return TypeAdapter.Adapt<TSource, TDestination>(source, destination);
    }
}
```

And wiring it up...

```c#
public class CustomerDataProxy : EF6DataProxyBase<Customer, tCustomer, long>
{
    public CustomerDataProxy() : base(new MapsterHelper())
    {
    }
        
    protected override DbContext GetDbContext()
    {
        return new OrdersDotComContext();
    }
}
```

Now the CustomerDataProxy will use Mapster instead of the preconfigured AutoMapper.
