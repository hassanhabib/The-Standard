# Services

# 0. Introduction
Services in general are the containers of all the business logic in any given software.

When we say business logic, we mainly refer to three main categories of operations, which are validation, processing and integration.

Validations focus on ensuring that the incoming or outgoing data match a particular set of rules, which can be structural, logical or dependency validations which we will go in details about in the upcoming sections.

Processing mainly focuses on the flow-control, mapping and computation to satisfy a business need - the processing operations specifically is what distinguishes one service from another, and in general one software from another.

Finally, the integration process is mainly focused on retrieving or pushing data from or to any integrated system dependencies.

Every one of these aspects will be discussed in details in the upcoming chapter, but the main thing that should be understood about services is that they should be built with the intent to be pluggable and configurable so they are easily integrated with any technology from a dependency standpoint and also be easily plugged into any exposure functionality from an API perspective.