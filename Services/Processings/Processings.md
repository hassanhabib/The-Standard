# Processing Services (Higher-Order)

## 0. Introduction
Processing services are the layer where a higher order of business logic is implemented. they may combine (or orchestrate) two primitive-level functions from their corresponding foundation service to introduce a newer functionality. they may also call one primitive function and change the outcome with a little bit of added business logic. and sometime processing services are there as a pass-through to introduce balance to the overall architecture.

Processing services are optional, depending on your business need - in a simple CRUD operations API, processing services and all the other categories of services beyond that point will sieze to exist as there is no need for a higher order of business logic at that point.

Here's an example of what a Processing service function would look like:

```csharp
public Student EnsureStudentExistsAsync(Student student) =>
TryCatch(async () => 
{
    ValidateStudent(student);
    
    IQueryable<Student> allStudents = 
        this.studentService.RetrieveAllStudents();
    
    bool isStudentExists = allStudents.Any(retrievedStudent => 
        retrievedStudent.Id == student.Id);

    return isStudentExsits switch {
        false => await this.studentService.RegisterStudentAsync(student),
        _ => await this.studentService.RetrieveStudentByIdAsync(student.Id)
    };
});

```

Processing services make Foundation services nothing but a layer of validation on top of the existing primitive operations. Which means that Processing services functions are beyond primitive, and they only deal with local models as we will discuss in the upcoming sections.

## 1. On The Map
When used, Processing services live between foundation services and the rest of the application. they may not call Entity or Business brokers, but they may call Utility brokers such as logging brokers, time brokers and any other brokers that offer supporting functionality and not specific to any particular business logic. here's a visual of where processing services are located on the map of our architecture:



