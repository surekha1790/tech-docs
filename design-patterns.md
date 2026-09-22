# Design Patterns

- ### Creational Design Patterns
    * **Factory Pattern**
    - To create object and hide object creation details to client.
    - Ex: Notifications - Email, Sms, Push, Whatsapp
    - Factory pattern is responsible for creating required object.
    * **Abstract Factory Pattern**
    - Create object for a family of objects
    - ```java
        public interface GatewayFactory {
          PaymentProcessor createPayment();
          RefundProcessor  createRefund();
          WebhookVerifier  createWebhookVerifier();
          }
        
          @Component("RAZORPAY")
          public class RazorpayFactory implements GatewayFactory {
          public PaymentProcessor createPayment()         { return new RazorpayPayment(); }
          public RefundProcessor  createRefund()          { return new RazorpayRefund(); }
          public WebhookVerifier  createWebhookVerifier() { return new RazorpayWebhookVerifier(); }
      }
    * **Builder Pattern**
    - Used to create an object with many parameters
    - ```java
       record Order(int id, String name, BigDecimal price, int quantity) {
          public static class Builder {
                int id, String name, BigDecimal price, int quantity;
                public Builder getId(int id) this.id = id;
                public Builder getName(String name) this.name = name;
                public Order build() { 
                   return new Order(id, name, price, quantity);
                }
          }
       }

    * **Singleton Pattern**
    - Create object once and return the same object everytime.
    - Reflection, Serialization, Clonable violates this feature.
    - Fix is to use enum 
    - ```java
        public enum StoreConfig {
             INSTANCE;   // the one and only instance
        }
        StoreConfig config = StoreConfig.INSTANCE;
      
    - Spring does not use this pattern for singleton. It save the bean in container and return same everytime.
    * **Prototype Pattern**
    - To create copy of the object
    - Creating a new object with data is costly so it is good option to create copy of the object.
    - There are two types. Shallow Copy, Deep Copy.
    - Shallow Copy: create copy with fields and uses same reference for lists, objects. So if one object is modified 
        then original will be effected.
    - Deep Copy: everything will be copied to new object, so it will be independent with original object.
    - Create a registry to store these object and return object
    - ```java
        templates.put("INTRADAY_BUY", new Order("BUY", "NSE", "LIMIT", "INTRADAY", "DAY",
                new RiskParams(new BigDecimal("1.0"), new BigDecimal("2.0"))));
        templates.put("DELIVERY_BUY", new Order("BUY", "NSE", "LIMIT", "DELIVERY", "DAY",
                new RiskParams(new BigDecimal("5.0"), new BigDecimal("10.0"))));
  - ### Structural Design Patterns
      * **Adapter Pattern**
      - To connect to incompatible objects
      - Ex: In Trading application, MDA acts as adapter to convert ME evets to trep events
      - In E-commerce, PaymentGatewayAdapter used to convert payment parameters to respective payment(Strip, UPI) parameters
      * **Bridge Pattern**
      - It acts a bridge between two independently growing objects 
      - Ex: There will be multiple reports and multiple formats, each may require different patterns
      - Here, reports can grow individually and formats like excel,csv, pdf can grow individually.
      - Some reports required PDF, some require excel
      - ```java
          public abstract class Report {
             protected final ReportExporter exporter;     // ← the bridge
             protected Report(ReportExporter exporter) { this.exporter = exporter; }
             public abstract byte[] generate(LocalDate from, LocalDate to);
          }
      * **Composite Pattern**
      - It is to build tree structure
      - Product categories and list of products
      - ```java
            public class ParentCategory implements CategoryNode {
               private final String name;
               private final List<CategoryNode> children = new ArrayList<>();
            }
      * **Decorator Pattern**
      - Add features during runtime.
      - Ex: E-commerce, fast delivery, special coupon, gift wrap options will be selected during checkout.
      - Cost will be calculated during runtime based on selection.
      - Existing examples: BufferedReader, unmodifiableList, Cacheable
      * **Flyweight Pattern**
      - Saves memory when there are huge number of same objects.
      - Ex: E-commerce contains warehouse which has huge number of products with same information.
      - ProductInfo holds common information 
      - ```java
        public class StockUnit {
           private final String serialNo;        // extrinsic
           private final String batchNo;         // extrinsic
           private final LocalDate expiryDate;   // extrinsic
           private final String shelfLocation;   // extrinsic
           private final ProductInfo product;    // shared flyweight (just a reference)
        }
      - In Trading, InstrumentCache which hold instruments information
      * **Facade Pattern**
      - It gives simple and single entry point and hides internal complex set of classes.
      - API Gateway in microservices architecture is example for this. It routes to other services.
      * **Proxy Pattern**
      - It sits in front of real object and controls access.
      - Ex: Spring AOP proxies, security layer with pre-auth.
- ### Behavioural Design Patterns
    * **Command Pattern**
    - Encapsulates request/action as an object, so the action can be queued, logged, retried.
    - Ex: In Trading, PlaceOrderCommand, CancelOrderCommand. Queued and processed in order.
    * **Chain of Responsibility Pattern**
    * **Strategy Pattern**
    * **State Pattern**
    * **Observer Pattern**
    * **Template Pattern**
    * **Visitor Pattern**
    * **Interpreter Pattern**
