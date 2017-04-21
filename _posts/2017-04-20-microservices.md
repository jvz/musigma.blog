---
layout: post
title: "Why Use a Microservice Architecture"
date: 2017-04-20 09:00:00 -0500
categories: architecture
---
In previous posts, we've gone over some newer frameworks and libraries for
developing reactive microservices. Today, we'll take a step back and look at why
developers should even consider using microservices in their projects.
Microservices are a general category of software architecture that are loosely
inspired by service oriented architecture (SOA) common to previous protocols
such as SOAP, but with a focus on physical separation between services. While
the general idea behind SOA that code should be organized and structured into
loosely coupled services was sound, each around a particular domain or category
of business logic, the execution left something to be desired.  Common
implementations of SOA relied on monolithic deployments, overly complicated
protocols that were never implemented quite the same from framework to
framework, and generally required expensive software licenses from established
IT companies.

Due in part to the existing complexity of SOA usage, many developers eventually
found themselves trapped by applications that could not be iterated on rapidly
in an agile development environment. Deployments required complicated build
systems easily described as "magic", and entire teams would need to be formed
just to ensure a single deployment worked properly. This sort of process worked
well enough in the waterfall development days (or at least as well as waterfall
development worked itself), but as the software industry migrated to agile
development methodologies, the friction of maintenance, deployment, and
onboarding new developers into overly complicated monoliths became untenable.

At this point, some developers began experimenting with what would later be
named microservices. Standardizing on the ubiquitous web standards, individual
services were written to communicate over HTTP using XML and later JSON, relying
on extensive library support for these standards. This borrowed the concept of
RESTful APIs, where services provided resources which were identifiable by URIs,
and each resource defined HTTP methods (or verbs) that could be performed on
those resources. There are other ideas on how to implement microservices besides
via HTTP (e.g., OSGi in the Java world provides their own style of microservices
which can communicate in-memory, but are still physically separated by the JVM
via separate ClassLoaders), but the general concept behind them all is that each
service requires physical separation and must communicate over well-defined
boundaries.  This provides several advantages over traditional monoliths, many
of which we will discuss.

It's important to note before we go into detail that in general, microservice
architecture provides far more benefits the larger an organization becomes.  A
small enterprise with only a single team or two working on these services may
find it overly complicated to break their systems into microservices,
particularly with purely CRUD-style applications. Many of the advantages
discussed here can also be extended into small monolithic applications with
proper developer discipline towards modularity, loose coupling of functionality,
and high cohesion of individual units of code as organized into modules.

First of all, breaking up a monolith into smaller pieces simplifies the
individual build and deployment systems of each microservice. While the overall
complexity of the system may increase, the savings in developer time and pain is
well worth the cost. Each individual microservice is much simpler to understand
in isolation, and in theory, this decreases the amount of time it takes to
onboard new developers into a system. This does generally require the adoption
of devops into an organization, but this is generally considered a good thing to
adopt regardless.

By breaking into microservices, another advantage is provided in that each
service may have different technologies that are more appropriate to use
depending on the domain in question. For example, simple CRUD microservices may
be more rapidly developed using a language like Kotlin, Ruby, or even
JavaScript, while more complicated asynchronous message-oriented services may be
more appropriately developed in Scala, Elixir, or another functional programming
language. Choosing appropriate frameworks for each service is also an important
advantage as developers no longer have to rely on overly broad enterprise style
frameworks. This allows more room for experimentation with newer technology and
design patterns which can then be shared with other developers to help improve
the overall system.

An important consequence of the physical separation between microservices is
that it makes it more difficult to create minimally cohesive systems. No longer
can code be simply dumped wherever the developer feels like it. Instead,
reusable libraries can be promoted where appropriate, and business logic remains
exactly where it belongs: within its appropriate domain. This can also provide a
prime opportunity for an organization to contribute back to the greater
community via open source software with said common library components.
Developers must make a conscious decision to make remote calls into other
microservices, so this helps enforce loose coupling. Even when developers
attempt to tightly couple microservices, this problem becomes apparent much
sooner rather than at the most inopportune time when a service hits a spike in
usage one day.

This leads into another advantage of microservices: the ability to independently
build, deploy, and scale these services. As alluded to before, build scripts can
be immensely simplified just due to the decrease in the amount of code needed to
be packaged together into a single distributable artifact.  Deployments can be
simplified out of necessity of supporting deployment of numerous microservices,
a goal that is often ignored when deploying monoliths. As for scalability, each
microservice can be independently copied and deployed multiple times across a
cluster of servers. In order for microservices to communicate in this scenario,
a form of client or server load balancing is required so that all instances of
each microservice can be evenly used. This relates to the elasticity aspect of
reactive architecture where services can scale up and down to meet the physical
requirements of that service at any given time. In general, scaling horizontally
like this is simpler than vertical scaling (improved hardware) or attempting to
use larger and larger thread pools where applicable.

There are disadvantages to using a microservice architecture, however.
Deployments are a double-edged sword in that they may be simpler on their own,
but in the real world, this doesn't negate the need for communication with other
teams. Due to limitations in creating RESTful services, maintaining backwards
compatibility can become difficult. Since REST APIs aren't exactly statically
typed, programmatic refactoring between services is not very possible at this
time.  If a backwards incompatible change is required in one service, then its
dependent services may also need to make a deployment at the same time with
updates, and this can slowly lead back into the problems of coordinating
monolithic deployments. This problem is being slowly solved through newer
RPC-style protocols that generate clients which can be programmatically
refactored, along other approaches.

A common pitfall when breaking a monolith into microservices is splitting them
up too finely. This can be compared to a similar problem of dropping the use of
relational databases purely in exchange for using a NoSQL style database.  While
certain domains may find this pattern lends well to use, other domains are
relational in nature and can't be physically decoupled without introducing
numerous performance problems. If this happens, though, not all is lost! Using a
distributed trace logging library such as Zipkin can help identify tightly
coupled microservices and performance issues in context of the physical flow of
data in relation to particular requests or events. After identifying these parts
of a system, they might be rejoined, or the overall architecture of how data
relate to each other can be revisited.

Another difficult issue in microservice architectures is that the divide between
frontend and backend needs can become ever greater. While the backend attempts
to remain logically pure, frontend needs begin to clash with UI requirements
that join data from numerous microservices into a single page.  Supporting such
UIs is generally much simpler in monolithic applications, but the microservice
world is quickly researching and experimenting with alternative approaches to
exposing APIs to applications.

An approach that stays pure to RESTful APIs is to layer APIs similar to how code
may be organized in a traditional SOA application. The base layer of APIs is
made up of the individual microservices in a system. Another layer of APIs can
be created to compose those microservices into more specific aggregate domains.
A third layer of APIs can then be added on to expose a more stable, UI-centric
API that in essence translates all the underlying domains into synthetic ones
that are relevant to the user experience. This approach is generally more
complicated, of course, but composite API layers, a common design pattern from
enterprise application integration, can be easily implemented using integration
libraries. This also allows for more intelligent caching and prefetching of data
based on common usage patterns.

Another approach is to use an alternative method to interacting with APIs such
as via GraphQL. Instead of exposing RESTful APIs directly to the frontend, a
GraphQL endpoint is provided which can be used to specify the data structures
required for a page along with filters on individual fields of the data, and
individual fields can be updated via the same mechanism. While this is not
RESTful in the slightest, it does tend to work better in practice when defining
an ever-changing API that both frontend and backend developers can agree on.
This also makes it much simpler to refactor individual microservices without
disrupting the frontend as the GraphQL server itself performs the data
translations into the requested structure. This also provides an interesting
opportunity for caching and prefetching of data.

A fully asynchronous approach (which does not currently have a buzzword name
that I'm aware of) may be to fully rely on using message queues and topics from
frontend to backend. This idea was original inspired by a joke I made with old
co-workers about implementing an "everything" API endpoint that would simply
return all the user data we know about so that the frontend developers would
stop requesting more and more combined REST APIs. To take this idea to its
logical conclusion, the only realistic way I could think to implement it without
an extremely long response latency would be to interpret the request as a
message, and then, using websockets, individual pieces of the user's data could
be sent back as messages which would feed into a Redux-style state management
library which would fill in relevant UI elements as it loaded.  While this
approach is not exactly realistic, the underlying idea of passing messages
between the frontend and backend is interesting and should be considered as
another alternative to traditional synchronous request/response APIs.
