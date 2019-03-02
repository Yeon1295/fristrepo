# Performance considerations for EF 4, 5, and 6

- Entity Framework 6 is an out of band release and does not depend on the Entity Framework components that ship with .NET

### Cold vs. Warm Query Execution

- The very first time any query is made against a given model, the Entity Framework does a lot of work behind the scenes to 
  load and validate the model. 
- We frequently refer to this first query as a  **"cold"** query.  
  Further queries against an already loaded model are known as **"warm"** queries, and are much faster.
  
##### What is View Generation?

- In order to understand what view generation is, we must first understand what “Mapping Views” are. 

- Mapping Views are executable representations of the transformations specified in the mapping for each entity set and association. 
- Internally, these mapping views take the shape of CQTs (canonical query trees). 

- There are two types of mapping views:
      - Query views: these represent the transformation necessary to go from the database schema to the conceptual model.
      - Update views: these represent the transformation necessary to go from the conceptual model to the database schema.

- Keep in mind that the conceptual model might differ from the database schema in various ways. For example, one single table 
  might be used to store the data for two different entity types. 
- Inheritance and non-trivial mappings play a role in the complexity of the mapping views.

- The process of computing these views based on the specification of the mapping is what we call view generation. 

  View generation can either take place dynamically when a model is loaded, or at build time, by using "pre-generated views"; 
  the latter are serialized in the form of Entity SQL statements to a C# or VB file.
  
 - When views are generated, they are also validated. From a performance standpoint, the vast majority of the cost of view 
   generation is actually the validation of the views which ensures that the connections between the entities make sense and 
   have the correct cardinality for all the supported operations.
   
 - When a query over an entity set is executed, the query is combined with the corresponding query view, and the result of 
   this composition is run through the plan compiler to create the representation of the query that the backing store can understand.
 - For SQL Server, the final result of this compilation will be a T-SQL SELECT statement. 
 
 - The first time an update over an entity set is performed, the update view is run through a similar process to transform 
   it into DML statements for the target database.
   
 ##### Factors that affect View Generation performance
 
 - The performance of view generation step not only depends on the size of your model but also on how interconnected the model is. 
 
 - If two Entities are connected via an inheritance chain or an Association, they are said to be connected. 
   Similarly if two tables are connected via a foreign key, they are connected. 
   
 - As the number of connected Entities and tables in your schemas increase, the view generation cost increases.
 
 - The algorithm that we use to generate and validate views is exponential in the worst case, though we do use some optimizations 
   to improve this.
   
- The biggest factors that seem to negatively affect performance are:
      - Model size, referring to the number of entities and the amount of associations between these entities.
      - Model complexity, specifically inheritance involving a large number of types.
    - Using Independent Associations, instead of Foreign Key Associations.
    
- For small, simple models the cost may be small enough to not bother using pre-generated views. 
  As model size and complexity increase, there are several options available to reduce the cost of view generation and validation.
  
##### Using Pre-Generated Views to decrease model load time

- Before the Entity Framework can execute a query or save changes to the data source, it must generate a set of mapping views 
  to access the database. 
- These mapping views are a set of Entity SQL statement that represent the database in an abstract way, and are part of the 
  metadata which is cached per application domain. 
  
- If you create multiple instances of the same context in the same application domain, they will reuse mapping views from the 
  cached metadata rather than regenerating them. 
- Because mapping view generation is a significant part of the overall cost of executing the first query, the Entity Framework 
  enables you to pre-generate mapping views and include them in the compiled project.

###### Generating Mapping Views with the EF Power Tools Community Edition

- The easiest way to pre-generate views is to use the EF Power Tools Community Edition. 
  Once you have the Power Tools installed you will have a menu option to Generate Views, as below.
      - For Code First models right-click on the code file that contains your DbContext class.
      - For EF Designer models right-click on your EDMX file.
      
- Once the process is finished you will have a class generated.
- Now when you run your application EF will use this class to load views as required. If your model changes and 
  you do not re-generate this class then EF will throw an exception.
  
###### Generating Mapping Views from Code - EF6 Onwards

- The other way to generate views is to use the APIs that EF provides. When using this method you have the freedom to 
  serialize the views however you like, but you also need to load the views yourself.

###### Generating Views
- The APIs to generate views are on the System.Data.Entity.Core.Mapping.StorageMappingItemCollection class. 
  You can retrieve a StorageMappingCollection for a Context by using the MetadataWorkspace of an ObjectContext. 
- If you are using the newer DbContext API then you can access this by using the IObjectContextAdapter like below, 
  in this code we have an instance of your derived DbContext called dbContext:
  
```
    var objectContext = ((IObjectContextAdapter) dbContext).ObjectContext;
    var  mappingCollection = (StorageMappingItemCollection)objectContext.MetadataWorkspace
                                                                        .GetItemCollection(DataSpace.CSSpace);
```

- Once you have the StorageMappingItemCollection then you can get access to the 
  GenerateViews and ComputeMappingHashValue methods.
```
    public Dictionary<EntitySetBase, DbMappingView> GenerateViews(IList<EdmSchemaError> errors)
    public string ComputeMappingHashValue()
```
- The first method creates a dictionary with an entry for each view in the container mapping. 
  The second method computes a hash value for the single container mapping and is used at runtime to validate 
  that the model has not changed since the views were pre-generated. 
- Overrides of the two methods are provided for complex scenarios involving multiple container mappings.

- When generating views you will call the GenerateViews method and then write out the resulting EntitySetBase 
  and DbMappingView. You will also need to store the hash generated by the ComputeMappingHashValue method.
  
###### Loading Views 

- In order to load the views generated by the GenerateViews method, you can provide EF with a class that inherits 
  from the DbMappingViewCache abstract class. 
- DbMappingViewCache specifies two methods that you must implement:
```
    public abstract string MappingHashValue { get; }
    public abstract DbMappingView GetView(EntitySetBase extent);
```
- The MappingHashValue property must return the hash generated by the ComputeMappingHashValue method. When EF is going 
  to ask for views it will first generate and compare the hash value of the model with the hash returned by this property. 
  If they do not match then EF will throw an EntityCommandCompilationException exception.
  
- The GetView method will accept an EntitySetBase and you need to return a DbMappingVIew containing the EntitySql that was 
  generated for that was associated with the given EntitySetBase in the dictionary generated by the GenerateViews method. 
  If EF asks for a view that you do not have then GetView should return null.
  
- The following is an extract from the DbMappingViewCache that is generated with the Power Tools as described above, in it 
  we see one way to store and retrieve the EntitySql required.
  
```
public override string MappingHashValue
    {
        get { return "a0b843f03dd29abee99789e190a6fb70ce8e93dc97945d437d9a58fb8e2afd2e"; }
    }

    public override DbMappingView GetView(EntitySetBase extent)
    {
        if (extent == null)
        {
            throw new ArgumentNullException("extent");
        }

        var extentName = extent.EntityContainer.Name + "." + extent.Name;

        if (extentName == "BlogContext.Blogs")
        {
            return GetView2();
        }

        if (extentName == "BlogContext.Posts")
        {
            return GetView3();
        }

        return null;
    }

    private static DbMappingView GetView2()
    {
        return new DbMappingView(@"
            SELECT VALUE -- Constructing Blogs
            [BlogApp.Models.Blog](T1.Blog_BlogId, T1.Blog_Test, T1.Blog_title, T1.Blog_Active, T1.Blog_SomeDecimal)
            FROM (
            SELECT
                T.BlogId AS Blog_BlogId,
                T.Test AS Blog_Test,
                T.title AS Blog_title,
                T.Active AS Blog_Active,
                T.SomeDecimal AS Blog_SomeDecimal,
                True AS _from0
            FROM CodeFirstDatabase.Blog AS T
            ) AS T1");
    }
```
- To have EF use your DbMappingViewCache you add use the DbMappingViewCacheTypeAttribute, specifying the context 
  that it was created for. In the code below we associate the BlogContext with the MyMappingViewCache class
```
    [assembly: DbMappingViewCacheType(typeof(BlogContext), typeof(MyMappingViewCache))]
```
- For more complex scenarios, mapping view cache instances can be provided by specifying a mapping view cache factory. 
  This can be done by implementing the abstract class System.Data.Entity.Infrastructure.MappingViews.DbMappingViewCacheFactory. 
  The instance of the mapping view cache factory that is used can be retrieved or set using the 
  StorageMappingItemCollection.MappingViewCacheFactoryproperty.

##### Reducing the cost of view generation

- Using pre-generated views moves the cost of view generation from model loading (run time) to design time. 

- While this improves startup performance at runtime, you will still experience the pain of view generation while you are developing.

- There are several additional tricks that can help reduce the cost of view generation, both at compile time and run time.

###### Using Foreign Key Associations to reduce view generation cost

- We have seen a number of cases where switching the associations in the model from Independent Associations 
  to Foreign Key Associations dramatically improved the time spent in view generation.
  
- It’s important to remark that pre-generating views in Entity Framework 4 and 5 can be done with EDMGen or the 
  Entity Framework Power Tools. For Entity Framework 6 view generation can be done via the Entity Framework Power Tools 
  or programmatically
  
###### How to use Foreign Keys instead of Independent Associations

- Entity Designer
    - After adding an association between two entities, make sure you have a referential constraint. 
      Referential constraints tell Entity Framework to use Foreign Keys instead of Independent Associations.
- EDMGen
    - When using EDMGen to generate your files from the database, your Foreign Keys will be respected and added to the model as such.
- Code First
    - Use the Code First Conventions
    
###### Moving your model to a separate assembly

- When your model is included directly in your application's project and you generate views through a pre-build event  or a 
  T4 template, view generation and validation will take place whenever the project is rebuilt, even if the model wasn't changed. 
- If you move the model to a separate assembly and reference it from your application's project, you can make other changes to 
  your application without needing to rebuild the project containing the model.
  
- Note:  when moving your model to separate assemblies remember to copy the connection strings for the model into the application 
  configuration file of the client project.

###### Disable validation of an edmx-based model

- EDMX models are validated at compile time, even if the model is unchanged. If your model has already been validated, 
  you can suppress validation at compile time by setting the "Validate on Build" property to false in the properties window. 
  When you change your mapping or model, you can temporarily re-enable validation to verify your changes.
  
- Note that performance improvements were made to the Entity Framework Designer for Entity Framework 6, and the cost of 
  the “Validate on Build” is much lower than in previous versions of the designer.
  
### Caching in the Entity Framework

- Entity Framework has the following forms of caching built-in:

      1. Object caching 
          the ObjectStateManager built into an ObjectContext instance keeps track in memory of the objects that have 
          been retrieved using that instance. This is also known as **first-level cache**.
          
      2. Query Plan Caching 
          reusing the generated store command when a query is executed more than once.
          
      3. Metadata caching 
          sharing the metadata for a model across different connections to the same model.
          
- Besides the caches that EF provides out of the box, a special kind of ADO.NET data provider known as a wrapping 
  provider can also be used to extend Entity Framework with a cache for the results retrieved from the database, 
  also known as **second-level caching**.
  
##### Object Caching

- By default when an entity is returned in the results of a query, just before EF materializes it, the ObjectContext 
  will check if an entity with the same key has already been loaded into its ObjectStateManager. 
- If an entity with the same keys is already present EF will include it in the results of the query. 
  Although EF will still issue the query against the database, this behavior can bypass much of the cost of 
  materializing the entity multiple times.

###### Getting entities from the object cache using DbContext Find

- Unlike a regular query, the Find method in DbSet will perform a search in memory before even issuing the query against 
  the database. 
- It’s important to note that two different ObjectContext instances will have two different ObjectStateManager instances, 
  meaning that they have separate object caches.
  
- Find uses the primary key value to attempt to find an entity tracked by the context. 
_ If the entity is not in the context then a query will be executed and evaluated against the database, 
  and null is returned if the entity is not found in the context or in the database.
- Note that Find also returns entities that have been added to the context but have not yet been saved to the database.

- There is a performance consideration to be taken when using Find. 
- Invocations to this method by default will trigger a validation of the object cache in order to detect changes that are 
  still pending commit to the database. 
- This process can be very expensive if there are a very large number of objects in the object cache or in a large object 
  graph being added to the object cache, but it can also be disabled. 

- In certain cases, you may perceive over an order of magnitude of difference in calling the Find method when you disable 
  auto detect changes. 
 -Yet a second order of magnitude is perceived when the object actually is in the cache versus when the object has to be 
  retrieved from the database.
  
- Example of Find with auto-detect changes disabled:
```
context.Configuration.AutoDetectChangesEnabled = false;
    var product = context.Products.Find(productId);
    context.Configuration.AutoDetectChangesEnabled = true;
    ...
```
- What you have to consider when using the Find method is:

  1. If the object is not in the cache the benefits of Find are negated, but the syntax is still simpler than a query by key.
  2. If auto detect changes is enabled the cost of the Find method may increase by one order of magnitude, or even more 
     depending on the complexity of your model and the amount of entities in your object cache.
     
- Also, keep in mind that Find only returns the entity you are looking for and it does not automatically loads its associated 
  entities if they are not already in the object cache. 
- If you need to retrieve associated entities, you can use a query by key with eager loading.

###### Performance issues when the object cache has many entities

- The object cache helps to increase the overall responsiveness of Entity Framework. However, when the object cache has a 
  very large amount of entities loaded it may affect certain operations such as Add, Remove, Find, Entry, SaveChanges and more. 
- In particular, operations that trigger a call to DetectChanges will be negatively affected by very large object caches. 
- DetectChanges synchronizes the object graph with the object state manager and its performance will determined directly 
  by the size of the object graph.
  
- When using Entity Framework 6, developers are able to call AddRange and RemoveRange directly on a DbSet, instead of 
  iterating on a collection and calling Add once per instance. 
- The advantage of using the range methods is that the cost of DetectChanges is only paid once for the entire set of 
  entities as opposed to once per each added entity.
  
 ##### Query Plan Caching
 
 - The first time a query is executed, it goes through the internal plan compiler to translate the conceptual query into 
   the store command (for example, the T-SQL which is executed when run against SQL Server).  
 - If query plan caching is enabled, the next time the query is executed the store command is retrieved directly from 
   the query plan cache for execution, bypassing the plan compiler.

- The query plan cache is shared across ObjectContext instances within the same AppDomain. You don't need to hold onto an 
  ObjectContext instance to benefit from query plan caching.
  
  ###### Some notes about Query Plan Caching
  
  - The query plan cache is shared for all query types: Entity SQL, LINQ to Entities, and CompiledQuery objects.
  
- By default, query plan caching is enabled for Entity SQL queries, whether executed through an EntityCommand or through 
  an ObjectQuery. It is also enabled by default for LINQ to Entities queries in Entity Framework on .NET 4.5, and in Entity 
  Framework 6 
- Query plan caching can be disabled by setting the EnablePlanCaching property (on EntityCommand or ObjectQuery) to false.
```
                    var query = from customer in context.Customer
                                where customer.CustomerId == id
                                select new
                                {
                                    customer.CustomerId,
                                    customer.Name
                                };
                    ObjectQuery oQuery = query as ObjectQuery;
                    oQuery.EnablePlanCaching = false;
```

- For parameterized queries, changing the parameter's value will still hit the cached query. 
  But changing a parameter's facets (for example, size, precision, or scale) will hit a different entry in the cache.
  
- When using Entity SQL, the query string is part of the key. Changing the query at all will result in different cache entries, 
  even if the queries are functionally equivalent. This includes changes to casing or whitespace.
  
- When using LINQ, the query is processed to generate a part of the key. Changing the LINQ expression will therefore 
  generate a different key.

###### Cache eviction algorithm

- Understanding how the internal algorithm works will help you figure out when to enable or disable query plan caching. 

- The cleanup algorithm is as follows:

    1. Once the cache contains a set number of entries (800), we start a timer that periodically (once-per-minute) 
      sweeps the cache.
    2. During cache sweeps, entries are removed from the cache on a LFRU (Least frequently – recently used) basis. 
       This algorithm takes both hit count and age into account when deciding which entries are ejected.
    3. At the end of each cache sweep, the cache again contains 800 entries.
    
- All cache entries are treated equally when determining which entries to evict. This means the store command for 
  a CompiledQuery has the same chance of eviction as the store command for an Entity SQL query.
- Note that the cache eviction timer is kicked in when there are 800 entities in the cache, but the cache is only 
  swept 60 seconds after this timer is started. That means that for up to 60 seconds your cache may grow to be quite large.    

##### Using CompiledQuery to improve performance with LINQ queries

- Our tests indicate that using CompiledQuery can bring a benefit of 7% over autocompiled LINQ queries
- Note that CompiledQueries are only compatible with ObjectContext-derived models, and not compatible with DbContext-derived models.

- There are two considerations you have to take when using a CompiledQuery, namely the requirement to use static instances and 
  the problems they have with composability.
  
###### Use static CompiledQuery instances

- Since compiling a LINQ query is a time-consuming process, we don’t want to do it every time we need to 
  fetch data from the database. 
- CompiledQuery instances allow you to compile once and run multiple times, but you have to be careful and 
  procure to re-use the same CompiledQuery instance every time instead of compiling it over and over again. 
- The use of static members to store the CompiledQuery instances becomes necessary; otherwise you won’t see any benefit.

- For example, suppose your page has the following method body to handle displaying the products for the selected category:
```
    // Warning: this is the wrong way of using CompiledQuery
    using (NorthwindEntities context = new NorthwindEntities())
    {
        string selectedCategory = this.categoriesList.SelectedValue;

        var productsForCategory = CompiledQuery.Compile<NorthwindEntities, string, IQueryable<Product>>(
            (NorthwindEntities nwnd, string category) =>
                nwnd.Products.Where(p => p.Category.CategoryName == category)
        );

        this.productsGrid.DataSource = productsForCategory.Invoke(context, selectedCategory).ToList();
        this.productsGrid.DataBind();
    }

    this.productsGrid.Visible = true;
```
- In this case, you will create a new CompiledQuery instance on the fly every time the method is called. 
  Instead of seeing performance benefits by retrieving the store command from the query plan cache, 
   the CompiledQuery will go through the plan compiler every time a new instance is created. 
 
 - In fact, you will be polluting your query plan cache with a new CompiledQuery entry every time the method is called.
 
 - Instead, you want to create a static instance of the compiled query, so you are invoking the same compiled query 
   every time the method is called.
   
 - One way to so this is by adding the CompiledQuery instance as a member of your object context.  
   You can then make things a little cleaner by accessing the CompiledQuery through a helper method:
 ```
    public partial class NorthwindEntities : ObjectContext
    {
        private static readonly Func<NorthwindEntities, string, IEnumerable<Product>> productsForCategoryCQ = CompiledQuery.Compile(
            (NorthwindEntities context, string categoryName) =>
                context.Products.Where(p => p.Category.CategoryName == categoryName)
            );

        public IEnumerable<Product> GetProductsForCategory(string categoryName)
        {
            return productsForCategoryCQ.Invoke(this, categoryName).ToList();
        }
 ```
 - This helper method would be invoked as follows:
 ```
        this.productsGrid.DataSource = context.GetProductsForCategory(selectedCategory);
 ```
 ###### Composing over a CompiledQuery
 
 - The ability to compose over any LINQ query is extremely useful; to do this, you simply invoke a method after the 
   IQueryable such as Skip() or Count(). However, doing so essentially returns a new IQueryable object. While there’s 
   nothing to stop you technically from composing over a CompiledQuery, doing so will cause the generation of a new 
   IQueryable object that requires passing through the plan compiler again.
   
 - Some components will make use of composed IQueryable objects to enable advanced functionality. For example, 
   ASP.NET’s GridView can be data-bound to an IQueryable object via the SelectMethod property. The GridView will then 
   compose over this IQueryable object to allow sorting and paging over the data model. As you can see, using a 
   CompiledQuery for the GridView would not hit the compiled query but would generate a new autocompiled query.
   
 - One place where you may run into this is when adding progressive filters to a query. 
 - For example, suppose you had a Customers page with several drop-down lists for optional filters (for example, Country 
   and OrdersCount). 
 - You can compose these filters over the IQueryable results of a CompiledQuery, but doing so will result in the new 
   query going through the plan compiler every time you execute it.
   
```
    using (NorthwindEntities context = new NorthwindEntities())
    {
        IQueryable<Customer> myCustomers = context.InvokeCustomersForEmployee();

        if (this.orderCountFilterList.SelectedItem.Value != defaultFilterText)
        {
            int orderCount = int.Parse(orderCountFilterList.SelectedValue);
            myCustomers = myCustomers.Where(c => c.Orders.Count > orderCount);
        }

        if (this.countryFilterList.SelectedItem.Value != defaultFilterText)
        {
            myCustomers = myCustomers.Where(c => c.Address.Country == countryFilterList.SelectedValue);
        }

        this.customersGrid.DataSource = myCustomers;
        this.customersGrid.DataBind();
    }
```

- To avoid this re-compilation, you can rewrite the CompiledQuery to take the possible filters into account:

```
      private static readonly Func<NorthwindEntities, int, int?, string, IQueryable<Customer>> customersForEmployeeWithFiltersCQ = CompiledQuery.Compile(
        (NorthwindEntities context, int empId, int? countFilter, string countryFilter) =>
            context.Customers.Where(c => c.Orders.Any(o => o.EmployeeID == empId))
            .Where(c => countFilter.HasValue == false || c.Orders.Count > countFilter)
            .Where(c => countryFilter == null || c.Address.Country == countryFilter)
        );
```

- Which would be invoked in the UI like:
```
    using (NorthwindEntities context = new NorthwindEntities())
    {
        int? countFilter = (this.orderCountFilterList.SelectedIndex == 0) ?
            (int?)null :
            int.Parse(this.orderCountFilterList.SelectedValue);

        string countryFilter = (this.countryFilterList.SelectedIndex == 0) ?
            null :
            this.countryFilterList.SelectedValue;

        IQueryable<Customer> myCustomers = context.InvokeCustomersForEmployeeWithFilters(
                countFilter, countryFilter);

        this.customersGrid.DataSource = myCustomers;
        this.customersGrid.DataBind();
    }
```

- A tradeoff here is the generated store command will always have the filters with the null checks, but these should 
  be fairly simple for the database server to optimize:
```
...
WHERE ((0 = (CASE WHEN (@p__linq__1 IS NOT NULL) THEN cast(1 as bit) WHEN (@p__linq__1 IS NULL) THEN cast(0 as bit) END)) OR ([Project3].[C2] > @p__linq__2)) AND (@p__linq__3 IS NULL OR [Project3].[Country] = @p__linq__4) 
```

##### Metadata caching

- The Entity Framework also supports Metadata caching. 
- This is essentially caching of type information and type-to-database mapping information across different connectionsto the same model.
- The Metadata cache is unique per AppDomain.

###### Metadata Caching algorithm

1. Metadata information for a model is stored in an ItemCollection for each EntityConnection.

  - As a side note, there are different ItemCollection objects for different parts of the model.   
  - For example, 
      - StoreItemCollections contains the information about the database model; 
      - ObjectItemCollection contains information about the data model;     
      - EdmItemCollection contains information about the conceptual model.
      
2. If two connections use the same connection string, they will share the same ItemCollection instance.

3. Functionally equivalent but textually different connection strings may result in different metadata caches. 
   We do tokenize connection strings, so simply changing the order of the tokens should result in shared metadata. 
   But two connection strings that seem functionally the same may not be evaluated as identical after tokenization.
   
4. The ItemCollection is periodically checked for use. If it is determined that a workspace has not been accessed recently, 
   it will be marked for cleanup on the next cache sweep.
   
5. Merely creating an EntityConnection will cause a metadata cache to be created (though the item collections in it will 
   not be initialized until the connection is opened). This workspace will remain in-memory until the caching algorithm 
   determines it is not “in use”.
   
###### The relationship between Metadata Caching and Query Plan Caching

- The query plan cache instance lives in the MetadataWorkspace's ItemCollection of store types. 
- This means that cached store commands will be used for queries against any context instantiated using a given MetadataWorkspace. 
- It also means that if you have two connections strings that are slightly different and don't match after tokenizing, 
  you will have different query plan cache instances.

##### Results caching

- With results caching (also known as "second-level caching"), you keep the results of queries in a local cache. 
  When issuing a query, you first see if the results are available locally before you query against the store. 
  While results caching isn't directly supported by Entity Framework, it's possible to add a second level cache by 
  using a wrapping provider.
  
- This implementation of second-level caching is an injected functionality that takes place after the LINQ expression 
  has been evaluated (and funcletized) and the query execution plan is computed or retrieved from the first-level cache. 
  The second-level cache will then store only the raw database results, so the materialization pipeline still executes afterwards.
  
### Autocompiled Queries

- When a query is issued against a database using Entity Framework, it must go through a series of steps before 
  actually materializing the results; one such step is Query Compilation. 
- Entity SQL queries were known to have good performance as they are automatically cached, so the second or third time 
  you execute the same query it can skip the plan compiler and use the cached plan instead.
  
- Entity Framework 5 introduced automatic caching for LINQ to Entities queries as well. 
- In past editions of Entity Framework creating a CompiledQuery to speed your performance was a common practice, 
  as this would make your LINQ to Entities query cacheable. 
- Since caching is now done automatically without the use of a CompiledQuery, we call this feature “autocompiled queries”.


  
  

  

  
  




  
  
   
