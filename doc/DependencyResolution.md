# Dependency resolution

- Starting with EF6, Entity Framework contains a general-purpose mechanism for obtaining implementations of servi that it requires. 
- That is, when EF uses an instance of some interfaces or base classes it will ask for a concrete implementation of the interface 
  or base class to use. 
  
  - This is achieved through use of the IDbDependencyResolver interface:
  
```
  public interface IDbDependencyResolver
{
    object GetService(Type type, object key);
}
```
- The GetService method is typically called by EF and is handled by an implementation of IDbDependencyResolver provided 
  either by EF or by the application. 
- When called, the type argument is the interface or base class type of the service being requested, 
  and the key object is either null or an object providing contextual information about the requested service.
  
- Unless otherwise stated any object returned must be thread-safe since it can be used as a singleton. 
  In many cases the object returned is a factory in which case the factory itself must be thread-safe but the object 
  returned from the factory does not need to be thread-safe since a new instance is requested from the factory for each use.
  
  
