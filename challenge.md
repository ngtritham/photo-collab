# Technical Architecture Work Sample

## Scenario

Imagine that you have joined CADDi in its current state. That is, you can make assumptions about the state
of the business, the organization, and the technical stack based on the slides here:
https://speakerdeck.com/caddi_eng/en-company-overview-for-engineers

Suppose that there is a desire to explore creating and launching a new online service to help manufacturing
companies communicate using photographs.

- The intended customer: large manufacturing companies, with 500+ employees
- Main use case:
  - Communication between mechanical engineers, procurement, and factory
- Vague requirements:
  - File storage and sharing, where the files are primarily PNG and JPEG photos
  - Be able to annotate the photos with circles and start a message thread
  - Allow multiple individuals to respond in the message thread
- Vague scenario: a factory worker notices a scratch on a part. They can take a photo, circle the
  scratch, annotate it with a question "Is this ok?" and an engineer can respond in a thread, similar to
  the Google Docs annotation/suggestion feature.
- There is quite a bit of time pressure to show that this software is viable and delivers value to users
- You are the first engineer assigned to this project, and no code has been written yet. Your design
  document will drive the resourcing strategy, including engineer reassignments and new hiring.

## Background

Thank you very much for your interest in CADDi. This is a technical architecture work sample intended for
experienced engineers, where we ask you to write a design document for a hypothetical situation. There are
three main objectives for us:

1. Understand your ability to communicate technical information
2. Understand how you prioritize and consider trade-offs in situations with high uncertainty
3. Evaluate your knowledge of both architecture and specific technologies

Please provide a design document as a Google Doc/PDF file describing how you would design a
system for the scenario below. Readers are engineering leads and managers.

- no more than 3 pages long
- take no longer than 90 minutes

This is intentionally open-ended, since we understand that there is often no correct answer. As such, we
ask that you be explicit about any assumptions you make and explain your design decisions. However,
please include below as a must item:

- overall architecture diagram, including both self and fully managed cloud resources

Also, consider covering some of the following:

- technical stack (languages, frameworks, specific databases, etc)
- database ER diagrams, and any other associated data models, if relevant
- any trade-offs you are making, whether it be working hours or technical debt
- non-functional requirements: security, performance, DevOps, testing, monitoring
