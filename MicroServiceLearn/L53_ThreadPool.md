| Metric                  | What it tells you                              | Suspicious sign                 |
| ----------------------- | ---------------------------------------------- | ------------------------------- |
| **Active threads**      | Threads currently doing work                   | Near maximum                    |
| **Pool size**           | Number of threads available                    | Too small for workload          |
| **Max pool size**       | Maximum threads allowed                        | Constantly reached              |
| **Queue size**          | Tasks waiting for threads                      | Continuously increasing         |
| **Completed tasks**     | Work being processed                           | Drops relative to incoming work |
| **Rejected tasks**      | Tasks rejected because pool is full            | Increasing                      |
| **Task execution time** | How long threads stay occupied                 | Increasing                      |
| **Request rate**        | Incoming workload                              | Sudden increase                 |
| **CPU**                 | Whether threads are CPU-bound                  | CPU near saturation             |
| **GC**                  | Whether threads are affected by GC             | Long/frequent GC                |
| **DB connection pool**  | Whether threads are waiting for DB             | Pending connections high        |
| **Downstream latency**  | Whether threads are blocked on another service | Downstream calls slow           |
