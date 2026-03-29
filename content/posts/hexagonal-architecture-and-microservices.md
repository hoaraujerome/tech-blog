+++
date = '2021-12-10T15:32:06-04:00'
title = 'Hexagonal Architecture and Microservices'
+++

# Presentation

Several technologies make it possible to expose or invoke business functions: SOA, REST / Web API, Messaging / JMS, and others. It is crucial to isolate the code that implements the business logic from the architectures used. The hexagonal architecture is an option to design microservices to address these challenges.

![Hexagonal Architecture Diagram](images/hexagonal_architecture_diagram.png)

Source: http://tpierrain.blogspot.com/2013/08/a-zoom-on-hexagonalcleanonion.html

P/A stands for "Port/Adapter" and UC stands for "Use Case"

High-level sequencing:

*   A service receives and sends events to the "outside" via ports. A port is specific to a technology or a protocol: Servlet API, SOAP endpoint, JMS listener, or a JDBC driver.
    
*   Adapters specific to each port make it possible to convert the event received into an invocation of methods of the business logic implementation classes - domain components.
    
*   Domain components, agnostic to the technologies and protocols used by the ports, focus on the implementation of the business logic.
    
*   The application layer (also called the service layer) is responsible, in particular, for the management of transactions.
    
*   Conversely, the service can send events to the "outside" via the ports: SOAP / REST service calls, new JMS messages, writing to a database.
    
*   Adapters dictate to ports how outgoing messages are formatted and submitted from domain or application layer components.
    
*   Context adapters by implementing the basic components. For example, the automatic code generation tools and the XML / Java conversion make the SOAP adapter implementation easier. Same thing with the URLs mapping tools and requests content to Java classes on the REST side.
    

# Why hexagonal architecture

The hexagonal architecture emphasizes a strict separation between the ports and adapters on the one hand and the components that implement the business logic on the other hand. It is fundamental because technologies and protocols can evolve faster than business logic.

> The hexagonal architecture forms the strong foundation for supporting any and all those additional architectural options like SOA, REST, event-driven, event-sourcing, CQRS. Vaughn Vernon - Domain Driven Design

In such an architecture, a service can have several ports which allow to receive and submit several events. It is an evolutionary design that allows supporting new sources of events without altering the existing components.

# Interactions within the service

The following diagram shows the components interactions within a REST service.

![Hexagonal Architecture Interactions](images/hexagonal_architecture_interactions.png)
