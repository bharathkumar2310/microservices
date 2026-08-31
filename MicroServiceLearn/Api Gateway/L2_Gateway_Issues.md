| Problem                  | Route Matched? | Service Reached? | Typical Result         |
| ------------------------ | -------------: | ---------------: |------------------------|
| No matching route        |              ❌ |                ❌ | 404                    |
| Wrong Path predicate     |              ❌ |                ❌ | 404                    |
| Wrong Method predicate   |              ❌ |                ❌ | 404                    |
| Missing Header           |              ❌ |                ❌ | 404                    |
| Wrong Query parameter    |              ❌ |                ❌ | 404                    |
| Wrong Host               |              ❌ |                ❌ | 404                    |
| Service not running      |              ✅ |                ❌ | 502 / connection error |
| Wrong port               |              ✅ |                ❌ | 502 / connection error |
| DNS/hostname failure     |              ✅ |                ❌ | 502                    |
| Backend endpoint missing |              ✅ |                ✅ | 404                    |
| Backend is slow          |              ✅ |                ✅ | 504 Slow/timeout       |
| Backend throws exception |              ✅ |                ✅ | 500                    |

    So for routing if the predicate donot match we get 404
    if predicte match and service is not running in the port / wrong port ---> 502 bad gateway connection refused
    if uri=http://user-servic:8081 doesnot resolve to a IP adress ---> 502 Bad Gateway Unknown host exception