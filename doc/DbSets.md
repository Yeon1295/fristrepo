# Defining DbSets

. When developing with the Code First workflow you define a derived DbContext that represents your session 
  with the database      and exposes a DbSet for each type in your model.
  
  ### DbContext with DbSet properties
  
  ```
  public class BloggingContext : DbContext
{
    public DbSet<Blog> Blogs { get; set; }
    public DbSet<Post> Posts { get; set; }
}
```

### DbContext with IDbSet properties

- There are situations, such as when creating mocks or fakes, where it is more useful to declare your set 
  properties using an interface. In such cases the IDbSet interface can be used in place of DbSet.
  
```
public class BloggingContext : DbContext
{
    public IDbSet<Blog> Blogs { get; set; }
    public IDbSet<Post> Posts { get; set; }
}
```

### DbContext with read-only set properties

- If you do not wish to expose public setters for your DbSet or IDbSet properties you can instead create 
  read-only properties and create the set instances yourself.
 
- Note that DbContext caches the instance of DbSet returned from the Set method so that each of these 
  properties will return the same instance every time it is called.
```
public class BloggingContext : DbContext
{
    public DbSet<Blog> Blogs
    {
        get { return Set<Blog>(); }
    }

    public DbSet<Post> Posts
    {
        get { return Set<Post>(); }
    }
}
```



