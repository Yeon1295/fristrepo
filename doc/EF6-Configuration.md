# EF6 configuration

- Configuration for an Entity Framework application can be specified in a config file (app.config/web.config) or through code.

- The config file takes precedence over code-based configuration. In other words, if a configuration option is set in both code 
  and in the config file, then the setting in the config file is used.

### Code-based configuration

##### Using DbConfiguration

- Code-based configuration in EF6 and above is achieved by creating a subclass of System.Data.Entity.Config.DbConfiguration.
  - Create only one DbConfiguration class for your application. This class specifies app-domain wide settings.
  - Place your DbConfiguration class in the same assembly as your DbContext class. 
  - Give your DbConfiguration class a public parameterless constructor.
  - Set configuration options by calling protected DbConfiguration methods from within this constructor.
  
- This class sets up EF to use the SQL Azure execution strategy - to automatically retry failed database operations - and 
  to use Local DB for databases that are created by convention from Code First.
```
using System.Data.Entity;
using System.Data.Entity.Infrastructure;
using System.Data.Entity.SqlServer;

namespace MyNamespace
{
    public class MyConfiguration : DbConfiguration
    {
        public MyConfiguration()
        {
            SetExecutionStrategy("System.Data.SqlClient", () => new SqlAzureExecutionStrategy());
            SetDefaultConnectionFactory(new LocalDbConnectionFactory("mssqllocaldb"));
        }
    }
}
```
##### Moving DbConfiguration
- There are cases where it is not possible to place your DbConfiguration class in the same assembly 
  as your DbContext class. For example, you may have two DbContext classes each in different assemblies. 
  
- There are two options for handling this.

- The first option is to use the config file to specify the DbConfiguration instance to use. To do this, 
  set the codeConfigurationType attribute of the entityFramework section.
  
 ```
 <entityFramework codeConfigurationType="MyNamespace.MyDbConfiguration, MyAssembly">
    ...Your EF config...
</entityFramework>
```
- The second option is to place DbConfigurationTypeAttribute on your context class.
- The value passed to the attribute can either be your DbConfiguration type or the 
  assembly and namespace qualified type name string.
```
[DbConfigurationType(typeof(MyDbConfiguration))]
public class MyContextContext : DbContext
{
}
```
```
[DbConfigurationType("MyNamespace.MyDbConfiguration, MyAssembly")]
public class MyContextContext : DbContext
{
}
```

##### Setting DbConfiguration explicitly

- There are some situations where configuration may be needed before any DbContext type has been used.
  - Using DbModelBuilder to build a model without a context
  - Using some other framework/utility code that utilizes a DbContext where that context 
    is used before your application context is used.
    
- In such situations EF is unable to discover the configuration automatically and you must instead do 
  one of the following:
    - Set the DbConfiguration type in the config file
    - Call the static DbConfiguration.SetConfiguration method during application startup
    
### Configuration File Settings

##### A Code-Based Alternative
- Starting in EF6 we introduced code-based configuration, which provides a central way of applying 
  configuration from code.
- The configuration file option allows these settings to be easily changed during deployment 
  without updating your code.
  
##### The Entity Framework Configuration Section

- In EF 4.3 we introduced the custom entityFramework section to handle the new settings.
- The entityFramework section was automatically added to the configuration file of your project 
  when you installed the EntityFramework NuGet package.
  
 ```
<?xml version="1.0" encoding="utf-8"?>
<configuration>
  <configSections>
    <!-- For more information on Entity Framework configuration, visit http://go.microsoft.com/fwlink/?LinkID=237468 -->
    <section name="entityFramework"
       type="System.Data.Entity.Internal.ConfigFile.EntityFrameworkSection, EntityFramework, Version=4.3.0.0, Culture=neutral, PublicKeyToken=b77a5c561934e089" />
  </configSections>
</configuration>
```

##### Connection Strings

- Connection strings go in the standard connectionStrings element and do not require the entityFramework section.

- Code First based models use normal ADO.NET connection strings

```
  <connectionStrings>
    <add name="BlogContext"  
          providerName="System.Data.SqlClient"  
          connectionString="Server=.\SQLEXPRESS;Database=Blogging;Integrated Security=True;"/>
  </connectionStrings>
```

- EF Designer based models use special EF connection strings. 

```
  <connectionStrings>
    <add name="BlogContext"  
      connectionString=
        "metadata=
          res://*/BloggingModel.csdl|
          res://*/BloggingModel.ssdl|
          res://*/BloggingModel.msl;
        provider=System.Data.SqlClient
        provider connection string=
          &quot;data source=(localdb)\mssqllocaldb;
          initial catalog=Blogging;
          integrated security=True;
          multipleactiveresultsets=True;&quot;"
       providerName="System.Data.EntityClient" />
  </connectionStrings>
```

##### Code-Based Configuration Type (EF6 Onwards)

- Starting with EF6, you can specify the DbConfiguration for EF to use for code-based configuration in your application. 
  In most cases you don't need to specify this setting as EF will automatically discover your DbConfiguration.
  
##### EF Database Providers (EF6 Onwards)

- Prior to EF6, Entity Framework-specific parts of a database provider had to be included as part of the core ADO.NET provider. 
  Starting with EF6, the EF specific parts are now managed and registered separately.
  
- Normally you won't need to register providers yourself. This will typically be done by the provider when you install it.

- Providers are registered by including a provider element under the providers child section of the entityFramework section. 
  There are two required attributes for a provider entry:
  - invariantName identifies the core ADO.NET provider that this EF provider targets
  - type is the assembly qualified type name of the EF provider implementation
  
```
  <providers>
    <provider invariantName="System.Data.SqlClient" type="System.Data.Entity.SqlServer.SqlProviderServices, EntityFramework.SqlServer" />
  </providers>
```
##### Interceptors (EF6.1 Onwards)

- Starting with EF6.1 you can register interceptors in the configuration file. Interceptors allow you to run additional logic 
  when EF performs certain operations, such as executing database queries, opening connections, etc.
  
- Interceptors are registered by including an interceptor element under the interceptors child section of the 
  entityFramework section. 
    
- For example, the following configuration registers the built-in DatabaseLogger interceptor that will 
    log all database operations to the Console.
    
```
  <interceptors>
    <interceptor type="System.Data.Entity.Infrastructure.Interception.DatabaseLogger, EntityFramework">
      <parameters>
        <parameter value="C:\Temp\LogOutput.txt"/>
      </parameters>
    </interceptor>
  </interceptors>
```
- By default this will cause the log file to be overwritten with a new file each time the app starts. 
  To instead append to the log file if it already exists use something like:

```
  <interceptors>
    <interceptor type="System.Data.Entity.Infrastructure.Interception.DatabaseLogger, EntityFramework">
      <parameters>
        <parameter value="C:\Temp\LogOutput.txt"/>
        <parameter value="true" type="System.Boolean"/>
      </parameters>
    </interceptor>
  </interceptors>
```

##### Code First Default Connection Factory

- The configuration section allows you to specify a default connection factory that Code First should use to 
  locate a database to use for a context. 
  
- The default connection factory is only used when no connection string has been added to the configuration file for a context.

- When you installed the EF NuGet package a default connection factory was registered that points to either SQL Express or LocalDB, 
  depending on which one you have installed.
  
 - To set a connection factory, you specify the assembly qualified type name in the defaultConnectionFactory element.
 
 ```
    <entityFramework>
    <defaultConnectionFactory type="MyNamespace.MyCustomFactory, MyAssembly"/>
  </entityFramework>
```
- The above example requires the custom factory to have a parameterless constructor. If needed, you can specify constructor 
  parameters using the parameters element.
  
 - For example, the SqlCeConnectionFactory, that is included in Entity Framework, requires you to 
    supply a provider invariant name to the constructor. The provider invariant name identifies the version 
    of SQL Compact you want to use.
   
```
  <entityFramework>
    <defaultConnectionFactory type="System.Data.Entity.Infrastructure.SqlCeConnectionFactory, EntityFramework">
      <parameters>
        <parameter value="System.Data.SqlServerCe.4.0" />
      </parameters>
    </defaultConnectionFactory>
  </entityFramework>
```

- If you don’t set a default connection factory, Code First uses the SqlConnectionFactory, pointing to .\SQLEXPRESS. 

- SqlConnectionFactory also has a constructor that allows you to override parts of the connection string. 
  If you want to use a SQL Server instance other than .\SQLEXPRESS you can use this constructor to set the server.
  
- The following configuration will cause Code First to use MyDatabaseServer for contexts that don’t have an explicit 
  connection string set.
  
```
  <entityFramework>
    <defaultConnectionFactory type="System.Data.Entity.Infrastructure.SqlConnectionFactory, EntityFramework">
      <parameters>
        <parameter value="Data Source=MyDatabaseServer; Integrated Security=True; MultipleActiveResultSets=True" />
      </parameters>
    </defaultConnectionFactory>
  </entityFramework>
```
- By default, it’s assumed that constructor arguments are of type string. You can use the type attribute to change this.
```
  <parameter value="2" type="System.Int32" />
```
##### Database Initializers

- Database initializers are configured on a per-context basis. 
- They can be set in the configuration file using the  context element. This element uses the assembly qualified name 
  to identify the context being configured.
  
- By default, Code First contexts are configured to use the CreateDatabaseIfNotExists initializer. 
- There is a disableDatabaseInitialization attribute on the context element that can be used to disable 
  database initialization.
  
- For example, the following configuration disables database initialization for the Blogging.BlogContext context 
   defined in MyAssembly.dll.

```
  <contexts>
    <context type=" Blogging.BlogContext, MyAssembly" disableDatabaseInitialization="true" />
  </contexts>
```
- You can use the databaseInitializer element to set a custom initializer.
```
  <contexts>
    <context type=" Blogging.BlogContext, MyAssembly">
      <databaseInitializer type="Blogging.MyCustomBlogInitializer, MyAssembly" />
    </context>
  </contexts>
```
- Constructor parameters use the same syntax as default connection factories.
```
  <contexts>
    <context type=" Blogging.BlogContext, MyAssembly">
      <databaseInitializer type="Blogging.MyCustomBlogInitializer, MyAssembly">
        <parameters>
          <parameter value="MyConstructorParameter" />
        </parameters>
      </databaseInitializer>
    </context>
  </contexts>
```

- You can configure one of the generic database initializers that are included in Entity Framework. 
  The type attribute uses the .NET Framework format for generic types.
  
- For example, if you are using Code First Migrations, you can configure the database to be migrated 
  automatically using the *MigrateDatabaseToLatestVersion<TContext, TMigrationsConfiguration>* initializer.
  
```
  <contexts>
    <context type="Blogging.BlogContext, MyAssembly">
      <databaseInitializer type="System.Data.Entity.MigrateDatabaseToLatestVersion`2[[Blogging.BlogContext, MyAssembly], [Blogging.Migrations.Configuration, MyAssembly]], EntityFramework" />
    </context>
  ```
  </contexts>
```
 ### Connection strings and models
 
 - Typically an Entity Framework application uses a class derived from DbContext. This derived class will call 
   one of the constructors on the base DbContext class to control:
    - How the context will connect to a database — that is, how a connection string is found/used
    - Whether the context will use calculate a model using Code First or load a model created with the EF Designer
    - Additional advanced options
    
##### Use Code First with connection by convention

- If you have not done any other configuration in your application, then calling the parameterless constructor on DbContext 
  will cause DbContext to run in Code First mode with a database connection created by convention.
  
- In this example DbContext uses the namespace qualified name of your derived context class—Demo.EF.BloggingContext—
  as the database name and creates a connection string for this database using either SQL Express or LocalDB. 
  - If both are installed, SQL Express will be used.
 
 ```
   namespace Demo.EF
  {
      public class BloggingContext : DbContext
      {
          public BloggingContext()
          // C# will call base class parameterless constructor by default
          {
          }
      }
  }
```
- Visual Studio 2010 includes SQL Express by default and Visual Studio 2012 and later includes LocalDB. 
  During installation, the EntityFramework NuGet package checks which database server is available. 
- The NuGet package will then update the configuration file by setting the default database server that Code First uses 
  when creating a connection by convention. 
- If SQL Express is running, it will be used. If SQL Express is not available then LocalDB will be registered 
  as the default instead. 
- No changes are made to the configuration file if it already contains a setting for the default connection factory.

##### Use Code First with connection by convention and specified database name

- If you have not done any other configuration in your application, then calling the string constructor on DbContext 
  with the database name you want to use will cause DbContext to run in Code First mode with a database connection 
  created by convention to the database of that name.

- In this example DbContext uses “BloggingDatabase” as the database name and creates a connection string for this database 
  using either SQL Express (installed with Visual Studio 2010) or LocalDB (installed with Visual Studio 2012). If both are 
  installed, SQL Express will be used.
  
 ```
  public class BloggingContext : DbContext
  {
      public BloggingContext()
          : base("BloggingDatabase")
      {
      }
  }
 
```

##### Use Code First with connection string in app.config/web.config file

- You may choose to put a connection string in your app.config or web.config file.
- This is an easy way to tell DbContext to use a database server other than SQL Express or LocalDB — 
  the example specifies a SQL Server Compact Edition database.
  
```
  <configuration>
    <connectionStrings>
      <add name="BloggingCompactDatabase"
           providerName="System.Data.SqlServerCe.4.0"
           connectionString="Data Source=Blogging.sdf"/>
    </connectionStrings>
  </configuration>
```
- If the name of the connection string matches the name of your context then it will be found by DbContext 
  when the parameterless constructor is used. 
- If the connection string name is different from the name of your context then you can tell DbContext to 
   use this connection in Code First mode by passing the connection string name to the DbContext constructor
   
```
  public class BloggingContext : DbContext
  {
      public BloggingContext()
          : base("BloggingCompactDatabase")
      {
      }
  }
```

- Alternatively, you can use the form “name=<connection string name>” for the string passed to the DbContext constructor.
- This form makes it explicit that you expect the connection string to be found in your config file. An exception will be 
  thrown if a connection string with the given name is not found.
  
```
  public class BloggingContext : DbContext
  {
      public BloggingContext()
          : base("name=BloggingCompactDatabase")
      {
      }
  }
```

##### Database/Model First with connection string in app.config/web.config file

- Models created with the EF Designer are different from Code First in that your model already exists and 
  is not generated from code when the application runs. 
- The model typically exists as an EDMX file in your project.

- The designer will add an EF connection string to your app.config or web.config file.
```
  <configuration>  
    <connectionStrings>  
      <add name="Northwind_Entities"  
           connectionString="metadata=res://*/Northwind.csdl|  
                                      res://*/Northwind.ssdl|  
                                      res://*/Northwind.msl;  
                             provider=System.Data.SqlClient;  
                             provider connection string=  
                                 &quot;Data Source=.\sqlexpress;  
                                       Initial Catalog=Northwind;  
                                       Integrated Security=True;  
                                       MultipleActiveResultSets=True&quot;"  
           providerName="System.Data.EntityClient"/>  
    </connectionStrings>  
  </configuration>
```

- The EF Designer will also generate code that tells DbContext to use this connection by passing 
  the connection string name to the DbContext constructor.
- DbContext knows to load the existing model (rather than using Code First to calculate it from code) 
  because the connection string is an EF connection string containing details of the model to use.  
```
  public class NorthwindContext : DbContext
  {
      public NorthwindContext()
          : base("name=Northwind_Entities")
      {
      }
  }
```
##### Other DbContext constructor options

- The DbContext class contains other constructors and usage patterns that enable some more advanced scenarios. 

  - You can use the DbModelBuilder class to build a Code First model without instantiating a DbContext instance. 
    The result of this is a DbModel object. You can then pass this DbModel object to one of the DbContext constructors 
    when you are ready to create your DbContext instance.
  - You can pass a full connection string to DbContext instead of just the database or connection string name. 
    By default this connection string is used with the System.Data.SqlClient provider; this can be changed by 
    setting a different implementation of IConnectionFactory onto context.Database.DefaultConnectionFactory.
  - You can use an existing DbConnection object by passing it to a DbContext constructor. If the connection object 
    is an instance of EntityConnection, then the model specified in the connection will be used rather than calculating 
    a model using Code First. If the object is an instance of some other type—for example, SqlConnection—then the context 
    will use it for Code First mode.
  - You can pass an existing ObjectContext to a DbContext constructor to create a DbContext wrapping the existing context. 
    This can be used for existing applications that use ObjectContext but which want to take advantage of DbContext in some 
    parts of the application.


  



   
   




  
  




 
  

