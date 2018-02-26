# Some thoughts around the code structure

## Background

I have been thinking and guessing about the shape of the source code of the new platform and here are some of the progress I made.  I was able to find several interesting projects and I believe we can learn a lot from them. Thus I will organize my thoughts around those projects and put everything else at the end.

## eShopOnContainers

[eShopOnContainer](https://github.com/dotnet-architecture/eShopOnContainers) is an educational repo on GitHub -- I ran into it while browsing [Trending C# Repos on GitHub](https://github.com/trending/c%23) out of boredom.  It has demonstrated a lot of aspects we want to engage on our new platform and there is a great deal of things we can learn or borrow from it directly or indirectly.  The [issues section](https://github.com/dotnet-architecture/eShopOnContainers/issues) is a goldmine of information. This Repo is from Microsoft so it is **Official** and many explanation could be found from [Microsoft Doc](https://docs.microsoft.com/en-us/dotnet/standard/microservices-architecture/) directly. 


* Main platform/toolkit

This solution is based upon the exact same toolkit/platform we have in mind - DotNetCore 2.0 + EFCore 2.0. 

* Micro-Services and Docker

The whole solution is organized upon Micro-Services architect -- which can be found under [src/Services](https://github.com/dotnet-architecture/eShopOnContainers/tree/dev/src/Services). Those web services are tiny (except for Ordering), Usually don't share code with any other services and build into a single Docker container which runs as a WebAPI.  

* Encapsulation of Message Queue

In our early survey experiments we have tried stage the ESB (Enterprise Service Bus) as the core of the system. And here we can see a more cloud-agnostic approach.  An abstract SubscriptionManager was declared and then implemented on both RabbitMQ and AzureServiceBus. So the on-premises deployment will probably go with RabbitMQ and Cloud based deployment will go with AzureServiceBus. However, I think we can find a more elegant way to handle the configuration differences other than the [if-then-else](https://github.com/dotnet-architecture/eShopOnContainers/blob/dev/src/Services/Ordering/Ordering.API/Startup.cs#L142). 

* Health Check

This is relatively new part. It's still unclear that eventually this will take the shape of nuget packages or staying as part of the solution but some basic explanation could be found [here](https://docs.microsoft.com/en-us/dotnet/standard/microservices-architecture/implement-resilient-applications/monitor-app-health).  Eventually those health information could be shown in the WebStatus web application. 


* Background Tasks

This is the fun part. Given the mixed experience we had with raw AzureFunction I am curious to find out how to handle them. This solution contains only [one](https://github.com/dotnet-architecture/eShopOnContainers/tree/dev/src/Services/Ordering/Ordering.BackgroundTasks). The structure of this project is very interesting, it takes the shape of an empty web project but an [long running singleton service](https://github.com/dotnet-architecture/eShopOnContainers/blob/dev/src/Services/Ordering/Ordering.BackgroundTasks/Startup.cs#L53) as added during initialization. The [Base.BackgroundService](https://github.com/dotnet-architecture/eShopOnContainers/blob/dev/src/Services/Ordering/Ordering.BackgroundTasks/Tasks/Base/BackgroundTask.cs) is especially interesting. From it you can see it is different than AzureFunction and traditional DotNetCore Daemon.  [Official Explanation](https://docs.microsoft.com/en-us/dotnet/standard/microservices-architecture/multi-container-microservice-net-applications/background-tasks-with-ihostedservice).


* ResilientHttpClient/ResilientTransaction

[More information](https://docs.microsoft.com/en-us/dotnet/standard/microservices-architecture/implement-resilient-applications/) could be found from Doc. Overall it is interesting to see how Http errors, database errors and distributed transactions are handled here.


* Identity, Account and Authorization

There is a IdentityServer4 based implementation in the solution so it seems the new normal now.  [Official Doc](https://docs.microsoft.com/en-us/dotnet/standard/microservices-architecture/secure-net-microservices-web-applications/) contains a lot of diagrams to help understand the flow. I guess the **biggest challenge** for us is during the mixed stage, how to encapsulate existing RL6 as the new Identity/Authentication/Authorization mechanism?  Could we re-vitalize part of the Phoenix API code for this? 

* DevOp and CLI

The [k8s](https://github.com/dotnet-architecture/eShopOnContainers/tree/dev/k8s) folder  contains some detailed information on how to automate/orchestrate the containers for Azure and VSTS. Still need to study how it will happen for on-premises installation though. 

* Parts I am not sure

Should we have multiple test projects, at least for unit test? Should those test projects go hand-in-hand with the original project? 


## CrmCore

I bumped into this [repo](https://github.com/vietnam-devs/crmcore) while digging for GraphQL and was amazed by how they organize the web application. The basic idea is to use the combination of *.targets and directory.build.prop to split a big web application into many smaller pieces. Traditionally split code is easier but split RazorPages and Views are harder. I think that'll be useful if we want to create separate teams to handle different parts (feedback,  tasks.... ).   


## Misc. 
* Multi-tenancy with Split Database

I ran into an [interesting article](https://dzone.com/articles/multi-tenant-api-based-on-swagger-entity-framework-1) and thought their way to handle multiple tenant/split databases might be useful if we decide to go down this path. 

* Single Repo vs. Multiple Repos

From the eShopOnContainer repo, all services are within the same solution, thus share the same CI pipeline.  However, there are more discussions around should we allow those micro-service be build/deployed separately and support version mismatch. (http://meuslivros.github.io/Building%20Microservices%20-%20Sam%20Newman/text/part0008_split_003.html)








