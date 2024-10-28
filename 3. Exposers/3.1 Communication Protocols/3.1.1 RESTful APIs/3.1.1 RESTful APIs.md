# 3.1.1 RESTful APIs

## 3.1.1.0 Introduction
RESTful API controllers are liaisons between the core business logic layer and the outside world. They sit on the other side of any application's core business realm. In a way, API Controllers are just like Brokers. They ensure a successful integration between our core logic and the rest of the world.

## 3.1.1.1 On the Map
Controllers sit at the edge of any system, regardless of whether this system is a monolithic platform or a simple microservice. API controllers today even apply to smaller lambdas or cloud functions. They act as a trigger to access these resources in any system through REST.

<br />
    <div align=center>
        <img src="https://user-images.githubusercontent.com/1453985/147741682-63434be5-3122-4484-b9a1-fd013f18b1b0.png" />
    </div>
<br />

The consumer side of controllers can vary. In production systems, these consumers can be other services that require data from a particular API endpoint. They can be libraries built as wrappers around the controller APIs to provide a local resource with external data. However, consumers can also be engineers testing endpoints and validating their behaviors through swagger documents.

## 3.1.1.2 Characteristics
Several rules and principles govern the implementation of RESTful API endpoints. Let's discuss those here.

### 3.1.1.2.0 Language
Controllers speak a different language when it comes to implementing their methods than services and brokers. For instance, if a broker that interfaces with a storage uses a language such as `InsertStudentAsync` and its corresponding service implementation uses something like `AddStudentAsync`, the controller equivalent will use a RESTful language such as `PostStudentAsync`.

A common controller would use only a handful of terminologies to express a particular operation. Let's draw the map here for clarity:

| Controllers							| Services                      	| Brokers								    |
|------------------------				|---------------------------		| ------------------------------------------|
| Post          				        | Add                               | Insert                     			    |
| Get               				    | Retrieve               			| Select                       			    |
| Put               					| Modify                          	| Update                			        |
| Delete                          		| Remove                            | Delete                                	|

But it must be important to understand that these operations can be extended and modified to fit the operation we are performing. We will talk about that shortly.
The language controllers speak is called HTTP Verbs. Their range is wider than the aforementioned CRUD operations. For instance, there is PATCH, which allows API consumers to update only portions of a particular document. From my experience in productionized applications, PATCH is rarely used today. However, I may expand on them in a special section in future versions of The Standard.

#### 3.1.1.2.0.0 Beyond CRUD Routines
As we mentioned before, a controller can interface with more than just a foundation service. They can interface with higher-order business logic functions. For instance, a processing service may offer an `Upsert` routine. In this case, a typical Http Verb wouldn't be able to satisfy a combinational routine such as an `Upsert`. In this case, we introduce custom http verbs that can support the aforementioned operation.

Custom http verbs can be anything as long as it doesn't conflict with an existing verb or contains non-supported characters. For instance, if we have a system that generate barcodes for some products, the generation process doesn't quite fit a `POST` or a `GET`. Especially when we factor in that these http verbs/methods might already be used for the same route for storing and retrieving barcodes from the system. This is a case where a new http verb can be introduced as `GENERATE` where you can pass the same route with a different verb that a server would understand.

We introduced this capabilty in RESTFulSense library where you can simply define your controller method as follows:

```csharp
    [HttpCustom("GENERATE")]
    public ActionResult<Barcode> GenerateBarcode() =>
        this.barcodeProcessingService.Generate();
```
A consumer client can call that endpoint also as follows:

```csharp
    Barcode generatedBarcode =
        await restfulHttpClient.SendHttpRequestAsync<Student>(
            method: "GENERATE",
            relativeUrl: "api/barcodes",
            cancellationToken: someOrNoneCancellationToken);
```

The same operations can also be done without using a particular library with the native `HttpClient`.

#### 3.1.1.2.0.1 Similar Verbs
Sometimes, especially with basic CRUD operations, you will need the same Http Verb to describe two different routines. For instance, integrating with both `RetrieveById` and `RetrieveAll` resolves to a `Get` operation on the RESTful realm. In this case, each function will have a different name and route while maintaining the same verb as follows:

```csharp
[HttpGet]
public async ValueTask<ActionResult<IQueryable<Student>>> GetAllStudentsAsync()
{
    ...
}

[HttpGet("{studentId}")]
public async ValueTask<ActionResult<Student>> GetStudentByIdAsync(Guid studentId)
{
    ...
}
```

As you can see above, the differentiator here is simultaneously the function name `GetAllStudents` versus `GetStudentByIdAsync` and the `Route`. We will discuss routes shortly, but the central aspect is the ability to implement multiple routines with different names, even if they resolve to the same Http Verb.

#### 3.1.1.2.0.2 Routes Conventions
RESTful API controllers are accessible through routes. A route is simply a URL combined with an Http verb, so the system knows which routine it needs to call to match that route. For instance, if I need to retrieve a student with id `123`, my API route would be `api/students/123`. And if I want to retrieve all the students in some system, I could just call `api/students` with the `GET` verb.

##### 3.1.1.2.0.2.0 Nouns & Verbs
Routes should never have verbs in them. That's the responsibility of the http verb. For instance, we should never name a route as such: `api/students/get` that is a violation of the naming conventions of The Standard. The rule here is that routes should also have nounes, and http methods should always have verbs. Http methods with customization as mentioned above could have endless number of custom methods against the very same route.

The Standard also enforces the single-noun principle. Routes should not combine mulitple nouns. For instance, instead of saying: `api/studentsubmissions` we should say: `api/student-submissions`.
On that same line, retreiving submissions for a particular student can be represented as follows: `api/students/{studentId}/submissions` the breakdown here is necessary to imply the intersection between resources compared to pulling everything in storage.

##### 3.1.1.2.0.2.1 Controller Routes
The controller class in a simple ASP.NET application can be set at the top of the controller class declaration with a decoration as follows:

```csharp
[ApiController]
[Route("api/[controller]")]
public class StudentsController
{
    ...
}
```

The route there is a template that defines the endpoint to start with `api` and trail by omitting the term "Controller" from the class name. So `StudentsController` would end up being `api/students`. All controllers must have a plural version of the contract they are serving. Unlike services where we say `StudentService`, controllers would be the plural version with `StudentsController`.

##### 3.1.1.2.0.2.2 Routine/Method-Specific Routes
The same idea applies to methods within the controller class. As the code snippet above says, we decorated `GetStudentByIdAsync` with an `HttpGet` decoration with a particular route identified to append to the existing controller overall route. For instance if the controller route is `api/students`, a routine with `HttpGet("{studentId})` would result in a route that looks like this: `api/students/{studentId}`.

The `studentId` then would be mapped in as an input parameter variable that *must* match the variable defined in the route as follows:

```csharp
[HttpGet("{studentId}")]
public async ValueTask<ActionResult<Student>> GetStudentByIdAsync(Guid studentId)
{
    ...
}
```

But sometimes, these routes are not just URL parameters. Sometimes, they contain a more specific parent resources information within them. For instance, we want to post a library card against a particular student record. Our endpoint would look like `api/students/{studentId}/librarycards` with a `POST` verb. In this case, we have to distinguish between these two input parameters with proper naming as follows:

```csharp
[HttpPost("{studentId}/librarycards")]
public async ValueTask<ActionResult<LibraryCard>> PostLibraryCardAsync(Guid studentId, LibraryCard libraryCard)
{
    ...
}
```

##### 3.1.1.2.0.2.3 Plural-Singular-Plural
When defining routes in a RESTful API, it is important to follow the global naming conventions for these routes. The general rule is to access a collection of resources, then target a particular entity, then again access a collection of resources within that entity, and so on and so forth. For instance, in the library card example above, `api/students/{studentId}/librarycards/{librarycardId}`, we started by accessing all students and then targeting a student with a particular ID. We wanted to access all library cards attached to that student and then target a very particular card by referencing its ID.

That convention works perfectly in one-to-many relationships. But what about one-to-one relationships? Let's assume a student may have one and only one library card at all times. In which case, our route would still look something like this: `api/students/{studentId}/librarycards` with a `POST` verb, and an error would occur as `CONFLICT` if a card is already in place regardless of whether the Ids match or not.

##### 3.1.1.2.0.2.4 Query Parameters & OData
But the route I recommend is the flat-model route. Where every resource lives on its own with its unique routes, in our case here, pulling a library card for a particular student would be as follows: `api/librarycards?studentId={studentId}` or use slightly advanced global technology such as OData where the query would be `api/librarycards?$filter=studentId eq '123'`.

Here's an example of implementing basic query parameters:

```csharp
[HttpPost()]
public async ValueTask<ActionResult<LibraryCard>> PostLibraryCardAsync(Guid studentId, LibraryCard libraryCard)
{
    ...
}
```

On the OData side, an implementation would be as follows:

```csharp
[HttGet]
[EnableQuery]
public async ValueTask<IQueryable<LibraryCard>> GetAllLibraryCardsAsync()
{
    ...
}
```

The same idea applies to `POST` for a model. Instead of posting towards: `api/students/{studentId}/librarycards` - we can leverage the contract itself to post against `api/librarycards` with a model that contains the student id within. This flat-route idea can simplify the implementation and aligns perfectly with the overall theme of The Standard. We are keeping things simple.

##### 3.1.1.2.0.2.5 X-WWW-Form-UrlEncoded Parameters
The Standard enfornces the concept of single-entity. For instance, we can't have a method as follows in a Standard-compliant system:
```csharp
ValueTask<Teacher> GetTeachersByStudentAsync(Student student);
```
The above is considered a violation because the service that supports this routine explicity handles multiple models or entity types. But The Standard also permits passing primitive parameters such as `string`, `bool` or any other primitive or native type. Controllers/API can also support the same pattern though x-www.form-urlencoded parameters as follows:

On the controller side, you can implement `x-www-form-urlencoded` as follows:

```csharp
    [HttpPost("login")]
    [Consumes("application/x-www-form-urlencoded")]
    public async ValueTask<ActionResult<UserAuthentication>> PostLoginUserAsync(
        [FromForm] string username,
        [FromForm] string password)
    {
        ....
    }
```
On the consumer side, the implementation would be:

```csharp
    var formUrlEncodedContent =
         new FormUrlEncodedContent(new[]
         {
            new KeyValuePair<string, string>("username", username),
            new KeyValuePair<string, string>("password", password)
         });
    
    HttpResponseMessage httpResponseMessage =
        await this.apiClient.ExecuteHttpCallAsync(this.httpClient.PostAsync(
            requestUri: $"{userAuthenticationsRelativeUrl}/login",
            content: formUrlEncodedContent));
```
The rule to Services applies to Controllers as well, a routine at this level cannot accept more than 3 parameters - and beyond that point engineers must design the system to accept an actual entity or model and return the same model in the response.

### 3.1.1.2.1 Codes & Responses
Responses from an API controller must be mapped towards codes and responses. For instance, if we are trying to add a new student to a schooling system. We are going to `POST` a student, and in return, we receive the same body we submitted with a status code `201`, which means the resource has been `Created`. 

There are three main categories into which responses can fall. The first is the success category. Both the user and the server have done their part, and the request has been successful. The second category is the User Error Codes, where the user request has an issue of any type. In this case, a `4xx` code will be returned with a detailed error message to help users fix their requests to perform a successful operation. The third case is the System Error Codes, where the system has run into an issue of any type, internal or external, and it needs to communicate a `5xx` code to indicate to the user that something internally has gone wrong with the system and they need to contact support.

Let's talk about those codes and their scenarios in detail here.

#### 3.1.1.2.1.0 Success Codes (2xx)
Success codes indicate a resource has been created, updated, deleted, or retrieved. In some cases, it indicates that a request has been submitted successfully in an eventual consistent manner that may or may not succeed. Here are the details for each:

| Code      							| Method                          	| Details								                            |
|---------------------------------------|-----------------------------------|-------------------------------------------------------------------|
| 200             				        | Ok                                | Used for successful GET, PUT, and DELETE operations.               |
| 201               				    | Created               			| Used for successful POST operations                               |
| 202               					| Accepted                          | Used for request that was delegated but may or may not succeed    |


Here are some examples for each:

In a retrieve non-post scenario, it's more befitting to return an `Ok` status code as follows:

```csharp
[HttpGet("{studentId}")]
public async ValueTask<ActionResult<Student>> GetStudentByIdAsync(Guid studentId)
{
    Student retrievedStudent = 
        await this.studentService.RetrieveStudentByIdAsync(studentId);

    return Ok(retrievedStudent);
}
```

But in a scenario where we have to create a resource, a `Created` is more befitting for this case as follows:

```csharp
[HttpPost)]
public async ValueTask<ActionResult<Student>> PostStudentAsync(Student student)
{
    Student retrievedStudent = 
        await this.studentService.AddStudentAsync(student);

    return Created(student);
}
```

In eventual consistency cases, where a resource posted is not persisted yet, we enqueue the request and return an `Accepted` status to indicate a process will start:
```csharp
[HttpPost)]
public async ValueTask<ActionResult<Student>> PostStudentAsync(Student student)
{
    Student retrievedStudent = 
        await this.studentEventService.EnqueueStudentEventAsync(student);

    return Accepted(student);
}
```

The Standard rule for eventual consistency scenarios is to ensure the submitter has a token of some type so requestors can inquire about the status of their request using a different API call. We will discuss these patterns in a different book called The Standard Architecture.


#### 3.1.1.2.1.1 User Error Codes (4xx)
This is the second category of API responses. In this category, a user request has an issue, and the system is required to help the user understand why their request was not successful. For instance, assume a client is submitting a new student to a schooling system. If the student ID is invalid, a `400` or `Bad Request` code should be returned with a problem detail that explains what exactly caused the request to fail.

Controllers are responsible for mapping the core layer categorical exceptions into proper status codes. Here's an example:

```csharp
[HttpGet("{studentId}")]
public async ValueTask<ActionResult<Student>> GetStudentByIdAsync(Guid studentId)
{
    try
    {
        ...
    }
    catch (StudentValidationException studentValidationException)
    {
        return BadRequest(studentValidationException.InnerException)
    }
}
```

So, as shown in this code snippet, we caught a categorical validation exception and mapped it into a `400` error code, which is `BadRequest`. Access to the inner exception here is for the purpose of extracting a problem detail out of the `Data` property on the inner exception, which contains all the dictionary values of the error report.

But sometimes, controllers have to dig deeper. Catching a particular local exception, not just the categorical. For instance, say we want to handle `NotFoundStudentException` with an error code `404` or `NotFound`. Here's how we would accomplish that:

```csharp
[HttpGet("{studentId}")]
public async ValueTask<ActionResult<Student>> GetStudentByIdAsync(Guid studentId)
{
    try
    {
        ...
    }
    catch (StudentValidationException studentValidationException)
        (when studentValidationException.InnerException is NotFoundStudentException)
    {
        return NotFound(studentValidationException.InnerException)
    }
}
```

In the code snippet above, we had to examine the inner exception type to validate the localized exception from within. This is the advantage of the unwrapping and wrapping process discussed in section 2.3.3.0.2 of The Standard. The controller may examine multiple types within the same block as well as follows:

```csharp
    ...
    catch (StudentCoordinationDependencyValidationException studentCoordinationDependencyValidationException)
        (when studentValidationException.InnerException 
            is NotFoundStudentException
            or NotFoundLibraryCardException
            or NotFoundStudentContactException)
    {
        return NotFound(studentValidationException.InnerException)
    }
    ...
```

With that in mind, let's detail the most common mappings from exceptions to codes:

| Code      							| Method                          	| Exception								                            |
|---------------------------------------|-----------------------------------|-------------------------------------------------------------------|
| 400             				        | BadRequest                        | ValidationException or DependencyValidationException              |
| 404               				    | NotFound               			| NotFoundException                                                 |
| 409               					| Conflict                          | AlreadyExistException                                             |
| 423               					| Locked                            | LockedException                                                   |
| 424               					| FailedDependency                  | InvalidReferenceException                                         |

There are more `4xx` status codes out there. But they can either be automatically generated by the web framework, like in ASP.NET, or there are no useful scenarios for them yet. For instance, a `401` or `Unauthorized` error can be automatically generated if the controller endpoint is decorated with an authorization requirement.

#### 3.1.1.2.1.2 System Error Codes (5xx)
System error codes are the third and last possible type of code that may occur or be returned from an API endpoint. Their main responsibility is to indicate that the API endpoint consumer is generally at no fault. Something terrible happened in the system, and the engineering team must get involved to resolve the issue. That's why we log our exceptions with a severity level at the core business logic layer so we know how urgent the matter may be.

The most common Http code that can be communicated on a server-side issue is the `500` or `InternalServerError` code. Let's take a look at a code snippet that deals with this situation:

```csharp
[HttpGet("{studentId}")]
public async ValueTask<ActionResult<Student>> GetStudentByIdAsync(Guid studentId)
{
    try
    {
        ...
    }
    ...
    catch (StudentDependencyException studentDependencyException)
    {
        return InternalServerError(studentValidationException)
    }
}
```

In the above snippet, we ignored the inner exception and mainly focused on the categorical exception for security reasons. Primarily to not allow internal server information to be exposed in an API response other than something as simple as `Dependency error occurred, contact support.` Since the API consumer is required to perform no action whatsoever other than creating a ticket for the support team, Ideally, these issues should be caught out of Acceptance Tests, which we will discuss shortly in this chapter. But there are times where there's a server blip that may cause a memory leakage of some sort or any other internal infrastrucrual issues that won't be caught by end-to-end testing in any way.

The types of exceptions that may be handled are smaller regarding server errors. Here are the details:

| Code      							| Method                          	| Exception								                            |
|---------------------------------------|-----------------------------------|-------------------------------------------------------------------|
| 500             				        | InternalServerError               | DependencyException or ServiceException                           |
| 507               				    | NotFound               			| InsufficientStorageException (Internal Only)                      |

There's also an interesting case where two teams agree on a specific swagger document, and the back-end API development team decides to build corresponding API endpoints with methods yet to be implemented to communicate to the other team that the work has yet to start. In this case, the error code `501` is sufficient, just a code for `NotImplemented`.

It is also important to mention that the native `500` error code can be communicated in ASP.NET applications through the `Problem` method. We are relying on a library, `RESTFulSense`, to provide more codes than the native implementation can offer, but more importantly, to provide a problem detail serialization option and deserialization option on the client side.

#### 3.1.1.2.1.3 All Codes
Other than the ones mentioned in previous sections, and for documentation purposes, here are all of the `4xx` and `5xx` codes an API could communicate according to the latest standardized API guidelines:

|Status|Code|
|--- |--- |
|BadRequest|400|
|Unauthorized|401|
|PaymentRequired|402|
|Forbidden|403|
|NotFound|404|
|UrlNotFound|404|
|MethodNotAllowed|405|
|NotAcceptable|406|
|ProxyAuthenticationRequired|407|
|RequestTimeout|408|
|Conflict|409|
|Gone|410|
|LengthRequired|411|
|PreconditionFailed|412|
|RequestEntityTooLarge|413|
|RequestUriTooLong|414|
|UnsupportedMediaType|415|
|RequestedRangeNotSatisfiable|416|
|ExpectationFailed|417|
|MisdirectedRequest|421|
|UnprocessableEntity|422|
|Locked|423|
|FailedDependency|424|
|UpgradeRequired|426|
|PreconditionRequired|428|
|TooManyRequests|429|
|RequestHeaderFieldsTooLarge|431|
|UnavailableForLegalReasons|451|
|InternalServerError|500|
|NotImplemented|501|
|BadGateway|502|
|ServiceUnavailable|503|
|GatewayTimeout|504|
|HttpVersionNotSupported|505|
|VariantAlsoNegotiates|506|
|InsufficientStorage|507|
|LoopDetected|508|
|NotExtended|510|
|NetworkAuthenticationRequired|511|

We will explore incorporating some of these codes in future revisions of The Standard as needed.

### 3.1.1.2.2 Single Dependency
Exposer components can have one and only one dependency. This dependency must be a Service component. It cannot be a Broker or any other native dependency that Brokers may use to pull configurations or any other type of dependencies.

When implementing a controller, the constructor can be implemented as follows:

```csharp
[ApiController]
[Route("api/[controller]")]
public class StudentsController : RESTFulController
{
    private readonly IStudentService studentService;

    public StudentsController(IStudentService studentService) =>
        this.studentService = studentService;

    ...
    ...
}
```

### 3.1.1.2.3 Single Contract
This characteristic comes out of the box with the single dependency rule. If Services can only serve and receive one contract, then the same rule will apply to controllers. When passing in IDs or queries, they can return a contract or a list of objects with the same contract or portion of the contract.

## 3.1.1.3 Organization
Controllers should be located under the `Controllers` folder and belong within a `Controllers` namespace. However, controllers do not need to have their own folders or namespaces as they perform a simple exposure task.

Here's an example of a controller namespace:

```csharp
namespace GitFyle.Core.Api.Controllers
{
    [ApiController]
    [Route("api/[controller]")]
    public class ContributionsController : RESTFulController
    {
        ...
    }
}
```

## 3.1.1.4 Home Controller
Every system should implement an API endpoint that we call `HomeController`. The controller's only responsibility is to return a simple message to indicate that the API is still alive. Here's an example:

```csharp
using Microsoft.AspNetCore.Mvc;

namespace OtripleS.Web.Api.Controllers
{
    [ApiController]
    [Route("api/[controller]")]
    public class HomeController : ControllerBase
    {
        [HttpGet]
        public ActionResult<string> Get() =>
            Ok("Hello Mario, the princess is in another castle!");
    }
}
```

Home controllers are not required to have any security. They open a gate for heartbeat tests to ensure the system as an entity is running without checking any external dependencies. This practice is very important to help engineers know when the system is down and quickly act on it.

## 3.1.1.5 Tests
There are three different types of tests that cover API controllers as well as any other exposure layer. These tests are: unit tests, acceptance tests and integration or end-to-end (E2E) tests. Integration tests can vary between smoke testing, availability testing, performance testing and many others. But for the purpose of this chapter, we will focus on unit and acceptance tests.

### 3.1.1.5.0 Unit Tests
Controllers have a similar type of logic that exists in Services. This logic is the mapping between exceptions coming from a dependency or internally and what these exceptions are being mapped to for the consumer of these APIs. For instance, a `StudentValidationException` can be mapped to a `BadRequest` status code. This logic is tested in unit tests. Let's take a look at an example:

Starting with the test before the implementation, let's assume we have a controller `StudentsController` that retrieves all students. When the call succeeds, the controller should return a `200 OK` status code. Let's write a test for that:

First, let's setup our `StudentsController` class as follows:

```csharp
namespace School.Core.Api.Controllers
{
    [ApiController]
    [Route("api/[controller]")]
    public class StudentsController : RESTFulController
    {
        private readonly IStudentService studentService;

        public StudentsController(IStudentService studentService) =>
            this.studentService = studentService;
        
        [HttpGet]
        public async ValueTask<ActionResult<IQueryable<Student>>> GetAllStudentsAsync()
        {
            return NotImplemented(new NotImplementedException());
        }
    }
```
In the above code snippet, we initialized `StudentsController`, we inherited `RESTFulController` so we can have support for all possible status codes. But additionally, we created a `GetAllStudentsAsync` method that returns `NotImplemented` status code with a `NotImplementedException` exception. Notice the difference here between throwing the exception `NotImplementedException` at the Services layer compared to controllers.

Now, let's move on to writing a controller unit test. Let's setup our `StudentsControllerTest` class as follows:
```csharp
    public partial class StudentsControllerTests : RESTFulController
    {
        private readonly Mock<IStudentService> studentServiceMock;
        private readonly StudentsController studentsController;

        public StudentsControllerTests()
        {
            this.studentServiceMock = new Mock<IStudentService>();

            this.studentsController = new StudentsController(
                studentService: this.studentServiceMock.Object);
        }

        ....
    }
```
In the above example, we did three important things:
1. We made sure the `SourcesControllerTests` class is partial so we can write other files that are still a part of this class but target particular areas and methods.
2. We inherited from `RESTFulController` which is a class that comes from `RESTFulSense` .NET library which we will use later to create the expected response such as `Ok(retrievedStudents)`.
3. We mocked the dependency so we don't actually call the `StudentService` but rather call a controlled mock so we can simulate responses and exceptions depends on the context of the unit test.

Now, let's write a unit test for `GetAllStudentsAsync` controller method as follows:

```csharp
    [Fact]
    public async Task ShouldReturnOkOnGetAllStudentsAsync()
    {
        // given
        List<Student> randomStudents =
            CreateRandomStudents();

        List<Student> returnedStudents =
            randomStudents;

        List<Student> expectedStudents =
            returnedStudents.DeepClone();

        OkObjectResult expectedObjectResult =
            Ok(expectedStudents);

        var expectedActionResult =
            new ActionResult<List<Student>>(
                expectedObjectResult);

        this.studentServiceMock.Setup(service =>
            service.RetrieveAllStudentsAsync())
                .ReturnsAsync(returnedStudents);

        // when
        ActionResult<List<Student>> actualActionResult =
            await this.studentsController
                .GetAllStudentsAsync();

        // then
        actualActionResult.ShouldBeEquivalentTo(
            expectedActionResult);

        this.studentServiceMock.Verify(service =>
            service.RetrieveAllStudentsAsync(),
                Times.Once);

        this.studentServiceMock.VerifyNoOtherCalls();
    }
```

In the above test, just like we did with Services unit tests we did the following:
1. We created a list of random students to simulate a response from the service.
2. We cloned the list of students to create an expected response.
3. We created an `OkObjectResult` object to simulate the expected response from the controller.
4. We setup the `studentServiceMock` to return the list of students when `RetrieveAllStudentsAsync` is called.
5. We called the `GetAllStudentsAsync` method on the controller.
6. We verified that response `expectedActionResult` is equivalent to the actual response `actualActionResult`.
7. We verified that the `RetrieveAllStudentsAsync` method was called once.
8. Lastly, we wanted to verify that the controller isn't making any additional unnecessary calls from the dependency.

The above test will fail with expected code being `200 OK` but instead the actual is `501 Not Implemented`. Now, let's make that test pass as following:

```csharp
namespace School.Core.Api.Controllers
{
    [ApiController]
    [Route("api/[controller]")]
    public class StudentsController : RESTFulController
    {
        private readonly IStudentService studentService;

        public StudentsController(IStudentService studentService) =>
            this.studentService = studentService;
        
        [HttpGet]
        public async ValueTask<ActionResult<IQueryable<Student>>> GetAllStudentsAsync()
        {
            List<Student> students =
                await this.studentService.RetrieveAllStudentsAsync();

            return Ok(students);
        }
    }
```

In the above code, we implemented `GetAllStudentsAsync` method and now our unit test will successfully pass. 

### 3.1.1.5.1 Acceptance Tests
Here's an example of an acceptance test:

```csharp
[Fact]
public async Task ShouldPostStudentAsync()
{
    // given
    Student randomStudent = CreateRandomStudent();
    Student inputStudent = randomStudent;
    Student expectedStudent = inputStudent;

    // when 
    await this.otripleSApiBroker.PostStudentAsync(inputStudent);

    Student actualStudent =
        await this.otripleSApiBroker.GetStudentByIdAsync(inputStudent.Id);

    // then
    actualStudent.Should().BeEquivalentTo(expectedStudent);
    await this.otripleSApiBroker.DeleteStudentByIdAsync(actualStudent.Id);
}
```

Acceptance tests are required to cover every available endpoint on a controller and are responsible for cleaning up any test data after the test is completed. It is also important to mention that resources not owned by the microservice, like the database, must be emulated with applications such as `WireMock` and many others.

Acceptance tests are also implemented after the fact, unlike unit tests. An endpoint has to be fully integrated and functional before a test is written to ensure implementation success is in place.

[*] [Controller Unit Tests](https://www.youtube.com/watch?v=Fc4LgUR2174)

[*] [Acceptance Tests (Part 1)](https://www.youtube.com/watch?v=WWN-9ahbdIU)

[*] [Acceptance Tests (Part 2)](https://www.youtube.com/watch?v=ANqj9pldfso)
