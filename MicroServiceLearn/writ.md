Scenario 1 — API suddenly becomes slow

Imagine you're on production support.

Interviewer:
“Users are reporting that the GET /orders/{id} API has suddenly become very slow. It normally takes 200 ms, but now users are seeing 5–10 seconds. How would you investigate this?”

First I will verify whether we still have this issue or this issue was present previously and not there now
Then I would look into all monitoring metric ike traffics, cpu, memory, latency, db latency, threads, gc etc
Based on theese metrics combinations I wil try to narrow down the problem
Once the problem is narrowed down I will look into distribute traicng to find where exactly we have the issue
Once aftre finding where I iwill ook into logs and code to find out why
after finding out wat, where and why I will fix the issue, and then verify it agian whether we have thes issue or not



for db the part to know

monitoring ---> apm/distributed tecing---> Explain/logs/code ---> fix ---> verif__-> monitoring
some scenatios like

index issues, db optimizer issue , n+11 query, pagination, huge db, connection ppooling, transaction issues, new deployments

then know metrics like p95, 99, query latency, connection pool metricses, db cpu, 



select emp_name feom employee e where salary = (select max(salary) from employee e1 where e.dept_id = e1.dept_id)