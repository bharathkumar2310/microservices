What is an API Gateway?

    An API Gateway is a single entry point for clients to access multiple microservices.
    An API Gateway is a centralized entry point that manages client requests and directs them to the appropriate backend services. It simplifies communication between clients and multiple microservices while enforcing security and performance policies.
    
    Routes client requests, handles authentication and rate limiting, while acting as a reverse proxy to hide internal service complexity.
    Provides a single, unified API interface that simplifies communication between clients and multiple backend services.


Instead of this:

        Client
        │
        ├── User Service
        ├── Order Service
        ├── Payment Service
        └── Product Service

The client communicates like this:

        Client
        │
        ▼
        API Gateway
        │
        ├── User Service
        ├── Order Service
        ├── Payment Service
        └── Product Service


Problem 1: Client needs to know every microservice


Imagine you have:

    User Service → user-service:8081
    Product Service → product-service:8082
    Order Service → order-service:8083
    Payment Service → payment-service:8084

Without Gateway:

    Frontend
    │
    ├── calls User Service
    ├── calls Product Service
    ├── calls Order Service
    └── calls Payment Service
Problems:

The frontend needs to know:

        Which services exist
        Their URLs
        Their ports
        How to communicate with each service

If a service changes location:

    Order Service:8083 → Order Service:9090

You may need to update clients.

With API Gateway:

    The client only knows:

            api.company.com

For example:

    /api/users
    /api/products
    /api/orders
    /api/payments

The gateway internally routes requests to the correct service.

    Client
    │
    ▼
    /api/orders
    │
    API Gateway
    │
    ▼
    Order Service

✅ The client doesn't need to know internal microservice locations.


Problem 2: Authentication gets duplicated

    Without a Gateway, potentially every service needs to deal with things like:

        Authentication
        JWT validation
        Security configuration
        CORS

Example:

        User Service     → JWT validation
        Order Service    → JWT validation
        Payment Service  → JWT validation
        Product Service  → JWT validation

This creates duplicated cross-cutting logic.

With Gateway:

        Client
        │ JWT
        ▼
        API Gateway
        │
        ├── User Service
        ├── Order Service
        └── Payment Service

The Gateway can handle common security concerns.



Problem 3: Cross-cutting concerns get repeated

        Many things are needed by multiple APIs:

        Authentication
        Rate limiting
        Logging
        Request tracing
        CORS
        SSL/TLS termination
        Request/response transformation

Without Gateway:

        User Service      → Rate limiting
        Order Service     → Rate limiting
        Payment Service   → Rate limiting
        Product Service   → Rate limiting

You may end up implementing similar infrastructure concerns repeatedly.

With Gateway:
    
                        ┌── Authentication
                        ├── Rate Limiting
    Client → Gateway ───┼── Logging
    ├── CORS
    └── Routing

Then requests go to the appropriate services.


--------------------------------------------------------------------------------------------------------------------------------------

1. What is a Route?

        A Route is simply a rule that tells the API Gateway:
        If a request matches this condition, send it to this destination.


2. Route ID

        Example:
        
            id=user-service-route
        
        The Route ID is simply a unique name for the route.

3. Predicate — How Gateway decides where to send the request

          A Predicate is a condition.

       Think of it like:

        if (condition is true) {
        select this route;
        }

For example:

        Path=/users/**

This means:

    If the request path starts with /users/, this route matches.


4. URI — Where should the request go?

        After finding the matching route, Gateway needs to know:
        
        Where should I forward this request?



TYPES OF PREDICATE 


Path Predicate

        This is the most commonly used routing condition.

Example:

    Path=/users/**

The ** means:

    Match everything after /users/.

Examples:

    /users/1
    /users/profile
    /users/address/home

All can match.

    

2. Method Predicate

        Route based on HTTP method.

For example:

        GET /users/**

could go to one route.

        POST /users/**

could potentially match another route.

Conceptually:

    Method=GET

The Gateway checks:

    Is the request method GET?

    YES → Route matches
    NO → Route doesn't match

3. Header Predicate

        Route based on a request header.

Example:

    Header=X-Version, v2

Meaning:

If request contains:

    X-Version: v2

then the route can match.

This can be useful for things like:

    API versioning
    Special clients
    Specific request types
    Query Predicate

4. Route based on query parameters.

Example request:

    /users?version=v2

Gateway can check:

    Query parameter version exists/matches

Again, this is just another condition.

5. Host Predicate

        Route based on domain.

Example:

    api.company.com

could route differently from:

    admin.company.com




        # ==========================================
        # API GATEWAY
        # ==========================================
        
        spring.application.name=api-gateway
        server.port=8080
        
        
        # ==========================================
        # 1. BASIC PATH ROUTING
        # ==========================================
        
        # USER SERVICE
        spring.cloud.gateway.routes[0].id=user-service-route
        spring.cloud.gateway.routes[0].uri=http://localhost:8081
        spring.cloud.gateway.routes[0].predicates[0]=Path=/users/**
        
        
        # ORDER SERVICE
        spring.cloud.gateway.routes[1].id=order-service-route
        spring.cloud.gateway.routes[1].uri=http://localhost:8082
        spring.cloud.gateway.routes[1].predicates[0]=Path=/orders/**
        
        
        # ==========================================
        # 2. METHOD PREDICATE
        # ==========================================
        
        # Only GET requests to /get-users/**
        spring.cloud.gateway.routes[2].id=get-users-route
        spring.cloud.gateway.routes[2].uri=http://localhost:8081
        spring.cloud.gateway.routes[2].predicates[0]=Path=/get-users/**
        spring.cloud.gateway.routes[2].predicates[1]=Method=GET
        
        
        # Only POST requests to /create-user/**
        spring.cloud.gateway.routes[3].id=create-user-route
        spring.cloud.gateway.routes[3].uri=http://localhost:8081
        spring.cloud.gateway.routes[3].predicates[0]=Path=/create-user/**
        spring.cloud.gateway.routes[3].predicates[1]=Method=POST
        
        
        # ==========================================
        # 3. HEADER PREDICATE
        # ==========================================
        
        # Route only when header X-Version has value v2
        spring.cloud.gateway.routes[4].id=user-v2-route
        spring.cloud.gateway.routes[4].uri=http://localhost:8081
        spring.cloud.gateway.routes[4].predicates[0]=Path=/v2/users/**
        spring.cloud.gateway.routes[4].predicates[1]=Header=X-Version, v2
        
        
        # ==========================================
        # 4. QUERY PARAMETER PREDICATE
        # ==========================================
        
        # Route when query parameter "version" exists
        spring.cloud.gateway.routes[5].id=user-query-route
        spring.cloud.gateway.routes[5].uri=http://localhost:8081
        spring.cloud.gateway.routes[5].predicates[0]=Path=/query/users/**
        spring.cloud.gateway.routes[5].predicates[1]=Query=version
        
        
        # Route when query parameter version matches v2
        spring.cloud.gateway.routes[6].id=user-query-v2-route
        spring.cloud.gateway.routes[6].uri=http://localhost:8081
        spring.cloud.gateway.routes[6].predicates[0]=Path=/query-v2/users/**
        spring.cloud.gateway.routes[6].predicates[1]=Query=version, v2
        
        
        # ==========================================
        # 5. HOST PREDICATE
        # ==========================================
        
        # Route requests based on Host header/domain
        spring.cloud.gateway.routes[7].id=user-host-route
        spring.cloud.gateway.routes[7].uri=http://localhost:8081
        spring.cloud.gateway.routes[7].predicates[0]=Host=users.example.com
        
        
        # ==========================================
        # 6. MULTIPLE PREDICATES (AND CONDITION)
        # ==========================================
        
        # Path AND Method must both match
        spring.cloud.gateway.routes[8].id=user-get-route
        spring.cloud.gateway.routes[8].uri=http://localhost:8081
        spring.cloud.gateway.routes[8].predicates[0]=Path=/special/users/**
        spring.cloud.gateway.routes[8].predicates[1]=Method=GET



-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------


What is a Gateway Filter?

      A Gateway Filter intercepts a request/response and performs some logic.

For example:

      Client Request
      ↓
      Filter
      ↓
      Microservice
      ↓
      Filter
      ↓
      Client Response

You can use filters for:

      Authentication
      Logging
      Adding headers
      Removing headers
      Modifying requests
      Modifying responses
      Rate limiting
      CORS-related handling
      Request validation



BUILT IN GATEWAY FILTERS :


| Filter                 | What it does                        |
| ---------------------- | ----------------------------------- |
| `AddRequestHeader`     | Adds a header to the request        |
| `AddResponseHeader`    | Adds a header to the response       |
| `RemoveRequestHeader`  | Removes a request header            |
| `RemoveResponseHeader` | Removes a response header           |
| `SetRequestHeader`     | Sets/replaces a request header      |
| `SetResponseHeader`    | Sets/replaces a response header     |
| `AddRequestParameter`  | Adds a query parameter              |
| `RewritePath`          | Changes the request URL path        |
| `StripPrefix`          | Removes parts of the URL path       |
| `PrefixPath`           | Adds a prefix to the URL path       |
| `RedirectTo`           | Redirects the client                |
| `Retry`                | Retries failed requests             |
| `RequestRateLimiter`   | Limits request rate                 |
| `CircuitBreaker`       | Handles failing downstream services |
| `ModifyRequestBody`    | Modifies request body               |
| `ModifyResponseBody`   | Modifies response body              |
| `RequestSize`          | Limits request size                 |
| `SaveSession`          | Saves session before forwarding     |



1️⃣ StripPrefix
What does it do?

      It removes a specified number of path segments from the beginning of the URL.

Example

      Client calls:

         /api/users/123

Gateway configuration:

      spring.cloud.gateway.routes[0].filters[0]=StripPrefix=1

      Gateway removes the first path segment:

      /api/users/123
      ↓
      /users/123

The microservice receives:

      GET /users/123
Why is this useful?

Suppose your public API is:

      /api/user-service/users/123

But your User Service endpoint is:

      /users/123

You don't want the User Service to know about:

         /api/user-service

So Gateway can remove those parts.

      spring.cloud.gateway.routes[0].filters[0]=StripPrefix=2
      /api/user-service/users/123
      ↓ Remove 2 segments
      /users/123
      Easy rule 🧠
      StripPrefix=N

👉 Remove the first N path segments.

2️⃣ PrefixPath

What does it do?

      It adds something to the beginning of the path.

Suppose the client calls:

      /users/123

Configuration:

         spring.cloud.gateway.routes[0].filters[0]=PrefixPath=/api

Gateway forwards:

      /api/users/123

So:

      /users/123
      ↓
      PrefixPath=/api
      ↓
      /api/users/123
When would we use this?

Suppose your microservice endpoints are:

      /api/users/**

But you want clients to call:

      /users/**

The Gateway adds /api.


3️⃣ RewritePath ⭐⭐⭐

      This is more flexible.

      It allows you to change the URL using a regular expression.

Example

Client calls:

      /old-api/users/123

But the backend expects:

      /users/123

Configuration:

      spring.cloud.gateway.routes[0].filters[0]=RewritePath=/old-api/(?<segment>.*), /${segment}

Conceptually:

      /old-api/users/123
      ↓
      /users/123
Another example

Client:

      /v1/users/123

Backend:

      /api/users/123

You can rewrite:

      /v1/users/123
      ↓
      /api/users/123

The important idea is:

      RewritePath uses regular expressions, so it can perform more complex URL transformations than StripPrefix.


4️⃣ RedirectTo

⚠️ This one is slightly different.

      The Gateway does not forward the request to the microservice.

      Instead, it tells the client to go somewhere else.

Example

      Client calls:

      /old-page

Gateway configuration:

      spring.cloud.gateway.routes[0].filters[0]=RedirectTo=302, https://example.com/new-page

Gateway responds:

      HTTP/1.1 302 Found
      Location: https://example.com/new-page

Then:

      Client
      │
      │ GET /old-page
      ▼
      Gateway
      │
      │ 302 Redirect
      │ Location: /new-page
      ▼
      Client makes another request
      │
      ▼
      New Location


---------------------------------------------------------------------------------------------------------------------------------------

CUSTOM FILTERS :

1️⃣ Custom GatewayFilter

      Suppose we want a filter that prints something before and after the request is processed.

Create the filter
      
      @Component
      public class CustomGatewayFilter implements GatewayFilter {
      
          @Override
          public Mono<Void> filter(ServerWebExchange exchange,
                                   GatewayFilterChain chain) {
      
              // BEFORE request goes to microservice
              System.out.println("Before calling microservice");
      
              return chain.filter(exchange)
                      .then(Mono.fromRunnable(() -> {
      
                          // AFTER response comes back
                          System.out.println("After receiving response");
      
                      }));
          }
      }

What is happening here?

      
      Client
      ↓
      Custom Filter
      ↓
      "Before calling microservice"
      ↓
      chain.filter(exchange)
      ↓
      Microservice
      ↓
      Response comes back
      ↓
      "After receiving response"
      ↓
      Client

The most important line is:

      chain.filter(exchange)

This means:

         Continue the filter chain and eventually forward the request to the next filter/downstream service.
         
         Without it, the request generally won't continue through the Gateway filter chain.



CONFIGURE IT :

      @Bean
      public RouteLocator customRouteLocator(
      RouteLocatorBuilder builder,
      MyCustomFilter myCustomFilter) {
      
          return builder.routes()
                  .route("user-service", r -> r
                          .path("/users/**")
                          .filters(f -> f.filter(myCustomFilter))
                          .uri("http://localhost:8081"))
                  .build();
      }


GlobalFilter

      A GlobalFilter applies to every request passing through the Gateway.

Example:

      @Component
      public class LoggingGlobalFilter implements GlobalFilter {
      
          @Override
          public Mono<Void> filter(ServerWebExchange exchange,
                                   GatewayFilterChain chain) {
      
              System.out.println(
                      "Request URI: " +
                      exchange.getRequest().getURI()
              );
      
              return chain.filter(exchange);
          }
      }

This automatically runs for:

      /users/**
      /orders/**
      /products/**
      /payments/**

because it is a GlobalFilter.

      
      # ================================
      # ROUTE 1 - AddRequestHeader
      # ================================
      spring.cloud.gateway.routes[0].id=add-request-header
      spring.cloud.gateway.routes[0].uri=http://localhost:8081
      spring.cloud.gateway.routes[0].predicates[0]=Path=/add-request/**
      spring.cloud.gateway.routes[0].filters[0]=AddRequestHeader=X-Gateway, API-Gateway
      
      
      # ================================
      # ROUTE 2 - AddResponseHeader
      # ================================
      spring.cloud.gateway.routes[1].id=add-response-header
      spring.cloud.gateway.routes[1].uri=http://localhost:8081
      spring.cloud.gateway.routes[1].predicates[0]=Path=/add-response/**
      spring.cloud.gateway.routes[1].filters[0]=AddResponseHeader=X-Gateway, API-Gateway
      
      
      # ================================
      # ROUTE 3 - RemoveRequestHeader
      # ================================
      spring.cloud.gateway.routes[2].id=remove-request-header
      spring.cloud.gateway.routes[2].uri=http://localhost:8081
      spring.cloud.gateway.routes[2].predicates[0]=Path=/remove-request/**
      spring.cloud.gateway.routes[2].filters[0]=RemoveRequestHeader=X-Internal-Header
      
      
      # ================================
      # ROUTE 4 - RemoveResponseHeader
      # ================================
      spring.cloud.gateway.routes[3].id=remove-response-header
      spring.cloud.gateway.routes[3].uri=http://localhost:8081
      spring.cloud.gateway.routes[3].predicates[0]=Path=/remove-response/**
      spring.cloud.gateway.routes[3].filters[0]=RemoveResponseHeader=X-Internal-Version
      
      
      # ================================
      # ROUTE 5 - SetRequestHeader
      # ================================
      spring.cloud.gateway.routes[4].id=set-request-header
      spring.cloud.gateway.routes[4].uri=http://localhost:8081
      spring.cloud.gateway.routes[4].predicates[0]=Path=/set-request/**
      spring.cloud.gateway.routes[4].filters[0]=SetRequestHeader=X-Source, API-Gateway
      
      
      # ================================
      # ROUTE 6 - SetResponseHeader
      # ================================
      spring.cloud.gateway.routes[5].id=set-response-header
      spring.cloud.gateway.routes[5].uri=http://localhost:8081
      spring.cloud.gateway.routes[5].predicates[0]=Path=/set-response/**
      spring.cloud.gateway.routes[5].filters[0]=SetResponseHeader=X-Version, v2
      
      
      # =================================================
      # ROUTE 7 - StripPrefix
      # Client: /api/users/123
      # Backend receives: /users/123
      # =================================================
      spring.cloud.gateway.routes[6].id=strip-prefix
      spring.cloud.gateway.routes[6].uri=http://localhost:8081
      spring.cloud.gateway.routes[6].predicates[0]=Path=/api/**
      spring.cloud.gateway.routes[6].filters[0]=StripPrefix=1
      
      
      # =================================================
      # ROUTE 8 - PrefixPath
      # Client: /users/123
      # Backend receives: /api/users/123
      # =================================================
      spring.cloud.gateway.routes[7].id=prefix-path
      spring.cloud.gateway.routes[7].uri=http://localhost:8081
      spring.cloud.gateway.routes[7].predicates[0]=Path=/users/**
      spring.cloud.gateway.routes[7].filters[0]=PrefixPath=/api
      
      
      # =================================================
      # ROUTE 9 - RewritePath
      # Client: /v1/users/123
      # Backend receives: /users/123
      # =================================================
      spring.cloud.gateway.routes[8].id=rewrite-path
      spring.cloud.gateway.routes[8].uri=http://localhost:8081
      spring.cloud.gateway.routes[8].predicates[0]=Path=/v1/**
      spring.cloud.gateway.routes[8].filters[0]=RewritePath=/v1/(?<segment>.*), /$\{segment}
      
      
      # =================================================
      # ROUTE 10 - RedirectTo
      # Client: /old-page
      # Gateway returns redirect to /new-page
      # =================================================
      spring.cloud.gateway.routes[9].id=redirect
      spring.cloud.gateway.routes[9].uri=http://localhost:8081
      spring.cloud.gateway.routes[9].predicates[0]=Path=/old-page
      spring.cloud.gateway.routes[9].filters[0]=RedirectTo=302, http://localhost:8081/new-page




--------------------------------------------------------------------------------------------------------------------------------------------------------------------

SERVICE DISCOVERY :

      Same as normal but include gateway also as a client of eureka server and register it with eureka server. 
      Then in the gateway application.properties instead of using uri use serviceId to route to the service.

      spring.cloud.gateway.routes[0].id=order-service
      spring.cloud.gateway.routes[0].uri=lb://ORDER-SERVICE
      spring.cloud.gateway.routes[0].predicates[0]=Path=/orders/**
      
      spring.cloud.gateway.routes[1].id=payment-service
      spring.cloud.gateway.routes[1].uri=lb://PAYMENT-SERVICE
      spring.cloud.gateway.routes[1].predicates[0]=Path=/payments/**