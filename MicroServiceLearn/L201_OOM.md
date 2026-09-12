   



    OutOfMemoryError (OOM) happens when the JVM cannot allocate enough memory for an object or JVM operation.

Example:

    java.lang.OutOfMemoryError: Java heap space

It means:

    Your application needs memory, but the JVM doesn't have enough available memory in the required memory area.

1️⃣ Common types of OutOfMemoryError


        java heap space
        GC overhead limit exceeded
        Metaspace
        Direct buffer memory    
        Pagination Issues



| OOM Error                                | What it means                                                  | Common cause                                              |
| ---------------------------------------- | -------------------------------------------------------------- | --------------------------------------------------------- |
| **`Java heap space`** ⭐                  | Heap is full                                                   | Too many objects, large collections, loading huge DB data |
| **`GC overhead limit exceeded`** ⭐       | JVM spends too much time doing GC but frees very little memory | Heap pressure / memory leak                               |
| **`Metaspace`** ⭐                        | Cannot allocate memory for class metadata                      | Too many classes / ClassLoader leak                       |
| **`Direct buffer memory`**               | Direct/off-heap buffer memory exhausted                        | NIO, Netty, buffer leaks                                  |
| **`Unable to create new native thread`** | JVM/OS cannot create another thread                            | Too many threads / native memory exhausted                |



A. Java heap space ⭐ Most common

Example:

    java.lang.OutOfMemoryError: Java heap space

What does it mean?

Objects are created in the Java Heap, but the heap is full.

Common causes

    ❌ Memory leak
    ❌ Loading huge data into memory
    ❌ Large collections
    ❌ Infinite object creation
    ❌ Cache growing forever
    ❌ Reading an entire file into memory

Example:

List<String> list = new ArrayList<>();

while (true) {
list.add("Some data");
}

Eventually:

OutOfMemoryError: Java heap space


2️⃣ GC overhead limit exceeded

Example:

    java.lang.OutOfMemoryError: GC overhead limit exceeded
What does this mean?

    The JVM is spending most of its time doing Garbage Collection, but it is unable to free enough memory.
    The JVM is spending too much time performing Garbage Collection but recovering very little memory.
Traditionally, the JVM detects this when roughly:

    More than 98% of JVM time is spent doing Garbage Collection
    GC recovers less than 2% of the heap

Basically:

    Application creates objects
    ↓
    Heap becomes full
    ↓
    GC runs
    ↓
    Very little memory is freed
    ↓
    GC runs again and again
    ↓
    Application becomes slow
    ↓
    OOM
Fix

    Don't simply increase memory first.

Investigate:

    Memory leak
    Large collections
    Unbounded cache
    Too many objects being retained
3️⃣ Metaspace

    java.lang.OutOfMemoryError: Metaspace
What is stored in Metaspace?

    Class metadata.
    
    Possible causes:
    
    Too many dynamically generated classes
    ClassLoader memory leak
    Excessive proxy generation

When can this happen?

        Example: Continuously creating new classes dynamically

Frameworks can generate classes dynamically using things such as:

    Proxies
    Bytecode generation
    Runtime code generation

For example, imagine continuously creating classes with new class loaders and never allowing them to be unloaded.

    This can sometimes happen with frameworks that dynamically generate classes, especially if class loaders are repeatedly created and never released.

The JVM stores metadata such as:

        Class name → User
        Methods → save()
        Fields → name
        Modifiers → public
        Method/class-related runtime metadata

This is broadly called class metadata.


Fix

        Investigate the classloader leak and generated classes.
        
        You can also increase metaspace:
        
        -XX:MaxMetaspaceSize=512m

But again:

    Increasing memory may hide the problem instead of fixing it.

4️⃣ Direct buffer memory

    Heap = Java objects live here
    Direct Memory = native memory used by Java/JVM for high-performance I/O
    Direct Buffer Memory is off-heap/native memory used by Java NIO and networking libraries to perform efficient I/O operations by reducing unnecessary copying between Java heap memory and native I/O buffers.

Example:

java.lang.OutOfMemoryError: Direct buffer memory

    This usually involves off-heap memory.

Common in:

    Netty
    Spring WebFlux
    NIO applications
Fix

Check:

    Direct buffer usage
    Buffer leaks
    Connection/resource leaks

You may need to configure:

        -XX:MaxDirectMemorySize

🔥 Production Scenario

Imagine your Spring Boot API does this:

    @GetMapping("/users")
    public List<User> getUsers() {
    return userRepository.findAll();
    }

Your database contains:

    10 million users

Now your application does:

        Database
        ↓
        Loads 10 million records
        ↓
        Creates Java objects
        ↓
        Stores everything in Heap
        ↓
        Heap becomes full
        ↓
        OutOfMemoryError 💥
✅ Correct Fix

    Use pagination.
    
    @GetMapping("/users")
    public Page<User> getUsers(
    @RequestParam int page,
    @RequestParam int size) {
    
        Pageable pageable = PageRequest.of(page, size);
    
        return userRepository.findAll(pageable);
    }

Now:

Instead of:

    10,000,000 users → Memory

We load:

    100 users → Memory

Much safer.


 DIRECT BUFFER MEMORY:

    Direct Buffer: Why do we need it?

    The main reason is to make certain native I/O operations more efficient.

    Imagine you have data on the Java Heap:

            Java Heap Buffer
            ↓
            Native/OS needs data

Native code generally needs memory it can access directly.

So conceptually, Java may need an extra copy:

        Java Heap Buffer
        ↓ Copy
        Native Buffer
        ↓
        Network / File / OS

With a direct buffer:

        Direct Buffer (Native Memory)
        ↓
        Network / File / OS

This can avoid or reduce extra copying in appropriate I/O operations.

Exactly when do YOU use it?

You explicitly use it when you write:

        ByteBuffer buffer = ByteBuffer.allocateDirect(1024);
        
        Then Java allocates the buffer outside the heap.
        You may not write allocateDirect() yourself.

Frameworks may allocate direct memory internally when doing lots of network I/O:

        Spring Cloud Gateway
        ↓
        Reactor Netty
        ↓
        Netty ByteBuf
        ↓
        Direct Memory

Other common high-I/O systems may use direct buffers too:

        Netty
        Kafka
        NIO applications
        Database/network drivers
        High-throughput servers


🔥 Unable to create new native thread — what is it?

This error means:

    Your Java application tried to create a new thread, but the JVM/operating system could not allocate the resources needed for another native thread.

Example:

    java.lang.OutOfMemoryError: unable to create new native thread




---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------



DEBUGGING STEPS :


1. FIrst look logs what exact OOM error we get

-----------------------------------------------------------------------------------------------------------------------------------------

📝 Java Heap Space OOM 
1️⃣ What is the error?

    java.lang.OutOfMemoryError: Java heap space

It means:

    The JVM tried to allocate an object in the Java Heap, but there was not enough heap memory available.

2️⃣ First important point ⭐

    Heap full does NOT automatically mean Memory Leak

There are two major possibilities:

    A. Memory Leak

Objects are no longer useful, but they are still referenced.

    Objects created
    ↓
    Still referenced
    ↓
    GC cannot remove them
    ↓
    Heap keeps growing
    ↓
    💥 Java heap space

B. Genuine High Memory Usage

    The application legitimately needs too much memory at that moment.

Example:

    Load 10 million DB records
    ↓
    Millions of objects created
    ↓
    Heap becomes full
    ↓
    💥 OOM

3️⃣ First debugging step: Check JVM Heap Metrics

When Heap OOM happens, check:

        ✅ Heap Used
        
        sum(jvm_memory_used_bytes{area="heap"})
        
        It tells us:
        
        How much heap memory is currently occupied.
        
        ✅ Heap Max
        sum(jvm_memory_max_bytes{area="heap"})
        
        It tells us:
        
        Maximum heap memory available to the JVM.
        
        Usually related to:
        
        -Xmx
        4️⃣ Heap Usage Percentage
        
        Useful PromQL:
        
        (
        sum(jvm_memory_used_bytes{area="heap"})
        /
        sum(jvm_memory_max_bytes{area="heap"})
        ) * 100
        
        Example:
        
        Heap Used = 95%
        
        🚨 Heap is under heavy pressure.
        
        But:
        
        95% Heap usage does NOT automatically mean memory leak.
        
        We need to investigate further.
        
        5️⃣ Check Heap Pattern Over Time ⭐⭐⭐
        
        This is very important.
        
        🟢 Healthy Pattern
        Memory
        
               /\       /\
              /  \     /  \
        _____/    \___/    \____
        GC
        
        Memory:
        
        Increases
        ↓
        GC happens
        ↓
        Memory decreases
        
        This usually means GC is successfully cleaning unused objects.
        
        🔴 Suspicious Pattern
        Memory
        
                               💥
                            /
                         /
                      /
                   /
        __________/
        
        Even after GC, memory does not come down properly.
        
        This can indicate:
        
        Memory Leak
        OR
        Objects continuously being retained
        6️⃣ Check GC Metrics
        
        We don't look only at Heap.
        
        We also check what GC is doing.
        
        A. GC Frequency
        
        Metric:
        
        jvm_gc_pause_seconds_count
        
        Rate:
        
        rate(jvm_gc_pause_seconds_count[5m])
        
        Question it answers:
        
        How frequently are GC pauses/events occurring?
        
        If GC frequency suddenly increases:
        
        Low
        ↓
        Medium
        ↓
        High
        ↓
        Very High 🚨
        
        Possible causes:
        
        High object allocation
        Heap pressure
        Memory leak
        
        ⚠️ High GC frequency alone does not prove a memory leak.
        
        B. GC Pause Time
        
        Metric:
        
        jvm_gc_pause_seconds_sum
        
        Rate:
        
        rate(jvm_gc_pause_seconds_sum[5m])
        
        Question it answers:
        
        How much time is being spent in recorded GC pauses?
        
        Example:
        
        GC Pause
        
        10 ms  → Normal
        50 ms
        500 ms → 🚨
        2 sec  → 🚨🚨
        
        High GC pauses can cause:
        
        Slow APIs
        Timeouts
        High latency
        Poor application performance


7️⃣ Memory After GC ⭐⭐⭐ MOST IMPORTANT

The key question:

    After GC runs, how much memory remains occupied?

Healthy example:
    
    GC #1
    
    Before = 900 MB
    After  = 300 MB
    
    GC successfully freed lots of memory.
    
    Suspicious example:
    After GC #1 → 300 MB
    After GC #2 → 450 MB
    After GC #3 → 600 MB
    After GC #4 → 800 MB
    After GC #5 → 950 MB

The after-GC baseline keeps increasing 📈.

This means:

More and more objects are surviving GC.

Possible causes:

    Memory leak
    Unbounded cache
    Growing collection
    Objects being retained



Step 3 : Take Heap Dump and Analyze

    If Heap OOM happens, take a heap dump.
    
    Analyze it using tools like:
    
    - Eclipse MAT (Memory Analyzer Tool)
    - VisualVM
    - YourKit
    - JProfiler





Heap Dump Analysis — What to Check
    
    1️⃣ Leak Suspects Report
    2️⃣ Histogram / Biggest objects
    3️⃣ Retained Size ⭐⭐⭐
    4️⃣ Number of instances
    5️⃣ Dominator Tree ⭐⭐⭐
    6️⃣ Path to GC Roots ⭐⭐⭐
    7️⃣ Trace it back to your code

Let's understand each slowly.


1️⃣ Classic Memory Leak ⭐⭐⭐

    You find objects that are no longer needed but are still being retained.

Heap dump:

    5,000,000 User objects

Path to GC Root:

    GC Root
    ↓
    static HashMap
    ↓
    User objects

You check the code:

    static Map<String, User> users = new HashMap<>();

Objects keep getting added:

    users.put(id, user);

But nothing removes them.

Issue found:

    ❌ Unwanted objects are still referenced
Fix:
    
    Remove references
    Fix object lifecycle
    Clear old data
2️⃣ Unbounded Cache ⭐⭐⭐

Heap dump may show:

CacheManager

        Retained Size = 4 GB

Inside:

    HashMap
    ↓
    10 million objects

The cache is intentionally retaining objects, but:

    ❌ No maximum size
    ❌ No TTL
    ❌ No eviction
Issue:
    
    Cache grows forever 📈
    Fix:
    Maximum size
    TTL
    Eviction policy

3️⃣ Huge Collection ⭐⭐⭐

You may find:

    ArrayList
    Retained Size = 3 GB

or:

    HashMap
    Retained Size = 5 GB

Then inspect it.

Maybe:

    ArrayList
    ↓
    20 million Orders
    Possible reasons:
    findAll()
    Loading entire DB table
    Accumulating results
    Unlimited queue
Example:

    List<Order> orders = orderRepository.findAll();
Fix:
    
    Pagination
    Batch processing
    Streaming
------------------------------------------------------------------------------------------------------------------------------------------------


1️⃣ What happens when Java creates a thread?

Suppose your code does:

    Thread thread = new Thread(() -> {
    // some work
    });

    thread.start();

Conceptually:

    Java Application
    ↓
    thread.start()
    ↓
    JVM requests the OS
    ↓
    OS creates a native thread
    ↓
    Memory is allocated for that thread's stack
    ↓
    Thread starts running
🧠 Important: Every Java thread needs memory

Suppose your application has:

    Thread 1
    Thread 2
    Thread 3
    Thread 4

Each thread needs its own stack.

Native Memory

    Thread 1 → Stack
    Thread 2 → Stack
    Thread 3 → Stack
    Thread 4 → Stack

Why does every thread need its own stack?

Because every thread has its own:

    Method calls
    Local variables
    Method execution state

For example:

    public void methodA() {
    methodB();
    }

The thread needs to remember:

    methodA()
    ↓
    methodB()

That execution information is stored in that thread's stack.

1️⃣ Too many threads

    Thread count = 10,000 😨
2️⃣ Not enough native memory

    Even with a lower thread count:
    
    Native memory is already heavily used
3️⃣ Thread stack size is too large

    Each thread has a configured stack size.
    
    For example:
    
    -Xss1m
    
    Conceptually:
    
    1 thread → up to ~1 MB stack
    
    Now:
    
    1,000 threads × ~1 MB
    
    Potentially a large amount of memory is needed.

4️⃣ OS or container thread/process limits

    The operating system or container may have a limit.
    
    JVM wants to create Thread #500
    ↓
    OS thread limit reached ❌
    
    So the real meaning of the error is:
    
    The JVM tried to create another native thread, but the OS/JVM could not allocate the resources needed to create it.

Next is Step 2: Thread Metrics 👍

We now know what the error means. When you get:

java.lang.OutOfMemoryError:
unable to create new native thread

the first debugging question is:

How many threads does my application have?

2️⃣ Check Thread Count ⭐⭐⭐

We want to observe:

    Thread Count over time

For example, a healthy application might look like:

    100
    105
    102
    108
    103

Threads are created and finished, so the count stays roughly stable. ✅

🚨 Thread leak pattern

Suppose monitoring shows:

        100
        ↓
        500
        ↓
        1,000
        ↓
        3,000
        ↓
        8,000
        ↓
        15,000
💥 unable to create new native thread

This is highly suspicious.

    Thread Count continuously increasing 📈

Now our next question becomes:

Why are threads continuously being created and not going away?

    In Spring Boot / Actuator

A useful metric is:

    jvm.threads.live

This tells you:

How many live threads currently exist in the JVM.

Another useful metric:

    jvm.threads.peak

This tells you:

What was the highest number of live threads since the JVM started.

What patterns do we look for?
        ✅ Normal
        Live Threads
        
        100 → 110 → 105 → 108 → 102
        
        Stable.
        
        🚨 Suspicious
        Live Threads
        
        100 → 500 → 1000 → 2000 → 5000
        
        Continuously growing.
        
        Possible thread leak.
        
        ⚠️ Sudden spike
        100 → 2000 → 2100 → 150
        
        This is different.
        
        Maybe a sudden traffic spike or temporary workload created many threads.
        
        So we don't just look at one number—we look at the trend over time.
-----------------------------------------------------------------------------------------------------------------------------------------------------------------


Main causes of Metaspace OOM

There are two major categories.

1️⃣ Too many classes being loaded ⭐⭐⭐
    
    Classes loaded
    ↓
    Class metadata stored in Metaspace
    ↓
    More and more classes
    ↓
    Metaspace fills
    ↓
    💥 Metaspace OOM

Possible situations:

        Dynamic class generation
        Runtime proxy generation
        Bytecode manipulation
        Frameworks/libraries generating classes
2️⃣ ClassLoader leak ⭐⭐⭐⭐⭐

This is the most important concept.

Normally:

    ClassLoader
    ↓
    Loads Classes

When that ClassLoader is no longer needed:

    ClassLoader becomes unreachable
    ↓
    Its classes can be unloaded
    ↓
    Metaspace can be reclaimed

But imagine:

    Something still references ClassLoader
    ↓
    ClassLoader cannot be GC'd
    ↓
    Classes cannot be unloaded
    ↓
    Metaspace keeps growing 📈
    ↓
    💥 Metaspace OOM

This is called a:
    
    🚨 ClassLoader Leak
    3️⃣ Other common causes
    🔹 Aggressive dynamic proxy/class generation

For example, frameworks can generate classes dynamically.

Conceptually:

    Request 1 → Generated Class A
    Request 2 → Generated Class B
    Request 3 → Generated Class C
...

If generated classes keep increasing without proper cleanup:

    Number of Classes 📈
    ↓
    Metaspace 📈
    ↓
    OOM
🔹 Application/plugin redeployment issues

Common in application servers or plugin systems:

    Application Version 1 deployed
    ↓
    ClassLoader 1
    
    Application Version 2 deployed
    ↓
    ClassLoader 2
    
    Application Version 3 deployed
    ↓
    ClassLoader 3

If old ClassLoaders are retained:

        ClassLoader 1 ❌ still alive
        ClassLoader 2 ❌ still alive
        ClassLoader 3 ❌ still alive

Then Metaspace keeps growing.

    🔹    Metaspace limit configured too low

Unlike the Java Heap, Metaspace uses native memory and can grow dynamically unless constrained.

But you may configure:

    -XX:MaxMetaspaceSize=256m

If your application legitimately needs more:

    Actual requirement = 500 MB
    Limit              = 256 MB ❌

Then:

💥 OutOfMemoryError: Metaspace

This doesn't necessarily mean there is a leak.



-----------------------------------------------------------------------------------------------------------------------------------

1️⃣ What is a Heap Dump?

    A Heap Dump is a snapshot of the Java Heap at a particular moment.

Imagine at this moment your heap contains:

Heap

        ├── 2,000,000 User objects
        ├── 500,000 Order objects
        ├── HashMap → 3 GB
        ├── Cache → 2 GB
        └── Other objects

    A heap dump captures information about these objects.

It helps answer:

    What is occupying my heap memory?

2️⃣ What information can we get from a Heap Dump?

A heap dump helps us investigate:

        ✅ Which objects consume the most memory?
        
        For example:
        
        byte[]        → 4 GB
        HashMap       → 2 GB
        User          → 1 GB
        String        → 800 MB
✅ How many objects exist?

Example:

    User objects = 5,000,000 😨

Now you can ask:

    Why do we have 5 million User objects?

✅ Who is retaining the objects? ⭐⭐⭐

This is the most important part.

Example:

    User Object
    ↑
    ArrayList
    ↑
    UserCache
    ↑
    static variable

Even if the User objects are no longer useful, they cannot be garbage collected because something still references them.

This is called the retention path.

🧠 Simple example

Suppose you have:

    public class UserCache {
    
        private static final List<User> users = new ArrayList<>();
    
    }

And the list keeps growing.

    Heap Dump might show something conceptually like:

    5,000,000 User objects
    ↑
    ArrayList
    ↑
    UserCache.users
    ↑
    static field
    ↑
    GC Root

Now you have found the important question:

    Why are these User objects still alive?

Answer:

Because UserCache.users is retaining them.
3️⃣ When should we take a Heap Dump?

There are a few situations.

Situation A: Automatically when OOM happens ⭐⭐⭐

This is the best preparation for production.

Start the JVM with:

    -XX:+HeapDumpOnOutOfMemoryError

When this happens:

    java.lang.OutOfMemoryError: Java heap space

The JVM automatically creates a heap dump.

You can also specify where:

    -XX:HeapDumpPath=/logs/heapdumps

So conceptually:

    OOM happens 💥
    ↓
    JVM automatically captures Heap Dump 📸
    ↓
    You analyze it
4️⃣ How do we configure this in Spring Boot?

Usually through JVM options.

For example:

    -XX:+HeapDumpOnOutOfMemoryError
    -XX:HeapDumpPath=/logs/heapdumps

If you're running locally from IntelliJ, you can add these to VM options.

Example:

    Run Configuration
    ↓
    VM Options
    ↓
    -XX:+HeapDumpOnOutOfMemoryError
    -XX:HeapDumpPath=C:\heapdumps

Then if an OOM happens, a heap dump file is generated.

Usually it looks something like:

    java_pid12345.hprof

The .hprof file is the heap dump.

⚠️ Important production point

Heap dumps can be very large.

If your JVM heap is:

    -Xmx4g

Your heap dump can be several GB.

So in production, make sure:

        Enough disk space exists
        
        Otherwise, when OOM happens:
        
        Application needs diagnostics
        +
        Disk is full
        ↓
        Heap dump may fail
5️⃣ Can we take a Heap Dump manually?

Yes.

    If the application is still running and memory is growing, you may capture one before it crashes.

For example:

    jcmd <PID> GC.heap_dump heap.hprof

Conceptually:

    Running Application
    ↓
    Memory keeps growing 📈
    ↓
    Before OOM
    ↓
    Take Heap Dump 📸

This can be useful because you don't always want to wait for the crash.


You open it using a heap analysis tool.

Common tools include:

    Eclipse MAT ⭐
    VisualVM
    Java Mission Control

2️⃣ If you're using IntelliJ IDEA

Go to:

    Run
    ↓
    Edit Configurations
    ↓
    Your Spring Boot Application
    ↓
    VM options

Add:

    -XX:+HeapDumpOnOutOfMemoryError -XX:HeapDumpPath=C:\heapdumps

Then run your Spring Boot application normally.


1️⃣ Leak Suspects Report ⭐ First thing

    When using Eclipse MAT, the first thing you can check is:

Leak Suspects Report

    MAT automatically analyzes the heap and tries to identify suspicious objects.

Example:

    Problem Suspect 1
    
    HashMap retains 2.5 GB

Then you investigate:

Why is this HashMap holding 2.5 GB?

⚠️ Important:

    The Leak Suspects Report gives you a clue, not guaranteed proof of a memory leak.

2️⃣ Histogram — Which objects exist in huge numbers?

Next, check the Histogram.

It might show:

    Class              Objects        Memory
    
    User               5,000,000     800 MB
    Order              3,000,000     600 MB
    String            10,000,000     1 GB
    byte[]             2,000,000     2 GB

Now ask:

Which object type is unexpectedly large?

For example:

Expected:

    User objects = 10,000

Actual:

    User objects = 5,000,000 😱

🚨 Something is suspicious.

3️⃣ Number of Instances

Suppose you see:

    User = 5 million objects

That might explain the OOM.

But sometimes:

    CacheManager = 1 object

Yet that one object could retain:

    3 GB of other objects
    
    So object count alone is not enough.

This brings us to the most important concept.

⭐ Retained Size
4️⃣ Retained Size ⭐⭐⭐

Ask:

    If this object disappeared, how much memory could potentially become eligible for garbage collection?

Example:

    UserCache object = 100 bytes

Its own size is tiny.

But:

    UserCache
    ↓
    HashMap
    ↓
    5 million Users

So:

    Shallow Size = 100 bytes

Retained Size = 3 GB

🔥 This is why Retained Size is extremely important.

It helps find objects that are holding huge object graphs alive.

5️⃣ Dominator Tree ⭐⭐⭐

    The Dominator Tree is one of the most useful views.

It helps answer:

    Which objects are retaining the largest amount of memory?

Example:

    UserCache
    ↓ retains
    HashMap
    ↓ retains
    5 million User objects

MAT may show:

Object                  Retained Heap

    UserCache               3 GB
    HashMap                  2.9 GB
    ArrayList                2.8 GB

You start investigating the objects with the largest retained heap.

6️⃣ Path to GC Roots ⭐⭐⭐ MOST IMPORTANT

Now suppose you find:

    5 million User objects

You ask:

    Why can't GC remove them?

Use:

    Path to GC Roots

It may show:

    GC Root
    ↓
    static UserCache
    ↓
    HashMap
    ↓
    User

💥 Now you found the reason!

The User object is still reachable because:

    static UserCache
    ↓
    HashMap
    ↓
    User

As long as that reference exists:

GC ❌ cannot remove User
🎯 Complete Example

Imagine you have:

    public class UserCache {
    
        private static Map<String, User> cache =
                new HashMap<>();
    }
    
    The heap dump shows:
    
    Histogram
    User = 5,000,000 instances
    
    Then:
    
        Dominator Tree
        UserCache → 3 GB retained
    
    Then:
    
    Path to GC Roots
    
            GC Root
            ↓
            UserCache.cache (static)
            ↓
            HashMap
            ↓
            5,000,000 User objects
    
    Now we can conclude:
    
    Root Cause:
    
        Static HashMap is retaining
        millions of User objects.
⭐ The exact workflow I want you to remember

    Open Heap Dump
    ↓
    1. Leak Suspects Report
       ↓
       2. Histogram
          "What objects are unusually large?"
          ↓
       3. Dominator Tree
          "Who retains the most memory?"
          ↓
       4. Check Retained Size
          "How much memory does this object keep alive?"
          ↓
       5. Path to GC Roots
          "WHY is this object still alive?"
          ↓
       6. Find the reference in application code
          ↓
       7. Fix root cause



First: What is a GC Root?

Before understanding Path to GC Roots, understand this:

Garbage Collector starts from certain special references called GC Roots.

Conceptually:

GC Root
↓
Object A
↓
Object B
↓
Object C

As long as Object C can be reached from a GC Root:

GC ❌ cannot delete Object C
Simple example

Suppose you have:

public class UserCache {

    private static final Map<String, User> cache = new HashMap<>();

    public static void addUser(User user) {
        cache.put(user.getId(), user);
    }
}

Requests keep coming:

Request 1 → User1
Request 2 → User2
Request 3 → User3
...
Request 1,000,000 → User1,000,000

Now let's see the references.

GC Root
↓
UserCache class
↓
static cache
↓
HashMap
↓
User1
User2
User3
User4
...
Why is User1 not garbage collected?

Suppose User1 is no longer needed.

You might think:

User1 is old
↓
GC should remove it

But GC doesn't decide based on whether an object is old or useful.

GC asks:

Can I still reach this object from a GC Root?

Let's trace:

GC Root
↓
UserCache class
↓
static cache
↓
HashMap
↓
User1

Answer:

YES, User1 is reachable.

Therefore:

GC ❌ cannot remove User1


------------------------------------------------------------------------------------------------------------------------------------

1️⃣ What is a Thread Dump?

A thread dump is a snapshot of all threads currently running inside the JVM.

Think of it like taking a photo 📸 of your application's threads at one moment.

JVM
│
├── Thread-1 → RUNNABLE
├── Thread-2 → WAITING
├── Thread-3 → BLOCKED
├── Thread-4 → TIMED_WAITING
├── http-nio-8080-exec-1 → RUNNABLE
├── pool-1-thread-1 → WAITING
└── ...

A thread dump helps us answer:

How many threads exist, what are they doing, and where were they created/executing?

2️⃣ What information does a Thread Dump contain?

For each thread, you typically see:

Thread Name
Thread State
Stack Trace
Locks

Example:

"pool-1-thread-45"
java.lang.Thread.State: WAITING

at java.util.concurrent...
at com.example.OrderService.process(OrderService.java:50)

This tells us:

Thread name → pool-1-thread-45

State → WAITING

What it is doing → waiting

Application code → OrderService.process()
3️⃣ How do we take a Thread Dump?
Method 1: jcmd ⭐ Recommended

First find the Java process PID.

On Windows:

jps -l

You might see:

12345 com.example.MyApplication

Then take the thread dump:

jcmd 12345 Thread.print

You can save it to a file:

jcmd 12345 Thread.print > thread-dump.txt
Method 2: jstack
jstack 12345 > thread-dump.txt

Where:

12345 = Java Process PID
Method 3: IntelliJ ⭐ Easy for local development

When debugging locally, IntelliJ can show running threads through its debugger tools.

But in production, command-line tools such as jcmd are commonly useful.

4️⃣ What do we observe inside the Thread Dump? ⭐⭐⭐

For our native thread OOM, we mainly check these things:

1. How many threads exist?
2. What are their names?
3. Are many threads of the same type?
4. What state are they in?
5. What are they doing?
6. Where is the application code involved?

Let's understand each.

5️⃣ Look for repeated thread names ⭐⭐⭐

Suppose your dump contains:

pool-1-thread-1
pool-1-thread-2
pool-1-thread-3
...
pool-1-thread-5000 😨

Immediately ask:

Why does this pool have 5,000 threads?

Maybe your code is creating an unlimited thread pool.

Example 🚨

Bad code:

ExecutorService executor = Executors.newCachedThreadPool();

This can create more threads as demand increases.

Under uncontrolled workload:

100 threads
↓
1,000 threads
↓
5,000 threads

Eventually:

unable to create new native thread 💥

The thread dump helps you see that many threads belong to the same pool.

6️⃣ Check Thread States

Common states:

RUNNABLE
BLOCKED
WAITING
TIMED_WAITING

For now, understand them simply.

RUNNABLE
Thread is executing or ready to execute
BLOCKED
Waiting to acquire a monitor lock

Example:

synchronized(lock) {
// work
}

If another thread owns the lock:

Thread → BLOCKED
WAITING
Waiting indefinitely for something

For example:

Waiting for a task
Waiting for another thread
Waiting on a queue
TIMED_WAITING
Waiting for a specified time

Examples:

Thread.sleep()
wait(timeout)
7️⃣ Look at what the threads are doing ⭐⭐⭐

Suppose you have 2,000 threads like:

"pool-1-thread-1"
WAITING
at java.util.concurrent.FutureTask.awaitDone()

"pool-1-thread-2"
WAITING
at java.util.concurrent.FutureTask.awaitDone()

"pool-1-thread-3"
WAITING
at java.util.concurrent.FutureTask.awaitDone()

Now you ask:

Why do we have 2,000 threads waiting for something?

Maybe a downstream service is slow.

So threads are accumulating.

🔥 Very important: Thread Dump helps identify the pattern

Suppose your application has:

5,000 threads

That alone is not enough.

We want to group them:

4,500 → HTTP client threads
300   → Tomcat threads
100   → Kafka threads
50    → JVM/system threads

Now we know:

⭐ Most of the threads belong to the HTTP client.

Then inspect their stack traces.

8️⃣ Example of a real investigation

You get:

unable to create new native thread
Step 1: Check metric
jvm.threads.live = 10,000 🚨
Step 2: Take Thread Dump
jcmd 12345 Thread.print > thread-dump.txt
Step 3: Look at thread names
http-client-1
http-client-2
http-client-3
...
http-client-9000

🚨 Suspicious!

Step 4: Check their stack traces

You find many are waiting around:

com.example.PaymentClient.call()
Step 5: Investigate the code

Maybe every request creates a new client/thread instead of reusing a properly configured pool.

Now you have a direction for finding the root cause.

🎯 What a Thread Dump tells us
What to check	Question
Thread count	Do we have too many threads?
Thread names	Which component created them?
Repeated patterns	Which thread group is growing?
Thread state	What are they doing?
Stack trace	Where are they executing/waiting?
Locks	Are they blocked by something?


---------------------------------------------------------------------------------------------------------------------------------------------



strongly suspect from a thread dump.

🔥 Common problems found from a Thread Dump
1️⃣ Thread leak ⭐⭐⭐

The application keeps creating threads, but they never terminate.

Thread dump:

worker-1
worker-2
worker-3
...
worker-5000
worker-10000

Metrics:

100 → 500 → 2000 → 5000 → 10000 📈
Likely cause

Code repeatedly creates threads:

new Thread(() -> process()).start();
Fix

Use a properly configured bounded thread pool instead of creating unlimited threads.

2️⃣ Unbounded thread pool ⭐⭐⭐

A common suspicious configuration:

Executors.newCachedThreadPool();

Under heavy load:

Requests ↑
↓
Tasks ↑
↓
New threads created ↑
↓
Thread count ↑↑

Thread dump might show:

pool-1-thread-1
pool-1-thread-2
...
pool-1-thread-5000
Fix

Use a bounded ThreadPoolExecutor with controlled limits.

3️⃣ Creating a new ExecutorService repeatedly ⭐⭐⭐

For example:

public void process() {
ExecutorService executor =
Executors.newFixedThreadPool(10);

    executor.submit(task);
}

Imagine this method is called repeatedly.

Request 1 → Creates Executor
Request 2 → Creates Executor
Request 3 → Creates Executor

You may end up with many pools and threads.

Thread dump pattern
pool-1-thread-1
pool-2-thread-1
pool-3-thread-1
pool-4-thread-1
...
Fix

Create and manage the executor centrally and reuse it.

4️⃣ Threads stuck waiting for a slow downstream service ⭐⭐⭐

This is a very realistic production scenario.

Request
↓
Calls Payment Service
↓
Payment Service is slow 🐌
↓
Threads remain occupied

Thread dump may show many threads around:

PaymentClient.call()

or network/HTTP waiting code.

500 threads
↓
All waiting for Payment Service
Root cause

Not necessarily a thread leak.

Instead:

Slow downstream service
↓
Threads remain busy for a long time
↓
More work accumulates
↓
Thread exhaustion
Fix
Timeouts
Circuit breaker
Bulkhead
Proper pool limits
Backpressure
5️⃣ Too many BLOCKED threads — Lock contention ⭐⭐⭐

Thread dump:

500 threads → BLOCKED

All waiting for the same lock:

synchronized(lock) {
process();
}

Conceptually:

Thread 1 → owns lock 🔒

Thread 2 → BLOCKED
Thread 3 → BLOCKED
Thread 4 → BLOCKED
...
Problem

Threads pile up waiting for one lock.

Fix
Reduce synchronization scope
Avoid slow operations inside synchronized blocks
Use better concurrency design
6️⃣ Deadlock

Thread dump may explicitly show something like:

Found one Java-level deadlock

Example:

Thread A holds Lock 1
↓ waits for Lock 2

Thread B holds Lock 2
↓ waits for Lock 1
A 🔒 → waiting for B's lock
B 🔒 → waiting for A's lock

Neither can continue.

Important distinction

Deadlock itself doesn't necessarily directly cause:

unable to create new native thread

But if your application keeps creating replacement/new threads while existing work is stuck, it can contribute to thread exhaustion.

7️⃣ Too many threads WAITING ⭐⭐⭐

You might see:

WAITING = 5,000 threads

Then check:

What are they waiting for?

Examples:

Waiting for database connection
Waiting for HTTP response
Waiting for Future.get()
Waiting for queue
Waiting for another thread

The stack trace tells you what they are waiting on.

8️⃣ Database connection pool exhaustion

Example flow:

500 request threads
↓
Only 20 DB connections
↓
480 threads waiting

Thread dump may show many threads waiting around connection acquisition.

Conceptually:

Thread
↓
DataSource.getConnection()
↓
WAITING
Root cause

The DB pool is exhausted, often because:

Queries are slow
Connections aren't returned
Pool size is too small for the workload
9️⃣ Recursive thread creation

Example:

void process() {
new Thread(() -> process()).start();
}

This is an obvious bug.

1 thread
↓
2 threads
↓
4 threads
↓
8 threads
↓
...

Eventually:

💥 unable to create new native thread
🔟 External library / framework creating too many threads

Sometimes your application code doesn't directly contain:

new Thread()

A library might create them.

Thread names help here:

kafka-producer-network-thread
HikariPool-1-housekeeper
http-nio-8080-exec-10

The thread name can tell you:

Which component owns these threads?

--------------------------------------------------------------------------------------------------------------------


Example: Dynamic Proxy Problem

Normally, proxies are fine.

For example, Spring creates proxies for things like:

@Transactional
@Async
@Cacheable

Spring may generate proxy classes.

But normally:

Application starts
↓
Spring creates required proxies
↓
Proxy classes stabilize ✅

You should not see unlimited growth.

🚨 The problematic scenario

Imagine something is generating new classes continuously:

Request 1
↓
Generate Proxy Class #1

Request 2
↓
Generate Proxy Class #2

Request 3
↓
Generate Proxy Class #3

Request 4
↓
Generate Proxy Class #4

Eventually:

10,000 classes
20,000 classes
50,000 classes
100,000 classes
↓
Metaspace 💥
1️⃣ How do we detect this?
Check Metaspace trend
200 MB
↓
300 MB
↓
500 MB
↓
800 MB

🚨 Metaspace continuously grows.

Then check loaded class count

For example:

10,000
↓
12,000
↓
15,000
↓
25,000
↓
50,000

The important thing is:

Does the number of loaded classes continuously increase while the application is running?

If yes, investigate dynamic class generation.

2️⃣ Correlate class growth with application activity ⭐⭐⭐

This is very important.

Suppose you see:

Application starts
Loaded classes = 15,000

Then the application is idle:

15,000
15,001
15,000

Fine. ✅

But under traffic:

100 requests → 20,000 classes

1,000 requests → 40,000 classes

10,000 requests → 100,000 classes 🚨

Then you have a huge clue:

Requests
↓
Something is generating classes/proxies
3️⃣ How do we identify WHICH classes are being generated?

This is where you need more detailed JVM diagnostics.

You can inspect loaded classes and look for suspicious naming patterns.

For example, generated proxies may have names resembling:

com.example.UserService$$SpringCGLIB$$...

or JDK dynamic proxy classes resembling:

jdk.proxy...

Or classes generated by libraries such as bytecode-generation frameworks.

The exact names depend on the proxy/generation mechanism.

The key is:

Which class names are continuously increasing?

For example:

UserService$$Proxy1
UserService$$Proxy2
UserService$$Proxy3
...
UserService$$Proxy50000

🚨 Now you know proxies are likely involved.

4️⃣ Find the code generating proxies

Now search your codebase for things such as:

Proxy.newProxyInstance()

Or libraries/framework APIs that dynamically generate classes.

You ask:

Why is this proxy being created repeatedly?

🚨 Bad conceptual design

Imagine doing something like this for every request:

public void process() {

    Object proxy = createNewProxy();

    proxy.execute();
}

Every request:

Request
↓
Create proxy
↓
Generate/load classes repeatedly

Depending on the mechanism and caching behavior, this can lead to excessive class generation.

✅ Better design

Create and reuse the proxy appropriately:

Application startup
↓
Create/configure proxy
↓
Reuse it for requests

Instead of:

Every request
↓
Generate new proxy/class ❌
⭐ Important correction about Spring proxies

Don't think:

@Transactional
↓
Every request creates a new proxy

❌ That is generally not how normal Spring AOP works.

Usually:

Spring starts
↓
Creates proxy infrastructure
↓
Proxy is reused
↓
Requests use the existing proxy

So if you see Metaspace continuously growing, don't immediately blame @Transactional or Spring proxies.

Investigate whether something is dynamically creating new proxy classes repeatedly.

5️⃣ What would we fix?

It depends on what we find.

What we find	Fix
Proxy generated repeatedly	Reuse/cache the proxy
New proxy per request	Create once instead of per request
Dynamic class generation bug	Fix generation logic
Library generating unlimited classes	Fix configuration/update library
Class generation based on unlimited unique inputs	Bound/cache/reuse generation appropriately
ClassLoader leak	Fix the reference retaining old ClassLoader
🎯 Production debugging flow
Metaspace 📈
↓
Loaded class count 📈
↓
Does growth correlate with traffic?
↓
YES
↓
Identify class-name pattern
↓
Are generated proxy classes growing?
↓
YES
↓
Which library/framework creates them?
↓
Why are new classes generated repeatedly?
↓
Reuse/cache/fix generation logic
🧠 The key distinction
ClassLoader leak:
Old classes cannot unload
because old ClassLoader is retained
Dynamic proxy/class generation issue:


-----------------------------------------------------------------------------------------------------------------------------------------------------


OOM scenarios you have covered
1️⃣ Java heap space ⭐⭐⭐⭐⭐

You know:

Symptoms
↓
Heap usage metrics
↓
GC behavior
↓
Memory trend
↓
Heap dump
↓
Histogram
↓
Dominator Tree
↓
Retained Size
↓
Path to GC Roots
↓
Root cause
↓
Fix

Common causes:

Memory leak / unwanted retention
Unbounded cache
Growing collections
Large byte[] / files
Large DB results
Missing pagination
Unbounded queues
Too many objects
High legitimate memory requirement
2️⃣ GC overhead limit exceeded ⭐⭐⭐⭐

You understand:

Heap almost full
↓
GC runs continuously
↓
GC frees very little memory
↓
Application spends excessive time in GC
↓
OOM

Debugging:

Heap usage
GC frequency
GC time
Memory after GC
Heap dump
3️⃣ Unable to create new native thread ⭐⭐⭐⭐⭐

You covered:

Check thread count
↓
Check thread trend
↓
Take Thread Dump
↓
Group threads
↓
Check thread states
↓
Check stack traces
↓
Find root cause

Common causes:

Direct:
Thread leak
Unbounded thread pool
Creating executors repeatedly
Recursive thread creation
OS/container limits
Native memory shortage
Large thread stacks
Indirect:
Slow downstream service
DB connection pool exhaustion
Lock contention
Deadlock
4️⃣ Metaspace ⭐⭐⭐⭐

You covered:

Check Metaspace trend
↓
Check loaded classes
↓
Check unloaded classes
↓
Identify whether classes keep increasing

Then distinguish:

ClassLoader leak
Old ClassLoader retained
↓
Classes cannot unload
↓
Metaspace grows
Dynamic class generation
New classes/proxies continuously generated
↓
Loaded class count grows
↓
Metaspace grows
🎯 For scenario-based interviews, ONE thing is still important

You should now learn how to answer an OOM scenario from beginning to end.

For example, the interviewer says:

🚨 "Our Spring Boot service crashed with OutOfMemoryError. How would you debug it?"

You should first say:

1. Identify the exact OOM type
   ↓
   Java heap space?
   GC overhead?
   Metaspace?
   Native thread?
   ↓
2. Check monitoring metrics
   ↓
3. Capture the correct diagnostic data
   ↓
   Heap Dump / Thread Dump / Class loading data
   ↓
4. Find the root cause
   ↓
5. Fix it
   ↓
6. Verify after deployment
   ↓
7. Add monitoring/alerts to prevent recurrence