### How do you handle payment failure?
- In microservice architecture it is not one transaction performed on one database.
- It is in distributed environment and managed with events.
- Order placed -> inventory deducted -> payment failed then event out failure to order which cancel the order and restore the inventory
### Outbox pattern
- It is not possible to make db commit and kafka event in the same transaction. So, use transaction outbox.
- Write event into the outbox table in the same transaction so that db and event will be in the same transaction.
- Then periodically check this table to read and publish events.
