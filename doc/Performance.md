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

- Entity Framework detects when a query requires to be recompiled, and does so when the query is invoked even if it 
  had been compiled before. Common conditions that cause the query to be recompiled are:
  
    - Changing the MergeOption associated to your query. The cached query will not be used, instead the plan compiler 
      will run again and the newly created plan gets cached.
    - Changing the value of ContextOptions.UseCSharpNullComparisonBehavior. You get the same effect as changing the MergeOption.

- Other conditions can prevent your query from using the cache. Common examples are:

    - Using IEnumerable<T>.Contains<>(T value).
    - Using functions that produce queries with constants.
    - Using the properties of a non-mapped object.
    - Linking your query to another query that requires to be recompiled.
  
###### Using IEnumerable<T>.Contains<T>(T value)
  
- Entity Framework does not cache queries that invoke IEnumerable<T>.Contains<T>(T value) against an in-memory collection, 
    since the values of the collection are considered volatile. 
  
- The following example query will not be cached, so it will always be processed by the plan compiler:
  
```
  int[] ids = new int[10000];
...
    using (var context = new MyContext())
    {
        var query = context.MyEntities
                        .Where(entity => ids.Contains(entity.Id));

        var results = query.ToList();
        ...
    }
```
- Note that the size of the IEnumerable against which Contains is executed determines how fast or how slow your query is compiled.       Performance can suffer significantly when using large collections such as the one shown in the example above.

- Entity Framework 6 contains optimizations to the way IEnumerable<T>.Contains<T>(T value) works when queries are executed. 
  The SQL code that is generated is much faster to produce and more readable, and in most cases it also executes faster in the server.
  
###### Using functions that produce queries with constants 

- The Skip(), Take(), Contains() and DefautIfEmpty() LINQ operators do not produce SQL queries with parameters but instead put the 
  values passed to them as constants. 
- Because of this, queries that might otherwise be identical end up polluting the query plan cache, both on the EF stack and on 
  the database server, and do not get reutilized unless the same constants are used in a subsequent query execution. 
  
- In this example, each time this query is executed with a different value for id the query will be compiled into a new plan.
```
var id = 10;
...
    using (var context = new MyContext())
    {
        var query = context.MyEntities.Select(entity => entity.Id).Contains(id);

        var results = query.ToList();
        ...
    }
```
- In particular pay attention to the use of Skip and Take when doing paging. 

- In EF6 these methods have a lambda overload that effectively makes the cached query plan reusable because EF can capture 
  variables passed to these methods and translate them to SQLparameters. 
- This also helps keep the cache cleaner since otherwise each query with a different constant for Skip and Take 
  would get its own query plan cache entry.
  
- Consider the following code, which is suboptimal but is only meant to exemplify this class of queries:
```
    var customers = context.Customers.OrderBy(c => c.LastName);
    for (var i = 0; i < count; ++i)
    {
        var currentCustomer = customers.Skip(i).FirstOrDefault();
        ProcessCustomer(currentCustomer);
    }
```
- A faster version of this same code would involve calling Skip with a lambda:
```
    var customers = context.Customers.OrderBy(c => c.LastName);
    for (var i = 0; i \< count; ++i)
    {
        var currentCustomer = customers.Skip(() => i).FirstOrDefault();
        ProcessCustomer(currentCustomer);
    }
```
- The second snippet may run up to 11% faster because the same query plan is used every time the query is run, which 
  saves CPU time and avoids polluting the query cache. 
- Furthermore, because the parameter to Skip is in a closure the code might as well look like this now:
```
    var i = 0;
    var skippyCustomers = context.Customers.OrderBy(c => c.LastName).Skip(() => i);
    for (; i < count; ++i)
    {
        var currentCustomer = skippyCustomers.FirstOrDefault();
        ProcessCustomer(currentCustomer);
    }
```
##### Using the properties of a non-mapped object

- When a query uses the properties of a non-mapped object type as a parameter then the query will not get cached.

```
    using (var context = new MyContext())
    {
        var myObject = new NonMappedType();

        var query = from entity in context.MyEntities
                    where entity.Name.StartsWith(myObject.MyProperty)
                    select entity;

       var results = query.ToList();
        ...
    }
```

- This query can easily be changed to not use a non-mapped type and instead use a local variable as the parameter to the query.
- In this case, the query will be able to get cached and will benefit from the query plan cache.
```
    using (var context = new MyContext())
    {
        var myObject = new NonMappedType();
        var myValue = myObject.MyProperty;
        var query = from entity in context.MyEntities
                    where entity.Name.StartsWith(myValue)
                    select entity;

        var results = query.ToList();
        ...
    }
```
##### Linking to queries that require recompiling

- if you have a second query that relies on a query that needs to be recompiled, your entire second query will also be recompiled.
```
    int[] ids = new int[10000];
    ...
    using (var context = new MyContext())
    {
        var firstQuery = from entity in context.MyEntities
                            where ids.Contains(entity.Id)
                            select entity;

        var secondQuery = from entity in context.MyEntities
                            where firstQuery.Any(otherEntity => otherEntity.Id == entity.Id)
                            select entity;

        var results = secondQuery.ToList();
        ...
}
```
- The example is generic, but it illustrates how linking to firstQuery is causing secondQuery to be unable to get cached. 
  If firstQuery had not been a query that requires recompiling, then secondQuery would have been cached.
  
### NoTracking Queries

##### Disabling change tracking to reduce state management overhead

- If you are in a read-only scenario and want to avoid the overhead of loading the objects into the ObjectStateManager, 
  you can issue "No Tracking" queries.  
- Change tracking can be disabled at the query level.

- Note though that by disabling change tracking you are effectively turning off the object cache. 
  When you query for an entity, we can't skip materialization by pulling the previously-materialized query results from the
  ObjectStateManager. 
- If you are repeatedly querying for the same entities on the same context, you might actually see a performance benefit 
  from enabling change tracking.
  
- When querying using ObjectContext, ObjectQuery and ObjectSet instances will remember a MergeOption once it is set, and 
  queries that are composed on them will inherit the effective MergeOption of the parent query.
  
- When using DbContext, tracking can be disabled by calling the AsNoTracking() modifier on the DbSet. 

###### Disabling change tracking for a query when using DbContext

- You can switch the mode of a query to NoTracking by chaining a call to the AsNoTracking() method in the query. 
  Unlike ObjectQuery, the DbSet and DbQuery classes in the DbContext API don’t have a mutable property for the MergeOption.
```
        var productsForCategory = from p in context.Products.AsNoTracking()
                                where p.Category.CategoryName == selectedCategory
                                select p;
```
###### Disabling change tracking at the query level using ObjectContext
```
      var productsForCategory = from p in context.Products
                                where p.Category.CategoryName == selectedCategory
                                select p;

    ((ObjectQuery)productsForCategory).MergeOption = MergeOption.NoTracking;
```
###### Disabling change tracking for an entire entity set using ObjectContext
```
    context.Products.MergeOption = MergeOption.NoTracking;

    var productsForCategory = from p in context.Products
                                where p.Category.CategoryName == selectedCategory
                                select p;
```

### Query Execution Options

- Entity Framework offers several different ways to query.

##### LINQ to Entities queries
```
  var q = context.Products.Where(p => p.Category.CategoryName == "Beverages");
```
##### No Tracking LINQ to Entities queries

- When the context derives ObjectContext:
```
  context.Products.MergeOption = MergeOption.NoTracking;
  var q = context.Products.Where(p => p.Category.CategoryName == "Beverages");
```
- When the context derives DbContext:
```
    var q = context.Products.AsNoTracking()
                        .Where(p => p.Category.CategoryName == "Beverages");
```
- Note that queries that project scalar properties are not tracked even if the NoTracking is not specified.
```
    var q = context.Products.Where(p => p.Category.CategoryName == "Beverages")
                            .Select(p => new { p.ProductName });
```
- This particular query doesn’t explicitly specify being NoTracking, but since it’s not materializing a type that’s 
  known to the object state manager then the materialized result is not tracked.
  
##### Entity SQL over an ObjectQuery
```
  ObjectQuery<Product> products = context.Products.Where("it.Category.CategoryName = 'Beverages'");

```
##### Entity SQL over an Entity Command
```
      EntityCommand cmd = eConn.CreateCommand();
      cmd.CommandText = "Select p From NorthwindEntities.Products As p Where p.Category.CategoryName = 'Beverages'";

      using (EntityDataReader reader = cmd.ExecuteReader(CommandBehavior.SequentialAccess))
      {
          while (reader.Read())
          {
              // manually 'materialize' the product
          }
      }
```
##### SqlQuery and ExecuteStoreQuery

- SqlQuery on Database:
```
  // use this to obtain entities and not track them
  var q1 = context.Database.SqlQuery<Product>("select * from products");
```
- SqlQuery on DbSet:
```
  // use this to obtain entities and have them tracked
  var q2 = context.Products.SqlQuery("select * from products");
```
- ExecyteStoreQuery:
```
  var beverages = context.ExecuteStoreQuery<Product>(
@"     SELECT        P.ProductID, P.ProductName, P.SupplierID, P.CategoryID, P.QuantityPerUnit, P.UnitPrice, P.UnitsInStock, P.UnitsOnOrder, P.ReorderLevel, P.Discontinued, P.DiscontinuedDate
       FROM            Products AS P INNER JOIN Categories AS C ON P.CategoryID = C.CategoryID
       WHERE        (C.CategoryName = 'Beverages')"
);
```
##### CompiledQuery
```
private static readonly Func<NorthwindEntities, string, IQueryable<Product>> productsForCategoryCQ = CompiledQuery.Compile(
    (NorthwindEntities context, string categoryName) =>
        context.Products.Where(p => p.Category.CategoryName == categoryName)
        );
…
var q = context.InvokeProductsForCategoryCQ("Beverages");  
``` 

### Design time performance considerations

##### Inheritance Strategies

- Another performance consideration when using Entity Framework is the inheritance strategy you use. 
- Entity Framework supports 3 basic types of inheritance and their combinations:

  - **Table per Hierarchy (TPH)** 
      where each inheritance set maps to a table with a discriminator column to indicate which particular type in the hierarchy 
      is being represented in the row.
      
   - **Table per Type (TPT)** 
      where each type has its own table in the database; the child tables only define the columns that the parent table 
      doesn’t contain.
      
  - **Table per Class (TPC)** 
    where each type has its own full table in the database; the child tables define all their fields, including those 
    defined in parent types.
    
- If your model uses TPT inheritance, the queries which are generated will be more complex than those that are generated 
  with the other inheritance strategies, which may result on longer execution times on the store.  
- It will generally take longer to generate queries over a TPT model, and to materialize the resulting objects.  

###### Avoiding TPT in Model First or Code First applications

- When you create a model over an existing database that has a TPT schema, you don't have many options. 
  But when creating an application using Model First or Code First, you should avoid TPT inheritance for performance concerns.
  
- When you use Model First in the Entity Designer Wizard, you will get TPT for any inheritance in your model.
- When using Code First to configure the mapping of a model with inheritance, EF will use TPH by default, 
  therefore all entities in the inheritance hierarchy will be mapped to the same table
  
##### Upgrading from EF4 to improve model generation time

- A SQL Server-specific improvement to the algorithm that generates the store-layer (SSDL) of the model is available in 
  Entity Framework 5 and 6, and as an update to Entity Framework 4 when Visual Studio 2010 SP1 is installed.
  
##### Splitting Large Models with Database First and Model First  

- As model size increases, the designer surface becomes cluttered and difficult to use. We typically consider a model with 
  more than 300 entities to be too large to effectively use the designer.
  
##### Performance considerations with the Entity Data Source Control

- We've seen cases in multi-threaded performance and stress tests where the performance of a web application using the 
  EntityDataSource Control deteriorates significantly. 
- The underlying cause is that the EntityDataSource repeatedly calls MetadataWorkspace.LoadFromAssembly on the assemblies 
  referenced by the Web application to discover the types to be used as entities.
  
- The solution is to set the ContextTypeName of the EntityDataSource to the type name of your derived ObjectContext class. 
  This turns off the mechanism that scans all referenced assemblies for entity types.
  
##### POCO entities and change tracking proxies

- Entity Framework enables you to use custom data classes together with your data model without making any modifications to 
  the data classes themselves. 
- This means that you can use "plain-old" CLR objects (POCO), such as existing domain objects, with your data model. 
- These POCO data classes (also known as persistence-ignorant objects), which are mapped to entities that are defined 
  in a data model, support most of the same query, insert, update, and delete behaviors as entity types that are 
  generated by the Entity Data Model tools.

- Entity Framework can also create proxy classes derived from your POCO types, which are used when you want to enable 
  features such as lazy loading and automatic change tracking on POCO entities. 
- Your POCO classes must meet certain requirements to allow Entity Framework to use proxies
  
- Chance tracking proxies will notify the object state manager each time any of the properties of your entities has its 
  value changed, so Entity Framework knows the actual state of your entities all the time. 
- This is done by adding notification events to the body of the setter methods of your properties, and having the object 
  state manager processing such events. 
  
- Note that creating a proxy entity will typically be more expensive than creating a non-proxy POCO entity due to the 
  added set of events created by Entity Framework.
  
- When a POCO entity does not have a change tracking proxy, changes are found by comparing the contents of your entities 
  against a copy of a previous saved state. 
- This deep comparison will become a lengthy process when you have many entities in your context, or when your entities 
  have a very large amount of properties, even if none of them changed since the last comparison took place.
  
- In summary: you’ll pay a performance hit when creating the change tracking proxy, but change tracking will help you 
  speed up the change detection process when your entities have many properties or when you have many entities in your model. 

- For entities with a small number of properties where the amount of entities doesn’t grow too much, having change tracking 
  proxies may not be of much benefit.
  
### Loading Related Entities

##### Lazy Loading vs. Eager Loading

- Entity Framework offers several different ways to load the entities that are related to your target entity. 
- For example, when you query for Products, there are different ways that the related Orders will be loaded into the 
  Object State Manager. 
- From a performance standpoint, the biggest question to consider when loading related entities will be whether to 
  use Lazy Loading or Eager Loading
  
- When using Eager Loading, the related entities are loaded along with your target entity set. You use an Include statement 
  in your query to indicate which related entities you want to bring in.

- When using Lazy Loading, your initial query only brings in the target entity set. But whenever you access a navigation property,
  another query is issued against the store to load the related entity.
  
- Once an entity has been loaded, any further queries for the entity will load it directly from the Object State Manager, 
  whether you are using lazy loading or eager loading.
  
##### How to choose between Lazy Loading and Eager Loading

- Using Eager Loading
- When using eager loading, you'll issue a single query that returns all customers and all orders.
```
using (NorthwindEntities context = new NorthwindEntities())
{
    var ukCustomers = context.Customers.Include(c => c.Orders).Where(c => c.Address.Country == "UK");
    var chosenCustomer = AskUserToPickCustomer(ukCustomers);
    Console.WriteLine("Customer Id: {0} has {1} orders", customer.CustomerID, customer.Orders.Count);
}
```
- Using Lazy Loading
- When using lazy loading, you will load customers table first
- And each time you access the Orders navigation property of a customer, the orders table will be loaded
```
using (NorthwindEntities context = new NorthwindEntities())
{
    context.ContextOptions.LazyLoadingEnabled = true;

    //Notice that the Include method call is missing in the query
    var ukCustomers = context.Customers.Where(c => c.Address.Country == "UK");

    var chosenCustomer = AskUserToPickCustomer(ukCustomers);
    Console.WriteLine("Customer Id: {0} has {1} orders", customer.CustomerID, customer.Orders.Count);
}
```

###### Performance concerns with multiple Includes

- While including related entities in a query is powerful, it's important to understand what's happening under the covers.

- It takes a relatively long time for a query with multiple Include statements in it to go through our internal 
  plan compiler to produce the store command. 
- The majority of this time is spent trying to optimize the resulting query. The generated store command will contain 
  an Outer Join or Union for each Include, depending on your mapping. 
- Queries like this will bring in large connected graphs from your database in a single payload, which will acerbate any bandwidth
  issues, especially when there is a lot of redundancy in the payload (for example, when multiple levels of Include are used to
  traverse associations in the one-to-many direction).
  
- You can check for cases where your queries are returning excessively large payloads by accessing the underlying TSQL for 
  the query by using ToTraceString and executing the store command in SQL Server Management Studio to see the payload size. 
  
- In such cases you can try to reduce the number of Include statements in your query to just bring in the data you need. 
  Or you may be able to break your query into a smaller sequence of subqueries.
  
 - Before breaking the query:

```
 using (NorthwindEntities context = new NorthwindEntities())
{
    var customers = from c in context.Customers.Include(c => c.Orders)
                    where c.LastName.StartsWith(lastNameParameter)
                    select c;

    foreach (Customer customer in customers)
    {
        ...
    }
}
```
- After breaking the query:

```
using (NorthwindEntities context = new NorthwindEntities())
{
    var orders = from o in context.Orders
                 where o.Customer.LastName.StartsWith(lastNameParameter)
                 select o;

    orders.Load();

    var customers = from c in context.Customers
                    where c.LastName.StartsWith(lastNameParameter)
                    select c;

    foreach (Customer customer in customers)
    {
        ...
    }
}
```
- This will work only on tracked queries, as we are making use of the ability the context has to perform identity resolution and
  association fixup automatically.
  
- As with lazy loading, the tradeoff will be more queries for smaller payloads. You can also use projections of 
  individual properties to explicitly select only the data you need from each entity, but you will not be loading 
  entities in this case, and updates will not be supported.
  
#####  Workaround to get lazy loading of properties

- Entity Framework currently doesn’t support lazy loading of scalar or complex properties. 
- However, in cases where you have a table that includes a large object such as a BLOB, you can use table splitting to 
  separate the large properties into a separate entity. 
- For example, suppose you have a Product table that includes a varbinary photo column. If you don't frequently need to 
  access this property in your queries, you can use table splitting to bring in only the parts of the entity that you
  normally need. The entity representing the product photo will only be loaded when you explicitly need it.
  
  
### Other considerations

##### Server Garbage Collection

- Some users might experience resource contention that limits the parallelism they are expecting when the Garbage Collector 
  is not properly configured. 
- Whenever EF is used in a multithreaded scenario, or in any application that resembles a server-side system, make sure to 
  enable Server Garbage Collection
  
```
  <?xmlversion="1.0" encoding="utf-8" ?>
  <configuration>
          <runtime>
                 <gcServer enabled="true" />
          </runtime>
  </configuration>
```

- This should decrease your thread contention and increase your throughput by up to 30% in CPU saturated scenarios. 
- In general terms, you should always test how your application behaves using the classic Garbage Collection (which is 
  better tuned for UI and client side scenarios) as well as the Server Garbage Collection.
  
##### AutoDetectChanges

- Entity Framework might show performance issues when the object cache has many entities. Certain operations, such as 
  Add, Remove, Find, Entry and SaveChanges, trigger calls to DetectChanges which might consume a large amount of CPU 
  based on how large the object cache has become. 
- The reason for this is that the object cache and the object state manager try to stay as synchronized as possible on 
  each operation performed to a context so that the produced data is guaranteed to be correct under a wide array of scenarios.
  
- It is generally a good practice to leave Entity Framework’s automatic change detection enabled for the entire life of your
  application. 
- If your scenario is being negatively affected by high CPU usage and your profiles indicate that the culprit is the call 
  to DetectChanges, consider temporarily turning off AutoDetectChanges in the sensitive portion of your code:
  
```
try
{
    context.Configuration.AutoDetectChangesEnabled = false;
    var product = context.Products.Find(productId);
    ...
}
finally
{
    context.Configuration.AutoDetectChangesEnabled = true;
}
```
- Before turning off AutoDetectChanges, it’s good to understand that this might cause Entity Framework to lose its ability 
  to track certain information about the changes that are taking place on the entities. 
- If handled incorrectly, this might cause data inconsistency on your application.

##### Context per request


- Entity Framework’s contexts are meant to be used as short-lived instances in order to provide the most optimal performance
  experience. 
- Contexts are expected to be short lived and discarded, and as such have been implemented to be very lightweight and 
  reutilize metadata whenever possible. 

- In web scenarios it’s important to keep this in mind and not have a context for more than the duration of a single request.
- Similarly, in non-web scenarios, context should be discarded based on your understanding of the different levels of caching 
  in the Entity Framework. 
  
- Generally speaking, one should avoid having a context instance throughout the life of the application, as well as 
  contexts per thread and static contexts.
  
##### Database null semantics

- Entity Framework by default will generate SQL code that has C# null comparison semantics
```
            int? categoryId = 7;
            int? supplierId = 8;
            decimal? unitPrice = 0;
            short? unitsInStock = 100;
            short? unitsOnOrder = 20;
            short? reorderLevel = null;

            var q = from p incontext.Products
                    wherep.Category.CategoryName == "Beverages"
                          || (p.CategoryID == categoryId
                                || p.SupplierID == supplierId
                                || p.UnitPrice == unitPrice
                                || p.UnitsInStock == unitsInStock
                                || p.UnitsOnOrder == unitsOnOrder
                                || p.ReorderLevel == reorderLevel)
                    select p;

            var r = q.ToList();
```

- In this example, we’re comparing a number of nullable variables against nullable properties on the entity, such as 
  SupplierID and UnitPrice. 
- The generated SQL for this query will ask if the parameter value is the same as the column value, or if both the parameter 
  and the column values are null. 
- This will hide the way the database server handles nulls and will provide a consistent C# null experience across 
  different database vendors. 
- On the other hand, the generated code is a bit convoluted and may not perform well when the amount of comparisons in the 
  where statement of the query grows to a large number.
  
- One way to deal with this situation is by using database null semantics. 
- Note that this might potentially behave differently to the C# null semantics since now Entity Framework will generate 
  simpler SQL that exposes the way the database engine handles null values. 
  
- Database null semantics can be activated per-context with one single configuration line against the context configuration:  
```
  context.Configuration.UseDatabaseNullSemantics = true;
```

- Small to medium sized queries will not display a perceptible performance improvement when using database null semantics, 
  but the difference will become more noticeable on queries with a large number of potential null comparisons.
  
 #####  Async
 
 - Entity Framework 6 introduced support of async operations when running on .NET 4.5 or later. For the most part, applications 
   that have IO related contention will benefit the most from using asynchronous query and save operations. 
 - If your application does not suffer from IO contention, the use of async will, in the best cases, run synchronously and 
   return the result in the same amount of time as a synchronous call, or in the worst case, simply defer execution to an 
    asynchronous task and add extra time to the completion of your scenario.
    
##### NGEN

- Entity Framework 6 does not come in the default installation of .NET framework. As such, the Entity Framework assemblies 
  are not NGEN’d by default which means that all the Entity Framework code is subject to the same JIT’ing costs as any other 
  MSIL assembly. 
- This might degrade the F5 experience while developing and also the cold startup of your application in the production 
  environments. 
- In order to reduce the CPU and memory costs of JIT’ing it is advisable to NGEN the Entity Framework images as appropriate

##### Code First versus EDMX

- Entity Framework reasons about the impedance mismatch problem between object oriented programming and relational databases 
  by having an in-memory representation of the conceptual model (the objects), the storage schema (the database) and a mapping 
  between the two. 
- This metadata is called an Entity Data Model, or EDM for short. From this EDM, Entity Framework will derive the views to 
  roundtrip data from the objects in memory to the database and back.
  
- When Entity Framework is used with an EDMX file that formally specifies the conceptual model, the storage schema, and the 
  mapping, then the model loading stage only has to validate that the EDM is correct (for example, make sure that no mappings 
  are missing), then generate the views, then validate the views and have this metadata ready for use. Only then can a query be
  executed or new data be saved to the data store.
  
- The Code First approach is, at its heart, a sophisticated Entity Data Model generator. The Entity Framework has to produce 
  an EDM from the provided code; it does so by analyzing the classes involved in the model, applying conventions and configuring 
  the model via the Fluent API. 
- After the EDM is built, the Entity Framework essentially behaves the same way as it would had an EDMX file been present in 
  the project. Thus, building the model from Code First adds extra complexity that translates into a slower startup time for 
  the Entity Framework when compared to having an EDMX. The cost is completely dependent on the size and complexity of the 
  model that’s being built.
  
- When choosing to use EDMX versus Code First, it’s important to know that the flexibility introduced by Code First increases 
  the cost of building the model for the first time. 
- If your application can withstand the cost of this first-time load then typically Code First will be the preferred way to go.

### Investigating Performance

##### Using the Visual Studio Profiler

- If you are having performance issues with the Entity Framework, you can use a profiler like the one built into Visual Studio 
  to see where your application is spending its time.
  
##### Application/Database profiling

- Tools like the profiler built into Visual Studio tell you where your application is spending time.  Another type of profiler 
  is available that performs dynamic analysis of your running application, either in production or pre-production depending on 
  needs, and looks for common pitfalls and anti-patterns of database access.
  
##### Database logger

- If you are using Entity Framework 6 also consider using the built-in logging functionality.
```
  using (var context = newQueryComparison.DbC.NorthwindEntities())
    {
        context.Database.Log = Console.WriteLine;
        var q = context.Products.Where(p => p.Category.CategoryName == "Beverages");
        q.ToList();
    }
```

- If you want to enable database logging without recompiling, and you are using Entity Framework 6.1 or later, you can do so 
  by adding an interceptor in the web.config or app.config file of your application.
  
```
<interceptors>
    <interceptor type="System.Data.Entity.Infrastructure.Interception.DatabaseLogger, EntityFramework">
      <parameters>
        <parameter value="C:\Path\To\My\LogOutput.txt"/>
      </parameters>
    </interceptor>
  </interceptors>
```

### Query performance comparison tests

- The Northwind model was used to execute these tests. It was generated from the database using the Entity Framework designer. 
  Then, the following code was used to compare the performance of the query execution options:

```
using System;
using System.Collections.Generic;
using System.Data;
using System.Data.Common;
using System.Data.Entity.Infrastructure;
using System.Data.EntityClient;
using System.Data.Objects;
using System.Linq;

namespace QueryComparison
{
    public partial class NorthwindEntities : ObjectContext
    {
        private static readonly Func<NorthwindEntities, string, IQueryable<Product>> productsForCategoryCQ = CompiledQuery.Compile(
            (NorthwindEntities context, string categoryName) =>
                context.Products.Where(p => p.Category.CategoryName == categoryName)
                );

        public IQueryable<Product> InvokeProductsForCategoryCQ(string categoryName)
        {
            return productsForCategoryCQ(this, categoryName);
        }
    }

    public class QueryTypePerfComparison
    {
        private static string entityConnectionStr = @"metadata=res://*/Northwind.csdl|res://*/Northwind.ssdl|res://*/Northwind.msl;provider=System.Data.SqlClient;provider connection string='data source=.;initial catalog=Northwind;integrated security=True;multipleactiveresultsets=True;App=EntityFramework'";

        public void LINQIncludingContextCreation()
        {
            using (NorthwindEntities context = new NorthwindEntities())
            {                 
                var q = context.Products.Where(p => p.Category.CategoryName == "Beverages");
                q.ToList();
            }
        }

        public void LINQNoTracking()
        {
            using (NorthwindEntities context = new NorthwindEntities())
            {
                context.Products.MergeOption = MergeOption.NoTracking;

                var q = context.Products.Where(p => p.Category.CategoryName == "Beverages");
                q.ToList();
            }
        }

        public void CompiledQuery()
        {
            using (NorthwindEntities context = new NorthwindEntities())
            {
                var q = context.InvokeProductsForCategoryCQ("Beverages");
                q.ToList();
            }
        }

        public void ObjectQuery()
        {
            using (NorthwindEntities context = new NorthwindEntities())
            {
                ObjectQuery<Product> products = context.Products.Where("it.Category.CategoryName = 'Beverages'");
                products.ToList();
            }
        }

        public void EntityCommand()
        {
            using (EntityConnection eConn = new EntityConnection(entityConnectionStr))
            {
                eConn.Open();
                EntityCommand cmd = eConn.CreateCommand();
                cmd.CommandText = "Select p From NorthwindEntities.Products As p Where p.Category.CategoryName = 'Beverages'";

                using (EntityDataReader reader = cmd.ExecuteReader(CommandBehavior.SequentialAccess))
                {
                    List<Product> productsList = new List<Product>();
                    while (reader.Read())
                    {
                        DbDataRecord record = (DbDataRecord)reader.GetValue(0);

                        // 'materialize' the product by accessing each field and value. Because we are materializing products, we won't have any nested data readers or records.
                        int fieldCount = record.FieldCount;

                        // Treat all products as Product, even if they are the subtype DiscontinuedProduct.
                        Product product = new Product();  

                        product.ProductID = record.GetInt32(0);
                        product.ProductName = record.GetString(1);
                        product.SupplierID = record.GetInt32(2);
                        product.CategoryID = record.GetInt32(3);
                        product.QuantityPerUnit = record.GetString(4);
                        product.UnitPrice = record.GetDecimal(5);
                        product.UnitsInStock = record.GetInt16(6);
                        product.UnitsOnOrder = record.GetInt16(7);
                        product.ReorderLevel = record.GetInt16(8);
                        product.Discontinued = record.GetBoolean(9);

                        productsList.Add(product);
                    }
                }
            }
        }

        public void ExecuteStoreQuery()
        {
            using (NorthwindEntities context = new NorthwindEntities())
            {
                ObjectResult<Product> beverages = context.ExecuteStoreQuery<Product>(
@"    SELECT        P.ProductID, P.ProductName, P.SupplierID, P.CategoryID, P.QuantityPerUnit, P.UnitPrice, P.UnitsInStock, P.UnitsOnOrder, P.ReorderLevel, P.Discontinued
    FROM            Products AS P INNER JOIN Categories AS C ON P.CategoryID = C.CategoryID
    WHERE        (C.CategoryName = 'Beverages')"
);
                beverages.ToList();
            }
        }

        public void ExecuteStoreQueryDbContext()
        {
            using (var context = new QueryComparison.DbC.NorthwindEntities())
            {
                var beverages = context.Database.SqlQuery\<QueryComparison.DbC.Product>(
@"    SELECT        P.ProductID, P.ProductName, P.SupplierID, P.CategoryID, P.QuantityPerUnit, P.UnitPrice, P.UnitsInStock, P.UnitsOnOrder, P.ReorderLevel, P.Discontinued
    FROM            Products AS P INNER JOIN Categories AS C ON P.CategoryID = C.CategoryID
    WHERE        (C.CategoryName = 'Beverages')"
);
                beverages.ToList();
            }
        }

        public void ExecuteStoreQueryDbSet()
        {
            using (var context = new QueryComparison.DbC.NorthwindEntities())
            {
                var beverages = context.Products.SqlQuery(
@"    SELECT        P.ProductID, P.ProductName, P.SupplierID, P.CategoryID, P.QuantityPerUnit, P.UnitPrice, P.UnitsInStock, P.UnitsOnOrder, P.ReorderLevel, P.Discontinued
    FROM            Products AS P INNER JOIN Categories AS C ON P.CategoryID = C.CategoryID
    WHERE        (C.CategoryName = 'Beverages')"
);
                beverages.ToList();
            }
        }

        public void LINQIncludingContextCreationDbContext()
        {
            using (var context = new QueryComparison.DbC.NorthwindEntities())
            {                 
                var q = context.Products.Where(p => p.Category.CategoryName == "Beverages");
                q.ToList();
            }
        }

        public void LINQNoTrackingDbContext()
        {
            using (var context = new QueryComparison.DbC.NorthwindEntities())
            {
                var q = context.Products.AsNoTracking().Where(p => p.Category.CategoryName == "Beverages");
                q.ToList();
            }
        }
    }
}
```




  
  









  
  

  
  


  
  
  
  








  
  

  

  
  




  
  
   
