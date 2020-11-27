# Foundation Services (Broker-Neighboring)

## 0. Introduction
Foundation services are the first point of contact between your business logic and the brokers.

In general, the broker-neighboring services are a hybrid of business logic and an abstraction layer for the processing operations where the higher-order business logic happens, which we will talk about further when we start exploring the processing services.

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


## 1. Characteristics
### 1.0 Pure-Primitive
Broker-neighboring services are not allowed to combine multiple primitive operations to achieve a higher-order business logic operation.

For instance, broker-neighboring services cannot offer an *upsert* function, to combine a `Select` operations with an `Update` or `Insert` operations based on the outcome.