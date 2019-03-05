# Saving Data with Entity Framework 6

### Automatic detect changes

- When using most POCO entities the determination of how an entity has changed (and therefore which updates need to be sent to 
  the database) is handled by the Detect Changes algorithm. 
- Detect Changes works by detecting the differences between the current property values of the entity and the original property 
  values that are stored in a snapshot when the entity was queried or attached.
  
- By default, Entity Framework performs Detect Changes automatically when the following methods are called:

        DbSet.Find
        DbSet.Local
        DbSet.Add
        DbSet.AddRange
        DbSet.Remove
        DbSet.RemoveRange
        DbSet.Attach
        DbContext.SaveChanges
        DbContext.GetValidationErrors
        DbContext.Entry
        DbChangeTracker.Entries  
        
##### Disabling automatic detection of changes

- If you are tracking a lot of entities in your context and you call one of these methods many times in a loop, then you may get 
  significant performance improvements by turning off detection of changes for the duration of the loop.
  
  ```
      using (var context = new BloggingContext())
      {
          try
          {
              context.Configuration.AutoDetectChangesEnabled = false;

              // Make many calls in a loop
              foreach (var blog in aLotOfBlogs)
              {
                  context.Blogs.Add(blog);
              }
          }
          finally
          {
              context.Configuration.AutoDetectChangesEnabled = true;
          }
      }
  ```
- Don’t forget to re-enable detection of changes after the loop — We've used a try/finally to ensure it is always re-enabled 
  even if code in the loop throws an exception.
 
- An alternative to disabling and re-enabling is to leave automatic detection of changes turned off at all times and either 
  call context.ChangeTracker.DetectChanges explicitly or use change tracking proxies diligently. 
- Both of these options are advanced and can easily introduce subtle bugs into your application so use them with care.

- If you need to add or remove many objects from a context, consider using DbSet.AddRange and DbSet.RemoveRange. 
- This methods automatically detect changes only once after the add or remove operations are completed.

### Working with entity states

- Entity Framework takes care of tracking the state of entities while they are connected to a context, but in disconnected 
  or N-Tier scenarios you can let EF know what state your entities should be in.
  
##### Entity states and SaveChanges

- An entity can be in one of five states as defined by the EntityState enumeration. These states are:

    - Added: 
      the entity is being tracked by the context but does not yet exist in the database
    
    - Unchanged: 
        the entity is being tracked by the context and exists in the database, and its property values have not changed from the 
        values in the database
        
    - Modified: 
        the entity is being tracked by the context and exists in the database, and some or all of its property values have 
        been modified
        
    - Deleted: 
        the entity is being tracked by the context and exists in the database, but has been marked for deletion from the database 
        the next time SaveChanges is called
        
    - Detached: 
          the entity is not being tracked by the context
          
 - SaveChanges does different things for entities in different states:
 
    - Unchanged entities are not touched by SaveChanges. Updates are not sent to the database for entities in the Unchanged state.

    - Added entities are inserted into the database and then become Unchanged when SaveChanges returns.
    
    - Modified entities are updated in the database and then become Unchanged when SaveChanges returns.
    
    - Deleted entities are deleted from the database and are then detached from the context.
    
##### Adding a new entity to the context

- A new entity can be added to the context by calling the Add method on DbSet. This puts the entity into the Added state, meaning 
  that it will be inserted into the database the next time that SaveChanges is called.
  
```
      using (var context = new BloggingContext())
      {
          var blog = new Blog { Name = "ADO.NET Blog" };
          context.Blogs.Add(blog);
          context.SaveChanges();
      }
```
- Another way to add a new entity to the context is to change its state to Added.

```
      using (var context = new BloggingContext())
      {
          var blog = new Blog { Name = "ADO.NET Blog" };
          context.Entry(blog).State = EntityState.Added;
          context.SaveChanges();
      }
```
- Finally, you can add a new entity to the context by hooking it up to another entity that is already being tracked. 
- This could be by adding the new entity to the collection navigation property of another entity or by setting a reference 
   navigation property of another entity to point to the new entity.
   
```
      using (var context = new BloggingContext())
      {
          // Add a new User by setting a reference from a tracked Blog
          var blog = context.Blogs.Find(1);
          blog.Owner = new User { UserName = "johndoe1987" };

          // Add a new Post by adding to the collection of a tracked Blog
          blog.Posts.Add(new Post { Name = "How to Add Entities" });

          context.SaveChanges();
      }
```
- Note that for all of these examples if the entity being added has references to other entities that are not yet tracked then 
  these new entities will also be added to the context and will be inserted into the database the next time that SaveChanges is 
  called.

##### Attaching an existing entity to the context

- If you have an entity that you know already exists in the database but which is not currently being tracked by the context then 
  you can tell the context to track the entity using the Attach method on DbSet. The entity will be in the Unchanged state in the 
  context.
  
```
      var existingBlog = new Blog { BlogId = 1, Name = "ADO.NET Blog" };

      using (var context = new BloggingContext())
      {
          context.Blogs.Attach(existingBlog);

          // Do some more work...  

          context.SaveChanges();
      }
```
- Note that no changes will be made to the database if SaveChanges is called without doing any other manipulation of the attached 
  entity. This is because the entity is in the Unchanged state.
  
- Another way to attach an existing entity to the context is to change its state to Unchanged.

```
      var existingBlog = new Blog { BlogId = 1, Name = "ADO.NET Blog" };

      using (var context = new BloggingContext())
      {
          context.Entry(existingBlog).State = EntityState.Unchanged;

          // Do some more work...  

          context.SaveChanges();
      }
```
- Note that for both of these examples if the entity being attached has references to other entities that are not yet tracked 
  then these new entities will also attached to the context in the Unchanged state.
  
##### Attaching an existing but modified entity to the context

- If you have an entity that you know already exists in the database but to which changes may have been made then you can tell 
  the context to attach the entity and set its state to Modified.
  
```
      var existingBlog = new Blog { BlogId = 1, Name = "ADO.NET Blog" };

      using (var context = new BloggingContext())
      {
          context.Entry(existingBlog).State = EntityState.Modified;

          // Do some more work...  

          context.SaveChanges();
      }
```
- When you change the state to Modified all the properties of the entity will be marked as modified and all the property values 
  will be sent to the database when SaveChanges is called.
  
- Note that if the entity being attached has references to other entities that are not yet tracked, then these new entities 
  will attached to the context in the Unchanged state—they will not automatically be made Modified. 
- If you have multiple entities that need to be marked Modified you should set the state for each of these entities individually.

##### Changing the state of a tracked entity

- You can change the state of an entity that is already being tracked by setting the State property on its entry.

```
      var existingBlog = new Blog { BlogId = 1, Name = "ADO.NET Blog" };

      using (var context = new BloggingContext())
      {
          context.Blogs.Attach(existingBlog);
          context.Entry(existingBlog).State = EntityState.Unchanged;

          // Do some more work...  

          context.SaveChanges();
      }
```
- Note that calling Add or Attach for an entity that is already tracked can also be used to change the entity state. 
- For example, calling Attach for an entity that is currently in the Added state will change its state to Unchanged.

##### Insert or update pattern

- A common pattern for some applications is to either Add an entity as new (resulting in a database insert) or Attach 
  an entity as existing and mark it as modified (resulting in a database update) depending on the value of the primary key. 
- For example, when using database generated integer primary keys it is common to treat an entity with a zero key as new 
  and an entity with a non-zero key as existing. 
- This pattern can be achieved by setting the entity state based on a check of the primary key value.

```
      public void InsertOrUpdate(Blog blog)
      {
          using (var context = new BloggingContext())
          {
              context.Entry(blog).State = blog.BlogId == 0 ?
                                         EntityState.Added :
                                         EntityState.Modified;

              context.SaveChanges();
          }
      }
```
- Note that when you change the state to Modified all the properties of the entity will be marked as modified and all the property 
  values will be sent to the database when SaveChanges is called.
  
### Working with property values

- Entity Framework keeps track of two values for each property of a tracked entity. 
- The current value is, as the name indicates, the current value of the property in the entity. 
- The original value is the value that the property had when the entity was queried from the database or attached to the context.

- There are two general mechanisms for working with property values:

    - The value of a single property can be obtained in a strongly typed way using the Property method.
    
    - Values for all properties of an entity can be read into a DbPropertyValues object. 
      DbPropertyValues then acts as a dictionary-like object to allow property values to be read and set. 
      The values in a DbPropertyValues object can be set from values in another DbPropertyValues object or from values in some 
      other object, such as another copy of the entity or a simple data transfer object (DTO).

##### Getting and setting the current or original value of an individual property

- The example below shows how the current value of a property can be read and then set to a new value:

```
      using (var context = new BloggingContext())
      {
          var blog = context.Blogs.Find(3);

          // Read the current value of the Name property
          string currentName1 = context.Entry(blog).Property(u => u.Name).CurrentValue;

          // Set the Name property to a new value
          context.Entry(name).Property(u => u.Name).CurrentValue = "My Fancy Blog";

          // Read the current value of the Name property using a string for the property name
          object currentName2 = context.Entry(blog).Property("Name").CurrentValue;

          // Set the Name property to a new value using a string for the property name
          context.Entry(blog).Property("Name").CurrentValue = "My Boring Blog";
      }
```

- Use the OriginalValue property instead of the CurrentValue property to read or set the original value.

- Note that the returned value is typed as “object” when a string is used to specify the property name. 
  On the other hand, the returned value is strongly typed if a lambda expression is used.
  
- Setting the property value like this will only mark the property as modified if the new value is different from the old value.

- When a property value is set in this way the change is automatically detected even if AutoDetectChanges is turned off.

##### Getting and setting the current value of an unmapped property

- The current value of a property that is not mapped to the database can also be read. 

- An example of an unmapped property could be an RssLink property on Blog. This value may be calculated based on the BlogId, 
  and therefore doesn't need to be stored in the database.
  
```
      using (var context = new BloggingContext())
      {
          var blog = context.Blogs.Find(1);
          // Read the current value of an unmapped property
          var rssLink = context.Entry(blog).Property(p => p.RssLink).CurrentValue;

          // Use a string to specify the property name
          var rssLinkAgain = context.Entry(blog).Property("RssLink").CurrentValue;
      }
```

- The current value can also be set if the property exposes a setter.

- Reading the values of unmapped properties is useful when performing Entity Framework validation of unmapped properties. 
- For the same reason current values can be read and set for properties of entities that are not currently being tracked by 
  the context.
  
```
      using (var context = new BloggingContext())
      {
          // Create an entity that is not being tracked
          var blog = new Blog { Name = "ADO.NET Blog" };

          // Read and set the current value of Name as before
          var currentName1 = context.Entry(blog).Property(u => u.Name).CurrentValue;
          context.Entry(blog).Property(u => u.Name).CurrentValue = "My Fancy Blog";
          var currentName2 = context.Entry(blog).Property("Name").CurrentValue;
          context.Entry(blog).Property("Name").CurrentValue = "My Boring Blog";
      }
```
 - Note that original values are not available for unmapped properties or for properties of entities that are not being tracked 
   by the context.
   
   ##### Checking whether a property is marked as modified
   
   - The example below shows how to check whether or not an individual property is marked as modified:
   
```
      using (var context = new BloggingContext())
      {
          var blog = context.Blogs.Find(1);

          var nameIsModified1 = context.Entry(blog).Property(u => u.Name).IsModified;

          // Use a string for the property name
          var nameIsModified2 = context.Entry(blog).Property("Name").IsModified;
      }
```
- The values of modified properties are sent as updates to the database when SaveChanges is called.

##### Marking a property as modified

- The example below shows how to force an individual property to be marked as modified:

```
      using (var context = new BloggingContext())
      {
          var blog = context.Blogs.Find(1);

          context.Entry(blog).Property(u => u.Name).IsModified = true;

          // Use a string for the property name
          context.Entry(blog).Property("Name").IsModified = true;
      }
```
- Marking a property as modified forces an update to be send to the database for the property when SaveChanges is called even 
  if the current value of the property is the same as its original value.
  
- It is not currently possible to reset an individual property to be not modified after it has been marked as modified.

##### Reading current, original, and database values for all properties of an entity

- The example below shows how to read the current values, the original values, and the values actually in the database for all mapped properties 
  of an entity.

```
      using (var context = new BloggingContext())
      {
          var blog = context.Blogs.Find(1);

          // Make a modification to Name in the tracked entity
          blog.Name = "My Cool Blog";

          // Make a modification to Name in the database
          context.Database.SqlCommand("update dbo.Blogs set Name = 'My Boring Blog' where Id = 1");

          // Print out current, original, and database values
          Console.WriteLine("Current values:");
          PrintValues(context.Entry(blog).CurrentValues);

          Console.WriteLine("\nOriginal values:");
          PrintValues(context.Entry(blog).OriginalValues);

          Console.WriteLine("\nDatabase values:");
          PrintValues(context.Entry(blog).GetDatabaseValues());
      }

      public static void PrintValues(DbPropertyValues values)
      {
          foreach (var propertyName in values.PropertyNames)
          {
              Console.WriteLine("Property {0} has value {1}",
                                propertyName, values[propertyName]);
          }
      }
```
- The current values are the values that the properties of the entity currently contain. 
- The original values are the values that were read from the database when the entity was queried. 
- The database values are the values as they are currently stored in the database. 
- Getting the database values is useful when the values in the database may have changed since the entity was queried such as 
  when a concurrent edit to the database has been made by another user.
  
##### Setting current or original values from another object

- The current or original values of a tracked entity can be updated by copying values from another object. For example:

```
      using (var context = new BloggingContext())
      {
          var blog = context.Blogs.Find(1);
          var coolBlog = new Blog { Id = 1, Name = "My Cool Blog" };
          var boringBlog = new BlogDto { Id = 1, Name = "My Boring Blog" };

          // Change the current and original values by copying the values from other objects
          var entry = context.Entry(blog);
          entry.CurrentValues.SetValues(coolBlog);
          entry.OriginalValues.SetValues(boringBlog);

          // Print out current and original values
          Console.WriteLine("Current values:");
          PrintValues(entry.CurrentValues);

          Console.WriteLine("\nOriginal values:");
          PrintValues(entry.OriginalValues);
      }

      public class BlogDto
      {
          public int Id { get; set; }
          public string Name { get; set; }
      }
```
- This technique is sometimes used when updating an entity with values obtained from a service call or a client in an n-tier 
  application. 
  
- Note that the object used does not have to be of the same type as the entity so long as it has properties whose names 
  match those of the entity. 
- In the example above, an instance of BlogDTO is used to update the original values.

- Note that only properties that are set to different values when copied from the other object will be marked as modified.

##### Setting current or original values from a dictionary

- The current or original values of a tracked entity can be updated by copying values from a dictionary or some other data structure.

```
      using (var context = new BloggingContext())
      {
          var blog = context.Blogs.Find(1);

          var newValues = new Dictionary<string, object>
          {
              { "Name", "The New ADO.NET Blog" },
              { "Url", "blogs.msdn.com/adonet" },
          };

          var currentValues = context.Entry(blog).CurrentValues;

          foreach (var propertyName in newValues.Keys)
          {
              currentValues[propertyName] = newValues[propertyName];
          }

          PrintValues(currentValues);
      }
```
- Use the OriginalValues property instead of the CurrentValues property to set original values.

##### Setting current or original values from a dictionary using Property

- An alternative to using CurrentValues or OriginalValues as shown above is to use the Property method to set the value of each property. 
- This can be preferable when you need to set the values of complex properties.

```
      using (var context = new BloggingContext())
      {
          var user = context.Users.Find("johndoe1987");

          var newValues = new Dictionary<string, object>
          {
              { "Name", "John Doe" },
              { "Location.City", "Redmond" },
              { "Location.State.Name", "Washington" },
              { "Location.State.Code", "WA" },
          };

          var entry = context.Entry(user);

          foreach (var propertyName in newValues.Keys)
          {
              entry.Property(propertyName).CurrentValue = newValues[propertyName];
          }
      }
```
##### Creating a cloned object containing current, original, or database values

- The DbPropertyValues object returned from CurrentValues, OriginalValues, or GetDatabaseValues can be used to create a clone 
  of the entity. 
- This clone will contain the property values from the DbPropertyValues object used to create it.

```
      using (var context = new BloggingContext())
      {
          var blog = context.Blogs.Find(1);

          var clonedBlog = context.Entry(blog).GetDatabaseValues().ToObject();
      }
```
- Note that the object returned is not the entity and is not being tracked by the context. 
- The returned object also does not have any relationships set to other objects.

- The cloned object can be useful for resolving issues related to concurrent updates to the database, especially where a UI that 
  involves data binding to objects of a certain type is being used.
  
##### Getting and setting the current or original values of complex properties

- The value of an entire complex object can be read and set using the Property method just as it can be for a primitive property. 
- In addition you can drill down into the complex object and read or set properties of that object, or even a nested object.

```
      using (var context = new BloggingContext())
      {
          var user = context.Users.Find("johndoe1987");

          // Get the Location complex object
          var location = context.Entry(user)
                             .Property(u => u.Location)
                             .CurrentValue;

          // Get the nested State complex object using chained calls
          var state1 = context.Entry(user)
                           .ComplexProperty(u => u.Location)
                           .Property(l => l.State)
                           .CurrentValue;

          // Get the nested State complex object using a single lambda expression
          var state2 = context.Entry(user)
                           .Property(u => u.Location.State)
                           .CurrentValue;

          // Get the nested State complex object using a dotted string
          var state3 = context.Entry(user)
                           .Property("Location.State")
                           .CurrentValue;

          // Get the value of the Name property on the nested State complex object using chained calls
          var name1 = context.Entry(user)
                             .ComplexProperty(u => u.Location)
                             .ComplexProperty(l => l.State)
                             .Property(s => s.Name)
                             .CurrentValue;

          // Get the value of the Name property on the nested State complex object using a single lambda expression
          var name2 = context.Entry(user)
                             .Property(u => u.Location.State.Name)
                             .CurrentValue;

          // Get the value of the Name property on the nested State complex object using a dotted string
          var name3 = context.Entry(user)
                             .Property("Location.State.Name")
                             .CurrentValue;
      }
```
- Use the OriginalValue property instead of the CurrentValue property to get or set an original value.

- Note that either the Property or the ComplexProperty method can be used to access a complex property. 
- However, the ComplexProperty method must be used if you wish to drill down into the complex object with additional Property 
  or ComplexProperty calls.

##### Using DbPropertyValues to access complex properties

- When you use CurrentValues, OriginalValues, or GetDatabaseValues to get all the current, original, or database values for an 
  entity, the values of any complex properties are returned as nested DbPropertyValues objects. 
- These nested objects can then be used to get values of the complex object. 

- For example, the following method will print out the values of all properties, including values of any complex properties and 
  nested complex properties.

```
      public static void WritePropertyValues(string parentPropertyName, DbPropertyValues propertyValues)
      {
          foreach (var propertyName in propertyValues.PropertyNames)
          {
              var nestedValues = propertyValues[propertyName] as DbPropertyValues;
              if (nestedValues != null)
              {
                  WritePropertyValues(parentPropertyName + propertyName + ".", nestedValues);
              }
              else
              {
                  Console.WriteLine("Property {0}{1} has value {2}",
                                    parentPropertyName, propertyName,
                                    propertyValues[propertyName]);
              }
          }
      }
```
- To print out all current property values the method would be called like this:

```
      using (var context = new BloggingContext())
      {
          var user = context.Users.Find("johndoe1987");

          WritePropertyValues("", context.Entry(user).CurrentValues);
      }
```

### Handling Concurrency Conflicts

- Optimistic concurrency involves optimistically attempting to save your entity to the database in the hope that the data there 
  has not changed since the entity was loaded. 
- If it turns out that the data has changed then an exception is thrown and you must resolve the conflict before attempting to save
  again. 
  
- Resolving concurrency issues when you are using independent associations (where the foreign key is not mapped to a property 
  in your entity) is much more difficult than when you are using foreign key associations. Therefore 
- if you are going to do concurrency resolution in your application it is advised that you always map foreign keys into your entities.

- A DbUpdateConcurrencyException is thrown by SaveChanges when an optimistic concurrency exception is detected while attempting 
  to save an entity that uses foreign key associations.
  
##### Resolving optimistic concurrency exceptions with Reload (database wins) 

- The Reload method can be used to overwrite the current values of the entity with the values now in the database. 
- The entity is then typically given back to the user in some form and they must try to make their changes again and re-save.

```
      using (var context = new BloggingContext())
      {
          var blog = context.Blogs.Find(1);
          blog.Name = "The New ADO.NET Blog";

          bool saveFailed;
          do
          {
              saveFailed = false;

              try
              {
                  context.SaveChanges();
              }
              catch (DbUpdateConcurrencyException ex)
              {
                  saveFailed = true;

                  // Update the values of the entity that failed to save from the store
                  ex.Entries.Single().Reload();
              }

          } while (saveFailed);
      }
```
- A good way to simulate a concurrency exception is to set a breakpoint on the SaveChanges call and then modify an entity that 
  is being saved in the database using another tool such as SQL Management Studio. 
  
- You could also insert a line before SaveChanges to update the database directly using SqlCommand.
```
    context.Database.SqlCommand(
        "UPDATE dbo.Blogs SET Name = 'Another Name' WHERE BlogId = 1");
```
- The Entries method on DbUpdateConcurrencyException returns the DbEntityEntry instances for the entities that failed to update. 
  (This property currently always returns a single value for concurrency issues. It may return multiple values for general update
  exceptions.) 
- An alternative for some situations might be to get entries for all entities that may need to be reloaded from the database and 
  call reload for each of these.
  
##### Resolving optimistic concurrency exceptions as client wins

- The example above that uses Reload is sometimes called database wins or store wins because the values in the entity are overwritten 
  by values from the database. 
- Sometimes you may wish to do the opposite and overwrite the values in the database with the values currently in the entity. This 
  is sometimes called client wins and can be done by getting the current database values and setting them as the original values 
  for the entity.
    
```
      using (var context = new BloggingContext())
      {
          var blog = context.Blogs.Find(1);
          blog.Name = "The New ADO.NET Blog";

          bool saveFailed;
          do
          {
              saveFailed = false;
              try
              {
                  context.SaveChanges();
              }
              catch (DbUpdateConcurrencyException ex)
              {
                  saveFailed = true;

                  // Update original values from the database
                  var entry = ex.Entries.Single();
                  entry.OriginalValues.SetValues(entry.GetDatabaseValues());
              }

          } while (saveFailed);
      }
```
##### Custom resolution of optimistic concurrency exceptions

- Sometimes you may want to combine the values currently in the database with the values currently in the entity. 
- This usually requires some custom logic or user interaction. For example, you might present a form to the user containing the 
  current values, the values in the database, and a default set of resolved values. 
- The user would then edit the resolved values as necessary and it would be these resolved values that get saved to the database. 
- This can be done using the DbPropertyValues objects returned from CurrentValues and GetDatabaseValues on the entity’s entry.

```
      using (var context = new BloggingContext())
      {
          var blog = context.Blogs.Find(1);
          blog.Name = "The New ADO.NET Blog";

          bool saveFailed;
          do
          {
              saveFailed = false;
              try
              {
                  context.SaveChanges();
              }
              catch (DbUpdateConcurrencyException ex)
              {
                  saveFailed = true;

                  // Get the current entity values and the values in the database
                  var entry = ex.Entries.Single();
                  var currentValues = entry.CurrentValues;
                  var databaseValues = entry.GetDatabaseValues();

                  // Choose an initial set of resolved values. In this case we
                  // make the default be the values currently in the database.
                  var resolvedValues = databaseValues.Clone();

                  // Have the user choose what the resolved values should be
                  HaveUserResolveConcurrency(currentValues, databaseValues, resolvedValues);

                  // Update the original values with the database values and
                  // the current values with whatever the user choose.
                  entry.OriginalValues.SetValues(databaseValues);
                  entry.CurrentValues.SetValues(resolvedValues);
              }
          } while (saveFailed);
      }

      public void HaveUserResolveConcurrency(DbPropertyValues currentValues,
                                             DbPropertyValues databaseValues,
                                             DbPropertyValues resolvedValues)
      {
          // Show the current, database, and resolved values to the user and have
          // them edit the resolved values to get the correct resolution.
      }
```

##### Custom resolution of optimistic concurrency exceptions using objects

- The code above uses DbPropertyValues instances for passing around current, database, and resolved values. 
- Sometimes it may be easier to use instances of your entity type for this. This can be done using the ToObject and SetValues 
  methods of DbPropertyValues.
  
```
      using (var context = new BloggingContext())
      {
          var blog = context.Blogs.Find(1);
          blog.Name = "The New ADO.NET Blog";

          bool saveFailed;
          do
          {
              saveFailed = false;
              try
              {
                  context.SaveChanges();
              }
              catch (DbUpdateConcurrencyException ex)
              {
                  saveFailed = true;

                  // Get the current entity values and the values in the database
                  // as instances of the entity type
                  var entry = ex.Entries.Single();
                  var databaseValues = entry.GetDatabaseValues();
                  var databaseValuesAsBlog = (Blog)databaseValues.ToObject();

                  // Choose an initial set of resolved values. In this case we
                  // make the default be the values currently in the database.
                  var resolvedValuesAsBlog = (Blog)databaseValues.ToObject();

                  // Have the user choose what the resolved values should be
                  HaveUserResolveConcurrency((Blog)entry.Entity,
                                             databaseValuesAsBlog,
                                             resolvedValuesAsBlog);

                  // Update the original values with the database values and
                  // the current values with whatever the user choose.
                  entry.OriginalValues.SetValues(databaseValues);
                  entry.CurrentValues.SetValues(resolvedValuesAsBlog);
              }

          } while (saveFailed);
      }

      public void HaveUserResolveConcurrency(Blog entity,
                                             Blog databaseValues,
                                             Blog resolvedValues)
      {
          // Show the current, database, and resolved values to the user and have
          // them update the resolved values to get the correct resolution.
      }
```

### Working with Transactions

##### What EF does by default

- In all versions of Entity Framework, whenever you execute SaveChanges() to insert, update or delete on the database the framework 
  will wrap that operation in a transaction. 
- This transaction lasts only long enough to execute the operation and then completes. When you execute another such operation a new
  transaction is started.
  
- Starting with EF6 Database.ExecuteSqlCommand() by default will wrap the command in a transaction if one was not already present. 
- There are overloads of this method that allow you to override this behavior if you wish. 
- Also in EF6 execution of stored procedures included in the model through APIs such as ObjectContext.ExecuteFunction() does the 
  same (except that the default behavior cannot at the moment be overridden).
  
- In either case, the isolation level of the transaction is whatever isolation level the database provider considers its default
  setting. By default, for instance, on SQL Server this is READ COMMITTED.
  
 - Entity Framework does not wrap queries in a transaction.
 
- This default functionality is suitable for a lot of users and if so there is no need to do anything different in EF6; just write 
the code as you always did.

- However some users require greater control over their transactions.

##### How the APIs work

- Prior to EF6 Entity Framework insisted on opening the database connection itself (it threw an exception if it was passed a 
  connection that was already open). 
- Since a transaction can only be started on an open connection, this meant that the only way a user could wrap several operations 
  into one transaction was either to use a TransactionScope or use the ObjectContext.Connection property and start calling Open() 
  and BeginTransaction() directly on the returned EntityConnection object. 
- In addition, API calls which contacted the database would fail if you had started a transaction on the underlying database
    connection on your own.
    
- The limitation of only accepting closed connections was removed in Entity Framework 6. 

- Starting with EF6 the framework now provides:

    1. Database.BeginTransaction() : 
        An easier method for a user to start and complete transactions themselves within an existing DbContext – allowing several
        operations to be combined within the same transaction and hence either all committed or all rolled back as one. 
        It also allows the user to more easily specify the isolation level for the transaction.
        
    2. Database.UseTransaction() : 
        which allows the DbContext to use a transaction which was started outside of the Entity Framework.
      
##### Combining several operations into one transaction within the same context

- Database.BeginTransaction() has two overrides – one which takes an explicit IsolationLevel and one which takes no arguments and 
  uses the default IsolationLevel from the underlying database provider. 
- Both overrides return a DbContextTransaction object which provides Commit() and Rollback() methods which perform commit and 
  rollback on the underlying store transaction.
  
- The DbContextTransaction is meant to be disposed once it has been committed or rolled back. One easy way to accomplish this is 
  the using(…) {…} syntax which will automatically call Dispose() when the using block completes:
  
```
      using System;
      using System.Collections.Generic;
      using System.Data.Entity;
      using System.Data.SqlClient;
      using System.Linq;
      using System.Transactions;

      namespace TransactionsExamples
      {
          class TransactionsExample
          {
              static void StartOwnTransactionWithinContext()
              {
                  using (var context = new BloggingContext())
                  {
                      using (var dbContextTransaction = context.Database.BeginTransaction())
                      {
                          try
                          {
                              context.Database.ExecuteSqlCommand(
                                  @"UPDATE Blogs SET Rating = 5" +
                                      " WHERE Name LIKE '%Entity Framework%'"
                                  );

                              var query = context.Posts.Where(p => p.Blog.Rating >= 5);
                              foreach (var post in query)
                              {
                                  post.Title += "[Cool Blog]";
                              }

                              context.SaveChanges();

                              dbContextTransaction.Commit();
                          }
                          catch (Exception)
                          {
                              dbContextTransaction.Rollback();
                          }
                      }
                  }
              }
          }
      }
```

- Beginning a transaction requires that the underlying store connection is open. So calling Database.BeginTransaction() will open 
  the connection if it is not already opened. If DbContextTransaction opened the connection then it will close it when Dispose() 
  is called.
  
  ##### Passing an existing transaction to the context
  
  - Sometimes you would like a transaction which is even broader in scope and which includes operations on the same database but 
    outside of EF completely. 
  - To accomplish this you must open the connection and start the transaction yourself and then tell EF a) to use the already-opened
    database connection, and b) to use the existing transaction on that connection.
    
- To do this you must define and use a constructor on your context class which inherits from one of the DbContext constructors which
  take i) an existing connection parameter and ii) the contextOwnsConnection boolean.
  
- The contextOwnsConnection flag must be set to false when called in this scenario. This is important as it informs Entity Framework
  that it should not close the connection when it is done with it  
  
```  
      using (var conn = new SqlConnection("..."))
      {
          conn.Open();
          using (var context = new BloggingContext(conn, contextOwnsConnection: false))
          {
          }
      }
```
- Furthermore, you must start the transaction yourself (including the IsolationLevel if you want to avoid the default setting) and 
  let Entity Framework know that there is an existing transaction already started on the connection.
  
- Then you are free to execute database operations either directly on the SqlConnection itself, or on the DbContext. All such 
  operations are executed within one transaction. 
- You take responsibility for committing or rolling back the transaction and for calling Dispose() on it, as well as for closing 
  and disposing the database connection.
  
```
      using System;
      using System.Collections.Generic;
      using System.Data.Entity;
      using System.Data.SqlClient;
      using System.Linq;
      sing System.Transactions;

      namespace TransactionsExamples
      {
           class TransactionsExample
           {
              static void UsingExternalTransaction()
              {
                  using (var conn = new SqlConnection("..."))
                  {
                     conn.Open();

                     using (var sqlTxn = conn.BeginTransaction(System.Data.IsolationLevel.Snapshot))
                     {
                         try
                         {
                             var sqlCommand = new SqlCommand();
                             sqlCommand.Connection = conn;
                             sqlCommand.Transaction = sqlTxn;
                             sqlCommand.CommandText =
                                 @"UPDATE Blogs SET Rating = 5" +
                                  " WHERE Name LIKE '%Entity Framework%'";
                             sqlCommand.ExecuteNonQuery();

                             using (var context =  
                               new BloggingContext(conn, contextOwnsConnection: false))
                              {
                                  context.Database.UseTransaction(sqlTxn);

                                  var query =  context.Posts.Where(p => p.Blog.Rating >= 5);
                                  foreach (var post in query)
                                  {
                                      post.Title += "[Cool Blog]";
                                  }
                                 context.SaveChanges();
                              }

                              sqlTxn.Commit();
                          }
                          catch (Exception)
                          {
                              sqlTxn.Rollback();
                          }
                      }
                  }
              }
          }
      }
```

##### Clearing up the transaction

- You can pass null to Database.UseTransaction() to clear Entity Framework’s knowledge of the current transaction. Entity Framework
  will neither commit nor rollback the existing transaction when you do this, so use with care and only if you’re sure this is what 
  you want to do.
  
##### Errors in UseTransaction

- You will see an exception from Database.UseTransaction() if you pass a transaction when:
      - Entity Framework already has an existing transaction
      - Entity Framework is already operating within a TransactionScope
      - The connection object in the transaction passed is null. 
        That is, the transaction is not associated with a connection – usually this is a sign that that transaction has already     
        completed
      - The connection object in the transaction passed does not match the Entity Framework’s connection.
      
##### Using transactions with other features      

###### Connection Resiliency

- The new Connection Resiliency feature does not work with user-initiated transactions. 

###### Asynchronous Programming

- Be aware that, depending on what you do within the asynchronous methods, this may result in long-running transactions – which 
  can in turn cause deadlocks or blocking which is bad for the performance of the overall application.
  
###### TransactionScope Transactions

- Prior to EF6 the recommended way of providing larger scope transactions was to use a TransactionScope object:
- The SqlConnection and Entity Framework would both use the ambient TransactionScope transaction and hence be committed together.

```
      using System.Collections.Generic;
      using System.Data.Entity;
      using System.Data.SqlClient;
      using System.Linq;
      using System.Transactions;

      namespace TransactionsExamples
      {
          class TransactionsExample
          {
              static void UsingTransactionScope()
              {
                  using (var scope = new TransactionScope(TransactionScopeOption.Required))
                  {
                      using (var conn = new SqlConnection("..."))
                      {
                          conn.Open();

                          var sqlCommand = new SqlCommand();
                          sqlCommand.Connection = conn;
                          sqlCommand.CommandText =
                              @"UPDATE Blogs SET Rating = 5" +
                                  " WHERE Name LIKE '%Entity Framework%'";
                          sqlCommand.ExecuteNonQuery();

                          using (var context =
                              new BloggingContext(conn, contextOwnsConnection: false))
                          {
                              var query = context.Posts.Where(p => p.Blog.Rating > 5);
                              foreach (var post in query)
                              {
                                  post.Title += "[Cool Blog]";
                              }
                              context.SaveChanges();
                          }
                      }

                      scope.Complete();
                  }
              }
          }
      }
```

- Starting with .NET 4.5.1 TransactionScope has been updated to also work with asynchronous methods via the use 
  of the TransactionScopeAsyncFlowOption enumeration:
  
```
      using System.Collections.Generic;
      using System.Data.Entity;
      using System.Data.SqlClient;
      using System.Linq;
      using System.Transactions;

      namespace TransactionsExamples
      {
          class TransactionsExample
          {
              public static void AsyncTransactionScope()
              {
                  using (var scope = new TransactionScope(TransactionScopeAsyncFlowOption.Enabled))
                  {
                      using (var conn = new SqlConnection("..."))
                      {
                          await conn.OpenAsync();

                          var sqlCommand = new SqlCommand();
                          sqlCommand.Connection = conn;
                          sqlCommand.CommandText =
                              @"UPDATE Blogs SET Rating = 5" +
                                  " WHERE Name LIKE '%Entity Framework%'";
                          await sqlCommand.ExecuteNonQueryAsync();

                          using (var context = new BloggingContext(conn, contextOwnsConnection: false))
                          {
                              var query = context.Posts.Where(p => p.Blog.Rating > 5);
                              foreach (var post in query)
                              {
                                  post.Title += "[Cool Blog]";
                              }

                              await context.SaveChangesAsync();
                          }
                      }
                  }
              }
          }
      }
```

- There are still some limitations to the TransactionScope approach:
      - Requires .NET 4.5.1 or greater to work with asynchronous methods.
      - It cannot be used in cloud scenarios unless you are sure you have one and only one connection (cloud scenarios do not 
        support distributed transactions).
     - It cannot be combined with the Database.UseTransaction() approach of the previous sections.
     - It will throw exceptions if you issue any DDL and have not enabled distributed transactions through the MSDTC Service.

- Advantages of the TransactionScope approach:
      - It will automatically upgrade a local transaction to a distributed transaction if you make more than one connection to 
        a given database or combine a connection to one database with a connection to a different database within the same 
        transaction (note: you must have the MSDTC service configured to allow distributed transactions for this to work).
      - Ease of coding. If you prefer the transaction to be ambient and dealt with implicitly in the background rather than 
        explicitly under you control then the TransactionScope approach may suit you better.
        
        
- In summary, with the new Database.BeginTransaction() and Database.UseTransaction() APIs above, the TransactionScope approach 
  is no longer necessary for most users. If you do continue to use TransactionScope then be aware of the above limitations. 
  
  ### Data Validation
  
  ##### The model
  ```
  public class Blog
      {
          public int Id { get; set; }
          public string Title { get; set; }
          public string BloggerName { get; set; }
          public DateTime DateCreated { get; set; }
          public virtual ICollection<Post> Posts { get; set; }
          }
      }

      public class Post
      {
          public int Id { get; set; }
          public string Title { get; set; }
          public DateTime DateCreated { get; set; }
          public string Content { get; set; }
          public int BlogId { get; set; }
          public ICollection<Comment> Comments { get; set; }
      }
  ```
  
  ##### Data Annotations
  
  - Code First uses annotations from the System.ComponentModel.DataAnnotations assembly as one means of configuring code first classes.
  - Among these annotations are those which provide rules such as the Required, MaxLength and MinLength. 
  - A number of .NET client applications also recognize these annotations, for example, ASP.NET MVC. 
  - You can achieve both client side and server side validation with these annotations

```
    [Required]
        public string Title { get; set; }
```
- In the post back method of this Create view, Entity Framework is used to save the new blog to the database, but MVC’s client-side
  validation is triggered before the application reaches that code.
  
- Client side validation is not bullet-proof however. Users can impact features of their browser or worse yet, a hacker might use 
  some trickery to avoid the UI validations. But Entity Framework will also recognize the Required annotation and validate it.
  
- A simple way to test this is to disable MVC’s client-side validation feature. You can do this in the MVC application’s web.config
  file. The appSettings section has a key for ClientValidationEnabled. Setting this key to false will prevent the UI from performing
  validations.
  
  ```
      <appSettings>
            <add key="ClientValidationEnabled"value="false"/>
            ...
        </appSettings>
  ```
 - Even with the client-side validation disabled, you will get the same response in your application. The error message “The Title
   field is required” will be displayed as. Except now it will be a result of server-side validation. Entity Framework will perform 
   the validation on the Required annotation (before it even bothers to build and INSERT command to send to the database) and return
   the error to MVC which will display the message.
   
##### Fluent API

- You can use code first’s fluent API instead of annotations to get the same client side & server side validation.

- Fluent API configurations are applied as code first is building the model from the classes. You can inject the configurations by
  overriding the DbContext class’ OnModelCreating  method.

- Here is a configuration specifying that the BloggerName property can be no longer than 10 characters.
- Validation errors thrown based on the Fluent API configurations will not automatically reach the UI, but you can capture it in 
  code and then respond to it accordingly.
```
      public class BlogContext : DbContext
      {
          public DbSet<Blog> Blogs { get; set; }
          public DbSet<Post> Posts { get; set; }
          public DbSet<Comment> Comments { get; set; }

          protected override void OnModelCreating(DbModelBuilder modelBuilder)
          {
              modelBuilder.Entity<Blog>().Property(p => p.BloggerName).HasMaxLength(10);
          }
        }
```

- Here’s some exception handling error code in the application’s BlogController class that captures that validation error when 
  Entity Framework attempts to save a blog with a BloggerName that exceeds the 10 character maximum.
  
  ```
  [HttpPost]
    public ActionResult Edit(int id, Blog blog)
    {
        try
        {
            db.Entry(blog).State = EntityState.Modified;
            db.SaveChanges();
            return RedirectToAction("Index");
        }
        catch(DbEntityValidationException ex)
        {
            var error = ex.EntityValidationErrors.First().ValidationErrors.First();
            this.ModelState.AddModelError(error.PropertyName, error.ErrorMessage);
            return View();
        }
    }
  ```
- The validation doesn’t automatically get passed back into the view which is why the additional code that uses
  ModelState.AddModelError is being used. This ensures that the error details make it to the view which will then use the
  ValidationMessageFor Htmlhelper to display the error.
  
 ```
    @Html.ValidationMessageFor(model => model.BloggerName)
 ```
 
 ##### IValidatableObject
 
- IValidatableObject is an interface that lives in System.ComponentModel.DataAnnotations. While it is not part of the Entity 
  Framework API, you can still leverage it for server-side validation in your Entity Framework classes. 
- IValidatableObject provides a Validate method that Entity Framework will call during SaveChanges or you can call yourself any 
  time you want to validate the classes.
  
- Configurations such as Required and MaxLength perform validaton on a single field. In the Validate method you can have even more
  complex logic, for example, comparing two fields.
  
- In the following example, the Blog class has been extended to implement IValidatableObject and then provide a rule that the 
  Title and BloggerName cannot match.
  
```
      public class Blog : IValidatableObject
     {
         public int Id { get; set; }
         [Required]
         public string Title { get; set; }
         public string BloggerName { get; set; }
         public DateTime DateCreated { get; set; }
         public virtual ICollection<Post> Posts { get; set; }

         public IEnumerable<ValidationResult> Validate(ValidationContext validationContext)
         {
             if (Title == BloggerName)
             {
                 yield return new ValidationResult
                  ("Blog Title cannot match Blogger Name", new[] { "Title", “BloggerName” });
             }
         }
     } 
```
- The ValidationResult constructor takes a string that represents the error message and an array of strings that represent the 
  member names that are associated with the validation. Since this validation checks both the Title and the BloggerName, both 
  property names are returned.
  
- Unlike the validation provided by the Fluent API, this validation result will be recognized by the View and the exception handler
  that I used earlier to add the error into ModelState is unnecessary. Because I set both property names in the ValidationResult, 
  the MVC HtmlHelpers display the error message for both of those properties.
  
##### DbContext.ValidateEntity

- DbContext has an Overridable method called ValidateEntity. When you call SaveChanges, 
- Entity Framework will call this method for each entity in its cache whose state is not Unchanged. 
- You can put validation logic directly in here or even use this method to call, for example, the Blog.Validate method added 
  in the previous section.
  
- Here’s an example of a ValidateEntity override that validates new Posts to ensure that the post title hasn’t been used already. 
  It first checks to see if the entity is a post and that its state is Added. If that’s the case, then it looks in the database 
  to see if there is already a post with the same title. If there is an existing post already, then a new DbEntityValidationResult 
  is created.
  
- DbEntityValidationResult houses a DbEntityEntry and an ICollection of DbValidationErrors for a single entity. At the start of 
  this method, a DbEntityValidationResult is instantiated and then any errors that are discovered are added into its ValidationErrors
  collection. 

```
      protected override DbEntityValidationResult ValidateEntity (
        System.Data.Entity.Infrastructure.DbEntityEntry entityEntry,
        IDictionary<object, object> items)
    {
        var result = new DbEntityValidationResult(entityEntry, new List<DbValidationError>());
        if (entityEntry.Entity is Post && entityEntry.State == EntityState.Added)
        {
            Post post = entityEntry.Entity as Post;
            //check for uniqueness of post title
            if (Posts.Where(p => p.Title == post.Title).Count() > 0)
            {
                result.ValidationErrors.Add(
                        new System.Data.Entity.Validation.DbValidationError("Title",
                        "Post title must be unique."));
            }
        }

        if (result.ValidationErrors.Count > 0)
        {
            return result;
        }
        else
        {
         return base.ValidateEntity(entityEntry, items);
        }
    }
```
##### Explicitly triggering validation

- A call to SaveChanges triggers all of the validations covered in this article. But you don’t need to rely on SaveChanges. 
  u may prefer to validate elsewhere in your application.

- DbContext.GetValidationErrors will trigger all of the validations, those defined by annotations or the Fluent API, the validation
  eated in IValidatableObject (for example, Blog.Validate), and the validations performed in the DbContext.ValidateEntity method.
  
- The following code will call GetValidationErrors on the current instance of a DbContext. ValidationErrors are grouped by entity 
  pe into DbValidationResults. The code iterates first through the DbValidationResults returned by the method and then through 
  each ValidationError inside.
  
```
        foreach (var validationResults in db.GetValidationErrors())
        {
            foreach (var error in validationResults.ValidationErrors)
            {
                Debug.WriteLine(
                                  "Entity Property: {0}, Error {1}",
                                  error.PropertyName,
                                  error.ErrorMessage);
            }
        }
```
##### Other considerations when using validation

- Here are a few other points to consider when using Entity Framework validation:

        - zy loading is disabled during validation.
        
        - EF will validate data annotations on non-mapped properties (properties that are not mapped to a column in the database).
        
        - Validation is performed after changes are detected during SaveChanges. If you make changes during validation it is your
          responsibility to notify the change tracker.
          
        - DbUnexpectedValidationException is thrown if errors occur during validation.
        
        - Facets that Entity Framework includes in the model (maximum length, required, etc.) will cause validation, even if there 
          are not data annotations on your classes and/or you used the EF Designer to create your model.
          
        - Precedence rules: 
            - Fluent API calls override the corresponding data annotations
            
        - Execution order: 
             - Property validation occurs before type validation
              - Type validation only occurs if property validation succeeds
              
        - If a property is complex its validation will also include: 
            - Property-level validation on the complex type properties
            - Type level validation on the complex type, including IValidatableObject validation on the complex type

##### Summary

- The validation API in Entity Framework plays very nicely with client side validation in MVC but you don't have to rely on 
  client-side validation. Entity Framework will take care of the validation on the server side for DataAnnotations or 
  configurations you've applied with the code first Fluent API.
  
- You also saw a number of extensibility points for customizing the behavior whether you use the IValidatableObject interface or 
  tap into the DbContet.ValidateEntity method. And these last two means of validation are available through the DbContext, whether 
  you use the Code First, Model First or Database First workflow to describe your conceptual model.  


  


