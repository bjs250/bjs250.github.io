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

### A comment tree model

### A comment event model

### System architecture

<p align="center">
  <img src="https://user-images.githubusercontent.com/27317800/160022667-4024acdd-f581-4272-973e-30543633e806.jpg" width="500">
</p>
