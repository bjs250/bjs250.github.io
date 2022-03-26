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

Notifications are an incredibly important and useful tool to get users back into the product at the right time. The problem is...well, there are too many problems to count! Dispatching notifications is a lot like tossing bottles into the ocean. We were sending them to users, but didn't know if they were actually using them or something had gone wrong along the way.

Then we started to get triage. A lot of triage...

Customers would constantly tell us notitifications stopped working, or were delayed, or they got them on their phone but not in their browswer (or vice versa)

Engineers needed a way to measure the reliability of notification delivery and performance latency

Customer support needed a way to troubleshoot notifications for users

Product managers needed a way to measure the engagement rate with notifications

So this is why we build a centralized notification audit log

## Functional requirements


## Technical requirements


## Functional constraints


## Technical design

### Access patterns

### A comment tree model

### A comment event model

### System architecture
