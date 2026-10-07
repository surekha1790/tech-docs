### Can you tell me about a time you had to solve a concurrency issue in a Java application?
I worked on RFQ flow where client requests for quote and market maker responds with quotes which firm for a few seconds.
We had a rece condition where a client accepts the quote just as it was expiring. Accept thread and scheduled expiry thread acts at the
same time. It causes to execute quote which no longer alive or execute twice. I fixed it by making the state transition atomic.
compareAndSet from quoted to executed or expired. So that only one thread can win and either execute or no op for expired one.

### What is RFQ flow and how it differs from original
"In the order book, MMs stream public quotes and the engine matches anonymously by price-time priority. In RFQ, the client asks for a price, an MM replies with a private firm quote that is valid for a few seconds, 
and the client chooses to accept it or let it expire. So RFQ is on-demand, private and one-to-one, and it needs a per-quote state machine with an expiry timer.
That's where my race condition came from: accept and expiry competing on the same quote, which I fixed with an atomic compare-and-set on the quote state.

### What Java concurrency tools have you used in production (e.g. CompletableFuture, thread pools, ExecutorService), and what were you using them for?

I used multiple concurrency tools such as ConcurrentHashMap, CompletableFuture, ThreadPool, Scheduled/ExecutorService and Atomic Variable in multiple projects.
My last project is Trading Application which should maintain low latency and concurrency wherever required.
Our hot path is the matching engine which should match orders within micro seconds, so I avoid using any concurrency tools in the hot path.
Remaining paths such as orders, publishing market updates, scheduled for snapshots are where I used these tools.

For Example: Order Service in our application should read fix messages and convert them to SBE then publish for matching. In such scenarios, executor service helps
to create a thread pool process them parallaly across clients but serial within one and publish for matching.
Instrument Service, which maps security id, trading group for new instrument and publish the newly created instrument data. Here, I used CompletableFuture to
map data using methods like thenApply, thenCompose and publish.
I also worked with scheduledExecutorServices to generate snapshot of matching engine to recovery after restart/crash.
I also used thread safe AtomicReferences where multiple threads need to see a shared variable

### Which design patterns have you used most in your Java projects, and where did you apply them?
Over the years, I have been working on multiple design patterns such as Factory, Builder, Adapter and CQRS design patterns.

In trading application, Used CQRS to keep the matching service light weight.
CQRS(Command Query Responsibility Segregarion) - It separates read and write sides to scale them independently.
Matching is command/write side where it accepts orders, match against order book and publish events to kafka.
Order Query service is read side where it consumes events and persist orders, trades for audit.
Adapter Pattern - It connects two incompatible objects. Our application uses Scila Adapter which is external tool, adapter pattern connects our system generated objects and external tool objects.
Similary, Market data adapter which connects matching events with Trep events.
Factory Pattern - Centralized object creation. Used to create object based on request type.
Builder Pattern - To construct objects with many fields. Objects such as Order, Execution Report contains many fields. Builder patterns suits best in this case.

### How do you ensure your code remains clean and maintainable as a project grows?

I keep it clean and maintainable through few habits. Firstly, enforcing strong boundaries, continuous testing of core logic and refactoring.
I follow SOLID principles such as single responsiblility to keep the service dedicated to particular operation, so as project grows they evolve independently. 
I program to interfaces rather than concete classes, so that a new implementation can be plugged in without touching the callers.
I also keep the objects on hot path immutable.
And beyond the code itself, I also created job which pulls the recent commits, run smoke tests and post it in dedicated channel. So, if any commit breaks the flow
then we catch and pin it down immediately.

### How do you typically test the code you write before it goes to production?

I test at few levels, Junit, Mockito to verify new or changed logic against expected result where mockito to mock out dependencies.
For integration tests, I use Cucumber, running behaviour driven scenarios against test data to confirm components works together end-to-end.
Smoke tests during releases to make sure critical paths are healthy before it goes live.
And for Matching Engine, there is a reconciliation check - backup order book before shutdown, once engine rebuilds from its snapshot plus kafka replay, 
I compare recovered book with back up. They have to match exactly which confirms recovery is correct.

### Have you used TestContainers or other integration testing frameworks? If so, how have you used them?

I haven't used test containers directly, but I have done integration test with Cucumber. My set up is layered with Junit, Mockito and Cucumber.
Junit, Mockito are for unit testing and Cucumber for integration test. I run behaviour driven scenarios against test data. I understand TestContainers
solve similar problem where spinning up real dependencies like kafka, DB in disposable containers instead of mocking them.
