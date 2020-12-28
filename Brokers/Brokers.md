# Brokers

## 0. Introduction
Brokers play the role of a liaison between the business logic and the outside world.
They are wrappers around any external libraries, resources or APIs to satisfy a local interface for the business to interact with these resources without having to be tightly coupled with any particular resources or external library implementation.

Brokers in general are meant to be disposable and replaceable - they are built with the understanding that technology evolves and changes all the time and therefore they shall be at some point in time in the lifecycle of a given application be replaced with a modern technology that gets the job done faster.

But Brokers also ensure that your business is pluggable by abstracting away any specific external resource dependencies from what your software is actually trying to accomplish.


## 1. On The Map
In any given application, mobile, desktop, web or simply just an API - brokers usually reside at the "tail" of any app - that's because they are the last point of contact between our custom code and the outside world.

Whether the outside world in this instance is just simply a local storage in memory, or an entirely independent system that resides behind an API, they all have to reside behind the Brokers in any application.

In the following low-level architecture for a given API - Brokers reside between our business logic and the external resource:

<br />
<p align=center>
    <img src="https://user-images.githubusercontent.com/1453985/100530268-c5ed6880-31a4-11eb-8fc7-129a5a7dda54.png" />
</p>
<br />

## 2. Characteristics
There are few simple rules that govern the implementation of any broker - these rules are:

### 2.0 Implements a Local Interface
Brokers have to satisfy a local contract. they have to implement a local interface to allow the decoupling between their implementation and the services that consume them.

For instance, given that we have a local contract `IStorageBroker` that requires an implementation for any given CRUD operation for a local model `Student` - the contract operation would be as follows:

```csharp
    public partial interface IStorageBroker
    {
        public ValueTask<Student> InsertStudentAsync(Student student);
    }
```

An implementation for a storage broker would be as follows:

```csharp
    public partial class StorageBroker
    {
        public DbSet<Student> Students { get; set; }

        public IQueryable<Student> SelectAllStudents() => this.Students.AsQueryable();
    }
```
A local contract implementation can be replaced at any point in time from utilizing the Entity Framework as shows in the previous example, to using a completely different technology like Dapper, or an entirely different infrastructure like an Oracle or Postgres database.

### 2.1 No Flow Control
Brokers should not have any form of flow-control such as if-statements, while-loops or switch cases - that's simply because flow-control code is considered to be business logic, and it fits better the services layer where business logic should reside not the brokers.

For instance, a broker method that retrieves a list of students from a database would look something like this:

```csharp
    public IQueryable<Student> SelectAllStudents() => this.Students.AsQueryable();
```
A simple fat-arrow function that calls the native EntityFramework `DbSet<T>` and return a local model like `Student`. 


### 2.2 No Exception Handling
Exception handling is somewhat a form of flow-control. Brokers are not supposed to handle any exceptions, but rather let the exception propagate to the broker-neighboring services where these exceptions are going to be properly mapped and localized.


### 2.3 Own Their Configurations
Brokers are also required to handle their own configurations - they may have a dependency injection from a configuration object, to retrieve and setup the configurations for whichever external resource they are integrating with.

For instance, connection strings in database communications are required to be retrieved and passed in to the database client to establish a successful connection, as follows:

```csharp
    public partial class StorageBroker : EFxceptionsContext, IStorageBroker
    {
        private readonly IConfiguration configuration;

        public StorageBroker(IConfiguration configuration)
        {
            this.configuration = configuration;
            this.Database.Migrate();
        }

        protected override void OnConfiguring(DbContextOptionsBuilder optionsBuilder)
        {
            string connectionString = this.configuration.GetConnectionString("DefaultConnection");
            optionsBuilder.UseSqlServer(connectionString);
        }
    }
```

### 2.4 Natives from Primitives
Brokers may construct an external model object based on primitive types passed from the broker-neighboring services.

For instance, in e-mail notifications broker, input parameters for a `.Send(...)` function for instance require the basic input parameters such as the subject, content or the address for instance, here's an example:

```csharp
    public async ValueTask SendMailAsync(List<string> recipients, string subject, string content)
    {
        Message message = BuildMessage(recipients, ccRecipients, subject, content);
        await SendEmailMessageAsync(message);
    }
```

The primitive input parameters will ensure there are no strong dependencies between the broker-neighboring services and the external models.
Even in situations where the broker is simply a point of integration between your application and an external RESTful API, it's very highly recommended that you build your own native models to reflect the same JSON object sent or returned from the API instead of relying on nuget libraries, dlls or shared projects to achieve the same goal.

### 2.5 Naming Conventions
The contracts for the brokers shall remain as generic as possible to indicate the overall functionality of a broker, for instance we say `IStorageBroker` instead of `ISqlStorageBroker` to indicate a particular technology or infrastructure.

But in case of concrete implementations of brokers, it all depends on how many brokers you have providing similar functionality, in case of having a single storage broker, it might be more convenient to to maintain the same name as the contract - in our case here a concrete implementation of `IStorageBroker` would be `StorageBroker`.

However, if your application supports multiple queues, storages or e-mail service providers you might need to start be specifying the overall target of the component, for instance, an `IQueueBroker` would have multiple implementations such as `NotificationQueueBroker` and `OrdersQueueBroker`.

But if the concrete implementations target the same model and business value, then a diversion to the technology might be more befitting in this case, for instance in the case of an `IStorageBroker` two different concrete implementations would be `SqlStorageBroker` and `MongoStroageBroker` this case is very possible in situations where environment costs are reduced in lower than production infrastructure for instance.

### 2.6 Language
Brokers speak the language of the technologies they support.
For instnace, in a storage broker, we say `SelectById` to match the SQL `Select` statement and in a queue broker we say `Enqueue` to match the language.

If a broker is supporting an API endpoint, then it shall follow the RESTFul operations language, such as `POST`, `GET` or `PUT`, here's an example:

```csharp

    public async ValueTask<Student> PostStudentAsync(Student student) =>
        await this.PostAsync(RelativeUrl, student);

```

### 2.7 Up & Sideways
Brokers cannot call other brokers. that's simply because brokers are the first point of abstraction, they require no additional abstractions and no additional dependencies other than a configuration access model.

Brokers can't also have services as dependencies as the flow in any given system shall come from the services to the brokers and not the other way around.

Even in situations where a microservice has to subscribe to a queue for instance, brokers will pass forward a listener method to process incoming events, but not call the services that provide the processing logic.

The general rule here then would be, that brokers can only be called by services, and they can only call external native dependencies.

## 3. Organization
Brokers that support multiple entities such as Storage brokers should leverage partial classes to break down the responsibilites per entities.

For instance, if we have a storage broker that provides all CRUD operations for both `Student` and `Teacher` models, then the organization of the files should be as follows:

- IStorageBroker.cs
  - IStorageBroker.Students.cs
  - IStorageBroker.Teachers.cs
- StorageBroker.cs
  - StorageBroker.Students.cs
  - StorageBroker.Teachers.cs

The main purpose of this particular organization leveraging partial classes is to seperate the concern for each entity to even a finer level, which should make the maintainability of the software much higher.

But brokers files and folders naming convention strictly focuses on the plurality of the entities they support and the singularity for the overall resource being supported.

For instance, we say `IStorageBroker.Students.cs`. and we also say `IEmailBroker` or `IQueueBroker.Notifications.cs` - singular for the resource and plural entities.

The same concept applies to the folders or namespaces containing these brokers.

For instance, we say:

```csharp
namespace OtripleS.Web.Api.Brokers.Storages
{
    ...
}
```

And we say:
```csharp
namespace OtripleS.Web.Api.Brokers.Queues
{
    ...
}
```

## 4. Broker Types
In most of the applications built today, there are some common Brokers that are usually needed to get an enterprise application up and running - some of these Brokers are like Storage, Time, APIs, Logging and Queues.

Some of these brokers interact with existing resources on the system such as time to allow broker-neighboring services to treat time as a dependency and control how a particular service would behave based on the value of time at any point in the past, present or the future.

### 4.0 Entity Brokers
Entity brokers are the brokers providing integration points with external resources that the system needs to fulfill a business requirements.

For instance, entity brokers are brokers that integrate with storage, providing capabilities to store or retrieve records from a database.

entity brokers are also like queue brokers, providing a point of integration to push messages to a queue for other services to consume and process to fulfill their business logic.

Entity brokers can only be called by broker-neighboring services, simply because they require a level of validation that needs to be performed on the data they receive or provide before proceeding any further.

### 4.1 Support Brokers
Support brokers are general purpose brokers, they provide a functionality to support services but they have no charactristic that makes them different from one system or another.

A good example of support brokers is the `DateTimeBroker` - a broker made specifically to abstract away the business layer strong dependency on the system date time.

Time brokers don't really target any specific entity, and they are almost the same across many systems out there.

Another example of support brokers is the `LoggingBroker` - they provide data to logging and monitoring systems to enable the system's engineers to visualize the overall flow of data across the system, and be notified in case any issues occur.

Unlike Entity Brokers - support brokers may be called across the entire business layer, they may be called on foundation, processing, orchestration, coordination, management or aggregation services. that's because logging brokers are required as a supporting component in the system to provide all the capabilities needed for services to log their errors or calculate a date or any other supporting functionality.

You can find real-world examples of brokers in the OtripleS project [here](https://github.com/hassanhabib/OtripleS/tree/master/OtripleS.Web.Api/Brokers).

## 5. Implementation
Here's a real-life implementation of a full storage broker for all CRUD operations for `Student` entity:

###### For IStorageBroker.cs:
```csharp
namespace OtripleS.Web.Api.Brokers.Storage
{
    public partial interface IStorageBroker
    {
    }
}

```

###### For StorageBroker.cs:
```csharp
using System;
using EFxceptions.Identity;
using Microsoft.EntityFrameworkCore;
using Microsoft.Extensions.Configuration;
using OtripleS.Web.Api.Models.Users;

namespace OtripleS.Web.Api.Brokers.Storage
{
    public partial class StorageBroker : EFxceptionsContext, IStorageBroker
    {
        private readonly IConfiguration configuration;

        public StorageBroker(IConfiguration configuration)
        {
            this.configuration = configuration;
            this.Database.Migrate();
        }

        protected override void OnConfiguring(DbContextOptionsBuilder optionsBuilder)
        {
            string connectionString = this.configuration.GetConnectionString("DefaultConnection");
            optionsBuilder.UseSqlServer(connectionString);
        }
    }
}
```

###### For IStorageBroker.Students.cs:
```csharp
using System;
using System.Linq;
using System.Threading.Tasks;
using OtripleS.Web.Api.Models.Students;

namespace OtripleS.Web.Api.Brokers.Storage
{
    public partial interface IStorageBroker
    {
        public ValueTask<Student> InsertStudentAsync(Student student);
        public IQueryable<Student> SelectAllStudents();
        public ValueTask<Student> SelectStudentByIdAsync(Guid studentId);
        public ValueTask<Student> UpdateStudentAsync(Student student);
        public ValueTask<Student> DeleteStudentAsync(Student student);
    }
}
``` 

###### For StorageBroker.Students.cs:
```csharp
using System;
using System.Linq;
using System.Threading.Tasks;
using Microsoft.EntityFrameworkCore;
using Microsoft.EntityFrameworkCore.ChangeTracking;
using OtripleS.Web.Api.Models.Students;

namespace OtripleS.Web.Api.Brokers.Storage
{
    public partial class StorageBroker
    {
        public DbSet<Student> Students { get; set; }

        public async ValueTask<Student> InsertStudentAsync(Student student)
        {
            EntityEntry<Student> studentEntityEntry = await this.Students.AddAsync(student);
            await this.SaveChangesAsync();

            return studentEntityEntry.Entity;
        }

        public IQueryable<Student> SelectAllStudents() => this.Students.AsQueryable();

        public async ValueTask<Student> SelectStudentByIdAsync(Guid studentId)
        {
            this.ChangeTracker.QueryTrackingBehavior = QueryTrackingBehavior.NoTracking;

            return await Students.FindAsync(studentId);
        }

        public async ValueTask<Student> UpdateStudentAsync(Student student)
        {
            EntityEntry<Student> studentEntityEntry = this.Students.Update(student);
            await this.SaveChangesAsync();

            return studentEntityEntry.Entity;
        }

        public async ValueTask<Student> DeleteStudentAsync(Student student)
        {
            EntityEntry<Student> studentEntityEntry = this.Students.Remove(student);
            await this.SaveChangesAsync();

            return studentEntityEntry.Entity;
        }
    }
}
```

## 6. Summary
Brokers are the first layer of abstraction between your business logic and the outside world, but they are not the only layer of abstraction. simply because there will still be few native models that leak through your brokers to your broker-neighboring services which is natural to avoid doing any mappings outside of the realm of logic, in our case here the foundation services.

For instnace, in a storage broker, regardless what ORM you are using, some native exceptions from your ORM (EntityFramework for instance) will occur, such as `DbUpdateException` or `SqlException` - in that case we need another layer of abstraction to play the role of a mapper between these exceptions and our core logic to convert them into local exception models. 

This responsibility lies in the hands of the broker-neighboring services, I also call them foundation services, these services are the last point of abstraction before your core logic, in which everything becomes nothing but local models and contracts.


## 7. FAQs
During the course of time, there have been some common questions that arised by the engineers that I had the opportunity to work with throughout my career - since some of these questions reoccurred on several occassions, I thought it might be useful to aggregate all of them in here for everyone to learn about some other perspectives around brokers.

#### 7.0 Is the brokers pattern the same as the repository pattern?
Not exactley, at least from an operational standpoint, brokers seems to be more generic than repositories.

Repositories usually target storage-like operations, mainly towards databases. but brokers can be an integration point with any external dependency such as e-mail services, queues, other APIs and such.

A more similar pattern for brokers is the Unit of Work pattern, it mainly focuses on the overall operation without having to tie the definition or the name with any particular operation.

All of these patterns in general try to achieve the same SOLID principles goal, which is the separation of concern, dependency injection and single responsibility.

But because SOLID are principles and not exact guidelines, it's expected to see all different kinds of implementations and patterns to achieve that principle.


#### 7.1 Why can't the brokers implement a contract for methods that return an interface instead of a concrete model?
That would be an ideal situaiton, but that would also require brokers to do a conversion or mapping between the native models returned from the external resource SDKs or APIs and the internal model that adheres to the local contract.

Doing that on the broker level will require pushing business logic into that realm, which is outside of the purpose of that component completely.

Brokers do not get unit tested because they have no business logic in them, they may be a part of an acceptance or an integration test, but certainly not a part of unit level tests - simply because they don't contain any business logic in them.

#### 7.2 If brokers were truely a layer of abstraction from the business logic, how come we allow external exceptions to leak through them onto the services layer?
Brokers are only the *the first* layer of abstraction, but not the only one - the broker neighboring services are responsible for converting the native exceptions occurring from a broker into a more local exception model that can be handled and processed internally within the business logic realm.

Full pure local code starts to occur on the processing, orchestration, coordination and aggregation layers where all the exceptions, all the returned models and all operations are localized to the system.

#### 7.3 Why do we use partial classes for brokers who handle multiple entities?
Since brokers are required to own their own configurations, it made more sense to partialize when possible to avoid reconfiguring every storage broker for each entity.

This is a feature in C# specifically as a language, but it should be possible to implement through inheritance in other programming languages.