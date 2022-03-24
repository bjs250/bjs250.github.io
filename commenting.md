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





## Functional requirements


