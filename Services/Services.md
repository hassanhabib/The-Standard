# Services

## 0. Introduction
Services in general are the containers of all the business logic in any given software - they are the core component of any system and the main component that makes one system different from another.

Our main goal with services is that to keep them completely agnostic from specific technologies or external dependencies.

Any business layer is more compliant with The Standard if it can be plugged into any other dependencies and exposure technologies with the least amount of integration efforts possible.


### 0.0 Services Operations
When we say business logic, we mainly refer to three main categories of operations, which are validation, processing and integration.

<br />
<p align=center>
    <img src="https://user-images.githubusercontent.com/1453985/100530065-4494d680-31a2-11eb-8393-32b21ab99a3d.png" />
</p>
<br />

Let's talk about these categories.

#### 0.0.0 Validations 
Validations focus on ensuring that the incoming or outgoing data match a particular set of rules, which can be structural, logical or dependency validations which we will go in details about in the upcoming sections.

#### 0.0.1 Processing
Processing mainly focuses on the flow-control, mapping and computation to satisfy a business need - the processing operations specifically is what distinguishes one service from another, and in general one software from another.

#### 0.0.2 Integration
Finally, the integration process is mainly focused on retrieving or pushing data from or to any integrated system dependencies.

Every one of these aspects will be discussed in details in the upcoming chapter, but the main thing that should be understood about services is that they should be built with the intent to be pluggable and configurable so they are easily integrated with any technology from a dependency standpoint and also be easily plugged into any exposure functionality from an API perspective.


### 0.1 Services Types
But services have several types based on where they stand in any given architecture, they fall under three main categories, which are: validators, orchestrators and aggregators.

<br />
<p align=center>
 <img src="https://user-images.githubusercontent.com/1453985/100529444-b23e0400-319c-11eb-816a-59c73154542b.png" />
</p>
<br />

#### 0.1.0 Validators
Validator services are mainly the broker-neighboring services or foundation services.

These services' main responsibility is to add a validation layer on top of the existing primitive operations such as the CRUD operations to ensure incoming and outgoing data is validated structurally, logically and externally before sending the data off outside or insider the system.

#### 0.1.1 Orchestrators
Orchestrator services are the core of the business logic layer, they can be processors, orchestrators, coordinators or management services depending on the type of their dependencies.

Orchestrator services mainly focuses on combining multiple primitive operations, or multiple high-order business logic operations to achieve an even higher goal.

Orchestrators services are the decision makers within any architecture, they are the owners of the flow-control in any system and they are the main component that makes one application or software different from the other.

Orchestrator services are also meant to be built and live longer than any other type of services in the system.

#### 0.1.2 Aggregators
Aggregator services main responsibility is to tie the outcome of multiple processing, orchestration, coordination or management services to expose one single API for any given API controller, or UI component to interact with the rest of the system.

Aggregators are the gatekeepers of the business logic layer, they ensure the data exposure components (like API controllers) are interacting with only one point of contact to interact with the rest of the system.

Aggregators in general don't really care about the order in which they call the operations that is attached to them, but sometimes it becomes a necessity to execute a particular operation, such as creating a student record before assigning a library card to them.

We will discuss each and every type of these services in detail in the next chapters.

### 0.2 Overall Rules
There are several rules that govern the overall architecture and design of services in any system.

These rules ensure the overall readability, maintainability, configurability of the system - in that particular order.

#### 0.2.0 Do or Delegate
Every service should either do the work or delegate the work but not both.

For instance, a processing service should delegate the work of persisting data to a foundation service and not try to do that work by itself.

#### 0.2.1 Two-Three (Florance Pattern)
For Orchestrator services, their dependencies of services (not brokers) should be limited to 2 or 3 but not 1 and not 4 or more.

The dependency on one service denies the very definition of orchestration. that's because orchestration by definition is the combination between multiple different operations from different sources to achieve a higher order of business-logic.

###### This pattern violates Florance Pattern
<br/>
<p align=center>
    <img src="https://user-images.githubusercontent.com/1453985/100561648-4926c100-326e-11eb-9028-96bcd3eb0b1d.png">
</p>
<br />

###### This pattern follows the symmetry of the Pattern
<br />
<p align=center>
    <img src="https://user-images.githubusercontent.com/1453985/100561978-2a74fa00-326f-11eb-9d05-404eed3eaf5f.png">
</p>
<br />

The Florance pattern also ensures the balance and symmetry of the overall architecture as well.

For instance, you can't orchestrate between a foundation and a processing services, it causes a form of unbalance in your architecture, and an uneasy disturbance in trying to combine one unified statement with the language each service speaks based on their level and type.

The only type of services that is allowed to violate this rule are the aggregators, where the combination and the order of services or their calls doesn't have any real impact.

We will be discussing the Florance pattern a bit further in detail in the upcoming sections of The Standard.

#### 0.2.2 Single Exposure Point
API controllers, UI components or any other form of data exposure from the system should have one single point of contact with the business-logic layer.

For instance, an API endpoint that offer endpoints for persisting and retrieving student data, should not have multiple integrations with multiple services, but rather one service that offers all of these features.

Sometimes, a single orchestration, coordination or management service does not offer everything related to a particular entity, in which case an aggregator service is necessary to combine all of these features into one service ready to be integrated with by an exposure technology.

#### 0.2.3 Same or Primitives I/O Model
For all services, they have to maintain a single contract in terms of their return and input types, except if they were primitives.

For instance, a service that provides any kind of operations for an entity type `Student` - should not return from any of it's methods any other entity type.

You may return an aggregation of the same entity whether it's custom or native such as `List<Student>` or `AggregatedStudents` models, or a premitive type like getting students count, or a boolean indicating whether a student exists or not but not any other non-primitive or non-aggregating contract.

For input parameters a similar rule applies - any service may receive an input parameter of the same contract or a virtual aggregation contract or a primitive type but not any other contract, that simply violates the rule.

This rule enforces the focus of any service to maintain it's responsibility on a single entity and all it's related operations.

Once a service returns a different contract, it simply violates it's own naming convention like a `StudentOrchestrationService` returning `List<Teacher>` - and it starts falling into the trap of being called by other services from a completely different data pipelines.

For primitive input parameters, if they belong to a different entity model, that is not necessarily a reference on the main entity, it begs the question to orchestrate between two processing or foundation services to maintain a unified model without break the pure-contracting rule.

If the combination between multiple different contracts in an orchestration service is required, then a new unified virtual model has to be the new unique contract for the orchestration service with mappings implemented underneath on the concrete level of that service to maintain compatibility and integration saftey.
