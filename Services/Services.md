# Services

## 0. Introduction
Services in general are the containers of all the business logic in any given software.


### 0.0 Services Operations
When we say business logic, we mainly refer to three main categories of operations, which are validation, processing and integration.

<br />
<p align=center>
    <img src="https://user-images.githubusercontent.com/1453985/100530065-4494d680-31a2-11eb-8393-32b21ab99a3d.png" />
</p>
<br />

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

