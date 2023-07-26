# Twelve Month Design Proposal

This document will propose the architectural changes necessary to achieve the performance requirements
below.

## Performance Requirements

**Expected Traffic:** 1,000,000 daily active users (DAU)

**Peak Time:** 12:00 PM to 2:00 PM and 8:00 PM to 11:00 PM (local time)

**Peak Usage Stats:** 500,000 concurrent users

**Response Time:** 300 milliseconds for critical operations during normal and peak traffic

## Micro Services

In order to efficiently scale our service to a much larger user base, our architecture should grow as
well. The first goal of these changes is to incrementally migrate each SMS into its own, independently 
running service for the following reasons:

### Independent SMS Scaling

By deploying each SMS as its own service allows us to horizontally scale each service to fit their
needs more granularly and efficiently. If Spacebook is wildly more popular than Shreddit, the service
can auto scale to meet the needs of Spacebook users without the overhead of scaling the Shreddit services.
We can also make finer grained adjustments to scaling thresholds to better match the needs of each service.

### SMS Specific Technical Solutions

Given the features and API's of each social media provider are going to be different, they may require
vastly different technical solutions. Here are some example:

Spacebook could provide its data in such a format that our current tooling is not optimized for. By isolating
this service from the others, we can choose a different language for more efficient data processing or
resource usage without the need to overhaul all of the other services that might be operating at an
accpetable level.

Shreddit's REST API might be notoriously slow, but their GRPC API is blazingly fast. The integration
into a GRPC service requires new toolilng and possibly updated server configurations. By creating a
dedicated Shreddit API Proxy service, we would be able to expose the correct ingress and egress channels
in our ECS, Cloud Run, or Kubernetes Deployment without affecting the other services.

### Terms of Service and Compliance

We will probably be implementing some sort of storage or caching layer to reduce the amount of round
trips to external service, or to comply with rate limits imposed by a social media provider. That said,
Twitter might have offline data storage requirements that are not compatible with Photogram's or would
otherwise introduce an amount of complexity to our storage solutions that could destabilize the other
services. By separating each provider into its own service, we can more easily implement systems needed
to comply with each provider's Terms of Service.

This also touches on the previous two reasons as we might need to implement very specific webhooks,
stream processors, batch processors, or other solutions to comply with data retention policies like requests
to delete data or other requirements set by a social media provider.

Overall, the idea of creating a "micro-service" architecture and isolating each social media provider
gives us much more flexibility to iterate and optimize for a provider's specific needs and usage patterns
without interfering with how other providers are running.

## Storage and Caching

By migrating an SMS into its own service, we are introducing an additional network hop into our system,
and potentially adding more latency into each request from the user. My propsed solution for this is to
introduce a caching layer where retrieved data from a provider is stored with an appropriate time to live.

My initial design for this would be a Redis cache where a user's feed is stored as a fixed sized list,
where newer feed items push out older feed items. Upon request for the feed we can check our cache to
see if we have a page's worth of data and if we do, the data stored data has already been transformed
into an aggregateable form and can be return right away. If we do not have sufficient data, we will
make a request to the provider, transform the data, update the cache, and return the data to the caller.

This solution removes the need for as much transformation processes by each provider as well as removing
redundant requests that add latency as well as drive up requests in regards to rate limits.

## Auth Changes

To improve the security of our integration credential handling, as well as offer a smoother experience
for the user, I believe we should store long-lived authentication tokens provided by each social media
provider. This could be implemented in the Aggregator service since it acts as an API gateway, or in
its own independent Authentication service.

This could allow us to create a sort of single-sign-on service where authenticating for a social media
provider on one device essentially authenticates you into that service across all devices. Then instead
of having the user store and pass each SMS's auth token, they simply store and pass a JWT like token
that we create for them to authenticate into our service.

Once authenticated into our service, we can use their stored long-lived auth tokens for each provider
to make API calls on behalf of the user.

### Storage

Given the relatively small scope of this database I would propose using a hosted NoSQL solution provided
by the cloud provider like DynamoDB.

## High Level Topology

![One Year High Level Topology](/assets/OneYearHighLevel.png)

## SMS Service Topology

![SMS Topology](/assets/OneYearSSMTopo.png)

## Feature Updates

### Following

This can be impelemented by expanding the SMS Interface to include integrations with each provider's
followers API's and expanding the Aggregator's API to expose endpoints for a client to use.


### Bookmarking

I believe this could be as simple as expanding the database to include a "Bookmarks" collection or table
that would associate a user with a Bookmark record including the following data

- Social Media Provider (Facebook, twitter, ...etc)
- Post id for bookmarking
- Timestamp for bookmark


### Push notification management

I envision this as a way for the user to get notifications from certain accounts that they are interested
in. In that case, I would first look at each provider's API to find a webhook or other form of push-based
integration for us to take advantage of.

Otherwise we would build an additional service or module that tracks accounts to watch for notifications
and at some interval request those account's new posts and send those notifications.

### Multiple Accounts Per Platform

The proposed Auth solution above should support this with some slight modifications. By storing the
account information and long lived auth token for each account, we should be able to keep them signed in
and fetch content for each account.
