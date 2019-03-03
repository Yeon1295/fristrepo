# Querying and Finding Entities

### Finding entities using a query

- DbSet and IDbSet implement IQueryable and so can be used as the starting point for writing a LINQ query against the database. 

```
      using (var context = new BloggingContext())
      {
          // Query for all blogs with names starting with B
          var blogs = from b in context.Blogs
                         where b.Name.StartsWith("B")
                         select b;

          // Query for the Blog named ADO.NET Blog
          var blog = context.Blogs
                          .Where(b => b.Name == "ADO.NET Blog")
                          .FirstOrDefault();
      }
```

- Note that DbSet and IDbSet always create queries against the database and will always involve a round trip to the database even 
  if the entities returned already exist in the context. 
  
- A query is executed against the database when:
  
    - It is enumerated by a foreach (C#) or For Each (Visual Basic) statement.
    - It is enumerated by a collection operation such as ToArray, ToDictionary, or ToList.
    - LINQ operators such as First or Any are specified in the outermost part of the query.
    - The following methods are called: the Load extension method on a DbSet, DbEntityEntry.Reload, and Database.ExecuteSqlCommand.
    
- When results are returned from the database, objects that do not exist in the context are attached to the context. 
- If an object is already in the context, the existing object is returned (the current and original values of the object's properties 
  in the entry are not overwritten with database values).
  
- When you perform a query, entities that have been added to the context but have not yet been saved to the database are not returned 
  as part of the result set. To get the data that is in the context, see Local Data.
  
- If a query returns no rows from the database, the result will be an empty collection, rather than null.

### Finding entities using primary keys

- The Find method on DbSet uses the primary key value to attempt to find an entity tracked by the context. 
- If the entity is not found in the context then a query will be sent to the database to find the entity there. 
- Null is returned if the entity is not found in the context or in the database.

- Find is different from using a query in two significant ways:

  - A round-trip to the database will only be made if the entity with the given key is not found in the context.
  - Find will return entities that are in the Added state. That is, Find will return entities that have been added to the 
    context but have not yet been saved to the database.
 
##### Finding an entity by primary key

```
      using (var context = new BloggingContext())
      {
          // Will hit the database
          var blog = context.Blogs.Find(3);

          // Will return the same instance without hitting the database
          var blogAgain = context.Blogs.Find(3);

          context.Blogs.Add(new Blog { Id = -1 });

          // Will find the new blog even though it does not exist in the database
          var newBlog = context.Blogs.Find(-1);

          // Will find a User which has a string primary key
          var user = context.Users.Find("johndoe1987");
      }
```
##### Finding an entity by composite primary key

- Entity Framework allows your entities to have composite keys - that's a key that is made up of more than one property. 

- For example, you could have a BlogSettings entity that represents a users settings for a particular blog. 
- Because a user would only ever have one BlogSettings for each blog you could chose to make the primary key of BlogSettings 
  a combination of BlogId and Username. 
  
- The following code attempts to find the BlogSettings with BlogId = 3 and Username = "johndoe1987":

```
      using (var context = new BloggingContext())
      {
          var settings = context.BlogSettings.Find(3, "johndoe1987");
      }
```
- Note that when you have composite keys you need to use ColumnAttribute or the fluent API to specify an ordering for the 
  properties of the composite key. The call to Find must use this order when specifying the values that form the key.
  
### The Load Method
  
- There are several scenarios where you may want to load entities from the database into the context without immediately doing 
  anything with those entities. 
- A good example of this is loading entities for data binding as described in Local Data. 
- One common way to do this is to write a LINQ query and then call ToList on it, only to immediately discard the created list. 
- The Load extension method works just like ToList except that it avoids the creation of the list altogether.
  
- Here are two examples of using Load.
  
- The first is taken from a Windows Forms data binding application where Load is used to query for entities before binding to 
  the local collection, as described in Local Data:
    
```
      protected override void OnLoad(EventArgs e)
      {
          base.OnLoad(e);

          _context = new ProductContext();

          _context.Categories.Load();
          categoryBindingSource.DataSource = _context.Categories.Local.ToBindingList();
      }
```
- The second example shows using Load to load a filtered collection of related entities, as described in Loading Related Entities:
```
      using (var context = new BloggingContext())
      {
          var blog = context.Blogs.Find(1);

          // Load the posts with the 'entity-framework' tag related to a given blog
          context.Entry(blog)
              .Collection(b => b.Posts)
              .Query()
              .Where(p => p.Tags.Contains("entity-framework"))
              .Load();
      }
```

### Local Data
   
- Running a LINQ query directly against a DbSet will always send a query to the database, 
- but you can access the data that is currently in-memory using the DbSet.Local property. 
- You can also access the extra information EF is tracking about your entities using the DbContext.Entry and 
  DbContext.ChangeTracker.Entries methods.

##### Using Local to look at local data

- The Local property of DbSet provides simple access to the entities of the set that are currently being tracked by the context 
  and have not been marked as Deleted. 
- Accessing the Local property never causes a query to be sent to the database. 
- This means that it is usually used after a query has already been performed. 
- The Load extension method can be used to execute a query so that the context tracks the results.

```
      using (var context = new BloggingContext())
      {
          // Load all blogs from the database into the context
          context.Blogs.Load();

          // Add a new blog to the context
          context.Blogs.Add(new Blog { Name = "My New Blog" });

          // Mark one of the existing blogs as Deleted
          context.Blogs.Remove(context.Blogs.Find(1));

          // Loop over the blogs in the context.
          Console.WriteLine("In Local: ");
          foreach (var blog in context.Blogs.Local)
          {
              Console.WriteLine(
                  "Found {0}: {1} with state {2}",
                  blog.BlogId,  
                  blog.Name,
                  context.Entry(blog).State);
          }

          // Perform a query against the database.
          Console.WriteLine("\nIn DbSet query: ");
          foreach (var blog in context.Blogs)
          {
              Console.WriteLine(
                  "Found {0}: {1} with state {2}",
                  blog.BlogId,  
                  blog.Name,
                  context.Entry(blog).State);
          }
      }
```
```
In Local:
Found 0: My New Blog with state Added
Found 2: The Visual Studio Blog with state Unchanged

In DbSet query:
Found 1: ADO.NET Blog with state Deleted
Found 2: The Visual Studio Blog with state Unchanged
```

##### Using Local to add and remove entities from the context

- The Local property on DbSet returns an ObservableCollection with events hooked up such that it stays in sync with the contents 
  of the context. 
- This means that entities can be added or removed from either the Local collection or the DbSet. 
- It also means that queries that bring new entities into the context will result in the Local collection being updated 
  with those entities.
  
```
      using (var context = new BloggingContext())
      {
          // Load some posts from the database into the context
          context.Posts.Where(p => p.Tags.Contains("entity-framework").Load();  

          // Get the local collection and make some changes to it
          var localPosts = context.Posts.Local;
          localPosts.Add(new Post { Name = "What's New in EF" });
          localPosts.Remove(context.Posts.Find(1));  

          // Loop over the posts in the context.
          Console.WriteLine("In Local after entity-framework query: ");
          foreach (var post in context.Posts.Local)
          {
              Console.WriteLine(
                  "Found {0}: {1} with state {2}",
                  post.Id,  
                  post.Title,
                  context.Entry(post).State);
          }

          var post1 = context.Posts.Find(1);
          Console.WriteLine(
              "State of post 1: {0} is {1}",
              post1.Name,  
              context.Entry(post1).State);  

          // Query some more posts from the database
          context.Posts.Where(p => p.Tags.Contains("asp.net").Load();  

          // Loop over the posts in the context again.
          Console.WriteLine("\nIn Local after asp.net query: ");
          foreach (var post in context.Posts.Local)
          {
              Console.WriteLine(
                  "Found {0}: {1} with state {2}",
                  post.Id,  
                  post.Title,
                  context.Entry(post).State);
          }
      }
```
```
In Local after entity-framework query:
Found 3: EF Designer Basics with state Unchanged
Found 5: EF Code First Basics with state Unchanged
Found 0: What's New in EF with state Added
State of post 1: EF Beginners Guide is Deleted

In Local after asp.net query:
Found 3: EF Designer Basics with state Unchanged
Found 5: EF Code First Basics with state Unchanged
Found 0: What's New in EF with state Added
Found 4: ASP.NET Beginners Guide with state Unchanged
```
- One final thing to note about Local is that because it is an ObservableCollection performance is not great for large numbers 
  of entities. 
- Therefore if you are dealing with thousands of entities in your context it may not be advisable to use Local.

##### Using Local for WPF data binding

- The Local property on DbSet can be used directly for data binding in a WPF application because it is an instance of 
  ObservableCollection. 
- This means that it will automatically stay in sync with the contents of the context and the contents of the context will 
  automatically stay in sync with it. 
- Note that you do need to pre-populate the Local collection with data for there to be anything to bind to since Local 
  never causes a database query.
  
- the key elements for WPF data binding are:

    - Setup a binding source
    - Bind it to the Local property of your set
    - Populate Local using a query to the database.
    
##### WPF binding to navigation properties

- If you are doing master/detail data binding you may want to bind the detail view to a navigation property of one of your entities. 
- An easy way to make this work is to use an ObservableCollection for the navigation property.

```
      public class Blog
      {
          private readonly ObservableCollection<Post> _posts =
              new ObservableCollection<Post>();

          public int BlogId { get; set; }
          public string Name { get; set; }

          public virtual ObservableCollection<Post> Posts
          {
              get { return _posts; }
          }
      }
```

##### Using Local to clean up entities in SaveChanges

- In most cases entities removed from a navigation property will not be automatically marked as deleted in the context. 
- For example, if you remove a Post object from the Blog.Posts collection then that post will not be automatically deleted when 
  SaveChanges is called. 
- If you need it to be deleted then you may need to find these dangling entities and mark them as deleted before calling 
  SaveChanges or as part of an overridden SaveChanges.
  
```
      public override int SaveChanges()
      {
          foreach (var post in this.Posts.Local.ToList())
          {
              if (post.Blog == null)
              {
                  this.Posts.Remove(post);
              }
          }

          return base.SaveChanges();
      }
```
- The code above uses the Local collection to find all posts and marks any that do not have a blog reference as deleted. 
- The ToList call is required because otherwise the collection will be modified by the Remove call while it is being enumerated. 
- In most other situations you can query directly against the Local property without using ToList first.

##### Using Local and ToBindingList for Windows Forms data binding

- Windows Forms does not support full fidelity data binding using ObservableCollection directly. 
- However, you can still use the DbSet Local property for data binding to get all the benefits. 
- This is achieved through the ToBindingList extension method which creates an IBindingList implementation backed by the Local 
  ObservableCollection.
  
- the key elements for Windows Forms data binding are:

  - Setup an object binding source
  - Bind it to the Local property of your set using Local.ToBindingList()
  - Populate Local using a query to the database
  
##### Getting detailed information about tracked entities

- Many of the examples in this series use the Entry method to return a DbEntityEntry instance for an entity. 
- This entry object then acts as the starting point for gathering information about the entity such as its current state, as 
  well as for performing operations on the entity such as explicitly loading a related entity.
  
- The Entries methods return DbEntityEntry objects for many or all entities being tracked by the context. 
- This allows you to gather information or perform operations on many entities rather than just a single entry.

```
      using (var context = new BloggingContext())
      {
          // Load some entities into the context
          context.Blogs.Load();
          context.Authors.Load();
          context.Readers.Load();

          // Make some changes
          context.Blogs.Find(1).Title = "The New ADO.NET Blog";
          context.Blogs.Remove(context.Blogs.Find(2));
          context.Authors.Add(new Author { Name = "Jane Doe" });
          context.Readers.Find(1).Username = "johndoe1987";

          // Look at the state of all entities in the context
          Console.WriteLine("All tracked entities: ");
          foreach (var entry in context.ChangeTracker.Entries())
          {
              Console.WriteLine(
                  "Found entity of type {0} with state {1}",
                  ObjectContext.GetObjectType(entry.Entity.GetType()).Name,
                  entry.State);
          }

          // Find modified entities of any type
          Console.WriteLine("\nAll modified entities: ");
          foreach (var entry in context.ChangeTracker.Entries()
                                    .Where(e => e.State == EntityState.Modified))
          {
              Console.WriteLine(
                  "Found entity of type {0} with state {1}",
                  ObjectContext.GetObjectType(entry.Entity.GetType()).Name,
                  entry.State);
          }

          // Get some information about just the tracked blogs
          Console.WriteLine("\nTracked blogs: ");
          foreach (var entry in context.ChangeTracker.Entries<Blog>())
          {
              Console.WriteLine(
                  "Found Blog {0}: {1} with original Name {2}",
                  entry.Entity.BlogId,  
                  entry.Entity.Name,
                  entry.Property(p => p.Name).OriginalValue);
          }

          // Find all people (author or reader)
          Console.WriteLine("\nPeople: ");
          foreach (var entry in context.ChangeTracker.Entries<IPerson>())
          {
              Console.WriteLine("Found Person {0}", entry.Entity.Name);
          }
      }
```
- You'll notice we are introducing a Author and Reader class into the example - both of these classes implement the 
  IPerson interface.
  
```
    public class Author : IPerson
    {
        public int AuthorId { get; set; }
        public string Name { get; set; }
        public string Biography { get; set; }
   }

    public class Reader : IPerson
    {
        public int ReaderId { get; set; }
        public string Name { get; set; }
        public string Username { get; set; }
    }

    public interface IPerson
    {
        string Name { get; }
    }
```

- Let's assume we have the following data in the database:
      Blog with BlogId = 1 and Name = 'ADO.NET Blog'
      Blog with BlogId = 2 and Name = 'The Visual Studio Blog'
      Blog with BlogId = 3 and Name = '.NET Framework Blog'
      Author with AuthorId = 1 and Name = 'Joe Bloggs'
      Reader with ReaderId = 1 and Name = 'John Doe'    

- The output from running the code would be:
```
All tracked entities:
Found entity of type Blog with state Modified
Found entity of type Blog with state Deleted
Found entity of type Blog with state Unchanged
Found entity of type Author with state Unchanged
Found entity of type Author with state Added
Found entity of type Reader with state Modified`

All modified entities:
Found entity of type Blog with state Modified
Found entity of type Reader with state Modified

Tracked blogs:
Found Blog 1: The New ADO.NET Blog with original Name ADO.NET Blog
Found Blog 2: The Visual Studio Blog with original Name The Visual Studio Blog
Found Blog 3: .NET Framework Blog with original Name .NET Framework Blog

People:
Found Person John Doe
Found Person Joe Bloggs
Found Person Jane Doe
```
- These examples illustrate several points:

  - The Entries methods return entries for entities in all states, including Deleted. Compare this to Local which excludes 
    Deleted entities.
  - Entries for all entity types are returned when the non-generic Entries method is used. 
    When the generic entries method is used entries are only returned for entities that are instances of the generic type. 
    This was used above to get entries for all blogs. It was also used to get entries for all entities that implement IPerson. 
    This demonstrates that the generic type does not have to be an actual entity type.
  - LINQ to Objects can be used to filter the results returned. This was used above to find entities of any type as long as 
    they are modified.
    
- Note that DbEntityEntry instances always contain a non-null Entity. Relationship entries and stub entries are not represented 
  as DbEntityEntry instances so there is no need to filter for these.
  
 ### No-Tracking Queries
  
 - Sometimes you may want to get entities back from a query but not have those entities be tracked by the context. 
   This may result in better performance when querying for large numbers of entities in read-only scenarios.

- A new extension method AsNoTracking allows any query to be run in this way. For example:

```
      using (var context = new BloggingContext())
      {
          // Query for all blogs without tracking them
          var blogs1 = context.Blogs.AsNoTracking();

          // Query for some blogs without tracking them
          var blogs2 = context.Blogs
                              .Where(b => b.Name.Contains(".NET"))
                              .AsNoTracking()
                              .ToList();
      }
```
### Raw SQL Queries

- Entity Framework allows you to query using LINQ with your entity classes. 
- However, there may be times that you want to run queries using raw SQL directly against the database. 
- This includes calling stored procedures, which can be helpful for Code First models that currently do not support mapping to 
  stored procedures.
  
 ##### Writing SQL queries for entities
 
 - The SqlQuery method on DbSet allows a raw SQL query to be written that will return entity instances. The returned objects will 
   be tracked by the context just as they would be if they were returned by a LINQ query.

```
      using (var context = new BloggingContext())
      {
          var blogs = context.Blogs.SqlQuery("SELECT * FROM dbo.Blogs").ToList();
      }
```

- Note that, just as for LINQ queries, the query is not executed until the results are enumerated.

- Care should be taken whenever raw SQL queries are written for two reasons.

- First, the query should be written to ensure that it only returns entities that are really of the requested type. 
  For example, when using features such as inheritance it is easy to write a query that will create entities that are of the 
      wrong CLR type.

- Second, some types of raw SQL query expose potential security risks, especially around SQL injection attacks. Make sure that 
  you use parameters in your query in the correct way to guard against such attacks.
  
###### Loading entities from stored procedures

- You can use DbSet.SqlQuery to load entities from the results of a stored procedure. 
- For example, the following code calls the dbo.GetBlogs procedure in the database:
```
      using (var context = new BloggingContext())
      {
          var blogs = context.Blogs.SqlQuery("dbo.GetBlogs").ToList();
      }
```
- You can also pass parameters to a stored procedure using the following syntax:
```
      using (var context = new BloggingContext())
      {
          var blogId = 1;

          var blogs = context.Blogs.SqlQuery("dbo.GetBlogById @p0", blogId).Single();
      }
```
##### Writing SQL queries for non-entity types

- A SQL query returning instances of any type, including primitive types, can be created using the SqlQuery method on the 
  Database class.
- The results returned from SqlQuery on Database will never be tracked by the context even if the objects are instances 
  of an entity type.  
  
```
      using (var context = new BloggingContext())
      {
          var blogNames = context.Database.SqlQuery<string>(
                             "SELECT Name FROM dbo.Blogs").ToList();
      }  
```
##### Sending raw commands to the database

- Non-query commands can be sent to the database using the ExecuteSqlCommand method on Database.
- Note that any changes made to data in the database using ExecuteSqlCommand are opaque to the context until entities are loaded 
  or reloaded from the database.

```
      using (var context = new BloggingContext())
      {
          context.Database.ExecuteSqlCommand(
              "UPDATE dbo.Blogs SET Name = 'Another Name' WHERE BlogId = 1");
      }
```
###### Output Parameters

- If output parameters are used, their values will not be available until the results have been read completely. This is due to the underlying behavior 
  of DbDataReader.
  
### Loading Related Entities

- Entity Framework supports three ways to load related data - eager loading, lazy loading and explicit loading.

##### Eagerly Loading

- Eager loading is the process whereby a query for one type of entity also loads related entities as part of the query.
- Eager loading is achieved by use of the Include method. 

- For example, the queries below will load blogs and all the posts related to each blog.

```
      using (var context = new BloggingContext())
      {
          // Load all blogs and related posts
          var blogs1 = context.Blogs
                              .Include(b => b.Posts)
                              .ToList();

          // Load one blogs and its related posts
          var blog1 = context.Blogs
                             .Where(b => b.Name == "ADO.NET Blog")
                             .Include(b => b.Posts)
                             .FirstOrDefault();

          // Load all blogs and related posts  
          // using a string to specify the relationship
          var blogs2 = context.Blogs
                              .Include("Posts")
                              .ToList();

          // Load one blog and its related posts  
          // using a string to specify the relationship
          var blog2 = context.Blogs
                             .Where(b => b.Name == "ADO.NET Blog")
                             .Include("Posts")
                             .FirstOrDefault();
      }
```
- Note that Include is an extension method in the System.Data.Entity namespace so make sure you are using that namespace.

###### Eagerly loading multiple levels

- It is also possible to eagerly load multiple levels of related entities. 

- The queries below show examples of how to do this for both collection and reference navigation properties.
```
      using (var context = new BloggingContext())
      {
          // Load all blogs, all related posts, and all related comments
          var blogs1 = context.Blogs
                              .Include(b => b.Posts.Select(p => p.Comments))
                              .ToList();

          // Load all users, their related profiles, and related avatar
          var users1 = context.Users
                              .Include(u => u.Profile.Avatar)
                              .ToList();

          // Load all blogs, all related posts, and all related comments  
          // using a string to specify the relationships
          var blogs2 = context.Blogs
                              .Include("Posts.Comments")
                              .ToList();

          // Load all users, their related profiles, and related avatar  
          // using a string to specify the relationships
          var users2 = context.Users
                              .Include("Profile.Avatar")
                              .ToList();
      }
```

- Note that it is not currently possible to filter which related entities are loaded. 
- Include will always bring in all related entities.

##### Lazy Loading

- Lazy loading is the process whereby an entity or collection of entities is automatically loaded from the database the first 
  time that a property referring to the entity/entities is accessed. 
- When using POCO entity types, lazy loading is achieved by creating instances of derived proxy types and then overriding 
  virtual properties to add the loading hook. 
  
- For example, when using the Blog entity class defined below, the related Posts will be loaded the first time the Posts 
  navigation property is accessed:
  
```
      public class Blog
      {  
          public int BlogId { get; set; }  
          public string Name { get; set; }  
          public string Url { get; set; }  
          public string Tags { get; set; }  

          public virtual ICollection<Post> Posts { get; set; }  
      }
```

###### Turn lazy loading off for serialization

- Lazy loading and serialization don’t mix well, 
- and if you aren’t careful you can end up querying for your entire database just because lazy loading is enabled. 
- Most serializers work by accessing each property on an instance of a type. 
- Property access triggers lazy loading, so more entities get serialized. 
- On those entities properties are accessed, and even more entities are loaded. 
- It’s a good practice to turn lazy loading off before you serialize an entity.


###### Turning off lazy loading for specific navigation properties

- Lazy loading of the Posts collection can be turned off by making the Posts property non-virtual:

```
  public class Blog
  {  
      public int BlogId { get; set; }  
      public string Name { get; set; }  
      public string Url { get; set; }  
      public string Tags { get; set; }  

      public ICollection<Post> Posts { get; set; }  
  }
```
- Loading of the Posts collection can still be achieved using eager loading or the Load method.

###### Turn off lazy loading for all entities

- Lazy loading can be turned off for all entities in the context by setting a flag on the Configuration property. 

```
  public class BloggingContext : DbContext
  {
      public BloggingContext()
      {
          this.Configuration.LazyLoadingEnabled = false;
      }
  }
```
- Loading of the Posts collection can still be achieved using eager loading or the Load method.- 

##### Explicitly Loading

- Even with lazy loading disabled it is still possible to lazily load related entities, but it must be done with an explicit call. 
- To do so you use the Load method on the related entity’s entry.

```
      using (var context = new BloggingContext())
      {
          var post = context.Posts.Find(2);

          // Load the blog related to a given post
          context.Entry(post).Reference(p => p.Blog).Load();

          // Load the blog related to a given post using a string  
          context.Entry(post).Reference("Blog").Load();

          var blog = context.Blogs.Find(1);

          // Load the posts related to a given blog
          context.Entry(blog).Collection(p => p.Posts).Load();

          // Load the posts related to a given blog  
          // using a string to specify the relationship
          context.Entry(blog).Collection("Posts").Load();
}
```

- Note that the Reference method should be used when an entity has a navigation property to another single entity. 
- On the other hand, the Collection method should be used when an entity has a navigation property to a collection of other entities.

###### Applying filters when explicitly loading related entities

- The Query method provides access to the underlying query that Entity Framework will use when loading related entities. 
- You can then use LINQ to apply filters to the query before executing it with a call to a LINQ extension method such as ToList, 
  Load, etc. 
- The Query method can be used with both reference and collection navigation properties but is most useful for collections where 
  it can be used to load only part of the collection.
  
```
      using (var context = new BloggingContext())
      {
          var blog = context.Blogs.Find(1);

          // Load the posts with the 'entity-framework' tag related to a given blog
          context.Entry(blog)
                 .Collection(b => b.Posts)
                 .Query()
                 .Where(p => p.Tags.Contains("entity-framework"))
                 .Load();

          // Load the posts with the 'entity-framework' tag related to a given blog  
          // using a string to specify the relationship  
          context.Entry(blog)
                 .Collection("Posts")
                 .Query()
                 .Where(p => p.Tags.Contains("entity-framework"))
                 .Load();
      }
```
- When using the Query method it is usually best to turn off lazy loading for the navigation property. 
- This is because otherwise the entire collection may get loaded automatically by the lazy loading mechanism either before 
  or after the filtered query has been executed.

- Note that while the relationship can be specified as a string instead of a lambda expression, the returned IQueryable is not 
  generic when a string is used and so the Cast method is usually needed before anything useful can be done with it.
  
##### Using Query to count related entities without loading them

- Sometimes it is useful to know how many entities are related to another entity in the database without actually incurring the 
  cost of loading all those entities. 
- The Query method with the LINQ Count method can be used to do this.

```
      using (var context = new BloggingContext())
      {
          var blog = context.Blogs.Find(1);

          // Count how many posts the blog has  
          var postCount = context.Entry(blog)
                                 .Collection(b => b.Posts)
                                 .Query()
                                 .Count();
      }
```


  

  



  



   







    
    
    
    
    


    
