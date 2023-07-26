# One Month Design Proposal

## Performance Requirements

**Expected Traffic:** 100,000 Daily Active Users (DAU)

**Peak Time:** 12:00 PM to 2PM and 6:00 PM to 9:00 PM (local time)

**Peak Usage Stats:** 50,000 Concurrent Users

**Response Time:** 500 milliseconds for essential operations during normal traffic

## High Level Architecture

The first iteration of the architecture is relatively simple. In order to keep both implementation and
architectural complexity low, the idea is to build everything in a monolithic codebase that will run
in the same server process. Even though we will keep code co-located the service will be designed as
if it were implemented in different, independent services. Initially this could be in the form of different
modules split across domain boundaries. This service can be deployed as a docker container in a managed
container service like AWS ECS/Fargate or Google Cloud Run or even as an AWS Lambda function deployed
as a container.

## Domain Boundaries

The main services or modules that will be implemented and integrated together will be an Aggregator
Service (Aggregator) and various Social Media Services (SMS).

### Aggregate Service

The job of the aggregator is to act as a gateway to the different SMS by receiving requests from the
user, determining which social media feeds the user wishes to see, make the appropriate calls or requests
to the specified SMS and aggregate the feeds together to be returned to the user.

### Social Media Service

A Social Media Service will act a s a proxy to a specific social media API while implementing a common
interface. The implementation of each service will be responsible for making API read requests for a
user's social media feed, transforming the data into a shared data model, and returning it to the caller
(Aggregator in most instances).

Different social media API's represent data in very different ways. One API could provide its data in
a graph based model, while other could have a flatter, resource oriented data model. This is the key
reason an SMS will need to transform its data to a more commonly understood data model.

#### Pros

By implementing a common API interface for each SMS we can achieve a simple integration process into
the aggregator. Since the SMS is concerned with the different complexities and implementation details
of its associated Social Media API, the Aggregator will only need to know of its availability to be called
and make requests like it would for any other SMS.

This also allows us to swap in new implementations for an SMS without much trouble as well as build
mock services for development and testing purposes.

#### Cons

Designing a common interface for a wide range of social media data models will introduce some complexities
as we try to achieve a scalable unionized data model. For that reason, extracting only the data needed
for aggregating different feed types is necessary when transforming the data and the raw data can be included
as-is for the client application to determine how its used.

### Data Model

Below is simple concept for a Data model representing an Aggregator Service and the transformed social
media feed data it aggregates.


```ts
// Collection of feed items from SMS
type Feed = {
  service: string; // Identifier for the service this feed came from (Spacebook, Photogram, Shreddit)
  cursor: string; // Ending feed item identifier used to retrieve next feed page;
  items: FeedItem[];
}

// FeedItem acts as a common data model that each SMS would transform its feed items into
// from its API response. This is simply for example purposes and would include much more
// data.
type FeedItem = {
  timestamp: Date; // Timestamp used for aggregation sorting
  service: string; // Which service did this feed come from
  data: object; // Raw data received from the SMS
}

// An SMS would implement this by making direct API calls at first but could
// be refactored to make those calls to an SMS proxy service running in a different
// container/cloud in the future
interface SocialMediaService {
  getFeed(userId: string): Feed;
  auth(token: string): unknown; // this will be covered in a different section
}

type AggregateFeed = {
  items: FeedItem[];
  cursor: string; // Base64 encoded json object describing which page of each service was included
}

interface AggregateService {
  getFeed(cursor: string): AggregateFeed;
}
```

## API

My initial inclination is to expose the aggregator service as a simple REST API. My other proposed option
would be GraphQL.

### Rest API

I believe this would be the best starting point. Ideally there are only a handful of endpoints that
would need to be implemented at first. This would also allow us to version the API in the event of
major changes down the road without breaking all clients.

#### Endpoints

##### `GET /feed`

Retrieves an aggregated feed of all social media services a user has requested.

This API would be paginated by optionally including a cursor for which the next page should be returned.

##### `GET /:service/post/:postId`

Gets a full post for a specific service.

### GraphQL

I do not believe GraphQL would be a good API interface for the Aggregator service. Given the broad range
of data models provided by each social media provider, the value of GQL's type safe schema would be
diminished, or, at the very least, overly complicated.

## Auth

For the initial design of this system, Authentication to the various Social Media integrations should
be kept relatively simple and will evolve into a more complex, secure, and reliable auth system in the
future.

The Aggregator client (app or website) will keep track of the authentication credentials for each
service a user wishes to see their social feed from. When making a request to the aggregator backend,
the secure auth credentials should be included via a request header property for the back end to use
when making the request to each SMS.

**Example**

```json
{
  "Spacebook-Auth-Token": "<SHORT LIVED USER AUTH TOKEN>",
  "Shreddit-Auth-Token": "<USER AUTH TOKEN>",
  "Twitter-Auth-Token": "<AUTH TOKEN>",
}
```

In this example, the user would like to aggregate their Spacebook, Shreddit, and Twitter feeds, but
notice they are not interested in receiving their Photogram feed. The exclusion of that auth token
will be enough for the Aggregator service to no request the feed from Photogram.

## Observability and Monitoring

This service should include standardized and extensive logging for information, warnings, errors, and
fatal issues. This logging will be integrated with the logging features of the chosen host (Cloudwatch,
GCP Logging, etc.). The logging libraries used throughout the project project should include information
such as timestamps, request ids, log level, and relevant information for its context. GCP and AWS both
include various tools for tracking metrics and errors which would provide sufficient warnings and other
important information.

### Logging Standards

* Timestamp
* Request ID
* Outgoing request timing
* Incoming request timing
* Incoming request parameters
* Descriptive message to provide context
* Log level

## Scalability

The service will be deployed so that it can be auto-scaled to meet the user load at any time. Thus far,
this service is completely stateless, thus horizontal scaling using the tools made available by ECS/Fargate,
Lambda, or Google Cloud Run should not introduce an extensive level of complexity.

Using the observability tooling and metrics outlined above, we will be able to fine tune scaling threshold
like min/max CPU and memory usage as well as request loads.

## Deployability

Deploying this service should be as automated as possible. All provisions needed to run the aggregator
in a production environment will be done so using Infrastructure As Code tooling like [Terraform](https://www.terraform.io/)
or [Pulumi](https://www.pulumi.com/)

Using a tool like Terraform will allow us to keep track of any changes to the infrastructure while also
providing an extensive and expressive API that can be learned by anyone, thus reducing the need for
any one person to hold some specific knowledge about how the service is deployed and provisioned.

![Month One High Level Diagram](./assets/MonthOneHighLevel.png)

## Updates

### Time Based Notifications

What is all in this activity summary?

I'm thinking some of the posts that had the most engagement that the user might have interacted with
will be included in this email.

How do we track that?

I think at this point we need to introduce some state or storage for this piece. Given the 12 month
requirement of allowing users to bookmark content, this would be a safe bet to implement now.

### Quick Ideas

When a user makes a request for a feed store the aggregate feed in a database or cache.
Emit an event like `user.feed.aggreaged` to which a subscriber can retrieve that feed data and analyze
it for engagement and popularity. This could then update some other table or cache record to create
a digest for the user for that day (or other time period). By filtering and ranking posts by engagement
levels we can start to curate a daily digest for them.

At the end of each day or time period (per time zone) we can query for the curated digests and create
emails or notifications to send to the user.

### Account Storage

We will need to expedite the storage proposal from the 12 month version into this version by implementing
a database for the user to create an account in, register social media accounts, and might as well implement
the new version of auth token storage.

This will allow us to tie digests to the user as we curate their daily summary emails.

### Polling

In addition to the events published when a user manually fetches the feed, we will have a polling job
that requests new content from each social media provider that a user is registered for. This is essentially
the same process just performed automatically.

I would propose instead of every 4 minutes, we run this every minute but separate the user base into 4
secions by some sort of identifier, location, or other data point too keep each job less intensive.

![Digest Email Infra](/assets/UpdatedDigestEmailInfra.png)
