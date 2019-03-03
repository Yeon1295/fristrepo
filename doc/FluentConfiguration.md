# Fluent Configuration

- When working with Code First, you define your model by defining your domain CLR classes. 
- By default, Entity Framework uses the Code First conventions to map your classes to the database schema. 
- If you use the Code First naming conventions, in most cases you can rely on Code First to set up relationships between your 
  tables based on the foreign keys and navigation properties that you define on the classes. 
- If you do not follow the conventions when defining your classes, or if you want to change the way the conventions work, 
  you can use the fluent API or data annotations to configure your classes so Code First can map the relationships between your tables.
  
## Fluent API - Relationships

- When configuring a relationship with the fluent API, you start with the EntityTypeConfiguration instance and then use the 
 HasRequired, HasOptional, or HasMany method to specify the type of relationship this entity participates in. 

- The HasRequired and HasOptional methods take a lambda expression that represents a reference navigation property. 
- The HasMany method takes a lambda expression that represents a collection navigation property. 
- You can then configure an inverse navigation property by using the WithRequired, WithOptional, and WithMany methods. 
- These methods have overloads that do not take arguments and can be used to specify cardinality with unidirectional navigations.

- You can then configure foreign key properties by using the HasForeignKey method. This method takes a lambda expression that 
  represents the property to be used as the foreign key.

###  Model Used in Samples
```
using System.Data.Entity;
using System.Data.Entity.ModelConfiguration.Conventions;
// add a reference to System.ComponentModel.DataAnnotations DLL
using System.ComponentModel.DataAnnotations;
using System.Collections.Generic;
using System;

public class SchoolEntities : DbContext
{
    public DbSet<Course> Courses { get; set; }
    public DbSet<Department> Departments { get; set; }
    public DbSet<Instructor> Instructors { get; set; }
    public DbSet<OfficeAssignment> OfficeAssignments { get; set; }

    protected override void OnModelCreating(DbModelBuilder modelBuilder)
    {
        // Configure Code First to ignore PluralizingTableName convention
        // If you keep this convention then the generated tables will have pluralized names.
        modelBuilder.Conventions.Remove<PluralizingTableNameConvention>();
    }
}

public class Department
{
    public Department()
    {
        this.Courses = new HashSet<Course>();
    }
    // Primary key
    public int DepartmentID { get; set; }
    public string Name { get; set; }
    public decimal Budget { get; set; }
    public System.DateTime StartDate { get; set; }
    public int? Administrator { get; set; }

    // Navigation property
    public virtual ICollection<Course> Courses { get; private set; }
}

public class Course
{
    public Course()
    {
        this.Instructors = new HashSet<Instructor>();
    }
    // Primary key
    public int CourseID { get; set; }

    public string Title { get; set; }
    public int Credits { get; set; }

    // Foreign key
    public int DepartmentID { get; set; }

    // Navigation properties
    public virtual Department Department { get; set; }
    public virtual ICollection<Instructor> Instructors { get; private set; }
}

public partial class OnlineCourse : Course
{
    public string URL { get; set; }
}

public partial class OnsiteCourse : Course
{
    public OnsiteCourse()
    {
        Details = new Details();
    }

    public Details Details { get; set; }
}

public class Details
{
    public System.DateTime Time { get; set; }
    public string Location { get; set; }
    public string Days { get; set; }
}

public class Instructor
{
    public Instructor()
    {
        this.Courses = new List<Course>();
    }

    // Primary key
    public int InstructorID { get; set; }
    public string LastName { get; set; }
    public string FirstName { get; set; }
    public System.DateTime HireDate { get; set; }

    // Navigation properties
    public virtual ICollection<Course> Courses { get; private set; }
}

public class OfficeAssignment
{
    // Specifying InstructorID as a primary
    [Key()]
    public Int32 InstructorID { get; set; }

    public string Location { get; set; }

    // When Entity Framework sees Timestamp attribute
    // it configures ConcurrencyCheck and DatabaseGeneratedPattern=Computed.
    [Timestamp]
    public Byte[] Timestamp { get; set; }

    // Navigation property
    public virtual Instructor Instructor { get; set; }
}
```
### Configuring a Required-to-Optional Relationship (One-to–Zero-or-One)

- The following example configures a one-to-zero-or-one relationship. 
- The OfficeAssignment has the InstructorID property that is a primary key and a foreign key, because the name of the property 
  does not follow the convention the HasKey method is used to configure the primary key.
  
```
    // Configure the primary key for the OfficeAssignment
    modelBuilder.Entity<OfficeAssignment>()
        .HasKey(t => t.InstructorID);

    // Map one-to-zero or one relationship
    modelBuilder.Entity<OfficeAssignment>()
        .HasRequired(t => t.Instructor)
        .WithOptional(t => t.OfficeAssignment);
```
### Configuring a Relationship Where Both Ends Are Required (One-to-One)

- In most cases Entity Framework can infer which type is the dependent and which is the principal in a relationship. 
- However, when both ends of the relationship are required or both sides are optional Entity Framework cannot identify the 
  dependent and principal. 
- When both ends of the relationship are required, use WithRequiredPrincipal or WithRequiredDependent after the HasRequired method. 
- When both ends of the relationship are optional, use WithOptionalPrincipal or WithOptionalDependent after the HasOptional method.

```
    // Configure the primary key for the OfficeAssignment
    modelBuilder.Entity<OfficeAssignment>()
        .HasKey(t => t.InstructorID);

    modelBuilder.Entity<Instructor>()
        .HasRequired(t => t.OfficeAssignment)
        .WithRequiredPrincipal(t => t.Instructor);
```
### Configuring a Many-to-Many Relationship

- The following code configures a many-to-many relationship between the Course and Instructor types. 
- In the following example, the default Code First conventions are used to create a join table. 
- As a result the CourseInstructor table is created with Course_CourseID and Instructor_InstructorID columns.

```
    modelBuilder.Entity<Course>()
        .HasMany(t => t.Instructors)
        .WithMany(t => t.Courses)
```
- If you want to specify the join table name and the names of the columns in the table you need to do additional configuration 
  by using the Map method. 
- The following code generates the CourseInstructor table with CourseID and InstructorID columns 

```
      modelBuilder.Entity<Course>()
        .HasMany(t => t.Instructors)
        .WithMany(t => t.Courses)
        .Map(m =>
        {
            m.ToTable("CourseInstructor");
            m.MapLeftKey("CourseID");
            m.MapRightKey("InstructorID");
        });
```

### Configuring a Relationship with One Navigation Property

- A one-directional (also called unidirectional) relationship is when a navigation property is defined on only one of the 
  relationship ends and not on both. 

- By convention, Code First always interprets a unidirectional relationship as one-to-many. 

- For example, if you want a one-to-one relationship between Instructor and OfficeAssignment, where you have a navigation property 
  on only the Instructor type, you need to use the fluent API to configure this relationship.
  
```
    // Configure the primary Key for the OfficeAssignment
    modelBuilder.Entity<OfficeAssignment>()
        .HasKey(t => t.InstructorID);

    modelBuilder.Entity<Instructor>()
        .HasRequired(t => t.OfficeAssignment)
        .WithRequiredPrincipal();
```

### Enabling Cascade Delete

- You can configure cascade delete on a relationship by using the WillCascadeOnDelete method. 
- If a foreign key on the dependent entity is not nullable, then Code First sets cascade delete on the relationship. 
- If a foreign key on the dependent entity is nullable, Code First does not set cascade delete on the relationship, and when the principal is deleted the foreign key will be set to null.

- You can remove these cascade delete conventions by using:

      modelBuilder.Conventions.Remove<OneToManyCascadeDeleteConvention>()
      modelBuilder.Conventions.Remove<ManyToManyCascadeDeleteConvention>()

- The following code configures the relationship to be required and then disables cascade delete.

```
      modelBuilder.Entity<Course>()
          .HasRequired(t => t.Department)
          .WithMany(t => t.Courses)
          .HasForeignKey(d => d.DepartmentID)
          .WillCascadeOnDelete(false);
```

### Configuring a Composite Foreign Key

- If the primary key on the Department type consisted of DepartmentID and Name properties, 
- you would configure the primary key for the Department and the foreign key on the Course types as follows:

```
    // Composite primary key
    modelBuilder.Entity<Department>()
    .HasKey(d => new { d.DepartmentID, d.Name });

    // Composite foreign key
    modelBuilder.Entity<Course>()  
        .HasRequired(c => c.Department)  
        .WithMany(d => d.Courses)
        .HasForeignKey(d => new { d.DepartmentID, d.DepartmentName });
```

### Renaming a Foreign Key That Is Not Defined in the Model

- If you choose not to define a foreign key on the CLR type, but want to specify what name it should have in the database, 
  do the following:
  
```
     modelBuilder.Entity<Course>()
        .HasRequired(c => c.Department)
        .WithMany(t => t.Courses)
        .Map(m => m.MapKey("ChangedDepartmentID")); 
```

### Configuring a Foreign Key Name That Does Not Follow the Code First Convention

- If the foreign key property on the Course class was called SomeDepartmentID instead of DepartmentID, 
  you would need to do the following to specify that you want SomeDepartmentID to be the foreign key:
  
```  
    modelBuilder.Entity<Course>()
             .HasRequired(c => c.Department)
             .WithMany(d => d.Courses)
             .HasForeignKey(c => c.SomeDepartmentID);
```

## Fluent API - Configuring and Mapping Properties and Types

- There are two main ways you can configure EF to use something other than conventions, namely annotations or EFs fluent API. 
- The annotations only cover a subset of the fluent API functionality, so there are mapping scenarios that cannot be achieved 
  using annotations.
  
- The code first fluent API is most commonly accessed by overriding the OnModelCreating method on your derived DbContext. 

### Model-Wide Settings

##### Default Schema

- Starting with EF6 you can use the HasDefaultSchema method on DbModelBuilder to specify the database schema to use for all 
  tables, stored procedures, etc
  
  ```
        modelBuilder.HasDefaultSchema("sales");
  ```
  
  ##### Custom Conventions
  
  - Starting with EF6 you can create your own conventions to supplement the ones included in Code First. 
  
  ### Property Mapping
  
- The Property method is used to configure attributes for each property belonging to an entity or complex type. 
- The Property method is used to obtain a configuration object for a given property. 
- The options on the configuration object are specific to the type being configured; IsUnicode is available only on string 
  properties for example.
  
##### Configuring a Primary Key
  
- The Entity Framework convention for primary keys is:
      1. Your class defines a property whose name is “ID” or “Id”
      2. or a class name followed by “ID” or “Id”
      
- To explicitly set a property to be a primary key, you can use the HasKey method. 

```
    modelBuilder.Entity<OfficeAssignment>().HasKey(t => t.InstructorID);
```

##### Configuring a Composite Primary Key

- The following example configures the DepartmentID and Name properties to be the composite primary key of the Department type.

```
    modelBuilder.Entity<Department>().HasKey(t => new { t.DepartmentID, t.Name });
```

##### Switching off Identity for Numeric Primary Keys

- The following example sets the DepartmentID property to System.ComponentModel.DataAnnotations.DatabaseGeneratedOption.None 
  to indicate that the value will not be generated by the database.
  
```
        modelBuilder.Entity<Department>().Property(t => t.DepartmentID)
                    .HasDatabaseGeneratedOption(DatabaseGeneratedOption.None);
```
##### Specifying the Maximum Length on a Property

- In the following example, the Name property should be no longer than 50 characters. 
- If you make the value longer than 50 characters, you will get a DbEntityValidationException exception. 
= If Code First creates a database from this model it will also set the maximum length of the Name column to 50 characters.

```
        modelBuilder.Entity<Department>().Property(t => t.Name).HasMaxLength(50);
```

##### Configuring the Property to be Required

- In the following example, the Name property is required. 
- If you do not specify the Name, you will get a DbEntityValidationException exception. 
- If Code First creates a database from this model then the column used to store this property will usually be non-nullable.

- In some cases it may not be possible for the column in the database to be non-nullable even though the property is required.
- For example, when using a TPH inheritance strategy data for multiple types is stored in a single table. 
- If a derived type includes a required property the column cannot be made non-nullable since not all types in the hierarchy 
  will have this property.
  
  ```
          modelBuilder.Entity<Department>().Property(t => t.Name).IsRequired();
  ```
  
  ##### Configuring an Index on one or more properties
  
  - Creating indexes isn't natively supported by the Fluent API, but you can make use of the support for IndexAttribute via the 
    Fluent API. 
  - Index attributes are processed by including a model annotation on the model that is then turned into an Index in the database 
    later in the pipeline. 
  = You can manually add these same annotations using the Fluent API.
  
  
  - The easiest way to do this is to create an instance of IndexAttribute that contains all the settings for the new index. 
  - You can then create an instance of IndexAnnotation which is an EF specific type that will convert the IndexAttribute settings 
    into a model annotation that can be stored on the EF model. 
  - These can then be passed to the HasColumnAnnotation method on the Fluent API, specifying the name Index for the annotation.
  
```
          modelBuilder
            .Entity<Department>()
            .Property(t => t.Name)
            .HasColumnAnnotation("Index", new IndexAnnotation(new IndexAttribute()));
```
- You can specify multiple index annotations on a single property by passing an array of IndexAttribute to the constructor of 
  IndexAnnotation.
  
```
        modelBuilder
            .Entity<Department>()
            .Property(t => t.Name)
            .HasColumnAnnotation(
                "Index",  
                new IndexAnnotation(new[]
                    {
                        new IndexAttribute("Index1"),
                        new IndexAttribute("Index2") { IsUnique = true }
                    })));
```

##### Specifying Not to Map a CLR Property to a Column in the Database

- The following example shows how to specify that a property on a CLR type is not mapped to a column in the database.
```
        modelBuilder.Entity<Department>().Ignore(t => t.Budget);
```

##### Mapping a CLR Property to a Specific Column in the Database

- The following example maps the Name CLR property to the DepartmentName database column.

```
        modelBuilder.Entity<Department>()
            .Property(t => t.Name)
            .HasColumnName("DepartmentName");
```

##### Renaming a Foreign Key That Is Not Defined in the Model

- If you choose not to define a foreign key on a CLR type, but want to specify what name it should have in the database, do 
  the following:
  
```
        modelBuilder.Entity<Course>()
            .HasRequired(c => c.Department)
            .WithMany(t => t.Courses)
            .Map(m => m.MapKey("ChangedDepartmentID"));
```

##### Configuring whether a String Property Supports Unicode Content

- By default strings are Unicode (nvarchar in SQL Server). 
- You can use the IsUnicode method to specify that a string should be of varchar type.

```
        modelBuilder.Entity<Department>()
            .Property(t => t.Name)
            .IsUnicode(false);
```

##### Configuring the Data Type of a Database Column

- The HasColumnType method enables mapping to different representations of the same basic type. 
- Using this method does not enable you to perform any conversion of the data at run time. 
- Note that IsUnicode is the preferred way of setting columns to varchar, as it is database agnostic.

```
        modelBuilder.Entity<Department>()   
            .Property(p => p.Name)   
            .HasColumnType("varchar");
````

##### Configuring Properties on a Complex Type

- There are two ways to configure scalar properties on a complex type

- You can call Property on ComplexTypeConfiguration.
```
        modelBuilder.ComplexType<Details>()
            .Property(t => t.Location)
            .HasMaxLength(20);
```

- You can also use the dot notation to access a property of a complex type.
```
        modelBuilder.Entity<OnsiteCourse>()
            .Property(t => t.Details.Location)
            .HasMaxLength(20);
```

##### Configuring a Property to Be Used as an Optimistic Concurrency Token

- To specify that a property in an entity represents a concurrency token, 
-  you can use either the ConcurrencyCheck attribute or the IsConcurrencyToken method.

```
        modelBuilder.Entity<OfficeAssignment>()
            .Property(t => t.Timestamp)
            .IsConcurrencyToken();
```
- You can also use the IsRowVersion method to configure the property to be a row version in the database. 
  Setting the property to be a row version automatically configures it to be an optimistic concurrency token.
```
        modelBuilder.Entity<OfficeAssignment>()
            .Property(t => t.Timestamp)
            .IsRowVersion();
```

### Type Mapping

##### Specifying That a Class Is a Complex Type

- By convention, a type that has no primary key specified is treated as a complex type. 
- There are some scenarios where Code First will not detect a complex type (for example, if you do have a property called ID, 
  but you do not mean for it to be a primary key). 
- In such cases, you would use the fluent API to explicitly specify that a type is a complex type.

```
        modelBuilder.ComplexType<Details>();
```

##### Specifying Not to Map a CLR Entity Type to a Table in the Database

- The following example shows how to exclude a CLR type from being mapped to a table in the database.

```
        modelBuilder.Ignore<OnlineCourse>();
```

##### Mapping an Entity Type to a Specific Table in the Database

- All properties of Department will be mapped to columns in a table called t_ Department.

```
        modelBuilder.Entity<Department>()  
            .ToTable("t_Department");
```
- You can also specify the schema name like this:
```
        modelBuilder.Entity<Department>()  
            .ToTable("t_Department", "school");
```

##### Mapping the Table-Per-Hierarchy (TPH) Inheritance

- In the TPH mapping scenario, all types in an inheritance hierarchy are mapped to a single table. 
- A discriminator column is used to identify the type of each row. When creating your model with Code First, 
- TPH is the default strategy for the types that participate in the inheritance hierarchy. 
- By default, the discriminator column is added to the table with the name “Discriminator” and the CLR type name 
  of each type in the hierarchy is used for the discriminator values. 
- You can modify the default behavior by using the fluent API.

```
        modelBuilder.Entity<Course>()  
            .Map<Course>(m => m.Requires("Type").HasValue("Course"))  
            .Map<OnsiteCourse>(m => m.Requires("Type").HasValue("OnsiteCourse"));
```

##### Mapping the Table-Per-Type (TPT) Inheritance

- In the TPT mapping scenario, all types are mapped to individual tables. 
- Properties that belong solely to a base type or derived type are stored in a table that maps to that type. 
- Tables that map to derived types also store a foreign key that joins the derived table with the base table.

```
        modelBuilder.Entity<Course>().ToTable("Course");  
        modelBuilder.Entity<OnsiteCourse>().ToTable("OnsiteCourse");
```

##### Mapping the Table-Per-Concrete Class (TPC) Inheritance

- In the TPC mapping scenario, all non-abstract types in the hierarchy are mapped to individual tables. 
- The tables that map to the derived classes have no relationship to the table that maps to the base class in the database. 
- All properties of a class, including inherited properties, are mapped to columns of the corresponding table.

- Call the MapInheritedProperties method to configure each derived type. 
- MapInheritedProperties remaps all properties that were inherited from the base class to new columns in the table for the 
  derived class.
 
 > **Note**  
 >  Note
> Note that because the tables participating in TPC inheritance hierarchy do not share a primary key there will be duplicate 
> entity keys when inserting in tables that are mapped to subclasses if you have database generated values with the same identity 
> seed. To solve this problem you can either specify a different initial seed value for each table or switch off identity on the 
> primary key property. Identity is the default value for integer key properties when working with Code First.

```
        modelBuilder.Entity<Course>()
            .Property(c => c.CourseID)
            .HasDatabaseGeneratedOption(DatabaseGeneratedOption.None);

        modelBuilder.Entity<OnsiteCourse>().Map(m =>
        {
            m.MapInheritedProperties();
            m.ToTable("OnsiteCourse");
        });

        modelBuilder.Entity<OnlineCourse>().Map(m =>
        {
            m.MapInheritedProperties();
            m.ToTable("OnlineCourse");
        });
```

##### Mapping Properties of an Entity Type to Multiple Tables in the Database (Entity Splitting)

- Entity splitting allows the properties of an entity type to be spread across multiple tables. 

- In the following example, the Department entity is split into two tables: Department and DepartmentDetails. 
- Entity splitting uses multiple calls to the Map method to map a subset of properties to a specific table.

```
        modelBuilder.Entity<Department>()
            .Map(m =>
            {
                m.Properties(t => new { t.DepartmentID, t.Name });
                m.ToTable("Department");
            })
            .Map(m =>
            {
                m.Properties(t => new { t.DepartmentID, t.Administrator, t.StartDate, t.Budget });
                m.ToTable("DepartmentDetails");
            });
```

##### Mapping Multiple Entity Types to One Table in the Database (Table Splitting)

- The following example maps two entity types that share a primary key to one table.

```
        modelBuilder.Entity<OfficeAssignment>()
            .HasKey(t => t.InstructorID);

        modelBuilder.Entity<Instructor>()
            .HasRequired(t => t.OfficeAssignment)
            .WithRequiredPrincipal(t => t.Instructor);

        modelBuilder.Entity<Instructor>().ToTable("Instructor");

        modelBuilder.Entity<OfficeAssignment>().ToTable("Instructor");
```

## Code First Insert, Update, and Delete Stored Procedures

- By default, Code First will configure all entities to perform insert, update and delete commands using direct table access. 
- Starting in EF6 you can configure your Code First model to use stored procedures for some or all entities in your model.

### Basic Entity Mapping

- You can opt into using stored procedures for insert, update and delete using the Fluent API.
```
        modelBuilder
          .Entity<Blog>()
          .MapToStoredProcedures();
```
- Doing this will cause Code First to use some conventions to build the expected shape of the stored procedures in the database.

    - Three stored procedures named <type_name>_Insert, <type_name>_Update and <type_name>_Delete (for example, Blog_Insert, 
      Blog_Update and Blog_Delete).
      
    - Parameter names correspond to the property names.
    - If you use HasColumnName() or the Column attribute to rename the column for a given property then this name is used for 
      parameters instead of the property name.
      
    - The insert stored procedure will have a parameter for every property, except for those marked as store generated (identity 
      or computed). The stored procedure should return a result set with a column for each store generated property.
      
    - The update stored procedure will have a parameter for every property, except for those marked with a store generated pattern 
      of 'Computed'. Some concurrency tokens require a parameter for the original value. The stored procedure should return a result 
      set with a column for each computed property.
      
    - The delete stored procedure should have a parameter for the key value of the entity (or multiple parameters if the entity 
      has a composite key). Additionally, the delete procedure should also have parameters for any independent association foreign 
      keys on the target table (relationships that do not have corresponding foreign key properties declared in the entity). Some 
      concurrency tokens require a parameter for the original value.     

- Using the following class as an example:
```
        public class Blog  
        {  
          public int BlogId { get; set; }  
          public string Name { get; set; }  
          public string Url { get; set; }  
        }
```
- The default stored procedures would be:
```
        CREATE PROCEDURE [dbo].[Blog_Insert]  
          @Name nvarchar(max),  
          @Url nvarchar(max)  
        AS  
        BEGIN
          INSERT INTO [dbo].[Blogs] ([Name], [Url])
          VALUES (@Name, @Url)

          SELECT SCOPE_IDENTITY() AS BlogId
        END
        CREATE PROCEDURE [dbo].[Blog_Update]  
          @BlogId int,  
          @Name nvarchar(max),  
          @Url nvarchar(max)  
        AS  
          UPDATE [dbo].[Blogs]
          SET [Name] = @Name, [Url] = @Url     
          WHERE BlogId = @BlogId;
        CREATE PROCEDURE [dbo].[Blog_Delete]  
          @BlogId int  
        AS  
          DELETE FROM [dbo].[Blogs]
          WHERE BlogId = @BlogId
```
##### Overriding the Defaults

- You can override part or all of what was configured by default.

- You can change the name of one or more stored procedures. This example renames the update stored procedure only.

```
      modelBuilder  
        .Entity<Blog>()  
        .MapToStoredProcedures(s =>  
          s.Update(u => u.HasName("modify_blog")));
```
- This example renames all three stored procedures.
```
      modelBuilder  
        .Entity<Blog>()  
        .MapToStoredProcedures(s =>  
          s.Update(u => u.HasName("modify_blog"))  
           .Delete(d => d.HasName("delete_blog"))  
           .Insert(i => i.HasName("insert_blog")));
```
- In these examples the calls are chained together, but you can also use lambda block syntax.
```
      modelBuilder  
        .Entity<Blog>()  
        .MapToStoredProcedures(s =>  
          {  
            s.Update(u => u.HasName("modify_blog"));  
            s.Delete(d => d.HasName("delete_blog"));  
            s.Insert(i => i.HasName("insert_blog"));  
          });
```
- This example renames the parameter for the BlogId property on the update stored procedure.
```
      modelBuilder  
        .Entity<Blog>()  
        .MapToStoredProcedures(s =>  
          s.Update(u => u.Parameter(b => b.BlogId, "blog_id")));
```
- These calls are all chainable and composable. Here is an example that renames all three stored procedures and their parameters.
```
      modelBuilder  
        .Entity<Blog>()  
        .MapToStoredProcedures(s =>  
          s.Update(u => u.HasName("modify_blog")  
                         .Parameter(b => b.BlogId, "blog_id")  
                         .Parameter(b => b.Name, "blog_name")  
                         .Parameter(b => b.Url, "blog_url"))  
           .Delete(d => d.HasName("delete_blog")  
                         .Parameter(b => b.BlogId, "blog_id"))  
           .Insert(i => i.HasName("insert_blog")  
                         .Parameter(b => b.Name, "blog_name")  
                         .Parameter(b => b.Url, "blog_url")));
```

- You can also change the name of the columns in the result set that contains database generated values.
```
      modelBuilder
        .Entity<Blog>()
        .MapToStoredProcedures(s =>
          s.Insert(i => i.Result(b => b.BlogId, "generated_blog_identity")));
```

```
      CREATE PROCEDURE [dbo].[Blog_Insert]  
        @Name nvarchar(max),  
        @Url nvarchar(max)  
      AS  
      BEGIN
        INSERT INTO [dbo].[Blogs] ([Name], [Url])
        VALUES (@Name, @Url)

        SELECT SCOPE_IDENTITY() AS generated_blog_id
      END
```
### Relationships Without a Foreign Key in the Class (Independent Associations)

- When a foreign key property is included in the class definition, the corresponding parameter can be renamed in the same way as 
  any other property. 
- When a relationship exists without a foreign key property in the class, the default parameter name is 
  <navigation_property_name>_<primary_key_name>.
  
- For example, the following class definitions would result in a Blog_BlogId parameter being expected in the stored procedures 
  to insert and update Posts. 
  
```
      public class Blog  
      {  
        public int BlogId { get; set; }  
        public string Name { get; set; }  
        public string Url { get; set; }

        public List<Post> Posts { get; set; }  
      }  

      public class Post  
      {  
        public int PostId { get; set; }  
        public string Title { get; set; }  
        public string Content { get; set; }  

        public Blog Blog { get; set; }  
      }
```

##### Overriding the Defaults

- You can change parameters for foreign keys that are not included in the class by supplying the path to the primary key 
  property to the Parameter method.
  
```
      modelBuilder
        .Entity<Post>()  
        .MapToStoredProcedures(s =>  
          s.Insert(i => i.Parameter(p => p.Blog.BlogId, "blog_id")));
```
- If you don’t have a navigation property on the dependent entity (i.e no Post.Blog property) then you can use the Association 
  method to identify the other end of the relationship and then configure the parameters that correspond to each of the key 
  property(s).
  
```
      modelBuilder
        .Entity<Post>()  
        .MapToStoredProcedures(s =>  
          s.Insert(i => i.Navigation<Blog>(  
            b => b.Posts,  
            c => c.Parameter(b => b.BlogId, "blog_id"))));
```

### Concurrency Tokens

- Update and delete stored procedures may also need to deal with concurrency:

    - If the entity contains concurrency tokens, the stored procedure can optionally have an output parameter that returns the 
      number of rows updated/deleted (rows affected). Such a parameter must be configured using the RowsAffectedParameter method.
      By default EF uses the return value from ExecuteNonQuery to determine how many rows were affected. Specifying a rows affected 
      output parameter is useful if you perform any logic in your sproc that would result in the return value of ExecuteNonQuery 
      being incorrect (from EF's perspective) at the end of execution.
      
    - For each concurrency token there will be a parameter named <property_name>_Original (for example, Timestamp_Original ). This 
      will be passed the original value of this property – the value when queried from the database.
      
      - Concurrency tokens that are computed by the database – such as timestamps – will only have an original value parameter.
      - Non-computed properties that are set as concurrency tokens will also have a parameter for the new value in the update 
        procedure. This uses the naming conventions already discussed for new values. An example of such a token would be using 
        a Blog's URL as a concurrency token, the new value is required because this can be updated to a new value by your code 
        (unlike a Timestamp token which is only updated by the database).
        
- This is an example class and update stored procedure with a timestamp concurrency token.

```
      public class Blog  
      {  
        public int BlogId { get; set; }  
        public string Name { get; set; }  
        public string Url { get; set; }  
        [Timestamp]
        public byte[] Timestamp { get; set; }
      }
```

```
      CREATE PROCEDURE [dbo].[Blog_Update]  
        @BlogId int,  
        @Name nvarchar(max),  
        @Url nvarchar(max),
        @Timestamp_Original rowversion  
      AS  
        UPDATE [dbo].[Blogs]
        SET [Name] = @Name, [Url] = @Url     
        WHERE BlogId = @BlogId AND [Timestamp] = @Timestamp_Original
```

- Here is an example class and update stored procedure with non-computed concurrency token.

```
      public class Blog  
      {  
        public int BlogId { get; set; }  
        public string Name { get; set; }  
        [ConcurrencyCheck]
        public string Url { get; set; }  
      }
```

```
      CREATE PROCEDURE [dbo].[Blog_Update]  
        @BlogId int,  
        @Name nvarchar(max),  
        @Url nvarchar(max),
        @Url_Original nvarchar(max),
      AS  
        UPDATE [dbo].[Blogs]
        SET [Name] = @Name, [Url] = @Url     
        WHERE BlogId = @BlogId AND [Url] = @Url_Original
```

##### Overriding the Defaults

- You can optionally introduce a rows affected parameter.

```
      modelBuilder  
        .Entity<Blog>()  
        .MapToStoredProcedures(s =>  
          s.Update(u => u.RowsAffectedParameter("rows_affected")));
```
- For database computed concurrency tokens – where only the original value is passed – you can just use the standard parameter 
  renaming mechanism to rename the parameter for the original value.
  
```
      modelBuilder  
        .Entity<Blog>()  
        .MapToStoredProcedures(s =>  
          s.Update(u => u.Parameter(b => b.Timestamp, "blog_timestamp")));
```

- For non-computed concurrency tokens – where both the original and new value are passed – you can use an overload of Parameter 
  that allows you to supply a name for each parameter.

```
      modelBuilder
       .Entity<Blog>()
       .MapToStoredProcedures(s => s.Update(u => u.Parameter(b => b.Url, "blog_url", "blog_original_url")));
```

### Many to Many Relationships

```
      public class Post  
      {  
        public int PostId { get; set; }  
        public string Title { get; set; }  
        public string Content { get; set; }  

        public List<Tag> Tags { get; set; }  
      }  

      public class Tag  
      {  
        public int TagId { get; set; }  
        public string TagName { get; set; }  

        public List<Post> Posts { get; set; }  
      }
```
- Many to many relationships can be mapped to stored procedures with the following syntax.
```
      modelBuilder  
        .Entity<Post>()  
        .HasMany(p => p.Tags)  
        .WithMany(t => t.Posts)  
        .MapToStoredProcedures();
```

- If no other configuration is supplied then the following stored procedure shape is used by default.
    
    - Two stored procedures named <type_one><type_two>_Insert and <type_one><type_two>_Delete (for example, PostTag_Insert 
      and PostTag_Delete).
    - The parameters will be the key value(s) for each type. The name of each parameter being <type_name>_<property_name> 
      (for example, Post_PostId and Tag_TagId).
      
- Here are example insert and update stored procedures.
```
      CREATE PROCEDURE [dbo].[PostTag_Insert]  
        @Post_PostId int,  
        @Tag_TagId int  
      AS  
        INSERT INTO [dbo].[Post_Tags] (Post_PostId, Tag_TagId)   
        VALUES (@Post_PostId, @Tag_TagId)
      CREATE PROCEDURE [dbo].[PostTag_Delete]  
        @Post_PostId int,  
        @Tag_TagId int  
      AS  
        DELETE FROM [dbo].[Post_Tags]    
        WHERE Post_PostId = @Post_PostId AND Tag_TagId = @Tag_TagId
```
##### Overriding the Defaults

- The procedure and parameter names can be configured in a similar way to entity stored procedures.

```
      modelBuilder  
        .Entity<Post>()  
        .HasMany(p => p.Tags)  
        .WithMany(t => t.Posts)  
        .MapToStoredProcedures(s =>  
          s.Insert(i => i.HasName("add_post_tag")  
                         .LeftKeyParameter(p => p.PostId, "post_id")  
                         .RightKeyParameter(t => t.TagId, "tag_id"))  
           .Delete(d => d.HasName("remove_post_tag")  
                         .LeftKeyParameter(p => p.PostId, "post_id")  
                         .RightKeyParameter(t => t.TagId, "tag_id")));
```


















  
  
  







