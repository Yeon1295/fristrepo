# Custom Code First Conventions

- When using Code First your model is calculated from your classes using a set of conventions. 

- The default Code First Conventions determine things like which property becomes the primary key of an entity, 
  the name of the table an entity maps to, and what precision and scale a decimal column has by default.
  
- Sometimes these default conventions are not ideal for your model, and you have to work around them by 
  configuring many individual entities using Data Annotations or the Fluent API. 
  
- Custom Code First Conventions let you define your own conventions that provide configuration defaults for your model.

### Our Model

```
using System;
    using System.Collections.Generic;
    using System.Data.Entity;
    using System.Linq;

    public class ProductContext : DbContext
    {
        static ProductContext()
        {
            Database.SetInitializer(new DropCreateDatabaseIfModelChanges<ProductContext>());
        }

        public DbSet<Product> Products { get; set; }
    }

    public class Product
    {
        public int Key { get; set; }
        public string Name { get; set; }
        public decimal? Price { get; set; }
        public DateTime? ReleaseDate { get; set; }
        public ProductCategory Category { get; set; }
    }

    public class ProductCategory
    {
        public int Key { get; set; }
        public string Name { get; set; }
        public List<Product> Products { get; set; }
    }
    ```
    
    ### Introducing Custom Conventions
    
    - Conventions are enabled on the model builder, which can be accessed by overriding 
      *OnModelCreating* in the context.
    
    - any property in our model named Key will be configured as the primary key of whatever entity its part of.
    ```
    public class ProductContext : DbContext
    {
        static ProductContext()
        {
            Database.SetInitializer(new DropCreateDatabaseIfModelChanges<ProductContext>());
        }

        public DbSet<Product> Products { get; set; }

        protected override void OnModelCreating(DbModelBuilder modelBuilder)
        {
            modelBuilder.Properties()
                        .Where(p => p.Name == "Key")
                        .Configure(p => p.IsKey());
        }
    }
    ```
    
    - This will configure all properties called Key to be the primary key of their entity, 
      but only if they are an integer.
      
```
      modelBuilder.Properties<int>()
                .Where(p => p.Name == "Key")
                .Configure(p => p.IsKey());
```   

- An interesting feature of the IsKey method is that it is additive. Which means that 
  if you call IsKey on multiple properties and they will all become part of a composite key.
  
```
    modelBuilder.Properties<int>()
                .Where(x => x.Name == "Key")
                .Configure(x => x.IsKey().HasColumnOrder(1));

    modelBuilder.Properties()
                .Where(x => x.Name == "Name")
                .Configure(x => x.IsKey().HasColumnOrder(2));
```

- configure all DateTime properties in my model to map to the datetime2 type in SQL Server instead of datetime.
```
    modelBuilder.Properties<DateTime>()
                .Configure(c => c.HasColumnType("datetime2"));
``` 

### Convention Classes

- Another way of defining conventions is to use a Convention Class to encapsulate your convention. 
  When using a Convention Class then you create a type that inherits from the Convention class in the 
  *System.Data.Entity.ModelConfiguration.Conventions* namespace.
  
```
public class DateTime2Convention : Convention
    {
        public DateTime2Convention()
        {
            this.Properties<DateTime>()
                .Configure(c => c.HasColumnType("datetime2"));        
        }
    }
```  
- To tell EF to use this convention you add it to the Conventions collection in OnModelCreating

```
protected override void OnModelCreating(DbModelBuilder modelBuilder)
    {
        modelBuilder.Properties<int>()
                    .Where(p => p.Name.EndsWith("Key"))
                    .Configure(p => p.IsKey());

        modelBuilder.Conventions.Add(new DateTime2Convention());
    }
```  

### Custom Attributes

- Another great use of conventions is to enable new attributes to be used when configuring a model.

- create an attribute that we can use to mark String properties as non-Unicode.
```
    [AttributeUsage(AttributeTargets.Property, AllowMultiple = false)]
    public class NonUnicode : Attribute
    {
    }
```
- With this convention we can add the NonUnicode attribute to any of our string properties, which 
  means the column in the database will be stored as varchar instead of nvarchar.
  
```
        modelBuilder.Properties()
                .Where(x => x.GetCustomAttributes(false).OfType<NonUnicode>().Any())
                .Configure(c => c.IsUnicode(false));
```

- One thing to note about this convention is that if you put the NonUnicode attribute on anything 
  other than a string property then it will throw an exception. 
- It does this because you cannot configure IsUnicode on any type other than a string. 
- If this happens, then you can make your convention more specific, so that it filters out anything that isn’t a string.

- There is another API that can be much easier to use, especially when you want to use properties from the attribute class.

```
    [AttributeUsage(AttributeTargets.Property, AllowMultiple = false)]
    internal class IsUnicode : Attribute
    {
        public bool Unicode { get; set; }

        public IsUnicode(bool isUnicode)
        {
            Unicode = isUnicode;
        }
    }
```
- Once we have this, we can set a bool on our attribute to tell the convention whether or not a property should be Unicode. 
  We could do this in the convention we have already by accessing the ClrProperty of the configuration class.
  
```
    modelBuilder.Properties()
                .Where(x => x.GetCustomAttributes(false).OfType<IsUnicode>().Any())
                .Configure(c => c.IsUnicode(c.ClrPropertyInfo.GetCustomAttribute<IsUnicode>().Unicode));
```
- This is easy enough, but there is a more succinct way of achieving this by using the Having method of the conventions API.

```
    modelBuilder.Properties()
                .Having(x =>x.GetCustomAttributes(false).OfType<IsUnicode>().FirstOrDefault())
                .Configure((config, att) => config.IsUnicode(att.Unicode));
```

### Configuring Types

- Type level conventions can be really useful for is changing the table naming convention, either to map to an existing schema 
  that differs from the EF default or to create a new database with a different naming convention.
  
  - This method takes a type and returns a string that uses lower case with underscores instead of CamelCase. 
    In our model this means that the ProductCategory class will be mapped to a table called product_category 
    instead of ProductCategories.
    
```
    private string GetTableName(Type type)
    {
        var result = Regex.Replace(type.Name, ".[A-Z]", m => m.Value[0] + "_" + m.Value[1]);

        return result.ToLower();
    }
```
- This convention configures every type in our model to map to the table name that is returned 
    from our GetTableName method.
    
```
    modelBuilder.Types()
                .Configure(c => c.ToTable(GetTableName(c.ClrType)));
```
 - One thing to note about this is that when you call ToTable EF will take the string that you provide as 
   the exact table name, without any of the pluralization that it would normally do when determining table names. 
   This is why the table name from our convention is product_category instead of product_categories. 
   
- We can resolve that in our convention by making a call to the pluralization service ourselves.

```
    private string GetTableName(Type type)
    {
        var pluralizationService = DbConfiguration.DependencyResolver.GetService<IPluralizationService>();

        var result = pluralizationService.Pluralize(type.Name);

        result = Regex.Replace(result, ".[A-Z]", m => m.Value[0] + "_" + m.Value[1]);

        return result.ToLower();
    }
```

### ToTable and Inheritance

- By default both employee and manager are mapped to the same table (Employees) in the database. 
  The table will contain both employees and managers with a discriminator column that will tell you 
  what type of instance is stored in each row.
- This is TPH mapping as there is a single table for the hierarchy.

```
    public class Employee
    {
        public int Id { get; set; }
        public string Name { get; set; }
    }

    public class Manager : Employee
    {
        public string SectionManaged { get; set; }
    }
```

- However, if you call ToTable on both classe then each type will instead be mapped to its own table, 
  also known as TPT since each type has its own table. 

```
    modelBuilder.Types()
                .Configure(c=>c.ToTable(c.ClrType.Name));
```

- You can avoid this, and maintain the default TPH mapping, in a couple ways:
  1. Call ToTable with the same table name for each type in the hierarchy.
  2. Call ToTable only on the base class of the hierarchy, in our example that would be employee.
  
### Execution Order

- Conventions operate in a last wins manner, the same as the Fluent API. 
  What this means is that if you write two conventions that configure the same option of the same property, 
  then the last one to execute wins.
  
- the code below the max length of all strings is set to 500 but we then configure all properties called Name 
  in the model to have a max length of 250.

```
    modelBuilder.Properties<string>()
                .Configure(c => c.HasMaxLength(500));

    modelBuilder.Properties<string>()
                .Where(x => x.Name == "Name")
                .Configure(c => c.HasMaxLength(250));
```

### Built-in Conventions

- Because custom conventions could be affected by the default Code First conventions, it can be useful to add 
  conventions to run before or after another convention.
- To do this you can use the AddBefore and AddAfter methods of the Conventions collection on your derived DbContext.

- This is going to be of the most use when adding conventions that need to run before or after the built in conventions

- The following code would add the convention class we created earlier so that it will run before the built in 
  key discovery convention.

```
    modelBuilder.Conventions.AddBefore<IdKeyDiscoveryConvention>(new DateTime2Convention());
```
- You can also remove conventions that you do not want applied to your model.

```
  protected override void OnModelCreating(DbModelBuilder modelBuilder)
    {
        modelBuilder.Conventions.Remove<PluralizingTableNameConvention>();
    }
```

  
  
 
  
 
  
  







