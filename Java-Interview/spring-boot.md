### Explain Spring Boot auto-configuration
- @SpringBootApplication comes with three annotations @SpringbootConfiguration, @EnableAutoConfiguration, @ComponentScan
- While loading, spring scans all the annotations like @Configuration, @Component, @Service, @Repository and initialize the beans
- It scans through dependencies, properties declared in application.yml file and initiate beans
- For Example, KafkaTemple, Datasource are auto configured based on the dependencies and properties
- Spring configure your own configuration beans first. If you declare bean in own configuration then only @ConditionalOnBean works because order is not guranteed.

### How spring decides which beans to create
- It decides based on @Conditional annotations like @ConditionalOnBean @ConditinalOnMissingBean, @ConditionalOnProperty etc

### What is @ConditionalOnBean
- A bean is created only if specified bean is already injected in the context.
- It is order specific, it can view only beans evaluation before this annotation.
- If this is used in own configuration then it can fragile as it does not follow the order and if bean is not available then it fails
### How Spring creates KafkaTemplate automatically
- Spring scans through dependencies and properties while starting.
- If it finds kafka dependency and bootstrap server config then it auto configure the kafka template.

### Construction Injection vs Field Injection
- Spring officially recommend construction injection only.
- Constructor arguments are final and assigned once during object construction, never be reassigned and can not be null
``` @RestController
  public class OrderController {
  
      private final OrderService service;   // final → set once, never reassigned
  
      // @Autowired is OPTIONAL for a single constructor since Spring 4.3
      public OrderController(OrderService service) {
          this.service = service;
      }
  }
 ```
  - This implies object can not be created without initializing dependencies (OrderService)
  - It can be easily testable in testing frameworks like Junit because it just behaves as normal object which can be created with constructor, no spring context is required.
  - Avoid circular dependency
 
### How spring resolves circular dependency
- When an Object A depends on B and B again depends on A then it causes circular dependency
- With constructor injection, it will throw BeanIsCurrentlyInCreationException while start up.
- With field or setter injection it can resolve it with three-level cache, where it caches(SingletonFactories) A with null reference and prepare B fully and then prepare A
- Spring2.6 does not allow this circular dependency, even with setter/field injection it throws exception.
- Circular dependency can be resolved by making one of the objects lazy.

### Explain Bean lifecycle ?
- Initialization: In this stage, it is just initialize the bean with constructor or with @Bean annotation. Here object is present but bean is not fully wired.
- Populate Properties: Spring inject dependencies and properties values (${}).
- BeanPostProcessor: Run bean processor before initializer
- Init: Run @PostConstructor
- BeanPostProcessor - AfterInitializers: Creates Proxies (@Transactional, @PreAuthorise, @Async)
- Now, bean is fully ready and injected in context.
- Destruction: @PreDestroy will be executed

### Spring Transaction Management
```
@Transactional
public void createOrder() {
    saveOrder();      // both run inside ONE transaction
    updateWallet();   // both commit together, or neither does
}
```
- When spring identifies `@Transaction` then it wraps a proxy class around the bean.
- When transaction method is called then it is deligated to proxy and it calls transaction interception which manages the transaction.
- Once the actual logic is executed then commit if no exception or else rollback if there is an exception.
- It rollback the transaction only if it unchecked exception or error, it will not rollback checked exceptions.
- If transaction needs to rolledback when checked exception then ``` @Transactional(rollbackfor="Exception.class")
- Spring bydefault create subclass (CGLIB) so final, static, and private methods can not be transaction as it can not be overriden
```
  Proxy object
      |
Transaction interceptor
      |
Begin transaction
      |
Execute method
      |
Commit/Rollback
```

### Why does this not rollback?
```
@Transactional
public void method(){

   try{
      save();
      throw new RuntimeException();
   }
   catch(Exception e){

   }
}
```

- Method is rethrowing exception and catch with empty block, so it is swalloing the exception and exit the method normally.
- Transaction interception rollback the transaction only exception propagated to it but in the above example it is not happening.
- should not catch with empty block or set promatically rollback only transaction.
```
@Service
class OrderService {
    @Autowired PaymentService paymentService;

    @Transactional                         // OUTER — starts the physical transaction T
    public void placeOrder() {
        saveOrder();
        try {
            paymentService.charge();       // INNER @Transactional — joins the SAME tx T
        } catch (Exception e) {
            log.warn("payment failed, continuing anyway", e);   // swallow it
        }
        // returns normally → outer interceptor tries to COMMIT T
    }                                      // 💥 throws UnexpectedRollbackException
}

@Service
class PaymentService {
    @Transactional                         // propagation REQUIRED (default)
    public void charge() {
        throw new RuntimeException("card declined");
    }
}
```

- In the above example inner method is also marked as transactional.
- Since default propagation is REQUIRED it will be wrapped under same outer transaction.
- When charge() throws exception it marks it as rollback only but outer method silently swallowed the exception.
- Outer method calls commit but nested method set rollback only, so it rolls back everything and throws UnexpectedRollbackException.

- mark nested method as REQUIRES_NEW, which makes it independent and rollsback on exception
  
      ```
      @Service
      class PaymentService {
          @Transactional(propagation = Propagation.REQUIRES_NEW)  // its OWN physical tx
          public void charge() {
              throw new RuntimeException("card declined");
          }
      }
      ```
  
      * Above solution uses two connections from pool, so it may exhaust the connection pool. So, should used cautiously.
- Correct fix here would be:
   * Do not catch the exception which makes it silently exit the method.
   * If both are independent then do not call another transaction inside a transaction.

### Transaction Propagations ?
- Required: Default option and all the inner calls bounded to the same transaction. Either everything is committed or rolledback.
- Required_New: Inner method call creates a new individual transaction and irrespective of the outer transaction it can commit or rollback. Caveat is it uses multiple connections from pool.
- Nested: Supports only with JDBC not with JPA.
- Supports: If the call is from trasanction then support otherwise just act as normal
- Not_Supported: Suspend the transaction and run normally.
- Mandatory: It must be in the transaction.
- Never: It must not be in transaction.

### @Transactional pitfalls ?
- Since transactional class are wrapped with proxy, self method invocation does not apply transaction
- Only public methods work under transaction. Private, static, final, protected methods are ignored silently.
- Only unchecked exceptions and errors are rolled back but checked exceptions commits the transaction. To handle this add ```@Transactional(requiredFor=Exception.class)```.
- Calling another transactional method inside a transaction uses same transaction and it adds rollback required when there is an exception and outer transaction can not commit.
- To avoid above case, use REQUIRED_NEW propagation type but it uses separate connection from pool which can cause issues at connection pool side.
- If transactional method swallos the exception then it will commit the transaction.
- Transactional method holds connection for entire call, so should concious on method scope.
- It does not propagate to the another thread in case of @Async call
- If kafka publish is there within transaction and kafka published but db rolled back then kafka still have the published event.


### About SpringBoot's Connection pool (Hikari Pool):
- Springboot auto creates Hikari connection pool when jpa dependency is added
- It is required to update max-connections, max-life, connection-time-out based on the prod infra
- max-connections : should be per service and depends on db infra. Total size should not exceed db max allowed connections size
- max-life : should be shorter than any upstram that kills ideal connections
- connection-time : how long it should wait for free connection.
- lead-detection-threshold: It is off by default and good to set this to catch the connection borrowed but never returned logs
