# Architecture Council Review Pre-Read

## The 5 W's

* Who (People Manager): Felix Sargent
* What (Product Owner): Michael Welles
* When (Project Manager): Michael Welles
* How (Technical Owner): Aleksandr Pasechnik
* Why (Executive Sponsor): Wil Schobeiri


## Vision Statement

Enable MediaMath to distribute and share the rules behind the access of sensitive customer and corporate data with trusted services, so that we can build decentralized services with consistent security.


## Product Charter

The Authorization Service normalizes and standardizes MediaMath user permissions models into a streamlined, performant, and globally-distributed service.

* Represents the current Adama user authorization model
* Supports new, more flexible authorization models that reflect real-world use cases and feature requests from product teams and customers
* Simplifies the integration of user authorization into new service development through the creation of code libraries and APIs
* Accelerates decomposition of T1DB user permissions tables/flags


## Solving for the class

An overview of the proposed architecture can be found in the [the Architecture
document](./architecture.md).

Essentially, the Authorization Service:
* reads user/entity-related data from various Kafka topics
* applies permission rules to this data
* writes the results to two Kafka topics (User Permissions and Entity Permissions)
* makes the Kafka topic data available through a permissions API, for those servcies that don't want to access the topics directly


This service is intended to provide a solution for all of the user-based entity
operation authorization questions at MediaMath. We have validated our model
against a fair sample of the endpoints in Adama, covering both entity
hierarchies and separate root entities
([MediaMath/permissions-service-experiments](https://github.com/MediaMath/permissions-service-experiments)).
We can express all of the authorization requirements for the tested entities
within our proposed model and are confident that we can handle any other
realistic authorization requirements that might come up as well.

We are not solving the question of Validation, i.e. operations that are allowed
or not for *any* user, especially based on the specific details of the
operations and their context.

The authorization rules only cover operations on specific entities or
non-cyclical hierarchies of entities. There is no way to express authorization
on arbitrary sub-trees that are all the same entity-type (a subset of
`target_values`, for example), or specific fields of an entity. The former can
often be expressed more correctly from a business perspective based on an
orthogonal pointer (the `target_dimension` of those `target_values`). The latter
can be handled by splitting the entity into a generally-accessible parent and
a restricted child.


## High Level Roadmap

* Initial minimally functional system by Moratorium 2017
* MVP by end of 2017
* Adama migrated by end 2018 Q1 (POSSIBLY?)

We have 2 full time and 2 part time engineers on the team today working on this
project, plus John Napiorkowski's assistance for the updates in Adama code.


## Dependencies

The service depends on the following:

* the snapshotting system of the changes team
* the users and org-tree tables for basic the User Permissions topic, plus any other topic that needs an Entity Permission topic

It would be nice to have:

* hardware in EWR to be close to the Kafka cluster of the primary source topics
* Kubernetes for easier deployment and scaling of both the Rules Engine and the Evaluator API nodes
* Service Discovery for downstream users to find appropriate Evaluator API nodes

Things that depend on the service:

* UI Middleware
* Search system
* Adama authorization cleanup
* Adama decomposition
* All the things that need to know if a user has access to some entity


## State

Primary state storage is in Kafka topics. The User Permissions topic contains
the various permissions that each user has. The Entity Permissions topics have
various required data for making authorization decisions. The topics are
pointer-based topics. They do not include the full state of the original
entities from the source topics, only the parts required for the authorizaiton
service. New messages are published only when the authorization-specific values
are different from the last message published for the same entity.

This state is available to all interested parties that wish to read the
associated Kafka topics and perform the authorization decisions. We will start
by at least providing an evaluator-lib go library. We will either write or
support the development of libraries in other languages. We will also expose
simple API nodes wrapping the go library for any services that cannot handle the
Kafka-based approach.


## Third party vendors

None.


## Hosting

The Rules Engine nodes will be hosted in the data centers of their primary Kafka
topic's primary clusters. This is done to minimize the latency between source
message publication and the Rules Engine action and message publication based on
it.

This means that the main Rules engine nodes will reside in EWR to start.

Evaluator API nodes will be deployed anywhere there is a mirror of the
authorization service Kafka topics.


## Performance, Scaling, and Failure

Ideally a warmed up Rules Engine node should be able to process incoming
messages and get them published to its Kafka topic within reasonable deadline of
the original message availability. Thus the Rules Engine nodes should to be able
to keep up with the rate of the original publisher even during periods of
sustained activity.

**TODO**: do we have more specific numbers around this?

The topics produced by the authorization service are pointer-topic and messages
are only published when the authorization-specific parts change. This means that
Entity Permissions topics will likely get not much more than one message per
source *entity*, not per source entity message. The User Permissions topic will
publish new messages whenever the relevant User fields or user-entity links
change. This means that we can have situations where we produce a spike of
messages when a single `api/v2.0/users/:id/permissions` call is made which
changes a large number of user-org-tree links. This might be alleviated by
delaying message publishing a short time (within our SLO) to see if there are
any more updates which can be batched up together for a single user.

Being primarily a Kafka topic processor, we have the possibility to scale out to
the number of consumed topic partitions.

If we do not reside in EWR near the primary Kafka cluster, we may see
bottlenecks on the publish side due to network latencies and our confirmed
message publishing requirements. It may be possible to relax some of the
publishing confirmation requirement constraints if that becomes necessary.

**TODO**: Define something about the speed of the evaluator-lib and maybe the
Evaluator API

Since the Rules Engine has no permanent local state, nodes should be able to
simply restart if they fail and catch up if they lose connection to their source
topics.

The Evaluator API nodes will continue responding to requests with stale data if
they stop getting updates.

**TODO**: update the paragraph above with whatever we finally decide on with the
"staleness" definition and indication question.


## Technical Challenges

A generic-enough but still usable Rules Engine. Ideally we'd have a consistent
experience for both the User Permissions topic and the Entity Permissions topic
handling. The general approach of looking at messages and applying rules to
decide what a new message should look like has applications beyond just the
authorization service.


## Metrics

The primary metrics of interest for the authorization service are the
correctness of the rule evaluation based on our published topics, the
client-perspective processing lag, and evaluator-lib and Evaluator API response
speed. All of these need to be monitored and the lag and response speed needs to
be minimized.


## Migration

We are duplicating some of the capabilities of Adama around answering
permissions questions. Ideally this service would make the current
api/v2.0/users/:id/permissions endpoint unnecessary for internal services making
it possible for those services to ask the specific authorization questions they
are actually interested in.


### Adama authorization migration

Furthermore we intend to replace the various custom and default permissions
lookup code in Adama with processes based on the data generated by the
authorization service. This migration will be done in a phased way with feature
flags to slowly roll out changes for each controller and component to users
while monitoring Adama performance and correctness.

The User Permissions and Entity Permissions topics will be cached to t1db tables
and an equivalent of the evaluator-lib will be implemented for Adama controllers
to use. This will be a combination of functions and updates to the SQL queries
that Adama makes to complete requests.

Once the code is in place, we can incorporate it into the controllers for
specific endpoints (or even EntityCRUD), protected by user feature flags, test
at the Perl and API level, and then progressively expand the user pool. We can
observe what effect these changes have on the timing and correctness of the
responses from Adama relative to the current approach.


## SRE Resource

Nobody yet.


## Language

go ([see also this note about Kafka Streams on the JVM](https://github.com/MediaMath/authorization-service/blob/architecture-fist-draft/docs/architecture.md#kafka-streams-on-the-jvm))


## Validation of entity to entity links

There is an angle of "Authorization" involving the rules  which dictate which
entities may be linked to which other entities. These rules may be based on
Organizations or other org-tree elements. They may arguably be considered
Validation rules, as opposed to Authorization rules, since they deal with entity
linking without regard for the user performing the operation. But these
link-rule questions become increasingly more important as we break Adama into
separate services and as we build new services outside of Adama. (For example:
in order to decide which deals may be linked to a strategy, Adama has to make a
potentially expensive call to the Deals service.)

Though the Authorization Service will not answer these questions, it may act as
a seed for a future service with a mandate focused on the link-rules.

