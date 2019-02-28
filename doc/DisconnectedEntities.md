# Working with disconnected entities

- In an Entity Framework-based application, a context class is responsible for detecting changes applied to tracked entities. 
  Calling the SaveChanges method persists the changes tracked by the context to the database. 
  
-  When working with n-tier applications, entity objects are usually modified while disconnected from the context, and you must 
   decide how to track changes and report those changes back to the context.
   
### Web service frameworks

- Web services technologies typically support patterns that can be used to persist changes on individual disconnected objects. 
- For example, ASP.NET Web API allows you to code controller actions that can include calls to EF to persist changes made to 
  an object on a database. 
- In fact, the Web API tooling in Visual Studio makes it easy to scaffold a Web API controller from your Entity Framework 6 model.

### Low-level EF APIs

- If you don't want to use an existing n-tier solution, or if you want to customize what happens inside a controller action 
  in a Web API services, Entity Framework provides APIs that allow you to apply changes made on a disconnected tier.
  
### Self-Tracking Entities

- Tracking changes on arbitrary graphs of entities while disconnected from the EF context is a hard problem. 
  One of the attempts to solve it was the Self-Tracking Entities code generation template. 
  This template generates entity classes that contain logic to track changes made on a disconnected tier as state in 
  the entities themselves. A set of extension methods is also generated to apply those changes to a context.
  
- This template can be used with models created using the EF Designer, but can not be used with Code First models.

- Microsoft no longer recommend using the self-tracking-entities template

- Self-Tracking Entities (STEs) can help you track changes in any tier and then replay these changes into a context to be saved.

- Use STEs only if the context is not available on a tier where the changes to the object graph are made. 
  If the context is available, there is no need to use STEs because the context will take care of tracking changes.
  
- This template item generates two .tt (text template) files:

    - The <model name>.tt file generates the entity types and a helper class that contains the change-tracking logic 
      that is used by self-tracking entities and the extension methods that allow setting state on self-tracking entities.
      
    - The <model name>.Context.tt file generates a derived context and an extension class that contains ApplyChanges 
      methods for the ObjectContext and ObjectSet classes. These methods examine the change-tracking information that 
      is contained in the graph of self-tracking entities to infer the set of operations that must be performed to save 
      the changes in the database.
   
##### Functional Considerations When Working with Self-Tracking Entities

- Make sure that your client project has a reference to the assembly containing the entity types.
  If you add only the service reference to the client project, the client project will use the WCF proxy types and not 
  the actual self-tracking entity types. This means that you will not get the automated notification features that manage 
  the tracking of the entities on the client.
  
- Calls to the service operation should be stateless and create a new instance of object context. We also recommend that 
  you create object context in a using block.
  
- When you send the graph that was modified on the client to the service and then intend to continue working with the 
  same graph on the client, you have to manually iterate through the graph and call the AcceptChanges method on each 
  object to reset the change tracker.
  
> If objects in your graph contain properties with database-generated values (for example, identity or concurrency values), 
> Entity Framework will replace values of these properties with the database-generated values after the SaveChanges method 
> is called. You can implement your service operation to return saved objects or a list of generated property values for the 
> objects back to the client. The client would then need to replace the object instances or object property values with the 
> objects or property values returned from the service operation.

- Merging graphs from multiple service requests may introduce objects with duplicate key values in the resulting graph. 

- When you change the relationship between objects by setting the foreign key property, the reference navigation property is 
  set to null and not synchronized to the appropriate principal entity on the client. After the graph is attached to the object 
  context (for example, after you call the ApplyChanges method), the foreign key properties and navigation properties are 
  synchronized.
  
> Not having a reference navigation property synchronized with the appropriate principal object could be an issue if you have 
> specified cascade delete on the foreign key relationship. If you delete the principal, the delete will not be propagated to 
> the dependent objects. If you have cascade deletes specified, use navigation properties to change relationships instead of 
> setting the foreign key property.

- Self-tracking entities are not enabled to perform lazy loading.

- Binary serialization and serialization to ASP.NET state management objects is not supported by self-tracking entities.

##### Security Considerations

- A service should not trust requests to retrieve or update data from a non-trusted client or through a non-trusted channel. 
  A client must be authenticated: a secure channel or message envelope should be used. Clients' requests to update or retrieve 
  data must be validated to ensure they conform to expected and legitimate changes for the given scenario.
  
- Avoid using sensitive information as entity keys (for example, social security numbers). This mitigates the possibility 
  of inadvertently serializing sensitive information in the self-tracking entity graphs to a client that is not fully trusted. 
  With independent associations, the original key of an entity that is related to the one that is being serialized might be 
  sent to the client as well.
  
- To avoid propagating exception messages that contain sensitive data to the client tier, calls to ApplyChanges and SaveChanges 
  on the server tier should be wrapped in exception-handling code.
  
  

  - 

