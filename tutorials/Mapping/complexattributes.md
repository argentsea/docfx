# Complex Attributes

The [Attribute tutorial](attributes.md) described how the Mapper populates simple Model class properties, like numbers, strings, etc. This guide extends this information to account for complex objects that combine mapped values into a new object.

There are three types of complex objects handled by the Mapper:

* Model Properties, via the [MapToModel](/api/ArgentSea.MapToModel.html) attribute
* List Properties
* Shard Identifiers, via the [ShardKey](/api/ArgentSea.ShardKey-2.html) and [ShardChild](/api/ArgentSea.ShardChild-3.html) attributes

## The [MapToModel](/api/ArgentSea.MapToModel.html) Attribute

Some Model objects may have properties that are objects with their *own* properties, which also need to be mapped to the underlying data.

For example, you might have an Address object that contains street address, city, region, etc. Since this data is used for customer addresses, store addresses, vendor addresses, etc. you might have a single, shared Address model. The Customer, Store, and Vendor Models would all have an Address property of type Address Model.

The [MapToModel](/api/ArgentSea.MapToModel.html) attribute tells the Mapper that this property contains an object that should also be mapped.

## [SQL Server](#tab/tabid-sql)

```csharp
public class Vendor
{
    [MapToSqlInt("VendorId")]
    public int VendorId  { get; set; }

    //include other properties...

    [MapToModel]
    public Address VendorAddress { get; set; }
}
public class Store
{
    [MapToSqlInt("StoreId")]
    public int StoreId  { get; set; }

    //include other properties...

    [MapToModel]
    public Address StoreAddress { get; set; }
}
public class Customer
{
    [MapToSqlInt("CustomerId")]
    public int CustomerId  { get; set; }

    //include other properties...

    [MapToModel]
    public Address CustomerAddress { get; set; }
}

public class Address
{
        [MapToSqlNVarChar("StreetAddress", 255)]
        public string StreetAddress { get; set; }

        [MapToSqlNVarChar("Locality", 100)]
        public string Locality { get; set; }

        [MapToSqlNVarChar("Region", 100)]
        public string Region { get; set; }

        [MapToSqlNVarChar("PostalCode", 25)]
        public string PostalCode { get; set; }
}

```

## [PostgreSQL](#tab/tabid-pg)

```csharp
public class Vendor
{
    [MapToPgInteger("VendorId")]
    public int VendorId  { get; set; }

    //include other properties...

    [MapToModel]
    public Address VendorAddress { get; set; }
}
public class Store
{
    [MapToPgInteger("StoreId")]
    public int StoreId  { get; set; }

    //include other properties...

    [MapToModel]
    public Address StoreAddress { get; set; }
}
public class Customer
{
    [MapToPgInteger("CustomerId")]
    public int CustomerId  { get; set; }

    //include other properties...

    [MapToModel]
    public Address CustomerAddress { get; set; }
}

public class Address
{
        [MapToPgVarChar("StreetAddress", 255)]
        public string StreetAddress { get; set; }

        [MapToPgVarChar("Locality", 100)]
        public string Locality { get; set; }

        [MapToPgVarChar("Region", 100)]
        public string Region { get; set; }

        [MapToPgVarChar("PostalCode", 25)]
        public string PostalCode { get; set; }
}
```

***

In this example, the Vendor, Store, and Customer Models each have a property with an Address Model type. When mapping Vendor database results, the Mapper will map the StreetAddress column or parameter to Vendor.VendorAddress.StreetAddress. Likewise for the Store and Customer Models.

In each case, the Address properties and attributes are simply included in the properties that the Mapper expects to see in the data results or parameters.

### Null Values

The Mapper will automatically set the property to a new instance of the model object, if:

* The property value is null
* The property is settable (not readonly)
* The property model class has a default constructor

Otherwise, simply instantiate the property’s object when the base Model class is created.

```csharp
public class Customer
{
    [MapToModel]
    public Address CustomerAddress { get;  } = new Address();
}
```

As [discussed earlier](attributes.md#required), the mapping attributes have a *required* argument — when set to *true*, a database null will cause the entire object to be null. This may not behave like you expect in this context: this does *not* cause the [MapToModel](/api/ArgentSea.MapToModel.html) property to become null; the entire parent model will be null.

Essentially, a [MapToModel](/api/ArgentSea.MapToModel.html) property simply extends the Model with additional properties. Consequently, a “required” column that has a null database value will cause the entire Model to be null, not just the [MapToModel](/api/ArgentSea.MapToModel.html) property.

### Chaining MapToModel Properties

In the example above, suppose the Address Model *also* had a [MapToModel](/api/ArgentSea.MapToModel.html) property — perhaps a `Coordinates` Model object. Theoretically, the `Coordinates` Model itself could also have a [MapToModel](/api/ArgentSea.MapToModel.html) property; the Mapper should be able to resolve nearly any number of chained [MapToModel](/api/ArgentSea.MapToModel.html) properties. All of these attributes are included in the data mapping.

## Lists

The Mapper can also map to properties that contain lists (`List<Model>` or `IList<Model>`) of Model objects. This allows for one-to-many relationships.

Mapping to a List works quite differently than [MapToModel](/api/ArgentSea.MapToModel.html). The List data can only come from a data reader result. Because data readers can contain multiple results — which might be mapped to different list properties — the mapping and result order is specified at query time. For this reason, no attribute is required.

Models in a List may themselves have child lists, but unlike [MapToModel](/api/ArgentSea.MapToModel.html) properties, they will be ignored by the Mapper. In cases where a result set must be transformed into a complete Model hierarchy, you can use the [Query](../querying/handling.md) methods to bypass the Mapper. This allows you to hydrate the results in your own code, optionally invoking Mapper on parts of the result stream.

For the details on how to populate Models with `List<Model>` properties, see the discussion on [querying](../querying/fetching.md).

## [MapShardKey](/api/ArgentSea.MapShardKeyAttribute.html) and [MapShardChild](/api/ArgentSea.MapShardChildAttribute.html) Attributes

A [ShardKey](/api/ArgentSea.ShardKey-2.html) is a record identifier containing a shard Id and a record Id. A [ShardChild](/api/ArgentSea.ShardChild-3.html) is a compound key consisting of a shard Id, a record Id, and a child Id.

The [MapShardKey](/api/ArgentSea.MapShardKeyAttribute.html) and [MapShardChild](/api/ArgentSea.MapShardChildAttribute.html) attributes are used to decorate properties of type [ShardKey](/api/ArgentSea.ShardKey-2.html) or [ShardChild](/api/ArgentSea.ShardChild-3.html) respectively.

These are special types are reviewed in detail in the [sharding](../sharding/shardkey.md) section.

Next: [Mapping Targets](targets.md)