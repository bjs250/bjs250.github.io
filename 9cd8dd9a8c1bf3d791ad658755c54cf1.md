# Designing a domain-agnostic commenting microservice

## Background

Drift is a B2B "conversational marketing and sales platform". It's main offering is to install a virtual assistant on a client company's marketing website ("a chatbot") to interact with site visitors, targeting two main scenarios: site visitors that want to initiate a business deal with the client company, and site visitors that require technical assistance from the client company:

<p align="center">
  <img src="https://user-images.githubusercontent.com/27317800/159860362-43b4ecc7-c0c1-43b2-acb2-725e81a9aa58.jpg" width="300">
</p>
<p align="center">
  An example of a Drift chatbot on the client company, Peloton
</p>

During the interaction between the chatbot and the site visitor, Drift collects, aggregates, and logs information about whom the site visitor might be (via third party intel software), the company they work for, and the conversation itself. This is fed into "Prospector", Drift's website analytics platform:

<p align="center">
  <img src="https://user-images.githubusercontent.com/27317800/159892391-eb567023-7428-4a68-8a6d-0ee302dc53f0.png" width="500">
</p>
<p align="center">
  An example of website analytics in Prospector
</p>

## Job to be done

One of Drift's main goals is to expand within client companies it already has a business relationship with. To do this, it needs to encourage more users within a client company's sales team to use the website analytics platform and get them using the platform habitually. One hypothesis for doing this was to add a commenting feature to resources within the analytics platform:

<p align="center">
  <img src="https://user-images.githubusercontent.com/27317800/159894437-df93e765-2e0b-429c-bfa4-62cc6521cc65.jpg" width="500">
</p>
<p align="center">
  Tagging another user to look at website activity
</p>

The idea here is that commenting will:
- Encourage users to get their coworkers into the analytics platform (expansion)
- Demonstrate value of the analytics platform to coworkers
- Get users back into the app and using the analytics platform

## Functional requirements

For this feature to be a success, these were the so-called "table stakes":
- Users should be able to comment and reply
- Users should be able to tag other users within their org
- If a user is tagged in a comment tree, they should be notified by push or email
- If comments are created in a comment tree in which the user is a participant, they should be notified by push or email
- A user should be able to control their notifications (e.g. silence them)
- A user's notifications should be persisted somewhere (so that they have a way of navigating back to the comment tree)
- A user should be able to edit or delete their own comments

## Technical requirements

- This pattern should be portable to almost any entity in the front end
- Notifications need to be persisted reliably, or user trust will be eroded
- Notifications need to be very quick, as users often want to act on the website activity ASAP

## Functional constraints

The role model here is LinkedIn, as opposed to something like Reddit. There are a number of UX simplifications we can make under that assumption

<p align="center">
  <img src="https://user-images.githubusercontent.com/27317800/159896472-4cd4c8f7-76df-4717-8a17-4e68ce7f2289.jpg" width="500">
</p>
<p align="center">
  Example LinkedIn comment tree
</p>

- The max depth of a comment tree is 1
- The total length of any given comment tree will be relatively short 
- Users want to be notified about any activity in the comment tree (this one in particular is key)

## Technical design

### Access patterns

- The entire comment tree for a resource needs to be retrieved and rendered by a user's browser, assuming they have access to that resource
- A user needs to be able to edit their own comment
- A user needs to be able to delete their own comment
- A user should be able to create a top level comment, a reply branch on the comment tree, or a reply in a branch

These requirements describe simple lookup operations, which suggest a NoSQL database like DynamoDB would be a good fit

### A comment tree model

Let's call any component throughout the Drift front end that can be commented on a "resource". In the example above, this happens to be a website activity card, but other product teams might want to implement commenting within their product domains. For this, we need to keep the comment tree model domain-agnostic.

As mentioned above, the most common access pattern will be to retrieve an entire comment tree pertaining to resource. For this reason, we want those records to be clustered together in the datastore we've chosen, DynamoDB. This can be done by assigning all the comments to the same partition key, which can just be the id of the resource that users are commenting on. The product team responsible for that resource type can use whatever kind of resource id they want!

Composite primary key:
- Partition key: resourceId
- Sort key: commentId

Great! But there's a problem. When we do a lookup on (resourceId), DynamoDB will just return our service a flat list of records. How can we transform that list into a tree? We can't just sort the list by timestamp due to replies in branches, as seen below:

<p align="center">
  <img src="https://user-images.githubusercontent.com/27317800/160234954-2598da8e-86d1-4a83-883b-2810bc819de7.jpg" width="400">
</p>
<p align="center">
  An example tree among three users
</p>

This can be solved by introducing 2 fields onto the model:
- Comment parent id
- Creation timestamp

To explain, we know that if we just had a single list of top-level comments, they would be well-ordered by timestamp. This is the kind of structure we  we have for any comment branch, given that the max depth of the comment tree is 1.

So top-level comments can be well-ordered by having a comment parent id of null combined with timestamp, and the branch-level comments can be well-ordered by having a comment parent id of their relative root combined with timestamp.

To complete our model, we just need 2 additional fields:
- user id, for the comment author
- content

Now, there are still some interesting considerations to make, particularly around comment deletion. What happens to a branch if the top-level comment is deleted, for example? Different product teams may have different requirements around this, so that pattern is defined by the domain.

### A comment event model

When something happens to a comment tree, we want to be able to notify users. At the most fundamental level, three things can happen to any given comment
- It's been created
- It's been edited
- It's been deleted

When that operation occurs, we make the appropriate change to the datastore and broadcast an event that looks like:
- comment operation (create, edit, delete)
- pre-operation comment (e.g. null for creation)
- post-operation comment (e.g. null for deletion)

Taken together, the pre-op and post-op comment constitutes a diff on the comment, and whatever service is consumming the comment event can do whatever it wants with that information. 

### System architecture

A topic and queue system provides the foundation for what we want here for a couple of reasons.

First, AWS queues have inherent retry logic. This isn't ideal for push notification processing (which we want to be realtime) but is important for the notification center, which needs to as accurate a record of events as possible.

Secondly, why broadcast the comment event to three separate queues? It's important that we keep the failure modes for downstream processing independent. The queue's inherent retry logic can be a double edged sword. For example, if something went wrong with push notification processing and processing was *not* independent, we could end up repeatedly spamming email delivery during retry attempts, which is terrible for obvious reasons. 

Thirdly, this allows us to get separate performance metrics for each process, as it's possible that each operation takes a different amount of time. Because we don't want to spam the email channel, we should buffer and batch updates together for some amount of time before dispatching the notification.

<p align="center">
  <img src="https://user-images.githubusercontent.com/27317800/160022667-4024acdd-f581-4272-973e-30543633e806.jpg" width="1000">
</p>
