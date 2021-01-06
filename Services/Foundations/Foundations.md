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

	return await this.storageBroker.InsertStudentAsync(student);
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

	return await this.storageBroker.InsertStudentAsync(student);
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
Foundation services are required to ensure incoming and outgoing data from and to the system are in a good state - they play the role of a gatekeeper between the system and the outside world to ensure the data that goes through is structurally, logically and externally valid before performing any further operations by upstream services.

#### 3.0.0 Structural Validations
Validations are three different layers. the first of these layers is the structural validations. to ensure certain properties on any given model or a primitive type are not in an invalid structual state.

For instance, a property of type `String` should not be empty, `null` or white space. another example would be for an input parameter of an `int` type, it should not be at it's `default` state which is `0`.

The structural validations ensure the data is in a good shape before moving forward with any further validations - for instance, we can't possible validate a student has the minimum number of characters in their names if their first name is structurally invalid.

Structural validations play the role of identifying the *required* properties on any given model, and while a lot of technologies offer the validation annotations, plugins or libraries to globally enforce data validation rules, I choose to perform the validation programmatically and manually to gain more control of what would be required and what wouldn't in a TDD fashion.

##### 3.0.0.0 Testing Structural Validations
Because I truly believe in the importance of TDD, I am going to start showing the implementation of structural validations by writing a failing test for it first.

Let's assume we have a student model, with the following details:

```csharp
public class Student 
{
	public Guid Id {get; set;}
}
```

We want to validate that the student Id is not a structurally invalid Id - such as an empty `Guid` - therefore we would write a unit test in the following fashion:

```csharp
[Fact]
public async void ShouldThrowValidationExceptionOnRegisterWhenIdIsInvalidAndLogItAsync()
{
	// given
	Student randomStudent = CreateRandomStudent();
	Student inputStudent = randomStudent;
	inputStudent.Id = Guid.Empty;

	var invalidStudentException = new InvalidStudentException(
		parameterName: nameof(Student.Id),
		parameterValue: inputStudent.Id);

	var expectedStudentValidationException =
		new StudentValidationException(invalidStudentException);

	// when
	ValueTask<Student> registerStudentTask =
		this.studentService.RegisterStudentAsync(inputStudent);

	// then
	await Assert.ThrowsAsync<StudentValidationException>(() =>
		registerStudentTask.AsTask());

	this.loggingBrokerMock.Verify(broker =>
		broker.LogError(It.Is(SameExceptionAs(expectedStudentValidationException))),
			Times.Once);

	this.storageBrokerMock.Verify(broker =>
		broker.InsertStudentAsync(It.IsAny<Student>()),
			Times.Never);

	this.dateTimeBrokerMock.VerifyNoOtherCalls();
	this.loggingBrokerMock.VerifyNoOtherCalls();
	this.storageBrokerMock.VerifyNoOtherCalls();
}
```

In the above test, we created a random student object then assigned the an invalid Id value of `Guid.Empty` to the student `Id`.

When the structural validation logic in our foundation service examines the `Id` property, it should throw an exception property describing the issue of validation in our student model. in this case we throw `InvalidStudentException`.

The exception is required to briefly describe the whats, wheres and whys of the validation operation. in our case here the what would be the validation issue occurring, the where would be the Student service and the why would be the property value.

Here's how an `InvalidStudentException` would look like:

```csharp
public class InvalidStudentException : Exception
{
	public InvalidStudentException(string parameterName, object parameterValue)
		: base($"Invalid Student, " +
			$"ParameterName: {parameterName}, " +
			$"ParameterValue: {parameterValue}.")
	{ }
}
```

The above unit test however, requires our `InvalidStudentException` to be wrapped up in a more generic system-level exception, which is `StudentValidationException` - these exceptions is what I call outer-exceptions, they encapsulate all the different situations of validations regardless of their category and communicates the error to upstream services or controllers so they can map that to the proper error code to the consumer of these services.

Our `StudentValidationException` would be implemented as follows:

```csharp
public class StudentValidationException : Exception
{
	public StudentValidationException(Exception innerException)
		: base("Invalid input, please check your input and then try again.", innerException) { }
}
```

The message in the outer-validation above indicates that the issue is in the input, and therefore it requires the input submitter to try again as there are no actions required from the system side to be adjusted.

##### 3.0.0.1 Implementing Structural Validations
Now, let's look at the other side of the validation process, which is the implementation.
Structural validations always come before each and every other type of validations. That's simply because structural validations are the cheapest from an execution and asymptotic time perspective. 
For instance, It's much cheaper to validation an `Id` is invalid structurally, than sending an API call across to get the exact same answer plus the cost of latency. this all adds up when multi-million requests per second start flowing in.
Structural and logical validations in general live in their own partial class to run these validations, for instance, if our service is called `StudentService.cs` then a new file should be created with the name `StudentService.Validations.cs` to encapsulate and visually abstracts away the validation rules to ensure clean data are coming in and going out.
Here's how an Id validation would look like:

###### StudentService.Validations.cs

```csharp

private void ValidateStudent(Student student)
{
	switch(student)
	{
		case {} when IsInvalid(student.Id):
			throw new InvalidStudentException(
				parameterName: nameof(Student.Id),
				parameterValue: student.Id);
	}
}
	
private static bool IsInvalid(Guid id) => id == Guid.Empty;
```

We have implemented a method to validate the entire student object, with a compilation of all the rules we need to setup to validate structurally and logically the student input object. The most important part to notice about the above code snippet is to ensure the encapsulation of any finer details further away from the main goal of a particular method.

That's the reason why we decided to implement a private static method `IsInvalid` to abstract away the details of what determines a property of type `Guid` is invalid or not. And as we move further in the implementation, we are going to have to implement multiple overloads of the same method to validate other value types structurally and logically.

The purpose of the `ValidateStudent` method is to simply set up the rules and take an action by throwing an exception if any of these rules are violated. there's always an opportunity to aggregated the violation errors rather than throwing too early at the first sign of structural or logically validation issue to be detected.

Now, with the implementation above, we need to call that method to structurally and logically validate our input. let's make that call in our `RegisterStudentAsync` method as follows:

###### StudentService.cs
```csharp
public ValueTask<Student> RegisterStudentAsync(Student student) =>
TryCatch(async () =>
{
	ValidateStudent(student);

	return await this.storageBroker.InsertStudentAsync(student);
});
```

At a glance, you will notice that our method here doesn't necessarily handle any type of exceptions at the logic level. that's because all the exception noise is also abstracted away in a method called `TryCatch`.

`TryCatch` is a concept I invented to allow engineers to focus on what matters that most based on which aspect of the servie that are looking at without having to take any shortcuts with the exception handling to make the code a bit more readable.

`TryCatch` methods in general live in another partial class, and an entirely new file called `StudentService.Exceptions.cs` - which is where all exception handling and error reporting happens as I will show you in the following example.

Let's take a look at what a `TryCatch` method looks like:

###### StudentService.Exceptions.cs
```csharp
private delegate ValueTask<Student> ReturningStudentFunction();

private async ValueTask<Student> TryCatch(ReturningStudentFunction returningStudentFunction)
{
	try
	{
		return await returningStudentFunction();
	}
	catch (InvalidStudentException invalidStudentInputException)
	{
		throw CreateAndLogValidationException(invalidStudentInputException);
	}
}

private StudentValidationException CreateAndLogValidationException(Exception exception)
{
	var studentValidationException = new StudentValidationException(exception);
	this.loggingBroker.LogError(studentValidationException);

	return studentValidationException;
}
```

The `TryCatch` exception noise-cancellation pattern beautifully takes in any function that returns a particular type as a delegate and handles any thrown exceptions off of that function or it's dependencies.

The main responsibility of a `TryCatch` function is to wrap up a service inner exceptions with outer exceptions to ease-up the reaction of external consumers of that service into only one of the three categories, which are Service Excetpions, Validations Excetpions and Dependency Excetpions. there are sub-types to these excetpions such as Dependency Validation Excetpions but these usually fall under the Validation Excetpion category as we will discuss in upcoming sections of The Standard.

In a `TryCatch` method, we can add as many inner and external excetpions as we want and map them into local exceptions for upstream services not to have a strong dependency on any particular libraries or external resource models, which we will talk about in detail when we move on to the Mapping responsibility of broker-neighboring (foundation) services.

#### 3.0.1 Logical Validations
Logical validations are the second in order to structural validations. their main responsibility by definition is to logically validate whether a structurally valid piece of data is logically valid.
For instance, a date of birth for a student could be structurally valid by having a value of `1/1/1800` but logically, a student that is over 200 years of age is an impposibility.

The most common logical validations are validations for audit fields such as `CreatedBy` and `UpdatedBy` - it's logically impossible that a new record can be inserted with two different values for the authors of that new record - simply because data can only be inserted by one person at a time.

Let's talk about how we can test-drive and implement logical validations:

##### 3.0.1.0 Testing Logical Validations
In the common case of testing logical validations for audit fields, we want to throw a validation exception that the `UpdatedBy` value is invalid simply because it doesn't match the `CreatedBy` field.

Let's assume our Student model looks as follows:
```csharp
public class Student {
	Guid CreatedBy {get; set;}
	Guid UpdatedBy {get; set;}
}
```

Our test to validate these values logically would be as follows:

 ```csharp
[Fact]
public async Task ShouldThrowValidationExceptionOnRegisterIfUpdatedByNotSameAsCreatedByAndLogItAsync()
{
	// given
	Student randomStudent = CreateRandomStudent();
	Student inputStudent = randomStudent;
	inputStudent.UpdatedBy = Guid.NewGuid();

	var invalidStudentException = new InvalidStudentException(
		parameterName: nameof(Student.UpdatedBy),
		parameterValue: inputStudent.UpdatedBy);

	var expectedStudentValidationException =
		new StudentValidationException(invalidStudentException);

	// when
	ValueTask<Student> registerStudentTask =
		this.studentService.RegisterStudentAsync(inputStudent);

	// then
	await Assert.ThrowsAsync<StudentValidationException>(() =>
		registerStudentTask.AsTask());

	this.loggingBrokerMock.Verify(broker =>
		broker.LogError(It.Is(
			SameExceptionAs(expectedStudentValidationException))),
				Times.Once);

	this.storageBrokerMock.Verify(broker =>
		broker.InsertStudentAsync(It.IsAny<Student>()),
			Times.Never);

	this.loggingBrokerMock.VerifyNoOtherCalls();
	this.dateTimeBrokerMock.VerifyNoOtherCalls();
	this.storageBrokerMock.VerifyNoOtherCalls();
}
 ```

 In the above test, we have changed the value of the `UpdatedBy` field to ensure it completely differs from the `CreatedBy` field - now we expect an `InvalidStudentException` with the `CreatedBy` to be the reason for this validation exception to occur.

 Let's go ahead an write an implementation for this failing test.

 ##### 3.0.1.1 Implementing Logical Validations
 Just like we did in the structural validations section, we are going to add more rules to our validation `switch case` as follows:

 ###### StudentService.Validations.cs
```csharp
private void ValidateStudent(Student student)
{
	switch(student)
	{
		case {} when IsNotSame(student.CreatedBy, student.UpdatedBy):
			throw new InvalidStudentException(
				parameterName: nameof(Student.UpdatedBy),
				parameterValue: student.UpdatedBy);
	}
}
	
private static bool IsNotSame(Guid firstId, Guid secondId) => firstId != secondId;
```

Everything else in both `StudentService.cs` and `StudentService.Exceptions.cs` continues to be exactly the same as we've done above in the structural validations.

Logical validations exceptions, just like any other exceptions that may occur are usually non-critical. However, it all depends on your business case to determine whether a particular logical, structural or even a dependency validation are critical or not, this is when you might need to create a special class of exceptions, something like `InvalidStudentCriticalException` then log it accordingly.