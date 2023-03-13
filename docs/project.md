# BackLogic Project
---

`BackLogic` is the project implementing the concept of `data access service`. It is such named as DAS provides data access logic as a backing service to the application.

## Scope

### Service Builder <!-- {docsify-ignore} -->

The tool for developing data access service.

### Remote Workspace <!-- {docsify-ignore} -->

The development environment, hosting the `Service Builder Backend` and the devtime `Data Access Server`.

### Runtime Instance <!-- {docsify-ignore} -->

The runtime `Data Access Server` launched in user VPC as managed service.

### Data Access Client <!-- {docsify-ignore} -->

Java, TypeScript and Python client for simplifying consumption of data access services.

## Current Status

DAS is a project in progress. Currently, I have the `Service Builder` available for install from VS Code marketplace and `Remote Workspace` available for request from inside of `Service Builder`. Essentially, I have the tool and hosted DEV environment ready for you to play and try. You may start with the tutorials, deep dives and user guide available here to get familiar with Service Builder and data access services in general, and then move on to your own real or hypothetic data access problems. I will appreciate any feedback you may have in this phase.

## Road Map

In the next phase, I will be implementing the `Data Access Client` for Java, TypeScript and Python, so that you will be able to easily consume data access service from your Java, TypeScript and Python application.

In the third phase, I will be adding the capability to launch `Runtime Instance` as managed service inside your own VPC, so that you can run your production load in your own VPC, close to your own database. 

## Feedback

I will appreciate any feedback you may have.

Please email me at:  
ken@backlogic.net

