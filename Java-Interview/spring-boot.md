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

    
