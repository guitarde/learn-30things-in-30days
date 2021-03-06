# Day 20: Saga pattern - implement business transactions using Microservices

This is the 20th day of my challenge, today I'm going to learn Saga pattern.

## Distributed Transactions

Transactions are an essential part of applications. Without them, it would be impossible to maintain data consistency., A transaction is a logically atomic unit of work which may across to multiple resources (database, message queue etc.,), in SOA or monoliths system, transaction has famous properties call ACID

- Atomicity: It means that either a transaction happens in full or doesn’t happen at all. At any point, if the transaction feels it can’t process, it’ll rollback.
- Consistency: It means that the state of the database remains consistent before a transaction begins and after the transaction ends.
- Isolation: It means that multiple transactions can run in parallel without disrupting the consistency of the database.
- Durability: It means that any changes made in the database actually persist.

The way to do it in a monolith system is the Two-Phase Commit. In a two-phase commit, we have a controlling node which houses most of the logic, and we have a few participating nodes on which the actions would be performed.

**But in Micorservices world, 2PC is no longer an option anymore**
>The microservice architecture structures an application as a set of loosely coupled services
 - 2 PC Guarantees consistency

**BUT**

 - 2PC coordinator is a single point of failure
 - Chatty: at least O(4n) messages, with retries O(n^2)
 - Reduced throughput due to locks
 - Not supported by many NoSQL databases (or message brokers)
 - CAP theorem -> 2PC impacts availability
 - ......
 
Here’s the thing: a distributed transaction is about having one event trying to commit changes to two or more sources of data. So whenever you find yourself in this situation of wanting to commit data to two places. But failure will happen, if you have distributed transactions, then when those fallacies come home to roost, you’ll end up committing data to only one of your two sources. Depending on how you’ve written your system, that could be a very, very bad situation. 

Consider the following high-level Microservices architecture of an e-commerce system:

<img width="600" src="https://blog.couchbase.com/wp-content/uploads/2018/01/e-commerce-architecture-768x520.png" />

In the example above, one can’t just place an order, charge the customer, update the stock and send it to delivery all in a single ACID transaction. To execute this entire flow consistently, you would be required to create a distributed transaction.

We all know how difficult is to implement anything distributed, and transactions, unfortunately, are not an exception. Dealing with transient states, eventual consistency between services, isolations, and rollbacks are scenarios that should be considered during the design phase.

## Solution

Implement each business transaction that spans multiple services as a saga. A saga is a sequence of local transactions. Each local transaction updates the database and publishes a message or event to trigger the next local transaction in the saga. If a local transaction fails because it violates a business rule then the saga executes a series of compensating transactions that undo the changes that were made by the preceding local transactions.

<img width="600" src="https://i2.wp.com/blog.knoldus.com/wp-content/uploads/2018/07/Screenshot-from-2018-07-12-20-03-57.png?resize=683%2C468&ssl=1" />

## What is Saga Pattern

A saga is a sequence of local transactions where each transaction updates data within a single service. The first transaction is initiated by an external request corresponding to the system operation, and then each subsequent step is triggered by the completion of the previous one.

Using our previous e-commerce example, in a very high-level design a saga implementation would look like the following:

<img width="800" src="https://blog.couchbase.com/wp-content/uploads/2018/01/Screen-Shot-2017-12-30-at-1.39.21-PM-768x471.png" />

A successful saga looks something like this :

1. Start Saga
2. Start T1
3. End T1
4. Start T2
5. End T2
6. Start T3
7. End T3
8. End Saga

## The ways to implement a saga transaction

- **Events/Choreography:** when there is no central coordination, each service produces and listen to other service’s events and decides if an action should be taken or not.

<img width="800" src="https://microservices.io/i/data/Saga_Choreography_Flow.001.jpeg" />

- **Command/Orchestration:** when a coordinator service is responsible for centralizing the saga’s decision making and sequencing business logic.

<img width="800" src="https://microservices.io/i/data/Saga_Orchestration_Flow.001.jpeg" />

## Rollbacks in saga transactions

But things rarely go as straight. Sometimes, we might not be in a position to perform a transaction in the middle of the saga. At that point, the previously successful transactions would’ve already committed. For this, we apply compensatory transactions, for each transaction Ti, we implement a compensatory transaction Ci, which tries to semantically nullify Ti. It’s not always possible to get back to the exact same state. For example, if Ti involves sending out an email, we can’t really undo that. So we send a corrective email which semantically undoes Ti. So a failed saga looks something like this:

1. Begin Saga
2. Start T1
3. End T1
4. Start T2
5. Abort Saga
6. Start C2
7. End C2
8. Start C1
9. End C1
10. End Saga

<img width="600" src="https://user-images.githubusercontent.com/3359299/47262751-95637c80-d4be-11e8-995a-d3afbe226fbc.PNG"/>

## Saga Pattern Tips

- Create a unique Id per Transaction
- Add the reply address within the command
- Idempotent operations
- Avoiding Synchronous Communications

## Saga behavior

- On create:
  - Invokes a saga participant
- On reply:
  - Determine which saga participant to invoke next
  - Invokes saga participant
  - Updates its state
  - …
  
**Saga must complete even if there are transient failures**  

**Sagas are ACD**
- Atomicity
  - Saga implementation ensures that all transactions are
  - executed OR all are compensated
- Consistency
  - Referential integrity within a service handled by local databases
  - Referential integrity across services handled by application
- Durability
  - Durability handled by local databases

## Summary

- Microservices tackle complexity and accelerate development
- Database per service is essential for loose coupling
- Use sagas to maintain data consistency across services
- Use transactional messaging to make sagas reliable

## Resources

For more details, here are some YouTube videos

- [Data Consistency in Microservice Using Sagas - Chris Richardson](https://www.youtube.com/watch?v=txlSrGVCK18)
- [Distributed Sagas: A Protocol for Coordinating Microservices - Caitie McCaffrey](https://www.youtube.com/watch?time_continue=2460&v=0UTOLRTwOX0)
