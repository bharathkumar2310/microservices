Metric #2: Request Rate / RPS

The key question is:

    How much traffic is my service receiving?

What RPS tells you

    RPS = Requests Per Second

Example:

    Normal → 500 RPS
    Suddenly → 2,000 RPS

That tells you traffic increased 4×.

But remember:

    High RPS itself is not necessarily a problem.

    A service might handle 10,000 RPS easily. Another might struggle at 1,000 RPS.

So you always correlate RPS with latency and resource utilization.

RPS troubleshooting table


| RPS        | Latency | CPU    | Other finding           | Likely meaning                           | What to do                         |
| ---------- | ------- | ------ | ----------------------- | ---------------------------------------- | ---------------------------------- |
| ↑          | ↑       | ↑      | —                       | Traffic causing CPU saturation           | Scale/optimize                     |
| ↑          | ↑       | Normal | DB latency ↑            | Traffic stressing DB                     | Investigate DB capacity/query      |
| ↑          | ↑       | Normal | DB normal, downstream ↑ | Traffic stressing dependency             | Investigate downstream             |
| ↑          | Normal  | ↑      | CPU near limit          | Service approaching capacity             | Consider scaling                   |
| ↑          | Normal  | Normal | Everything healthy      | Service handling traffic                 | No immediate action                |
| Normal     | ↑       | ↑      | —                       | Not a traffic problem                    | Investigate CPU/code               |
| Normal     | ↑       | Normal | DB latency ↑            | DB bottleneck                            | Investigate DB                     |
| Normal     | ↑       | Normal | Downstream latency ↑    | Dependency bottleneck                    | Investigate downstream             |
| ↓ suddenly | ↑       | Normal | Errors ↑                | Traffic may actually be failing/rejected | Check gateway/load balancer/errors |
| ↑ suddenly | ↑       | ↑      | DB connections ↑        | Traffic overload propagating to DB       | Scale carefully + protect DB       |





The most important scenario

Suppose:

Before:
RPS = 500
CPU = 40%
Latency = 200 ms


Now:
RPS = 2,000
CPU = 95%
Latency = 2 sec

Your reasoning:

Traffic increased → CPU became saturated → latency increased.

Likely solution:

Scale horizontally, then verify DB/downstream capacity.

But look at this
Before:
RPS = 500
CPU = 40%
Latency = 200 ms


Now:
RPS = 500
CPU = 90%
Latency = 2 sec

RPS didn't change.

So traffic isn't the explanation.

Now investigate:

Recent deployment?
CPU-intensive code?
Infinite/expensive loop?
GC?
Thread behavior?
One important interview concept: Capacity

Think of every service as having some practical capacity.

For example:

Service capacity ≈ 1,000 RPS

At:

500 RPS → healthy
800 RPS → healthy
1,000 RPS → near saturation
1,500 RPS → latency/errors increase

So when RPS rises, you're really asking:

"Has traffic exceeded the service's sustainable capacity?"

That's why RPS should almost always be correlated with CPU, latency, threads, DB and downstream metrics.

What you should say in an interview

If asked:

"Traffic suddenly increased. What do you do?"

Don't list 20 things.

Say:

"First I'd quantify the increase in RPS and compare it with the service's normal baseline. Then I'd check whether the increased traffic is causing saturation in CPU, thread pools, database connections or downstream dependencies. If the service itself is the bottleneck and dependencies have capacity, I'd scale horizontally and verify the metrics after scaling."