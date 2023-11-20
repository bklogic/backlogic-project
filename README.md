# BackLogic Data Access Service

The project site for BackLogic data access service.

## Data Access Service

Data access service is a cross-platform data access solution for relational database. It solves the object-relational impedance mismatch problem by:

- Separating SQL from the programming language; and
- Abstracting object-relational transformation away from developer.

So that the developer can have the comfort to compose SQL in a database-centric environment and the flexibility to query and persist objects of any shape and complexity.

Data access service provides data access logic as a backing service to the application. For serverless and microservices application, it additionally provides connection pooling as a service.

### Get Started

- Get Started Tutorials:
    - Get started with Service Builder
    - Get Started with Data Access Service
- Data Access Deep Dives
    - Query service
    - SQL service
    - CRUD service
- Develop and deploy your own application in `DEV` workspace
- Deploy your own application onto Service Runtime

### Examples

An example data access application showcasing the various simple and complex query, command (aka SQL) and repository (aka CRUD) services:

https://github.com/bklogic/data-access-service-example

### Documentation

- Data access service concepts
- Get started tutorials
- Data access service deep dives
- Reference guide

https://www.backlogic.net/documentation/DataAccessService

## BackLogic

BackLogic is the platform for developing and running data access service.

### Service Builder

The development tool for data access service. A free VS Code extension.

https://github.com/bklogic/ServiceBuilder

May install from VS Code Marketplace by searching `backlogic`.

### BackLogic Workspace

A virtual private DEV environment for developing and deploying data access application.

May request from inside of Service Builder.

### Runtime Instance

Runtime data access server. Listed as free container product in AWS Marketplace.

May deploy in user's VPC as ECS service by searching `backlogic`.

### Data Access Client

#### HTTP Client

Any http client, as data access service is deployed as HTTP service.

#### Java Client

For each query, command and repository interface that you define, the Java client provides a proxy that handles all HTTP related stuff in background. With Spring Boot starter, all data access interfaces are registered as Spring Beans and can be readily injected into your service classes.

For Spring Boot user:  
https://github.com/bklogic/jdac-spring-boot-starter

For pure Java user:  
https://github.com/bklogic/java-data-access-client


## Website

https://www.backlogic.net

## Support

GitHub `Discussions` for questions and discussions. GitHub `Issues` for issues and feature requests associated with Service Builder and Runtime.

