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

<div align=center>
    <img src="https://user-images.githubusercontent.com/1453985/100049682-c4025e80-2dcc-11eb-88f4-3c6087f7eb1e.png" width=70% />
</div>

## 2. Characteristics
There are few simple rules that govern the implementation of any broker - these rules are:

#### 2.0 Implements a Local Interface
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

#### 2.1 No Flow Control
Brokers should not have any form of flow-control such as if-statements, while-loops or switch cases - that's simply because flow-control code is considered to be business logic, and it fits better the services layer where business logic should reside not the brokers.

For instance, a broker method that retrieves a list of students from a database would look something like this:

```csharp
    public IQueryable<Student> SelectAllStudents() => this.Students.AsQueryable();
```
A simple fat-arrow function that calls the native EntityFramework `DbSet<T>` and return a local model like `Student`. 


#### 2.2 No Exception Handling
Exception handling is somewhat a form of flow-control. Brokers are not supposed to handle any exceptions, but rather let the exception propagate to the broker-neighboring services where these exceptions are going to be properly mapped and localized.


#### 2.3 Own Their Configurations
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

#### 2.4 Natives from Primitives
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

#### 2.5 Naming Conventions
The contracts for the brokers shall remain as generic as possible to indicate the overall functionality of a broker, for instance we say `IStorageBroker` instead of `ISqlStorageBroker` to indicate a particular technology or infrastructure.

But in case of concrete implementations of brokers, it all depends on how many brokers you have providing similar functionality, in case of having a single storage broker, it might be more convenient to to maintain the same name as the contract - in our case here a concrete implementation of `IStorageBroker` would be `StorageBroker`.

However, if your application supports multiple queues, storages or e-mail service providers you might need to start be specifying the overall target of the component, for instance, an `IQueueBroker` would have multiple implementations such as `NotificationQueueBroker` and `OrdersQueueBroker`.

But if the concrete implementations target the same model and business value, then a diversion to the technology might be more befitting in this case, for instance in the case of an `IStorageBroker` two different concrete implementations would be `SqlStorageBroker` and `MongoStroageBroker` this case is very possible in situations where environment costs are reduced in lower than production infrastructure for instance.

#### 2.6 Language
Brokers speak the language of the technologies they support.
For instnace, in a storage broker, we say `SelectById` to match the SQL `Select` statement and in a queue broker we say `Enqueue` to match the language.

If a broker is supporting an API endpoint, then it shall follow the RESTFul operations language, such as `POST`, `GET` or `PUT`, here's an example:

```csharp

    public async ValueTask<Student> PostStudentAsync(Student student) =>
        await this.PostAsync(RelativeUrl, student);

```

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

## 4. Common Brokers
In most of the applications built today, there are some common Brokers that are usually needed to get an enterprise application up and running - some of these Brokers are like Storage, Time, APIs, Logging and Queues.

Some of these brokers interact with existing resources on the system such as time to allow broker-neighboring services to treat time as a dependency and control how a particular service would behave based on the value of time at any point in the past, present or the future.

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