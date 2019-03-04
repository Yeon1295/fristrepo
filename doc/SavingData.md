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


  


