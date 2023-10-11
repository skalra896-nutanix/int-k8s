# Design Challenges

## Overview
While learning the tools such as kubectl and the schema of the manifest of the
various objects you can create in Kubernetes is important, it's also important
to consider how Kubernetes might affect the design of your applications.

Kubernetes encourages a stateless, modular approach to application design -
whether you want to describe that as "cloud first" or "microservices" or 
"12 Factor Apps" is much less important than understanding the underlying
concepts and the "Why?" of the design.

Each of the following challenges is designed to accompany the material for the
course, and challenges you to apply these concepts to a more real-world type
scenario.

## Challenge 1: Running an application

Using what you've learned so far, create a design for the following problem.

You have been tasked with aiding a team in deploying a new application to
Kubernetes. As an initial step, the team has asked you to help them test
some tools and gather some informal performance metrics as a way of
helping to protoype their application. The team would like to use a RabbitMQ
service running in Kubernetes to support their application. Design a method
to quickly get RabbitMQ running in Kubernetes. You may assume any testing or
metrics gathering tools are also running in the same cluster.

Make sure to address the following:
* Where should the container image come from?
* How do you get the image into Kubernetes?
* What Kubernetes API objects do you need to run the application?
* What are the limitations of this method?


## Challenge 2: Designing for uptime

Using what you've learned so far, create a design for the following problem.

The team you're supporting has decided to proceed with building their
application to be deployed to Kubernetes. You are asked to help them to design
the K8s architecture of their primary component, the REST API. This REST API
will support multiple customer-facing applications, such as a mobile app and an
Electron-based desktop app. For now, the team have provided you a single
container image called "challenge-core-api" While they are currently unclear on
exactly what the level of traffic might be, they want to ensure resiliancy and
uptime in the design as much as possible.

Make sure to address the following:
* What should you include in the design to aid in ensuring uptime?
* How will external applications such as the mobile and desktop apps access the API?
* Assuming the REST component will need to access the previously tested RabbitMQ tool, how would you recommend that access is configured?
* What limitations should be considered in your approach?
* BONUS: Practice writing the YAML manifests to match your design


## Challenge 3: Updates

Using what you've learned so far, create a design for the following problem.

The team decides to employ your prior designs for the application, as it seems
to meet their needs as far as uptime and overall scalability. However, one of
the engineers on the team has brought up and excellent point - what about updates?
The application will be somewhat regularly updated to provide new REST API
endpoints in support of new features or enhancements. It may also be updated from
time to time for urgent bugfixes. Help the team decide what approach to use in
the Kubernetes ecosystem to ensure redeploying updates goes smoothly. Justify
your design choices.

Make sure to address the following:
* Assuming updates are backward compatible, how can you minimize or eliminate downtime?
* How can the team perform limited testing of the application deployment before
rolling it out to all users?
* If the deployment fails, how can the previous versions be restored quickly?
* What are the limitations of the methods you have selected?


## Challenge 4: Maintaining State

Using what you've learned so far, create a design for the following problem.

The team you're supporting has been happy with your designs so far, but they
now bring another concern to your attention - maintaining state. They have
built the REST API as a stateless application as per best practices, but the
application still needs to maintain some state somewhere to be useful. The team
has decided that user data and associated application data should be stored in
MongoDB, and that the RabbitMQ messages should also be persistent so that no
loss of messages in transit can occur. Provide a design that supports storing
the data for this application according to the team's needs.

Make sure to address the following:
* How would running MongoDB in the cluster alter your earlier designs?
* How can the data for MongoDB and RabbitMQ be stored so that restarts and
failovers do not cause data loss?
* Assume credentials for MongoDB and RabbitMQ need to be provided to the API
component and periodically rotated. What approach would you use for that data?
* What are the limitations in your approach?


## BONUS Challenge: Automation

Using what you've learned so far, create a design for the following problem.

The team is happy with your design and will deploy according to your
recommendations. Now that they have seen what you can do with the cluster, they
are curious about any automation that can be applied to help reduce engineering
time in maintaining the application. They would like any advice you can provide
for the following concerns:

* The team has finished performance testing and determined that the API
component runs well with 500MB of memory and 0.5 CPU. They want to know if you
can use this information to automatically scale the application up and down as
traffic changes.
* An engineer is usually responsible for running reports on an hourly basis to
determine if there are any failed messages in the RabbitMQ dead letter queue
and to categorize them and send a report to the team lead.  They would like
to automate this.
* Some of the REST API endpoints trigger long-running tasks, which the team
manages through RabbitMQ messages. However, they'd like to avoid running
another component 24/7 just to run these periodic tasks. What approach might
they be able to take?
* What are the limitations of your approaches to these issues?
