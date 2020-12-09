# Foundation Services (Broker-Neighboring)

## 0. Introduction
Foundation services are the first point of contact between your business logic and the brokers.

In general, the broker-neighboring services are a hybrid of business logic and an abstraction layer for the processing operations where the higher-order business logic happens, which we will talk about further when we start exploring the processing services in the next section.

Broker-neighboring services main responsibility is to ensure the incoming and outgoing data through the system is validated and vetted structurally, logically and externally.

You can also think of broker-neighboring services as a layer of validation on top of the primitive operations the brokers already offer.

For instnace, if a storage broker is offering `InsertStudentAsync(Student student)` as a method, then the broker-neighboring service will offer something as follows:

```csharp
public async ValueTask<Student> AddStudentAsync(Student student)
{
	ValidateStudent(student);

	return this.storageBroker.InsertStudentAsync(student);
}
```

This makes broker-neighboring services nothing more than an extra layer of validation on top of the existing primitive operations brokers already offer.

## 1. On The Map
The broker-neighboring services reside between your brokers and the rest of your application, on the left side higher-order business logic processing services, orchestration, coordination, aggregation or management servies may live, or just simply a controller, a UI component or any other data exposure technology.

<br />
<p align=center>
	<img src="https://user-images.githubusercontent.com/1453985/100716772-00eec800-336e-11eb-9064-8bfe2f8e3be2.png" />
</p>
<br/>

## 2. Characteristics
Foundation or Broker-Neighboring services in general have very specific charactristics that stricly governs their development and integration.

Foundation services in general focus more on validations than anything else - simply because that's their purpose, to ensure all incoming and outgoing data through the system is in a good state for the system to process it saftely without any issues.

Here's the charactristics and rules that govern broker-neighboring services:

### 2.0 Pure-Primitive
Broker-neighboring services are not allowed to combine multiple primitive operations to achieve a higher-order business logic operation.

For instance, broker-neighboring services cannot offer an *upsert* function, to combine a `Select` operations with an `Update` or `Insert` operations based on the outcome to ensure an entity exists and is update to date in any storage.

But they offer a validation and exception handling (and mapping) wrapper around the dependency calls, here's an example:

```csharp
public ValueTask<Student> AddStudentAsync(Student student) =>
TryCatch(async () => 
{
	ValidateStudent(student);

	return this.storageBroker.InsertStudentAsync(student);
});
```

In the above method, you can see `ValidateStudent` function call preceded by a `TryCatch` block.
The `TryCatch` block is what I call Exception Noise Cancellation pattern, which we will discuss soon in this very section.

But the validation function ensures each and every property in the incoming data is validated before passing it forward to the primitive broker operation, which is the `InsertStudentAsync` in this very instance.

### 2.1 Single Entity Integration
Services strongly ensure the single responsibility principle is implemented by not integrating with any other entity brokers except for the one that it supports.

This rule doesn't necessarily apply to support brokers like `DateTimeBroker` or `LoggingBroker` since they don't specifically target any particular business entity and they are almost generic across the entire system.

For instance, a `StudentService` may integrate with a `StorageBroker` as long as it only targets only the functionality offered by the partial class in the `StorageBroker.Students.cs` file.

Foundation services should not integrate with more than one entity broker of any kind simply because it will increase the complexity of validation and orchestration which goes beyond the main purpose of the service which is just simply validation. we push this responsibility further to the orchestration-type services to handle it.

### 2.2 Business Language
Broker-neighboring services speak primitive business language for their operations.
For instance, while a Broker may provide a method with the name `InsertStudentAsync` - the equivelant of that on the service layer would be `AddStudentAsync`.

In general, most of the CRUD operations shall be converted from a storage lanaugage to a business language, and the same goes for non-storage operations such as Queues, for instance we say `PostQueueMessage` but on the business layer we shall say `EnqueueMessage`.

Since the CRUD operations the most common ones in every system, our mapping to these CRUD operations would be as follows:

| Brokers  	|   Services 	|
|----------	|:-------------:|
| Insert	|  	Add 		|
| Select 	|   Retrieve   	|
| Update 	|	Modify 		|
| Delete 	| 	Remove 		|

As we move forward towards higher-order business logic services, the language of the methods beings used will lean more towards a business language rather than a technology language as we will see in the upcoming sections.

## 3. Responsibilities
Broker-neighboring services play two very important roles in any system.
The first and most important role is to offer a layer of validation on top of the existing primitive operations a broker already offers to ensure incoming and outgoing data is valid to be processed or persisted by the system.
The second role is to play the role of a mapper of all other native models and contracts that may be needed to completed any given operation while interfacing with a broker.
Foundation services are the last point of abstraction between the core business logic of any system and the rest of the world, let's discuss these roles in detail.

### 3.0 Validation
Broker-neighboring services are required to ensure incoming and outgoing data from and to the system are in a good state - they play the role of a gatekeeper between the system and the outside world from a communication, data persistance and all other dependency needs a given system may have.

#### 3.0.0 Structural Validations
Validations are three different layers. the first of these layers is the structural validations. to ensure certain properties on any given model or a primitive type are not in an invalid structual state.

For instance, a property of type `String` should not be empty, `null` or white space. another example would be for an input parameter of an `int` type, it should not be at it's `default` state which is `0`.

The structural validations ensure the data is in a good shape before moving forward with any further validations - for instance, we can't possible validate a student has the minimum number of characters in their names if their first name is structurally invalid.

Structural validations play the role of identifying the *required* properties on any given model, and while a lot of technologies offer the validation annotations, plugins or libraries to globally enforce data validation rules, I choose to perform the validation programmatically and manually to gain more control of what would be required and what wouldn't in a TDD fashion.
