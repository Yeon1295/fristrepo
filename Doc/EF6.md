#  Entity Framework 6

- object-relational mapper (O/RM) for .NET

### Get Entity Framework
```
Install-Package EntityFramework
Install-Package EntityFramework -Version <number>
Install-Package EntityFramework -Pre 
```   
### Working with DbContext
##### Defining a DbContext derived class

```
public class ProductContext : DbContext
{
  public DbSet<Category> Categories { get; set; }
  public DbSet<Product> Products { get; set; }
}
```

##### Lifetime
```
public void UseProducts()
{
  using (var context = new ProductContext())
	{     
	  // Perform data access using the context
  }
}
```
- When working with Web applications, use a context instance per request.

- When working with Windows Presentation Foundation (WPF) or Windows Forms, use a context 
  instance per form.

- If the context instance is created by a dependency injection container, it is usually 
    the responsibility of the container to dispose the context.

- If the context is created in application code, remember to dispose of the context 
   when it is no longer required.

- The context is not thread-safe, therefore it should not be shared across multiple threads 
   doing work on it concurrently.

##### Connections
			   
- By default, the context manages connections to the database. The context opens and closes 
  connections as needed.

