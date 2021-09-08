[![Build status](https://ci.appveyor.com/api/projects/status/4xq5y34wxn9ygd10?svg=true)](https://ci.appveyor.com/project/kasperdaff/datacontextscope)

DataContextScope
==============
NOTE: THIS IS A FORK OF [DbContextScope](https://github.com/mehdime/DbContextScope) attempting to bring its goodness to Linq to SQL. Certain advanced features has not been ported yet.

A simple and flexible way to manage your Linq to SQL DataContext instances.

`DataContextScope` was created out of the need for a better way to manage DataContext instances in Linq to SQL-based applications. 

The commonly advocated method of injecting DataContext instances works fine for single-threaded web applications where each web request implements exactly one business transaction. But it breaks down quite badly when console apps, Windows Services, parallelism and requests that need to implement multiple independent business transactions make their appearance.

The alternative of manually instantiating DataContext instances and manually passing them around as method parameters is (speaking from experience) more than cumbersome. 

`DataContextScope` implements the ambient context pattern for DataContext instances. It's something that NHibernate users or anyone who has used the `TransactionScope` class to manage ambient database transactions will be familiar with.

It doesn't force any particular design pattern or application architecture to be used. It works beautifully with dependency injection. And it works beautifully without.

# Using DataContextScope

I would highly recommend reading the following blog post first. It examines in great details the most commonly used approaches to manage DataContext instances and explains how `DataContextScope` addresses their shortcomings and simplifies DataContext management: [Managing DataContext the right way with Entity Framework 6: an in-depth guide](http://mehdi.me/ambient-dbcontext-in-ef6/). 

# Overview

This is the `DataContextScope` interface:

```csharp
public interface IDataContextScope : IDisposable
{
    void SaveChanges();
    
    void RefreshEntitiesInParentScope(IEnumerable entities);
    
    IDataContextCollection DataContexts { get; }
}
```

The purpose of a `DataContextScope` is to create and manage the `DataContext` instances used within a code block. A `DataContextScope` therefore effectively defines the boundary of a business transaction. 

Wondering why DataContextScope wasn't called "UnitOfWork" or "UnitOfWorkScope"? The answer is here: [Why DataContextScope and not UnitOfWork?](http://mehdi.me/ambient-dbcontext-in-ef6/#whydbcontextscopeandnotunitofwork)

You can instantiate a `DataContextScope` directly. Or you can take a dependency on `IDataContextScopeFactory`, which provides convenience methods to create a `DataContextScope` with the most common configurations:

```csharp
public interface IDataContextScopeFactory
{
    IDataContextScope Create(DataContextScopeOption joiningOption = DataContextScopeOption.JoinExisting);
    IDataContextReadOnlyScope CreateReadOnly(DataContextScopeOption joiningOption = DataContextScopeOption.JoinExisting);

    IDataContextScope CreateWithTransaction(IsolationLevel isolationLevel);
    IDataContextReadOnlyScope CreateReadOnlyWithTransaction(IsolationLevel isolationLevel);

    IDisposable SuppressAmbientContext();
}
```

# Typical usage
With `DataContextScope`, your typical service method would look like this:

```language-csharp
public void MarkUserAsPremium(Guid userId)
{
    using (var dataContextScope = _dataContextScopeFactory.Create())
    {
        var user = _userRepository.Get(userId);
        user.IsPremiumUser = true;
        dataContextScope.SaveChanges();
    }
}
```

Within a `DataContextScope`, you can access the `DataContext` instances that the scope manages in two ways. You can get them via the `DataContextScope.DataContexts` property like this:

```csharp
public void SomeServiceMethod(Guid userId)
{
    using (var dataContextScope = _dataContextScopeFactory.Create())
    {
        var user = dataContextScope.DataContexts.Get<MyDataContext>.GetTable<User>.Find(userId);
        [...]
        dataContextScope.SaveChanges();
    }
}
```

But that's of course only available in the method that created the `DataContextScope`. If you need to access the ambient `DataContext` instances anywhere else (e.g. in a repository class), you can just take a dependency on `IAmbientDataContextLocator`, which you would use like this:

```csharp
public class UserRepository : IUserRepository
{
    private readonly IAmbientDataContextLocator _contextLocator;

    public UserRepository(IAmbientDataContextLocator contextLocator)
    {
        if (contextLocator == null) throw new ArgumentNullException("contextLocator");
        _contextLocator = contextLocator;
    }

    public User Get(Guid userId)
    {
        return _contextLocator.Get<MyDataContext>.GetTable<User>().Find(userId);
    }
}
```

Those `DataContext` instances are created lazily and the `DataContextScope` keeps track of them to ensure that only one instance of any given DataContext type is ever created within its scope. 

You'll note that the service method doesn't need to know which type of `DataContext` will be required during the course of the business transaction. It only needs to create a `DataContextScope` and any component that needs to access the database within that scope will request the type of `DataContext` they need. 

# Nesting scopes
A `DataContextScope` can of course be nested. Let's say that you already have a service method that can mark a user as a premium user like this:

```csharp
public void MarkUserAsPremium(Guid userId)
{
    using (var dataContextScope = _dataContextScopeFactory.Create())
    {
        var user = _userRepository.Get(userId);
        user.IsPremiumUser = true;
        dataContextScope.SaveChanges();
    }
}
```

You're implementing a new feature that requires being able to mark a group of users as premium within a single business transaction. You can easily do it like this:

```csharp
public void MarkGroupOfUsersAsPremium(IEnumerable<Guid> userIds)
{
    using (var dataContextScope = _dataContextScopeFactory.Create())
    {
        foreach (var userId in userIds)
        {
        	// The child scope created by MarkUserAsPremium() will
            // join our scope. So it will re-use our DataContext instance(s)
            // and the call to SaveChanges() made in the child scope will
            // have no effect.
        	MarkUserAsPremium(userId);
        }
        
        // Changes will only be saved here, in the top-level scope,
        // ensuring that all the changes are either committed or
        // rolled-back atomically.
        dataContextScope.SaveChanges();
    }
}
```

(this would of course be a very inefficient way to implement this particular feature but it demonstrates the point)

This makes creating a service method that combines the logic of multiple other service methods trivial. 

# Read-only scopes
If a service method is read-only, having to call `SaveChanges()` on its `DataContextScope` before returning can be a pain. But not calling it isn't an option either as: 

1. It will make code review and maintenance difficult (did you intend not to call `SaveChanges()` or did you forget to call it?) 
2. If you requested an explicit database transaction to be started (we'll see later how to do it), not calling `SaveChanges()` will result in the transaction being rolled back. Database monitoring systems will usually interpret transaction rollbacks as an indication of an application error. Having spurious rollbacks is not a good idea.

The `DataContextReadOnlyScope` class addresses this issue. This is its interface:

```csharp
public interface IDataContextReadOnlyScope : IDisposable
{
    IDataContextCollection DataContexts { get; }
}
```

And this is how you use it:

```csharp
public int NumberPremiumUsers()
{
    using (_dataContextScopeFactory.CreateReadOnly())
    {
        return _userRepository.GetNumberOfPremiumUsers();
    }
}
```

In the example above, the `OrderRepository.GetOrdersForUserAsync()` method will be able to see and access the ambient DataContext instance despite the fact that it's being called in a separate thread than the one where the `DataContextScope` was initially created.

This is made possible by the fact that `DataContextScope` stores itself in the CallContext. The CallContext automatically flows through async points. If you're curious about how it all works behind the scenes, Stephen Toub has written [an excellent blog post about it](http://blogs.msdn.com/b/pfxteam/archive/2012/06/15/executioncontext-vs-synchronizationcontext.aspx). But if all you want to do is use `DataContextScope`, you just have to know that: it just works.

**WARNING**: There is one thing that you *must* always keep in mind when using any async flow with `DataContextScope`. Just like `TransactionScope`, `DataContextScope` only supports being used within a single logical flow of execution. 

I.e. if you attempt to start multiple parallel tasks within the context of a `DataContextScope` (e.g. by creating multiple threads or multiple TPL `Task`), you will get into big trouble. This is because the ambient `DataContextScope` will flow through all the threads your parallel tasks are using. If code in these threads need to use the database, they will all use the same ambient `DataContext` instance, resulting the same the `DataContext` instance being used from multiple threads simultaneously. 

In general, parallelizing database access within a single business transaction has little to no benefits and only adds significant complexity. Any parallel operation performed within the context of a business transaction should not access the database.

However, if you really need to start a parallel task within a `DataContextScope` (e.g. to perform some out-of-band background processing independently from the outcome of the business transaction), then you **must** suppress the ambient context before starting the parallel task. Which you can easily do like this:

```csharp
public void RandomServiceMethod()
{
    using (var dataContextScope = _dataContextScopeFactory.Create())
    {
        // Do some work that uses the ambient context
        [...]
        
        using (_dataContextScopeFactory.SuppressAmbientContext())
        {
            // Kick off parallel tasks that shouldn't be using the
            // ambient context here. E.g. create new threads,
            // enqueue work items on the ThreadPool or create 
            // TPL Tasks. 
            [...]
        }

		// The ambient context is available again here.
        // Can keep doing more work as usual.
        [...]

        dataContextScope.SaveChanges();
    }
}
```

# Creating a non-nested DataContextScope
This is an advanced feature that I would expect most applications to never need. Tread carefully when using this as it can create tricky issues and quickly lead to a maintenance nightmare. 

Sometimes, a service method may need to persist its changes to the underlying database regardless of the outcome of overall business transaction it may be part of. This would be the case if:

- It needs to record cross-cutting concern information that shouldn't be rolled-back even if the business transaction fails. A typical example would be logging or auditing records.
- It needs to record the result of an operation that cannot be rolled back. A typical example would be service methods that interact with non-transactional remote services or APIs. E.g. if your service method uses the Facebook API to post a new status update on Facebook and then records the newly created status update in the local database, that record must be persisted even if the overall business transaction fails because of some other error occurring after the Facebook API call. The Facebook API isn't transactional - it's impossible to "rollback" a Facebook API call. The result of that API call should therefore never be rolled back. 

In that case, you can pass a value of `DataContextScopeOption.ForceCreateNew` as the `joiningOption` parameter when creating a new `DataContextScope`. This will create a `DataContextScope` that will not join the ambient scope even if one exists:

```csharp
public void RandomServiceMethod()
{
    using (var dataContextScope = _dataContextScopeFactory.Create(DataContextScopeOption.ForceCreateNew))
    {
        // We've created a new scope. Even if that service method
        // was called by another service method that has created its 
        // own DataContextScope, we won't be joining it. 
        // Our scope will create new DataContext instances and won't
        // re-use the DataContext instances that the parent scope uses.
        [...]
        
		// Since we've forced the creation of a new scope,
        // this call to SaveChanges() will persist
        // our changes regardless of whether or not the
        // parent scope (if any) saves its changes or rolls back.
        dataContextScope.SaveChanges();
    }
}
```

The major issue with doing this is that this service method will use separate `DataContext` instances than the ones used in the rest of that business transaction. Here are a few basic rules to always follow in that case in order to avoid weird bugs and maintenance nightmares:

## 1. Persistent entity returned by a service method must always be attached to the ambient context

If you force the creation of a new `DataContextScope` (and therefore of new `DataContext` instances) instead of joining the ambient one, your service method must **never** return persistent entities that were created / retrieved within that new scope. This would be completely unexpected and will lead to humongous complexity.

The client code calling your service method may be a service method itself that created its own `DataContextScope` and therefore expects all service methods it calls to use that same ambient scope (this is the whole point of using an ambient context). It will therefore expect any persistent entity returned by your service method to be attached to the ambient `DataContext`.

Instead, either:

- Don't return persistent entities. This is the easiest, cleanest, most foolproof method. E.g. if your service creates a new domain model object, don't return it. Return its ID instead and let the client load the entity in its own `DataContext` instance if it needs the actual object.
- If you absolutely need to return a persistent entity, switch back to the ambient context, load the entity you want to return in the ambient context and return that.

## 2. Upon exit, a service method must make sure that all modifications it made to persistent entities have been replicated in the parent scope

If your service method forces the creation of a new `DataContextScope` and then modifies persistent entities in that new scope, it must make sure that the parent ambient scope (if any) can "see" those modification when it returns. 

I.e. if the `DataContext` instances in the parent scope had already loaded the entities you modified in their first-level cache (ObjectStateManager), your service method must force a refresh of these entities to ensure that the parent scope doesn't end up working with stale versions of these objects.

The `DataContextScope` class has a handy helper method that makes this fairly painless:

```csharp
public void RandomServiceMethod(Guid accountId)
{
	// Forcing the creation of a new scope (i.e. we'll be using our 
	// own DataContext instances)
    using (var dataContextScope = _dataContextScopeFactory.Create(DataContextScopeOption.ForceCreateNew))
    {
        var account = _accountRepository.Get(accountId);
        account.Disabled = true;
        
        // Since we forced the creation of a new scope,
        // this will persist our changes to the database
        // regardless of what the parent scope does.
        dataContextScope.SaveChanges();

        // If the caller of this method had already
        // loaded that account object into their own
        // DataContext instance, their version
        // has now become stale. They won't see that
        // this account has been disabled and might
        // therefore execute incorrect logic.
        // So make sure that the version our caller
        // has is up-to-date.
        dataContextScope.RefreshEntitiesInParentScope(new[] { account });
    }
}
```
