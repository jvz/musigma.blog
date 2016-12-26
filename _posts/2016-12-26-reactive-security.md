---
layout: post
title: "Security in a reactive microservices architecture"
date: 2016-12-26 09:00:00 -0600
categories: security
---
Applying the [reactive manifesto][manifesto] to microservice architecture is a
difficult problem to solve. One of the more difficult facets of this type of
architecture is designing secure microservices. One common way to secure
microservices is by using [JSON web tokens][jwt] to securely share
authentication information between services and clients. This standard is
great as the authentication information can be encoded directly in the token
instead of needing to be queried from a central location on every service
call. For example, upon logging in to an SSO page, the authentication service
can create a JWT using a private RSA key to sign the token, and this JWT is
returned to the client. The client can then use this JWT on subsequent calls,
and the integrity of this token can be checked against the public RSA key of
the authentication service. As a neat side effect, microservices do not need
to be proxied by an authentication gateway as the authentication information
is now self-contained and verifiable in a distributed fashion.

However, security is not only authentication (authn). Security also covers
authorization (authz), the concept of what roles a user has along with what
roles are required to perform a certain action. These user roles can be
encoded into the JWT payload in addition to the authentication info. This way,
each microservice can decode the provided JWT, verify it against the public
key of the auth service, and perform local authorization checks using that
information. Again, no centralized authorization proxy is required!

Notice, however, that this architectural design seems to rely heavily on
stuffing everything to do with authentication and authorization into a single
microservice. This means that all secured microservices need to share
information with the auth service. Well, in a reactive system, we still want
our microservices to work even when other microservices go down (resiliency).
There is also the concern that a typical auth service tends to become a
catch-all service for any cross-cutting concerns, so in designing a
microservices architecture, it's important to prevent such a problem from
evolving. To preemptively avoid this mess, it may be useful to separate the
concerns of authentication and authorization entirely.

One important architectural principle to follow when creating reactive
microservices is that each service should own its own data, and other services
should obtain that data through its public API rather than through a shared
data source. Such a design tends to create the opposite of relational data
normalization as much data tends to be duplicated across the entirety of your
services. In order to make microservices responsive and resilient, this
replication of data is completely alright. In that regard, every microservice
contains resources and actions on those resources, and each of those
resource/action pairs require a particular user role to be present in order
to authorize a user to perform said action on said resource. With that in mind,
it only seems natural that each microservice owns the authorization data for
each authenticated user.

Following this idea to its logical conclusion, let's design a reactive
authorization architecture. Recall that a reactive system is responsive,
resilient, elastic, and message driven. This architecture will have to fulfill
all of these assets.

## Message Driven Security
Beginning with messages, what would such a system look like? First, we'll have
an authentication microservice which owns a user login database. This service
will also be responsible for generating OAuth tokens whether they be simple
bearer tokens or JWTs. Upon successful authentication (e.g., via a web form),
this login service will broadcast a login event message to all interested
microservices. For a more concrete implementation, we could use
[Apache Kafka][kafka] to broadcast this login event. This event would be
encoded as a JWT, and the payload would contain an access token, a login
username, an expiration timestamp, and the requested OAuth scopes that were
authorized by the user. These scopes will later correspond to local
authorization roles in each service.

This login message might not be entirely necessary, but it does allow each
microservice to preemptively cache relevant user data that is expected to be
queried soon after login. Using a login message would allow for larger
payloads than allowed in an HTTP header which is particularly useful if a
large payload is required. This also reduces message overhead by allowing
JWTs to be accessed by access tokens. In general, the decision on whether to
use JWTs directly or indirectly is similar to the decision in certain programming
languages on whether to pass data by value or by reference to functions. Do keep
in mind that network traffic is more expensive than in-memory traffic, so the
more fine-grained roles you have, the better an idea it is to use JWTs
indirectly.

## Elastic Security
In this design, every microservice (including the login service) can be
independently scaled in resource usage in response to changes in input rate
to each service. The fact that we will not be relying on this service to
proxy service requests makes this a lot simpler to scale. Since this service
only contains a small amount of data for each user (along with some additional
role/scope configuration), replication of data is also simpler to handle.

## Responsive Security
As each service will be performing local authorization checks using data
sent to it in each request, the responses generated by each service should
not be affected by complicated authorization checks involving multiple
services or proxies. In order to maintain low overhead in authorization checks,
each service should contain a local cache of authorization information that can
be recomputed from a hard data store on startup (e.g., by utilizing CQRS
patterns). For example, we could use an event log to store the login message
broadcast earlier, and using a read side processor to maintain an in memory
table of unexpired authentication states. [Lagom][lagom] provides a great way
to design a Java microservice around this pattern.

## Resilient Security
This is perhaps the most interesting aspect of this design. The goal here is
to stay responsive even in the face of failure. For example, if the login
service crashes or is overloaded, we still want the rest of our services to
continue working for authenticated users during the time it takes to fix,
restart, or replicate the login service. To do this, the idea is to apply a
sort of inversion of control over a centralized directory server. In a typical
directory service such as LDAP, authorization checks are queried against this
central server. While easier to configure and easier to code against, this
provides a central point of failure. Instead, we'll design our services to
maintain their own local stores of authorization data. As each service knows
what roles are available for its APIs, these roles can be broadcast in a sort
of "role discovery" mechanism similar to service discovery. This feature can
be used for an administrative service for configuring which users are allowed
which roles. Said service is unneeded for continued operation of every other
service, so we maintain service isolation.

Each service is now responsible for maintaining two aggregate roots related
to security: user roles and login events. The login events can be uniquely
identified by access tokens for a low overhead message header, or JWTs can be
used directly. While this may sound like a bit of extra coding needed in
every microservice, it can be abstracted into a security provider library
used by every microservice. For example, a custom Spring
[`AuthenticationProvider`][authprovider] and [`Authentication`][auth] class
could be written to handle translations of these access tokens. The
[Spring Security OAuth2][ssoauth] library can be used to implement a lot of
this functionality, though custom code would still be necessary to
configure it to work in this distributed environment.

When a service calls another service, the access token (or JWT) is provided
in a message header. For example, an HTTP REST call would use an HTTP header,
while a generic message broker would use either a message header or a custom
payload format that allows for encoding message headers. For example, the
[Message Security Layer][msl] framework can be used for securely encoding and
decoding messages in such an architecture.

## Is This Overkill?
This security architecture started as a shower thought on how to implement
security in a reactive microservices architecture. As of the end of 2016, I
have been unable to find any resources online detailing any sort of solution
to this problem. However, it may be unlikely that most applications do not
need this level of resiliency if enough resources are thrown at a secure API
gateway proxy that all microservices are required to communicate with. It is
also possible to allow microservices to trust each other completely and perform
all necessary access control checks at the gateway level. However, any single
microservice can theoretically be compromised, and if all it takes is
anonymous access to one microservice, then that compromised microservice could
be used to perform actions on any other microservice. On the other hand,
proxying all inter-service communication creates a single point of failure,
though some people may find that acceptable.

Implementing proper security is a difficult problem to solve, especially since
the development process tends to follow the steps of: make it work, make it
right, make it fast, and make it secure. Security is oftentimes an afterthought,
and proper implementation of security also requires a carefully designed
balance between security and usability. If a security architecture is too
cumbersome to use as a user or developer, then users or developers will take
shortcuts around the system or else lose an unacceptable amount of
productivity. In that regard, it is important not to go overboard creating too
many different authorization roles, and it is also important to make use of
your security layer easy to use for developers. By localizing authorization
to each service, this makes it easier to test in isolation as well while
also maintaining some sort of security over time instead of waiting until
integration testing to find out that something is insecure.

[manifesto]: http://www.reactivemanifesto.org/
[jwt]: https://jwt.io/
[kafka]: https://kafka.apache.org/
[lagom]: http://www.lagomframework.com/
[authprovider]: http://docs.spring.io/spring-security/site/docs/current/apidocs/org/springframework/security/authentication/AuthenticationProvider.html
[auth]: http://docs.spring.io/spring-security/site/docs/current/apidocs/org/springframework/security/core/Authentication.html
[ssoauth]: https://projects.spring.io/spring-security-oauth/docs/oauth2.html
[msl]: https://github.com/netflix/msl
