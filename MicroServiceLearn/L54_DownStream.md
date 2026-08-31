| Metric                    | What it tells you                       | Suspicious          |
| ------------------------- | --------------------------------------- | ------------------- |
| **Downstream latency**    | How long dependency takes               | P95/P99 increases   |
| **Request rate**          | How many calls you're making            | Unexpected increase |
| **Error rate**            | Dependency failures                     | 5xx/4xx increase    |
| **Timeout count**         | Calls exceeding timeout                 | Increasing          |
| **Retry count**           | Calls being repeated                    | Increasing          |
| **Connection pool**       | Available connections                   | Pool exhausted      |
| **Circuit breaker state** | Dependency health                       | OPEN                |
| **HTTP status codes**     | Type of failure                         | 5xx, 429 etc.       |
| **Network latency**       | Time spent communicating                | Increased           |
| **Dependency CPU/memory** | Whether downstream itself is overloaded | Saturation          |
| **Dependency DB latency** | Downstream may itself be waiting on DB  | Increased           |
