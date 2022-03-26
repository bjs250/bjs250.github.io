# A centralized notification audit log

## Background

Drift is a B2B "conversational marketing and sales platform". It's main offering is to install a virtual assistant on a client company's marketing website ("a chatbot") to interact with site visitors, targeting two main scenarios: site visitors that want to initiate a business deal with the client company, and site visitors that require technical assistance from the client company:

<p align="center">
  <img src="https://user-images.githubusercontent.com/27317800/159860362-43b4ecc7-c0c1-43b2-acb2-725e81a9aa58.jpg" width="300">
</p>
<p align="center">
  An example of a Drift chatbot on the client company, Peloton
</p>

While chatbots are capable of inititiating basic conversation with a visitor and answering primitive questions, at some point it often becomes important for a real human being to be able to jump into chat with a visitor. In the Drift app, we have an inbox that sales reps can visit, which contains all the conversations that are currently happening between site visitors and chatbot instances on Drift.com

<p align="center">
  <img src="https://user-images.githubusercontent.com/27317800/160237279-1491e85b-34fb-4922-af80-23cbfd856ae4.jpg" width="300">
</p>
<p align="center">
  The Drift.com inbox
</p>

But life as a sales rep is busy, and no one is going to sit in front of the inbox all day waiting for replies. So how does one get back to a chat when a reply is warranted? The answer is notifications: when a site visitor replies to you in chat, you're sent a push notification which you can follow back to that conversation within the inbox.

Now, there were two main channels we were using for notification delivery at Drift:
- Browser
- Mobile

<p align="center">
  <img src="https://user-images.githubusercontent.com/27317800/160237111-277a1398-7b8d-44b6-82c8-d6e17ebc515f.jpg" width="300">
</p>
<p align="center">
  A browser notification
</p>

<p align="center">
  <img src="https://user-images.githubusercontent.com/27317800/160237117-13076522-fd5a-436d-8a1a-925440e118c1.jpg" width="300">
</p>
<p align="center">
  A mobile notification
</p>

## Job to be done

Notifications are an incredibly important and useful tool for getting users back into the product at the right time. The problem was...well, there were too many problems to count! Dispatching notifications was a lot like tossing bottles into the ocean. We were sending them to users, but didn't know if they were making it to them. We also didn't know if users were actually using them.

Then we started to get triage. A lot of triage...

Customers would constantly tell us notitifications stopped working, or were delayed, or they got them on their phone but not in their browser (or vice versa). At the end of the day:

1. Engineers needed a way to measure the reliability of notification delivery and performance latency

2. Customer support needed a way to troubleshoot notifications for users

3. Product managers needed a way to measure the engagement rate with notifications

So this is why we built a centralized notification audit log. This would be an internal tool to assess the health of notifications in aggregate, and the outcomes of notifications sent to individual users.

## How notifications work

Browser notifications are built on top of the [Service Worker API](https://developer.mozilla.org/en-US/docs/Web/API/Service_Worker_API), which is generally a Javascript file responsible for processing notifications. You can register a service worker to a domain (e.g. app.drift.com), and when the user's browser accesses a web page on that domain, it will download and install the service worker snippet. You can then use the [Push API](https://developer.mozilla.org/en-US/docs/Web/API/Push_API) to control how you consume notifications.

There's some more complications around subscriptions, but I'll omit that for the time being. 

<p align="center">
  <img src="https://user-images.githubusercontent.com/27317800/160240377-e02f7b50-2053-45e2-9d7f-6705634b2053.jpg" width="800">
</p>
<p align="center">
  A simplified version of notifications
</p>

## Functional requirements

As an engineer:
- I want to know if my service dispatched a notification (or didn't!)
- I want to know how long it took my service to process the notification-generating event
- I want to know the duration between the underlying event happening and the user seeing the notification
- I want to know if the user's browser got the dispatched notification (delivery rate)
- I want to know if the user's browser rendered the notification (shown rate)
- I want these metrics in aggregate!

As a product manager:
- I want to know if the user interacted with the notification (click rate)
- I want these metrics in aggregate!

As an customer support specialist:
- I want to know where in the notification delivery pipeline things broke down for a user
- I want to know what I can do to put a user back into a good state

## Technical requirements

Notifications are one of the highest cardinality objects at Drift. Most services that interact with users end up dispatching them, and the total throughput of all notifications going through Drift within a week is in the millions. Since this is an internal tool, we can make some performance compromises that we wouldn't typically make with customers. Likewise, this isn't necessarily an alerting system

- We must handle many more write operations than read operations
- We need to store a large number of records cheaply
- We want to be able to retrieve an individual user's notification audit trail quickly
- We don't need instant access to notification information in aggregate 
- We don't want audit logging to add any overhead to notification delivery

So there is a bit of an inherent tradeoff here. DynamoDB is an ideal datastore for cost of storage (and an expiry policy can be set) and also easy lookup of individual information, but poor or difficult to use for querying in aggregate. Autoscaling will let us deal with heavy write activity. It's for these reasons we'll choose it over a relational database, but there are significant cons here.

## Technical design

### A new microservice

The first thing we need to do is move from a world where each microservice manages its own notifications...

<p align="center">
  <img src="https://user-images.githubusercontent.com/27317800/160254696-02eeae80-7f32-443a-a5a1-0f74f8a3ffef.jpg" width="500">
</p>

To a world where all notification requests go to one place before being sent outbound to users. This new service will be responsible for managing interactions to the audit datastore in DynamoDB

<p align="center">
  <img src="https://user-images.githubusercontent.com/27317800/160254720-2ce950ee-f138-4229-970e-174649fdac39.jpg" width="500">
</p>

### An audit trail model

A second major question is what the audit trail object looks like. We can thing of an audit trail for a notification as a series of events. But this model quickly becomes problematic. Considering the following case:

<p align="center">
  <img src="https://user-images.githubusercontent.com/27317800/160255049-e521a478-cc4e-4264-8086-0d1f3135a51d.png" width="500">
</p>

A single generating event (a message in a conversation involving multiple participants) results in a tenfold increase in events that we care about (ultimately resulting in delivery to 4 channels across 2 users). Given the scale of notitifications, the number of indidividual audit trail events can become unbounded if each of them is a record in a DynamoDB table.

For this reason, we prefer representing an audit trail object as a 1:1 relationship between a generating event and a receiving user. So in this case, 1 generating event would result in 2 audit trail objects (and 2 records in DynamoDB), one for Alice and one for Charlie. We can do this by growing the record columwise arbitrarily:

<p align="center">
  <img src="https://user-images.githubusercontent.com/27317800/160255278-7c46fbf6-fb06-40a7-9895-1283057ece29.jpg" width="500">
</p>

Now, we probably also want timestamps on these individual events as well. The client that reads and visualizes the audit trail for users will have some non trivial work to do but...well that's why developers get paid. There is no free lunch

### Audit events

Remember that one of the reasons we want to do this is to have some observability into the service worker. We can now do this by exposing an endpoint within the notification service that will upsert audit log events onto audit trails records in the DynamoDB table. Consider the "happy path" below:

<p align="center">
  <img src="https://user-images.githubusercontent.com/27317800/160255741-823588aa-ad10-4df2-ac19-89d88d80275b.jpg" width="700">
</p>
<p align="center">
  The service worker can write receipts when certain things happen
</p>

