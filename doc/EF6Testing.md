# Testing with EF 6 

- When writing tests for your application it is often desirable to avoid hitting the database. 
- Entity Framework allows you to achieve this by creating a context – with behavior defined by your tests – that makes use of 
  in-memory data.
  
### Options for creating test doubles

- There are two different approaches that can be used to create an in-memory version of your context.

    - Create your own test doubles –   
        This approach involves writing your own in-memory implementation of your context and DbSets. 
        This gives you a lot of control over how the classes behave but can involve writing and owning a reasonable amount of code.
          
    - Use a mocking framework to create test doubles –   
          Using a mocking framework (such as Moq) you can have the in-memory implementations of you context and sets created 
          dynamically at runtime for you.
          
 - The easiest way to get Moq is to install the Moq package from NuGet. 
      Install-Package Moq -Version 4.10.1 
      
### Testing with pre-EF6 versions

##### Testing With a Fake DbContext

###### The Problem

- Say we have a very simple model for Employees and Departments and we are using DbContext to persist and query our data:
```
      public class EmployeeContext : DbContext
      {
          public DbSet<Department> Departments { get; set; }
          public DbSet<Employee> Employees { get; set; }
      }

      public class Department
      {
          public int DepartmentId { get; set; }
          public string Name { get; set; }

          public ICollection<Employee> Employees { get; set; }
      }

      public class Employee
      {
          public int EmployeeId { get; set; }
          public int FirstName { get; set; }
          public int LastName { get; set; }
          public int Position { get; set; }

          public Department Department { get; set; }
      }
```
- In an MVC application we could add a controller that can display all departments in alphabetical order:
```
      public class DepartmentController : Controller
      {
          private EmployeeContext db = new EmployeeContext();

          public ViewResult Index()
          {
              return View(db.Departments.OrderBy(d => d.Name).ToList());
          }

          protected override void Dispose(bool disposing)
          {
              db.Dispose();
              base.Dispose(disposing);
          }
      }
```
- The issue is that our controller now has a hard dependency on EF because it needs an EmployeeContext which derives from DbContext 
  and exposes DbSets. 
- We have no way to replace this with any other implementation and we are forced to run any unit tests against a real database, 
  which means they really aren’t unit tests at all.

##### Adding an Interface

- DbSet<T> happens to implement IDbSet<T> so we can pretty easily create an interface that our derived context implements:
```
      public interface IEmployeeContext
      {
          IDbSet<Department> Departments { get; }
          IDbSet<Employee> Employees { get; }
          int SaveChanges();
      }

      public class EmployeeContext : DbContext, IEmployeeContext
      {
          public IDbSet<Department> Departments { get; set; }
          public IDbSet<Employee> Employees { get; set; }
      }
```
- Now we can update our controller to be based on this interface rather than the EF specific implementation.
```
      public class DepartmentController : Controller
      {
          private IEmployeeContext db;

          public DepartmentController()
          {
              this.db = new EmployeeContext();
          }

          public DepartmentController(IEmployeeContext context)
          {
              this.db = context;
          }

          public ViewResult Index()
          {
              return View(db.Departments.OrderBy(d => d.Name).ToList());
          }

          protected override void Dispose(bool disposing)
          {
              if (db is IDisposable)
              {
                  ((IDisposable)db).Dispose();
              }
              base.Dispose(disposing);
          }
      }

```
##### Building Fakes

- The first thing we need is a fake implementation of IDbSet<TEntity>, this is pretty easy to implement.
```
      public class FakeDbSet<T> : IDbSet<T>
          where T : class
      {
          ObservableCollection<T> _data;
          IQueryable _query;

          public FakeDbSet()
          {
              _data = new ObservableCollection<T>();
              _query = _data.AsQueryable();
          }

          public virtual T Find(params object[] keyValues)
          {
              throw new NotImplementedException("Derive from FakeDbSet<T> and override Find");
          }

          public T Add(T item)
          {
              _data.Add(item);
              return item;
          }

          public T Remove(T item)
          {
              _data.Remove(item);
              return item;
          }

          public T Attach(T item)
          {
              _data.Add(item);
              return item;
          }

          public T Detach(T item)
          {
              _data.Remove(item);
              return item;
          }

          public T Create()
          {
              return Activator.CreateInstance<T>();
          }

          public TDerivedEntity Create<TDerivedEntity>() where TDerivedEntity : class, T
          {
              return Activator.CreateInstance<TDerivedEntity>();
          }

          public ObservableCollection<T> Local
          {
              get { return _data; }
          }

          Type IQueryable.ElementType
          {
              get { return _query.ElementType; }
          }

          System.Linq.Expressions.Expression IQueryable.Expression
          {
              get { return _query.Expression; }
          }

          IQueryProvider IQueryable.Provider
          {
              get { return _query.Provider; }
          }

          System.Collections.IEnumerator System.Collections.IEnumerable.GetEnumerator()
          {
              return _data.GetEnumerator();
          }

          IEnumerator<T> IEnumerable<T>.GetEnumerator()
          {
              return _data.GetEnumerator();
          }
      }
```
- There isn’t really a good way to generically implement Find, so I’ve left it as a virtual method that throws if called. 
  If our application makes use of the Find method we can create an implementation specific to each type.

```
    public class FakeDepartmentSet : FakeDbSet<Department>
    {
        public override Department Find(params object[] keyValues)
        {
            return this.SingleOrDefault(d => d.DepartmentId == (int)keyValues.Single());
        }
    }

    public class FakeEmployeeSet : FakeDbSet<Employee>
    {
        public override Employee Find(params object[] keyValues)
        {
            return this.SingleOrDefault(e => e.EmployeeId == (int)keyValues.Single());
        }
    }
```
- Now we can create a fake implementation of our context:
```
      public class FakeEmployeeContext : IEmployeeContext
      {
          public FakeEmployeeContext()
          {
              this.Departments = new FakeDepartmentSet();
              this.Employees = new FakeEmployeeSet();
          }

          public IDbSet<Department> Departments { get; private set; }

          public IDbSet<Employee> Employees { get; private set; }

          public int SaveChanges()
          {
              return 0;
          }
      }
```
###### Testing Against Fakes

- Now that we have our fakes defined we can use them to write a unit test for our controller, that doesn’t use EF:
```
      [TestMethod]
      public void IndexOrdersByName()
      {
          var context = new FakeEmployeeContext
          {
              Departments =
              {
                  new Department { Name = "BBB"},
                  new Department { Name = "AAA"},
                  new Department { Name = "ZZZ"},
              }
          };

          var controller = new DepartmentController(context);
          var result = controller.Index();

          Assert.IsInstanceOfType(result.ViewData.Model, typeof(IEnumerable<Department>));
          var departments = (IEnumerable<Department>)result.ViewData.Model;
          Assert.AreEqual("AAA", departments.ElementAt(0).Name);
          Assert.AreEqual("BBB", departments.ElementAt(1).Name);
          Assert.AreEqual("ZZZ", departments.ElementAt(2).Name);
      }
```
- There are a number of reasons to use in-memory fakes for unit testing but some key benefits are stable and robust tests that 
  execute quickly and exercise a single component, making failures easy to isolate.
  
### Testing with a mocking framework

##### The EF model
```
      using System.Collections.Generic;
      using System.Data.Entity;

      namespace TestingDemo
      {
          public class BloggingContext : DbContext
          {
              public virtual DbSet<Blog> Blogs { get; set; }
              public virtual DbSet<Post> Posts { get; set; }
          }

          public class Blog
          {
              public int BlogId { get; set; }
              public string Name { get; set; }
              public string Url { get; set; }

              public virtual List<Post> Posts { get; set; }
          }

          public class Post
          {
              public int PostId { get; set; }
              public string Title { get; set; }
              public string Content { get; set; }

              public int BlogId { get; set; }
              public virtual Blog Blog { get; set; }
          }
      }
```
- Note that the DbSet properties on the context are marked as virtual.
- This will allow the mocking framework to derive from our context and overriding these properties with a mocked implementation.

##### Service to be tested

- To demonstrate testing with in-memory test doubles we are going to be writing a couple of tests for a BlogService. The service is 
  capable of creating new blogs (AddBlog) and returning all Blogs ordered by name (GetAllBlogs). 
- In addition to GetAllBlogs, we’ve also provided a method that will asynchronously get all blogs ordered by name (GetAllBlogsAsync).

```
      using System.Collections.Generic;
      using System.Data.Entity;
      using System.Linq;
      using System.Threading.Tasks;

      namespace TestingDemo
      {
          public class BlogService
          {
              private BloggingContext _context;

              public BlogService(BloggingContext context)
              {
                  _context = context;
              }

              public Blog AddBlog(string name, string url)
              {
                  var blog = _context.Blogs.Add(new Blog { Name = name, Url = url });
                  _context.SaveChanges();

                  return blog;
              }

              public List<Blog> GetAllBlogs()
              {
                  var query = from b in _context.Blogs
                              orderby b.Name
                              select b;

                  return query.ToList();
              }

              public async Task<List<Blog>> GetAllBlogsAsync()
              {
                  var query = from b in _context.Blogs
                              orderby b.Name
                              select b;

                  return await query.ToListAsync();
              }
          }
      }
```
##### Testing non-query scenarios

- The following test uses Moq to create a context. It then creates a DbSet<Blog> and wires it up to be returned from the context’s 
  Blogs property. 
- Next, the context is used to create a new BlogService which is then used to create a new blog – using the AddBlog method. 
- Finally, the test verifies that the service added a new Blog and called SaveChanges on the context.

```
      using Microsoft.VisualStudio.TestTools.UnitTesting;
      using Moq;
      using System.Data.Entity;

      namespace TestingDemo
      {
          [TestClass]
          public class NonQueryTests
          {
              [TestMethod]
              public void CreateBlog_saves_a_blog_via_context()
              {
                  var mockSet = new Mock<DbSet<Blog>>();

                  var mockContext = new Mock<BloggingContext>();
                  mockContext.Setup(m => m.Blogs).Returns(mockSet.Object);

                  var service = new BlogService(mockContext.Object);
                  service.AddBlog("ADO.NET Blog", "http://blogs.msdn.com/adonet");

                  mockSet.Verify(m => m.Add(It.IsAny<Blog>()), Times.Once());
                  mockContext.Verify(m => m.SaveChanges(), Times.Once());
              }
          }
      }
```

##### Testing query scenarios

- In order to be able to execute queries against our DbSet test double we need to setup an implementation of IQueryable. 
- The first step is to create some in-memory data – we’re using a List<Blog>. 
- Next, we create a context and DBSet<Blog> then wire up the IQueryable implementation for the DbSet – they’re just delegating 
  to the LINQ to Objects provider that works with List<T>.
  
- We can then create a BlogService based on our test doubles and ensure that the data we get back from GetAllBlogs is ordered by name.

```
      using Microsoft.VisualStudio.TestTools.UnitTesting;
      using Moq;
      using System.Collections.Generic;
      using System.Data.Entity;
      using System.Linq;

      namespace TestingDemo
      {
          [TestClass]
          public class QueryTests
          {
              [TestMethod]
              public void GetAllBlogs_orders_by_name()
              {
                  var data = new List<Blog>
                  {
                      new Blog { Name = "BBB" },
                      new Blog { Name = "ZZZ" },
                      new Blog { Name = "AAA" },
                  }.AsQueryable();

                  var mockSet = new Mock<DbSet<Blog>>();
                  mockSet.As<IQueryable<Blog>>().Setup(m => m.Provider).Returns(data.Provider);
                  mockSet.As<IQueryable<Blog>>().Setup(m => m.Expression).Returns(data.Expression);
                  mockSet.As<IQueryable<Blog>>().Setup(m => m.ElementType).Returns(data.ElementType);
                  mockSet.As<IQueryable<Blog>>().Setup(m => m.GetEnumerator()).Returns(data.GetEnumerator());

                  var mockContext = new Mock<BloggingContext>();
                  mockContext.Setup(c => c.Blogs).Returns(mockSet.Object);

                  var service = new BlogService(mockContext.Object);
                  var blogs = service.GetAllBlogs();

                  Assert.AreEqual(3, blogs.Count);
                  Assert.AreEqual("AAA", blogs[0].Name);
                  Assert.AreEqual("BBB", blogs[1].Name);
                  Assert.AreEqual("ZZZ", blogs[2].Name);
              }
          }
      }
```
##### Testing with async queries

- Entity Framework 6 introduced a set of extension methods that can be used to asynchronously execute a query. 
- Examples of these methods include ToListAsync, FirstAsync, ForEachAsync, etc.

- Because Entity Framework queries make use of LINQ, the extension methods are defined on IQueryable and IEnumerable. 
- However, because they are only designed to be used with Entity Framework you may receive the following error if you try to use 
  them on a LINQ query that isn’t an Entity Framework query:
  
        The source IQueryable doesn't implement IDbAsyncEnumerable{0}. Only sources that implement IDbAsyncEnumerable 
        can be used for Entity Framework asynchronous operations.
        
- Whilst the async methods are only supported when running against an EF query, you may want to use them in your unit test when 
  running against an in-memory test double of a DbSet.
  
- In order to use the async methods we need to create an in-memory DbAsyncQueryProvider to process the async query. Whilst it 
  would be possible to setup a query provider using Moq, it is much easier to create a test double implementation in code. 
  
```
      using System.Collections.Generic;
      using System.Data.Entity.Infrastructure;
      using System.Linq;
      using System.Linq.Expressions;
      using System.Threading;
      using System.Threading.Tasks;

      namespace TestingDemo
      {
          internal class TestDbAsyncQueryProvider<TEntity> : IDbAsyncQueryProvider
          {
              private readonly IQueryProvider _inner;

              internal TestDbAsyncQueryProvider(IQueryProvider inner)
              {
                  _inner = inner;
              }

              public IQueryable CreateQuery(Expression expression)
              {
                  return new TestDbAsyncEnumerable<TEntity>(expression);
              }

              public IQueryable<TElement> CreateQuery<TElement>(Expression expression)
              {
                  return new TestDbAsyncEnumerable<TElement>(expression);
              }

              public object Execute(Expression expression)
              {
                  return _inner.Execute(expression);
              }

              public TResult Execute<TResult>(Expression expression)
              {
                  return _inner.Execute<TResult>(expression);
              }

              public Task<object> ExecuteAsync(Expression expression, CancellationToken cancellationToken)
              {
                  return Task.FromResult(Execute(expression));
              }

              public Task<TResult> ExecuteAsync<TResult>(Expression expression, CancellationToken cancellationToken)
              {
                  return Task.FromResult(Execute<TResult>(expression));
              }
          }

          internal class TestDbAsyncEnumerable<T> : EnumerableQuery<T>, IDbAsyncEnumerable<T>, IQueryable<T>
          {
              public TestDbAsyncEnumerable(IEnumerable<T> enumerable)
                  : base(enumerable)
              { }

              public TestDbAsyncEnumerable(Expression expression)
                  : base(expression)
              { }

              public IDbAsyncEnumerator<T> GetAsyncEnumerator()
              {
                  return new TestDbAsyncEnumerator<T>(this.AsEnumerable().GetEnumerator());
              }

              IDbAsyncEnumerator IDbAsyncEnumerable.GetAsyncEnumerator()
              {
                  return GetAsyncEnumerator();
              }

              IQueryProvider IQueryable.Provider
              {
                  get { return new TestDbAsyncQueryProvider<T>(this); }
              }
          }

          internal class TestDbAsyncEnumerator<T> : IDbAsyncEnumerator<T>
          {
              private readonly IEnumerator<T> _inner;

              public TestDbAsyncEnumerator(IEnumerator<T> inner)
              {
                  _inner = inner;
              }

              public void Dispose()
              {
                  _inner.Dispose();
              }

              public Task<bool> MoveNextAsync(CancellationToken cancellationToken)
              {
                  return Task.FromResult(_inner.MoveNext());
              }

              public T Current
              {
                  get { return _inner.Current; }
              }

              object IDbAsyncEnumerator.Current
              {
                  get { return Current; }
              }
          }
      }
```

- Now that we have an async query provider we can write a unit test for our new GetAllBlogsAsync method.

```
      using Microsoft.VisualStudio.TestTools.UnitTesting;
      using Moq;
      using System.Collections.Generic;
      using System.Data.Entity;
      using System.Data.Entity.Infrastructure;
      using System.Linq;
      using System.Threading.Tasks;

      namespace TestingDemo
      {
          [TestClass]
          public class AsyncQueryTests
          {
              [TestMethod]
              public async Task GetAllBlogsAsync_orders_by_name()
              {

                  var data = new List<Blog>
                  {
                      new Blog { Name = "BBB" },
                      new Blog { Name = "ZZZ" },
                      new Blog { Name = "AAA" },
                  }.AsQueryable();

                  var mockSet = new Mock<DbSet<Blog>>();
                  mockSet.As<IDbAsyncEnumerable<Blog>>()
                      .Setup(m => m.GetAsyncEnumerator())
                      .Returns(new TestDbAsyncEnumerator<Blog>(data.GetEnumerator()));

                  mockSet.As<IQueryable<Blog>>()
                      .Setup(m => m.Provider)
                      .Returns(new TestDbAsyncQueryProvider<Blog>(data.Provider));

                  mockSet.As<IQueryable<Blog>>().Setup(m => m.Expression).Returns(data.Expression);
                  mockSet.As<IQueryable<Blog>>().Setup(m => m.ElementType).Returns(data.ElementType);
                  mockSet.As<IQueryable<Blog>>().Setup(m => m.GetEnumerator()).Returns(data.GetEnumerator());

                  var mockContext = new Mock<BloggingContext>();
                  mockContext.Setup(c => c.Blogs).Returns(mockSet.Object);

                  var service = new BlogService(mockContext.Object);
                  var blogs = await service.GetAllBlogsAsync();

                  Assert.AreEqual(3, blogs.Count);
                  Assert.AreEqual("AAA", blogs[0].Name);
                  Assert.AreEqual("BBB", blogs[1].Name);
                  Assert.AreEqual("ZZZ", blogs[2].Name);
              }
          }
      }
```
### Testing with your own test doubles

##### Creating a context interface

- In order to be able to replace our EF context with an in-memory version for testing, we'll define an interface that our EF 
  context (and it's in-memory double) will imeplement.

- The service we are going to test will query and modify data using the DbSet properties of our context and also call SaveChanges 
  to push changes to the database. So we're including these members on the interface.
  
```
    using System.Data.Entity;

    namespace TestingDemo
    {
        public interface IBloggingContext
        {
            DbSet<Blog> Blogs { get; }
            DbSet<Post> Posts { get; }
            int SaveChanges();
        }
    }
```
##### The EF model

- The service we're going to test makes use of an EF model made up of the BloggingContext and the Blog and Post classes.

```
      using System.Collections.Generic;
      using System.Data.Entity;

      namespace TestingDemo
      {
          public class BloggingContext : DbContext, IBloggingContext
          {
              public DbSet<Blog> Blogs { get; set; }
              public DbSet<Post> Posts { get; set; }
          }

          public class Blog
          {
              public int BlogId { get; set; }
              public string Name { get; set; }
              public string Url { get; set; }

              public virtual List<Post> Posts { get; set; }
          }

          public class Post
          {
              public int PostId { get; set; }
              public string Title { get; set; }
              public string Content { get; set; }

              public int BlogId { get; set; }
              public virtual Blog Blog { get; set; }
          }
      }
```

##### Service to be tested

- To demonstrate testing with in-memory test doubles we are going to be writing a couple of tests for a BlogService. 
  The service is capable of creating new blogs (AddBlog) and returning all Blogs ordered by name (GetAllBlogs). 
  In addition to GetAllBlogs, we’ve also provided a method that will asynchronously get all blogs ordered by name (GetAllBlogsAsync).

```
      using System.Collections.Generic;
      using System.Data.Entity;
      using System.Linq;
      using System.Threading.Tasks;

      namespace TestingDemo
      {
          public class BlogService
          {
              private IBloggingContext _context;

              public BlogService(IBloggingContext context)
              {
                  _context = context;
              }

              public Blog AddBlog(string name, string url)
              {
                  var blog = new Blog { Name = name, Url = url };
                  _context.Blogs.Add(blog);
                  _context.SaveChanges();

                  return blog;
              }

              public List<Blog> GetAllBlogs()
              {
                  var query = from b in _context.Blogs
                              orderby b.Name
                              select b;

                  return query.ToList();
              }

              public async Task<List<Blog>> GetAllBlogsAsync()
              {
                  var query = from b in _context.Blogs
                              orderby b.Name
                              select b;

                  return await query.ToListAsync();
              }
          }
      }
```

- Now that we have the real EF model and the service that can use it, it's time to create the in-memory test double that we can 
  use for testing. - We've created a TestContext test double for our context. 
- In test doubles we get to choose the behavior we want in order to support the tests we are going to run. 

- In this example we're just capturing the number of times SaveChanges is called, but you can include whatever logic is needed 
  to verify the scenario you are testing.
  
- We've also created a TestDbSet that provides an in-memory implementation of DbSet. 
- We've provided a complete implemention for all the methods on DbSet (except for Find), but you only need to implement the members 
  that your test scenario will use.
  
- TestDbSet makes use of some other infrastructure classes that we've included to ensure that async queries can be processed.

```
using System;
using System.Collections.Generic;
using System.Collections.ObjectModel;
using System.Data.Entity;
using System.Data.Entity.Infrastructure;
using System.Linq;
using System.Linq.Expressions;
using System.Threading;
using System.Threading.Tasks;

namespace TestingDemo
{
    public class TestContext : IBloggingContext
    {
        public TestContext()
        {
            this.Blogs = new TestDbSet<Blog>();
            this.Posts = new TestDbSet<Post>();
        }

        public DbSet<Blog> Blogs { get; set; }
        public DbSet<Post> Posts { get; set; }
        public int SaveChangesCount { get; private set; }
        public int SaveChanges()
        {
            this.SaveChangesCount++;
            return 1;
        }
    }

    public class TestDbSet<TEntity> : DbSet<TEntity>, IQueryable, IEnumerable<TEntity>, IDbAsyncEnumerable<TEntity>
        where TEntity : class
    {
        ObservableCollection<TEntity> _data;
        IQueryable _query;

        public TestDbSet()
        {
            _data = new ObservableCollection<TEntity>();
            _query = _data.AsQueryable();
        }

        public override TEntity Add(TEntity item)
        {
            _data.Add(item);
            return item;
        }

        public override TEntity Remove(TEntity item)
        {
            _data.Remove(item);
            return item;
        }

        public override TEntity Attach(TEntity item)
        {
            _data.Add(item);
            return item;
        }

        public override TEntity Create()
        {
            return Activator.CreateInstance<TEntity>();
        }

        public override TDerivedEntity Create<TDerivedEntity>()
        {
            return Activator.CreateInstance<TDerivedEntity>();
        }

        public override ObservableCollection<TEntity> Local
        {
            get { return _data; }
        }

        Type IQueryable.ElementType
        {
            get { return _query.ElementType; }
        }

        Expression IQueryable.Expression
        {
            get { return _query.Expression; }
        }

        IQueryProvider IQueryable.Provider
        {
            get { return new TestDbAsyncQueryProvider<TEntity>(_query.Provider); }
        }

        System.Collections.IEnumerator System.Collections.IEnumerable.GetEnumerator()
        {
            return _data.GetEnumerator();
        }

        IEnumerator<TEntity> IEnumerable<TEntity>.GetEnumerator()
        {
            return _data.GetEnumerator();
        }

        IDbAsyncEnumerator<TEntity> IDbAsyncEnumerable<TEntity>.GetAsyncEnumerator()
        {
            return new TestDbAsyncEnumerator<TEntity>(_data.GetEnumerator());
        }
    }

    internal class TestDbAsyncQueryProvider<TEntity> : IDbAsyncQueryProvider
    {
        private readonly IQueryProvider _inner;

        internal TestDbAsyncQueryProvider(IQueryProvider inner)
        {
            _inner = inner;
        }

        public IQueryable CreateQuery(Expression expression)
        {
            return new TestDbAsyncEnumerable<TEntity>(expression);
        }

        public IQueryable<TElement> CreateQuery<TElement>(Expression expression)
        {
            return new TestDbAsyncEnumerable<TElement>(expression);
        }

        public object Execute(Expression expression)
        {
            return _inner.Execute(expression);
        }

        public TResult Execute<TResult>(Expression expression)
        {
            return _inner.Execute<TResult>(expression);
        }

        public Task<object> ExecuteAsync(Expression expression, CancellationToken cancellationToken)
        {
            return Task.FromResult(Execute(expression));
        }

        public Task<TResult> ExecuteAsync<TResult>(Expression expression, CancellationToken cancellationToken)
        {
            return Task.FromResult(Execute<TResult>(expression));
        }
    }

    internal class TestDbAsyncEnumerable<T> : EnumerableQuery<T>, IDbAsyncEnumerable<T>, IQueryable<T>
    {
        public TestDbAsyncEnumerable(IEnumerable<T> enumerable)
            : base(enumerable)
        { }

        public TestDbAsyncEnumerable(Expression expression)
            : base(expression)
        { }

        public IDbAsyncEnumerator<T> GetAsyncEnumerator()
        {
            return new TestDbAsyncEnumerator<T>(this.AsEnumerable().GetEnumerator());
        }

        IDbAsyncEnumerator IDbAsyncEnumerable.GetAsyncEnumerator()
        {
            return GetAsyncEnumerator();
        }

        IQueryProvider IQueryable.Provider
        {
            get { return new TestDbAsyncQueryProvider<T>(this); }
        }
    }

    internal class TestDbAsyncEnumerator<T> : IDbAsyncEnumerator<T>
    {
        private readonly IEnumerator<T> _inner;

        public TestDbAsyncEnumerator(IEnumerator<T> inner)
        {
            _inner = inner;
        }

        public void Dispose()
        {
            _inner.Dispose();
        }

        public Task<bool> MoveNextAsync(CancellationToken cancellationToken)
        {
            return Task.FromResult(_inner.MoveNext());
        }

        public T Current
        {
            get { return _inner.Current; }
        }

        object IDbAsyncEnumerator.Current
        {
            get { return Current; }
        }
    }
}
```
##### Implementing Find

- The Find method is difficult to implement in a generic fashion. If you need to test code that makes use of the Find method 
  it is easiest to create a test DbSet for each of the entity types that need to support find. You can then write logic to find 
  that particular type of entity, as shown below.
  
  ```
        using System.Linq;

      namespace TestingDemo
      {
          class TestBlogDbSet : TestDbSet<Blog>
          {
              public override Blog Find(params object[] keyValues)
              {
                  var id = (int)keyValues.Single();
                  return this.SingleOrDefault(b => b.BlogId == id);
              }
          }
      }
  ```
##### Writing some tests
  
- That’s all we need to do to start testing. The following test creates a TestContext and then a service based on this context. 
  - The service is then used to create a new blog – using the AddBlog method. Finally, the test verifies that the service added 
    a new Blog to the context's Blogs property and called SaveChanges on the context.
    
```
      using Microsoft.VisualStudio.TestTools.UnitTesting;
      using System.Linq;

      namespace TestingDemo
      {
          [TestClass]
          public class NonQueryTests
          {
              [TestMethod]
              public void CreateBlog_saves_a_blog_via_context()
              {
                  var context = new TestContext();

                  var service = new BlogService(context);
                  service.AddBlog("ADO.NET Blog", "http://blogs.msdn.com/adonet");

                  Assert.AreEqual(1, context.Blogs.Count());
                  Assert.AreEqual("ADO.NET Blog", context.Blogs.Single().Name);
                  Assert.AreEqual("http://blogs.msdn.com/adonet", context.Blogs.Single().Url);
                  Assert.AreEqual(1, context.SaveChangesCount);
              }
          }
      }
```
- Here is another example of a test - this time one that performs a query. The test starts by creating a test context with some 
  data in its Blog property - 
- note that the data is not in alphabetical order. We can then create a BlogService based on our test context and ensure that the 
  data we get back from GetAllBlogs is ordered by name.

```
      using Microsoft.VisualStudio.TestTools.UnitTesting;

      namespace TestingDemo
      {
          [TestClass]
          public class QueryTests
          {
              [TestMethod]
              public void GetAllBlogs_orders_by_name()
              {
                  var context = new TestContext();
                  context.Blogs.Add(new Blog { Name = "BBB" });
                  context.Blogs.Add(new Blog { Name = "ZZZ" });
                  context.Blogs.Add(new Blog { Name = "AAA" });

                  var service = new BlogService(context);
                  var blogs = service.GetAllBlogs();

                  Assert.AreEqual(3, blogs.Count);
                  Assert.AreEqual("AAA", blogs[0].Name);
                  Assert.AreEqual("BBB", blogs[1].Name);
                  Assert.AreEqual("ZZZ", blogs[2].Name);
              }
          }
      }
```

- Finally, we'll write one more test that uses our async method to ensure that the async infrastructure we included in TestDbSet 
  is working.
  
  ```
        using Microsoft.VisualStudio.TestTools.UnitTesting;
      using System.Collections.Generic;
      using System.Linq;
      using System.Threading.Tasks;

      namespace TestingDemo
      {
          [TestClass]
          public class AsyncQueryTests
          {
              [TestMethod]
              public async Task GetAllBlogsAsync_orders_by_name()
              {
                  var context = new TestContext();
                  context.Blogs.Add(new Blog { Name = "BBB" });
                  context.Blogs.Add(new Blog { Name = "ZZZ" });
                  context.Blogs.Add(new Blog { Name = "AAA" });

                  var service = new BlogService(context);
                  var blogs = await service.GetAllBlogsAsync();

                  Assert.AreEqual(3, blogs.Count);
                  Assert.AreEqual("AAA", blogs[0].Name);
                  Assert.AreEqual("BBB", blogs[1].Name);
                  Assert.AreEqual("ZZZ", blogs[2].Name);
              }
          }
      }
  ```

  



  
  
  
  
  






  
  
  






 
 
  
