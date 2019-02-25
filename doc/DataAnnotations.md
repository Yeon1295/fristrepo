# Code First Data Annotations

- Entity Framework Code First allows you to use your own domain classes to represent the model that EF relies on to 
  perform querying, change tracking, and updating functions. 
  
- Code First leverages a programming pattern referred to as 'convention over configuration.' 
- Code First will assume that your classes follow the conventions of Entity Framework, and in that case, will 
  automatically work out how to perform it's job. However, 
  
- if your classes do not follow those conventions, you have the ability to add configurations to your classes to 
  provide EF with the requisite information.
  
- Code First gives you two ways to add these configurations to your classes. One is using simple attributes called DataAnnotations, 
  and the second is using Code First’s Fluent API.
  
- DataAnnotations are also understood by a number of .NET applications, such as ASP.NET MVC which allows these applications 
  to leverage the same annotations for client-side validations.
  
### The model

- As they are, the Blog and Post classes conveniently follow code first convention and require no tweaks to enable EF compatability. 
  However, you can also use the annotations to provide more information to EF about the classes and the database to which they map.
```
    public class Blog
    {
        public int Id { get; set; }
        public string Title { get; set; }
        public string BloggerName { get; set;}
        public virtual ICollection<Post> Posts { get; set; }
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
### Key
- Entity Framework relies on every entity having a key value that is used for entity tracking. 
- One convention of Code First is implicit key properties; Code First will look for a property named “Id”, or a combination of 
  class name and “Id”, such as “BlogId”. This property will map to a primary key column in the database.
- You can use the key annotation to specify which property is to be used as the EntityKey. 
 
 ```
    public class Blog
    {
        [Key]
        public int PrimaryTrackingKey { get; set; }
        public string Title { get; set; }
        public string BloggerName { get; set;}
        public virtual ICollection<Post> Posts { get; set; }
    }
 ```
 
 ### Composite keys
 - Entity Framework supports composite keys - primary keys that consist of more than one property
 
 ```
 C#

Copy
    public class Passport
    {
        [Key]
        [Column(Order=1)]
        public int PassportNumber { get; set; }
        [Key]
        [Column(Order = 2)]
        public string IssuingCountry { get; set; }
        public DateTime Issued { get; set; }
        public DateTime Expires { get; set; }
    }
```
- If you have entities with composite foreign keys, then you must specify the same column ordering that 
  you used for the corresponding primary key properties.
  
```
    public class PassportStamp
    {
        [Key]
        public int StampId { get; set; }
        public DateTime Stamped { get; set; }
        public string StampingCountry { get; set; }

        [ForeignKey("Passport")]
        [Column(Order = 1)]
        public int PassportNumber { get; set; }

        [ForeignKey("Passport")]
        [Column(Order = 2)]
        public string IssuingCountry { get; set; }

        public Passport Passport { get; set; }
    }
 ```
 
 ### Required
 - The Required annotation tells EF that a particular property is required.
 - The Required attribute will also affect the generated database by making the mapped property non-nullable. 
   Notice that the Title field has changed to “not null”.
 
 ```
     [Required]
    public string Title { get; set; }
 ```
 
 ### MaxLength and MinLength
 - The MaxLength and MinLength attributes allow you to specify additional property validations, just as you did with Required.
 - The MaxLength annotation will impact the database by setting the property’s length to 10.
 
 ```
   [MaxLength(10, ErrorMessage="BloggerName must be 10 characters or less"),MinLength(5)]
    public string BloggerName { get; set; }
 ```
 
### NotMapped
- That property can be created dynamically and does not need to be stored.
 
 ```
     [NotMapped]
    public string BlogCode
    {
        get
        {
            return Title.Substring(0, 1) + ":" + BloggerName.Substring(0, 1);
        }
    }
 ```
 ### ComplexType
 - It’s not uncommon to describe your domain entities across a set of classes and then layer those classes 
   to describe a complete entity.
   
```

    [ComplexType]
    public class BlogDetails
    {
        public DateTime? DateCreated { get; set; }

        [MaxLength(250)]
        public string Description { get; set; }
    }
 ```
 
 - Notice that BlogDetails does not have any type of key property. 
 - In domain driven design, BlogDetails is referred to as a *value object*. Entity Framework refers to value objects 
   as *complex types*.  
  
  - Complex types cannot be tracked on their own.
  - However as a property in the Blog class, BlogDetails it will be tracked as part of a Blog object. 
  ```
          public BlogDetails BlogDetail { get; set; }
```

- In the database, the Blog table will contain all of the properties of the blog including the properties contained 
  in its BlogDetail property. By default, each one is preceded with the name of the complex type, BlogDetail.
  
 ### ConcurrencyCheck
 
- The ConcurrencyCheck annotation allows you to flag one or more properties to be used for concurrency checking in 
  the database when a user edits or deletes an entity. 
 - In EF Designer, this aligns with setting a property's ConcurrencyMode to Fixed.
 
 ```
     [ConcurrencyCheck, MaxLength(10, ErrorMessage="BloggerName must be 10 characters or less"),MinLength(5)]
    public string BloggerName { get; set; }
    
  
  -- The *SaveChanges* will success only the following conditions are made. 
   ```
       where (([PrimaryTrackingKey] = @4) and ([BloggerName] = @5))
       @4=1,@5=N'Julie'
 ```
 
### TimeStamp
 - It's more common to use rowversion or timestamp fields for concurrency checking. 
 - But rather than using the ConcurrencyCheck annotation, you can use the more specific TimeStamp annotation as long as 
   the type of the property is byte array. 
 - Code first will treat Timestamp properties the same as ConcurrencyCheck properties, but it will also ensure that the 
   database field that code first generates is non-nullable. 
- You can only have one timestamp property in a given class.

```
    [Timestamp]
    public Byte[] TimeStamp { get; set; }
```

### Table and Column

- My class is named Blog and by convention, code first presumes this will map to a table named Blogs. 
  If that's not the case you can specify the name of the table with the Table attribute

```
    [Table("InternalBlogs")]
    public class Blog
 ```
 
 - The Column annotation is a more adept in specifying the attributes of a mapped column. You can 
   stipulate a name, data type or even the order in which a column appears in the table.
 - Don’t confuse Column’s TypeName attribute with the DataType DataAnnotation. 
   DataType is an annotation used for the UI and is ignored by Code First.
```
    [Column("BlogDescription", TypeName="ntext")]
    public String Description {get;set;}
 ```
 
 ### DatabaseGenerated
 
- An important database features is the ability to have computed properties. 
 If you're mapping your Code First classes to tables that contain computed columns, 
 you don't want Entity Framework to try to update those columns. 
 But you do want EF to return those values from the database after you've inserted or updated data.
 
```
    [DatabaseGenerated(DatabaseGeneratedOption.Computed)]
    public DateTime DateCreated { get; set; }
```
- You can use database generated on byte or timestamp columns when code first is generating the database, 
  otherwise you should only use this when pointing to existing databases because code first won't be able 
   to determine the formula for the computed column.
  
 - You read above that by default, a key property that is an integer will become an identity key in the database. 
   That would be the same as setting DatabaseGenerated to DatabaseGeneratedOption.Identity. 
   If you do not want it to be an identity key, you can set the value to DatabaseGeneratedOption.None.
   
### Index

- The Index attribute was introduced in Entity Framework 6.1.
- You can create an index on one or more columns using the IndexAttribute.

```
    public class Post
    {
        public int Id { get; set; }
        public string Title { get; set; }
        public string Content { get; set; }
        [Index]
        public int Rating { get; set; }
        public int BlogId { get; set; }
    }
```
- By default, the index will be named IX_<property name>. You can also specify a name for the index though.
```
    [Index("PostRatingIndex")]
    public int Rating { get; set; }
```
- By default, indexes are non-unique, but you can use the IsUnique named parameter to specify that an index should be unique.
```
    public class User
    {
        public int UserId { get; set; }

        [Index(IsUnique = true)]
        [StringLength(200)]
        public string Username { get; set; }

        public string DisplayName { get; set; }
    }
```
### Multiple-Column Indexes

- Indexes that span multiple columns are specified by using the same name in multiple Index annotations for a given table. 
  When you create multi-column indexes, you need to specify an order for the columns in the index.
  
  ```
    public class Post
    {
        public int Id { get; set; }
        public string Title { get; set; }
        public string Content { get; set; }
        [Index("IX_BlogIdAndRating", 2)]
        public int Rating { get; set; }
        [Index("IX_BlogIdAndRating", 1)]
        public int BlogId { get; set; }
    }
```

### Relationship Attributes: InverseProperty and ForeignKey

- Code first convention will take care of the most common relationships in your model, 
  but there are some cases where it needs help.
  
- When generating the database, code first sees the BlogId property in the Post class and recognizes it, 
  by the convention that it matches a class name plus “Id”, as a foreign key to the Blog class. 
  But there is no BlogId property in the blog class.
 
- The solution for this is to create a navigation property in the Post and use the Foreign DataAnnotation 
  to help code first understand how to build the relationship between the two classes —using the Post.BlogId property — 
  as well as how to specify constraints in the database.
  
 ```
     public class Post
    {
            public int Id { get; set; }
            public string Title { get; set; }
            public DateTime DateCreated { get; set; }
            public string Content { get; set; }
            public int BlogId { get; set; }
            [ForeignKey("BlogId")]
            public Blog Blog { get; set; }
            public ICollection<Comment> Comments { get; set; }
    }
```

- The InverseProperty is used when you have multiple relationships between classes.

- In the Post class, you may want to keep track of who wrote a blog post as well as who edited it.
```
    public Person CreatedBy { get; set; }
    public Person UpdatedBy { get; set; }
```

- The Person class has navigation properties back to the Post, one for all of the posts written by the person 
  and one for all of the posts updated by that person.
```
    public class Person
    {
            public int Id { get; set; }
            public string Name { get; set; }
            public List<Post> PostsWritten { get; set; }
            public List<Post> PostsUpdated { get; set; }
    }
```
- Code first is not able to match up the properties in the two classes on its own. The database table for Posts 
  should have one foreign key for the CreatedBy person and one for the UpdatedBy person but code first will create 
  four foreign key properties: Person_Id, Person_Id1, CreatedBy_Id and UpdatedBy_Id.
  
- To fix these problems, you can use the InverseProperty annotation to specify the alignment of the properties.
```
    [InverseProperty("CreatedBy")]
    public List<Post> PostsWritten { get; set; }

    [InverseProperty("UpdatedBy")]
    public List<Post> PostsUpdated { get; set; }
```

- Because the PostsWritten property in Person knows that this refers to the Post type, it will build the relationship to 
  Post.CreatedBy. Similarly, PostsUpdated will be connected to Post.UpdatedBy. And code first will not create the extra 
  foreign keys.
  
  ### Summary
  
  - DataAnnotations not only let you describe client and server side validation in your code first classes, 
    but they also allow you to enhance and even correct the assumptions that code first will make about your classes 
    based on its conventions. 
 - With DataAnnotations you can not only drive database schema generation, but you can also map your code first classes 
   to a pre-existing database.


  

  
  












 
 
 
 
 
 

 
 
 
 
 


   
 
 - 




