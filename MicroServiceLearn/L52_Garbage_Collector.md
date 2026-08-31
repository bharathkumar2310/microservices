1. First understand what GC does

        Java objects are created in the JVM heap:

        Request
        ↓
        Create objects
        ↓
        Objects become unused
        ↓
        Garbage Collector finds them
        ↓
        Reclaims memory

        GC itself is normal. The problem is excessive or long GC activity.

2. How GC causes API latency

Imagine your API normally takes:

        200 ms

Now the application needs to perform a GC cycle.

    For some GC phases, application threads may be paused:

    Request arrives
    ↓
    Application starts processing
    ↓
    GC pause starts
    ↓
    Application threads pause ⏸️
    ↓
    GC finishes
    ↓
    Application resumes
    ↓
    Request completes

If the GC pause is:

    2 seconds

your API could suddenly take:

    Normal processing = 200 ms
    GC pause           = 2,000 ms
    --------------------------------
    Total              ≈ 2,200 ms

So your P99 latency can shoot up even though the application code itself isn't necessarily slow.

3. But why does excessive GC happen?

        This is the really important part.

Case 1 — Excessive object creation

Suppose every request creates a huge number of temporary objects:

        Request
        ├── Object A
        ├── Object B
        ├── Object C
        ├── 1000 temporary objects
        └── Large JSON objects

Those objects quickly become unreachable.

    Allocation rate ↑
    ↓
    Garbage ↑
    ↓
    GC runs more frequently
    ↓
    CPU spent on GC ↑
    ↓
    Application has less CPU
    ↓
    Latency ↑

Case 2 — Heap pressure

    Suppose your heap is getting close to full:

        Heap
        
        ███████████████████░ 95%
        
        The JVM has to work harder to reclaim memory.

You may see:

    GC frequency ↑
    GC duration ↑
    CPU ↑
    API latency ↑

Case 3 — Memory leak / excessive object retention

    Something keeps references to objects that are no longer logically needed.

For example:

    Cache
    ├── Object 1
    ├── Object 2
    ├── Object 3
    ├── ...
    └── millions of objects

The GC can't reclaim objects that are still referenced.

Eventually:

    Old Gen usage ↑
    ↓
    Major/old-generation GC ↑
    ↓
    Longer pauses
    ↓
    Latency ↑

4. What would you see in monitoring?

Suppose your API latency suddenly changes:

P99:
300 ms → 5 sec

You check GC:

    GC count        ↑↑
    GC pause time   ↑↑
    Heap usage      ↑
    CPU             ↑

That gives you a strong indication that GC is contributing to latency.

But remember:

    High heap alone does not automatically mean GC is the cause.

You want to correlate:

    Latency spike ↔ GC activity ↔ GC pause duration

5. How do we prevent GC-related latency?

Don't start by blindly increasing the heap.

First find why so much garbage is being created.

Prevention hierarchy

1. Reduce unnecessary object creation

        Avoid creating huge numbers of temporary objects unnecessarily.

2. Fix memory leaks

Check things such as:

        Static collections
        Unbounded caches
        Objects being retained unnecessarily
        Incorrect cache configuration

3. Optimize large allocations

For example:

Bad:
    
    Load 1 million records
    ↓
    Create objects for all 1 million
    ↓
    Huge allocation

Instead:

    Pagination
    ↓
    Fetch smaller batches
    ↓
    Process
    ↓
    Release

4. Tune heap appropriately

        If the application genuinely needs more memory, configure an appropriate heap size.

But:

    Increasing heap is not a substitute for fixing a memory leak.

5. Choose/tune the GC appropriately

        Modern JVMs provide collectors such as G1 and ZGC, depending on your Java version and latency requirements.

6. The interview troubleshooting flow

If interviewer says:

"API latency increased and GC is high. What will you do?"

Your answer should be:

    Latency increased
    ↓
    Check GC metrics
    ↓
    GC frequency increased?
    GC pause duration increased?
    Heap usage increased?
    ↓
    Yes
    ↓
    Find why allocation/retention increased
    ↓
    Check recent deployment
    ↓
    Heap dump / allocation profiling
    ↓
    Identify excessive allocation or memory leak
    ↓
    Fix code/configuration
    ↓
    Verify GC + latency returned to normal




| Scenario                              | What you observe                            | How you investigate                         | Typical root cause                             |
| ------------------------------------- | ------------------------------------------- | ------------------------------------------- | ---------------------------------------------- |
| **1. Traffic increased**              | Traffic ↑, GC ↑                             | Compare GC increase with traffic increase   | More legitimate workload                       |
| **2. Allocation rate increased**      | Traffic same, allocation ↑                  | Allocation profiling / APM                  | Code creating excessive temporary objects      |
| **3. Recent deployment**              | GC ↑ immediately after release              | Compare old vs new version; profiling       | Code regression                                |
| **4. Heap filling quickly**           | Heap ↑ rapidly, GC ↑                        | Heap/GC metrics, allocation profiling       | Excessive allocation                           |
| **5. Old Gen continuously increases** | Old Gen doesn't return to baseline after GC | Heap dump, retained-object analysis         | Memory leak / objects retained                 |
| **6. Large objects**                  | Large allocation spikes                     | Allocation profiler / heap analysis         | Large arrays, huge JSON, large responses       |
| **7. Cache growth**                   | Heap ↑ as cache grows                       | Inspect cache size/entries                  | Unbounded cache                                |
| **8. Excessive JSON processing**      | Allocation ↑ during API calls               | Profile serialization/deserialization       | Huge payloads / inefficient mapping            |
| **9. Large DB result sets**           | Allocation ↑ during DB calls                | Check query result size and object creation | Loading too many records                       |
| **10. Temporary object explosion**    | Short-lived objects ↑                       | Allocation profiling                        | Loops/mapping/conversion creating many objects |



| Tool/concept             | Question it answers                                                               |
| ------------------------ | --------------------------------------------------------------------------------- |
| **Allocation profiling** | **What is creating objects?**                                                     |
| **Heap dump**            | **What objects are currently occupying memory, and why are they still retained?** |
| **GC metrics**           | **How frequently/how long is GC happening?**                                      |


1. "Too many temporary objects are being created"

        You cannot usually identify this from normal GC metrics alone.

You first notice:

    GC ↑
    Allocation rate ↑
    Traffic → normal
    Latency ↑

Now ask:

Which code is allocating these objects?

Where do you check?

        JFR / allocation profiling

You run a JFR recording and look at allocation hotspots.

For example:

Allocation hotspot

    OrderService.getOrders()
    ↓
    OrderMapper.map()
    ↓
    OrderDTO
    ↓
    500 MB allocated

Then you go to the actual application code:

    for (Order order : orders) {
    OrderDTO dto = new OrderDTO(...);
    }

Maybe you're creating hundreds of thousands of DTOs unnecessarily.

So the investigation is:

        GC ↑
        ↓
        Allocation ↑
        ↓
        JFR allocation profiling
        ↓
        Which class/method allocates?
        ↓
        Open that code
        ↓
        Understand why so many objects are created
2. "Memory leak / objects being retained"

This is different.

You observe:

    Heap usage
    2 GB
    ↓
    2.5 GB
    ↓
    3 GB
    ↓
    3.5 GB

Even after GC → memory doesn't come down significantly

Now you're asking:

    Why aren't these objects being garbage collected?

Where do you check?

    Heap dump

Take a heap dump and open it in something like Eclipse MAT.

You might find:

        HashMap
        ↓
        Cache
        ↓
        1,000,000 Order objects

Then inspect the retention/reference chain.

You discover:

    static Map<String, Order> cache;
    
    And the cache has no size limit.
    
    Now you've found the root cause:
    
    Unbounded cache
    ↓
    Objects remain referenced
    ↓
    GC cannot reclaim them
    ↓
    Old Gen ↑
    ↓
    GC pressure ↑
    ↓
    Latency ↑
3. "Loading 1 million records"

This one is slightly different.

Suppose:

        API request
        ↓
        DB
        ↓
        1,000,000 rows
        ↓
        JPA creates 1,000,000 entities
        ↓
        Huge memory allocation
        ↓
        GC ↑
Where do you check?

You don't necessarily need a heap dump first.

You investigate the request → DB → object creation path.

Look at:

A. DB metrics

Check:

    Rows returned
    Query execution time
    Result-set size
B. Application tracing/APM

Trace:

        API
        ↓
        Repository
        ↓
        SQL query
        ↓
        Entity mapping
C. Code

You might find:

    List<Order> orders = orderRepository.findAll();

That's your problem.

Instead:

        Page<Order> orders =
        orderRepository.findAll(pageable);

Now you're fetching smaller batche