# Introduction

The current cloud ecosystem has made huge strides in making powerful computing resources accessible. There is a cloud service for almost any use case that you can come up with and using it is as easy as plugging it into your application.

However despite the advances, the cloud ecosytem is lacking tooling to reason across these different services. This makes it difficult to create applications that are correct (i.e. respect invariants, satisfy property based tests across distributed components), secure, efficient and that align with business requirements. Briefly some of these shortcomings -

* Infrastructure declaration, configuration and application logic exist in **separate paradigms** - AWS CDK is making great progress in fixing this [^2](#design-and-implementation-of-aws-cdk).
* Infrastructure components are **not derived from application** logic - Punchcard project is trying to solve this [^5](#5-punchcard).
* Most cloud services do not allow defining **application constraints** in terms of business metrics, for e.g. latency, throughput, cost, availability. - Kubernetes has some concepts of constraints, when it works within user defined distruption budget for new deployments [^6](#6-kubernetes-disruption-budgets).
* Cloud systems lack tools to reason about performance bottlenecks, estimate cost and check logical invariants across multiple components **without running the application**.
* Cloud systems lack tools to generate the permissions required to a run an application **from the application logic** itself. This makes the process of creating secure policies tedious and very error prone.

Let's consider the journey of a fictional (is it?) startup called YAP (Yet Another App). They are making an app that allows users to upload photographs and find similar photographs. We will see the kind of challenges YAP faces as they grow and expand on the cloud.

## Constraint based programming [^1](#1-new-direction-in-cloud-programming)

### Story

YAP bootstraps itself on a low budget and starts with Functions as a Service (Faas - AWS Lambda, Azure functions), a database and a blob storage (for photos). Their users are mostly local and the application can easily handle the peak 2000 req/s  that come at it. User activity is very bursty and can often be as low as 10 req/s, using stateless functions to handle requests YAP can scale their costs linearly with the number of requests.

Soon YAP starts getting popular and is used across the country. It's daily traffic steadily increases and they start hitting bottlenecks. To handle bursty traffic between 10,000 req/s and 30,000 req/s they start serving traffic through stateful servers running in Docker containers. Kubernetes like orchestration services allow them to easily scale up and down their compute capacity with cost scaling linearly with traffic.

In a few months YAP goes viral and they start getting users from across the world. They find that photos search requests are have regular patterns based on geographic location. They want to keep their end user latency to be less than 1.5 secs and setup CDNs for major locations.

YAP has now become a popular global search service and consistently gets 100,000 reqs/s. YAP now decides to use dedicated hardware to serve consistent traffic efficiently. Bursty traffic can still be handled through autoscaling and setting appropriate thresholds.

In YAP's journey we see business requirements and user behaviour drive architectural changes. The application logic constantly has to adapt to integrate with architectural components that are best suited to the situation as YAP usage grows.

### Idea

A modern programming system should understand changing requirements from high level constraints and choose appropriate components. Here's a table mapping the relationship in the above table.

| Situation | Component |
| -- | -- |
| 1000 - 3000 req/s | stateless functions |
| 10,000 - 30,000 req/s | servers running in containers |
| 1.5 secs end user latency | CDN |
| 100,000 req/s | servers running on reserved VMs with thresholds and autoscaling |

While this is a straight forward real world components modelling will consider numerous properties including cost, throughput, latency, integrations, operational overhead. A programming system with this information will be able to derive the architectural components from the application annotated by constraints.

## Automated least priviledge security [^4](#4-zelkova) [^7](#7-capabilities-effects-for-free)

### Story

YAP stores photographs submitted by users for queries. It guarantees users that the data is only used for queries and then deleted from their servers. Moreover YAP also maintains a database of uploaded photographs to search against. They want to keep their user data and database secure.

YAP creates a security policy for their query server. So that even if an attacker gets into one of query servers, they won't be able to delete or bulk download all the images since the runtime environment does not have those permissions. However a new component got added for the caching layer and because of the urgent release deadline they were not able to create a least priviledge policy in time and deployed the service with basic checks.

### Idea

A common theme across cloud provider is ensuring least priviledge access. This means giving an entity only the permissions required for it to perform it's actions. The main reasoning behind this is that cloud runtime environments can run arbitrary code. A system with extra priviledges can potentially be used as an avenue for attacking the other parts of the system. Currently this is process of figuring out required permissions is completely manual and extremely tedious and error prone.

A modern programming system should generate the permissions from application logic. It can analyse the application and it's dependencies to find the set of external API calls being made. Constant values used as parameters can be used directly while dynamic values can be kept as parameters. This set of API calls can even indicate what kind of security the application needs and determine the best runtime environment for it.

Capability-effect systems are another technique used to reason about the effects of a piece of code. It can also be used to check if the logic is in violation of it's given capabilities by making incapable effectul calls. The challenge with these systems is to balance between verbosity i.e. every capability of every function is declared vs observability i.e. capabilities are declared at a high level function/api and lower level functions do not need furhter annnotation reducing cognitive burden. An addition benefit of such a system is that it can be retrofitted on to existing type systems and even be modified to use with languages without type systems.


## Reasoning about systems better [^2](#2-design-and-implementation-of-aws-cdk) [^3](#3-protocol-combinators)

## Story

YAP has developed a sound business model where they let other companies upload their photographs to their dataset. When user search for the photographs they show up with the companies branding.

The dataset submitted by third party providers runs through to pre-processing and vettings stages before being accepted. The pre-processing metadata is written to a persistent key value store, the analysis stage generates tags for the image which is commited to a graph database and rejected images are stored in a file storage to be sent back to provider. YAP wants to ensure the **sequence of steps is maintained across multiple components** (to avoid legal headaches obviously 😝). However the number and complexity of the integrations make it very difficult to maintain the test cases.

Furthermore some third party vendors want to integrate the YAP search with their own applications. They want to create an image query and then get notified when any new images matching the query are uploaded. However each vendor have various limitations on the amount and rate of traffic they can handle, so YAP builds in rate limits and circuit breakers patterns for the different vendors. In this complex architecture, YAP finds it **difficult to reason about performance bottlenecks and when and where they can occur in the application**, especially when adding new integrations.

Furthermore now YAP has become a multi-tenant search engine while itself running on top a multi-tenant infrastructure provider. YAP is facing considerable difficulty understanding the cost of onboarding new tenants, **modelling the cost of building new features** across tenants. This is further complicated cloud providers have granular charges (upto per request or per GB consumed cost) and tenants have diverse traffic patterns.

### Idea

The three problems mentioned above, namely reasoning about performance, enforcing invariants and reasing about cost are inter-related and point to the lack of analysis tools that reason across cloud components. Most of these properties can only be determined by through logs, traces and metrics generated by running the system which is costly and error prone.

A modern programming system should do allow the following -

* Declare logical invariants for the data flow specification. Simluate the application as a state machine to check if invariants are satisfied. Derive the working implementation from the specification [^3](#3-protocol-combinators) to ensure there is no gap.

* Perform data flow analysis and consider performance characteristics of various architectural components to identify bottlenecks.
* Use cost models of various architectural components and sample data traces or probabilistic traffic patterns to derive cost estimates without deploying/running the application.

In cloud environments, small changes can have big downstream impact. The idea is that simulations and static analysis tools, can help analyse the difference a change will make. Adding a new service can be modelled as changing the flow of data through the service. The changed flow affects performance and associated cost. Better programming systems can help architects and designers in understanding changes better by showing the delta with current behaviour. This can let them iterate faster and make more informed decisions.

## Dynamic application logic based on infrastructure state

### Story

YAP has found a clever way to deal with peak traffic without spending etc money. YAP dynamically changes their query logic accuracy based on the traffic they are receiving. When they have auto-scaled to 8 query server instances and have reached the 70% CPU utilization threshold they drop search accuracy from 95% to 90%. They cannot compromise on end user latency, but the 5% drop in accuracy is barely noticed so they use this technique to handle brief bursty traffic without incurring extra cost.

### Idea

The underlying runtime state needs to expose properties to the application logic. Then the application logic can change it's behaviour based on the state. Coupled with some ideas mentioned above like checking invariants and static analysis this approach will make it possible to truly utilize the cloud system as an extension of the application.

# Discussing the solution

The above ideas point towards creating a language and runtime more suited to tackling the challenges for creating applications on the cloud. This system might have some of the following features -

1. Types that also model performance, availability and similar "auxiliary" properties. Any component is composed a few core types and inherits from their auxiliary properties.
2. Type checker can solve constraints to find approprirate representation for types.
3. External API that are modelled as effects and are part of the function/solution signature. Making it possible to derive the permissions required by application runtime.
4. Core network, storage APIs are defined and have polymorphic implementations to derive working implementation, state machine implementation and invariant checking implementations from the same specification. [^3](#3-protocol-combinators)
5. The high level specification generates a human readable DSL that can be changed to make specific optimizations or changes. [^2](#2-design-and-implementation-of-aws-cdk)

# References

#### 1. [New direction in Cloud programming](http://cidrdb.org/cidr2021/papers/cidr2021_paper16.pdf)
#### 2. [Design and implementation of AWS CDK](https://github.com/aws/aws-cdk)
#### 3. [Protocol combinators](https://ilyasergey.net/assets/pdf/papers/dpc-jfp.pdf)
#### 4. [Zelkova](https://www.cs.utexas.edu/users/hunt/FMCAD/FMCAD18/papers/paper3.pdf)
Zelkova can create permission policies based on historical usage patterns. It's still missing a way to generate policy by statically analyzing code.
#### 5. [Punchcard](https://github.com/punchcard/punchcard)  
#### 6. [Kubernetes disruption budgets](https://kubernetes.io/docs/concepts/workloads/pods/disruptions/)
#### 7. [Capabilities Effects for free](https://link.springer.com/chapter/10.1007/978-3-030-02450-5_14)
