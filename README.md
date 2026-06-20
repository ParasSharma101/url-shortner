# URL Shortener

A REST API for shortening URLs, expanding short codes back to original URLs, and tracking access counts. The project is a **Spring Boot backend** backed by **PostgreSQL** and **Redis**.

> **Note:** The repository currently contains only the `backend/` module. There is no `frontend/` directory in this codebase.

## Features

- Shorten a long URL into a 6-character alphanumeric code prefixed with `http://devportal.com/`
- Return an existing short URL if the original URL was already shortened
- Expand a short URL back to its original URL
- Paginated listing of all URL mappings
- Hit count tracking via Redis buffering and periodic PostgreSQL aggregation
- Redis-backed URL lookup cache
- Per-IP rate limiting on the shorten endpoint (2 requests per 60 seconds)
- Centralized API response format with status, message, success flag, and timestamp
- Application and audit logging to file and console

## Tech Stack

| Layer | Technology |
|-------|------------|
| Runtime | Java 8 |
| Framework | Spring Boot 2.7.0 |
| Web | Spring Web (REST), WAR packaging |
| Persistence | Hibernate 5 (SessionFactory + Criteria API), PostgreSQL |
| Connection Pool | Apache Commons DBCP2 |
| Cache / Counters | Spring Data Redis |
| Cross-cutting | Spring AOP (rate limiting), Bean Validation |
| Logging | SLF4J + Logback (`logging.xml`) |
| Build | Maven |

## Project Structure

```
url-shortner/
├── backend/
│   ├── src/
│   │   ├── main/
│   │   │   ├── java/
│   │   │   │   ├── URLShortenerApplication.java      # App entry, beans (DataSource, SessionFactory, Redis)
│   │   │   │   ├── ServletInitializer.java           # WAR deployment support
│   │   │   │   └── com/devportal/
│   │   │   │       ├── annotation/                   # @RateLimit
│   │   │   │       ├── aspect/                       # RateLimitAspect
│   │   │   │       ├── async/                        # HitCountUpdater (Redis increment)
│   │   │   │       ├── bean/                         # URLMapping entity
│   │   │   │       ├── config/                       # LogDirectoryInitializer
│   │   │   │       ├── constants/                    # URLConstants
│   │   │   │       ├── controller/                   # URLController
│   │   │   │       ├── dao/                          # URLMappingRepository
│   │   │   │       ├── exceptions/                   # Custom exceptions + GlobalExceptionHandling
│   │   │   │       ├── scheduler/                    # HitCountAggregator (30s cron)
│   │   │   │       ├── service/                      # URLShortenerService
│   │   │   │       ├── to/
│   │   │   │       │   ├── request/                  # ShortenURLRequest
│   │   │   │       │   └── response/                 # ApiResponse, ShortenURLResponse
│   │   │   │       ├── util/                         # Util, AuditLogger
│   │   │   │       └── validation/                   # URLValidation
│   │   │   └── resources/
│   │   │       ├── application.properties
│   │   │       └── logging.xml
│   │   └── test/
│   │       └── java/com/devportal/
│   │           └── UrlShortenerApplicationTests.java
│   ├── logs/                                         # Runtime log output (application + audit)
│   ├── pom.xml
│   ├── mvnw / mvnw.cmd
│   └── .mvn/
└── README.md
```

## Architecture Overview

The application follows a layered REST architecture. HTTP requests enter through `URLController`, business logic lives in `URLShortenerService`, and data access is handled by `URLMappingRepository` using Hibernate's `SessionFactory`. Redis is used for three concerns: URL caching, hit-count buffering, and rate-limit tracking.

```mermaid
flowchart TB
    Client["HTTP Client"]

    subgraph SpringBoot["Spring Boot Application (port 8081)"]
        Controller["URLController"]
        Service["URLShortenerService"]
        Repo["URLMappingRepository"]
        Aspect["RateLimitAspect"]
        HitUpdater["HitCountUpdater"]
        Aggregator["HitCountAggregator"]
        ExceptionHandler["GlobalExceptionHandling"]
        Audit["AuditLogger"]
    end

    Redis[("Redis")]
    PG[("PostgreSQL<br/>projects.url_mapping")]

    Client --> Controller
    Aspect -.->|before /api/shorten| Controller
    Controller --> Service
    Controller --> Audit
    Service --> Repo
    Service --> HitUpdater
    Service --> Util["Util (Redis cache)"]
    HitUpdater --> Redis
    Aggregator --> Redis
    Aggregator --> Repo
    Repo --> PG
    Util --> Redis
    Aspect --> Redis
    Controller -.-> ExceptionHandler
    Service -.-> ExceptionHandler
```

### Layer Responsibilities

| Layer | Class(es) | Responsibility |
|-------|-----------|----------------|
| Controller | `URLController` | REST endpoints, request validation, audit logging |
| Service | `URLShortenerService` | Short-code generation, cache/DB lookup, hit recording |
| Repository | `URLMappingRepository` | Hibernate Criteria/SQL queries against `url_mapping` |
| Entity | `URLMapping` | JPA entity mapped to `projects.url_mapping` |
| Async | `HitCountUpdater` | Increments Redis counter on each expand |
| Scheduler | `HitCountAggregator` | Flushes Redis hit counts to PostgreSQL every 30 seconds |
| AOP | `RateLimitAspect` | Sliding-window rate limit via Redis sorted sets |
| Exception | `GlobalExceptionHandling` | Maps exceptions to `ApiResponse` / `ShortenURLResponse` |

## UML Diagrams

### Class Diagram

```mermaid
classDiagram
    class URLController {
        -URLShortenerService service
        -AuditLogger auditLogger
        -HttpServletRequest httpRequest
        +shortenUrl(ShortenURLRequest) ShortenURLResponse
        +expandUrl(ShortenURLRequest) ShortenURLResponse
        +getAllURLMappings(int offset, int limit) ShortenURLResponse
    }

    class URLShortenerService {
        -SessionFactory sessionFactroy
        -HitCountUpdater hitCountUpdater
        -URLMappingRepository repository
        -Util util
        +shortenUrl(String originalUrl) String
        +getOriginalUrl(String shortCode) String
        +getAllURLMappings(ShortenURLResponse, int, int) void
    }

    class URLMappingRepository {
        +findByShortCode(String, SessionFactory) URLMapping
        +findByOriginalUrl(String, SessionFactory) URLMapping
        +save(URLMapping, SessionFactory) void
        +isShortCodeExist(String, SessionFactory) boolean
        +getAllURLMappings(int, int, SessionFactory) List~URLMapping~
        +getAllURLMappingsCount(SessionFactory) Long
        +updateHitCount(String, SessionFactory) void
        +updateHitCount(String, Long, SessionFactory) void
        +findUrlValue(String, String, String, SessionFactory) String
    }

    class URLMapping {
        -String shortCode
        -String originalUrl
        -Timestamp createdAt
        -int hitCount
    }

    class HitCountUpdater {
        -SessionFactory sessionFactroy
        -URLMappingRepository repository
        -RedisTemplate redisTemplate
        +recordHitCount(String shortCode) void
        +updateHitCountAsync(String shortCode) void
    }

    class HitCountAggregator {
        -SessionFactory sessionFactory
        -URLMappingRepository repository
        -RedisTemplate redisTemplate
        +aggregateHitCount() void
    }

    class RateLimitAspect {
        -HttpServletRequest req
        -RedisTemplate redisTemplate
        +validateRateLimit(RateLimit) void
    }

    class GlobalExceptionHandling {
        +handleDataAlreadyExists(DataAlreadyExists) ShortenURLResponse
        +handleRateLimitException(RateLimitException) ApiResponse
        +handleDataNotFoundException(DataNotFoundException) ApiResponse
        +handleException(Exception) ApiResponse
    }

    class ShortenURLRequest {
        -String url
    }

    class ApiResponse {
        -HttpStatus status
        -int statusCode
        -String message
        -boolean success
        -Timestamp timestamp
    }

    class ShortenURLResponse {
        -String shortURL
        -String originalURL
        -List urlMappingList
        -Long count
    }

    URLController --> URLShortenerService
    URLController --> AuditLogger
    URLShortenerService --> URLMappingRepository
    URLShortenerService --> HitCountUpdater
    URLShortenerService --> Util
    URLMappingRepository --> URLMapping
    HitCountUpdater --> URLMappingRepository
    HitCountAggregator --> URLMappingRepository
    ShortenURLResponse --|> ApiResponse
    URLController ..> ShortenURLRequest
    URLController ..> ShortenURLResponse
```

### Sequence Diagram — Shorten URL

```mermaid
sequenceDiagram
    participant Client
    participant Aspect as RateLimitAspect
    participant Controller as URLController
    participant Service as URLShortenerService
    participant Repo as URLMappingRepository
    participant Redis
    participant DB as PostgreSQL

    Client->>Controller: POST /api/shorten { url }
    Aspect->>Redis: Check rate limit (ZSET per IP)
    alt Rate limit exceeded
        Aspect-->>Client: 429 Too Many Requests
    end
    Controller->>Service: shortenUrl(originalUrl)
    Service->>Repo: findUrlValue(shortCode, originalUrl)
    Repo->>DB: SELECT short_code WHERE original_url = ?
    alt URL already exists
        Repo-->>Service: existing shortCode
        Service-->>Controller: throw DataAlreadyExists
        Controller-->>Client: 200 OK with existing shortURL
    else New URL
        Service->>Service: generateUniqueShortCode() (6 chars)
        Service->>Repo: save(URLMapping)
        Repo->>DB: INSERT into url_mapping
        Service->>Redis: SET shortCode → originalUrl (TTL 300s)
        Service-->>Controller: shortCode
        Controller-->>Client: 201 Created { shortURL }
    end
```

### Sequence Diagram — Expand URL

```mermaid
sequenceDiagram
    participant Client
    participant Controller as URLController
    participant Service as URLShortenerService
    participant Util
    participant Redis
    participant Repo as URLMappingRepository
    participant HitUpdater as HitCountUpdater
    participant Aggregator as HitCountAggregator
    participant DB as PostgreSQL

    Client->>Controller: POST /api/expand { url }
    Controller->>Controller: Validate prefix & short code format
    Controller->>Service: getOriginalUrl(shortCode)
    Service->>Util: getDataFromRedis(shortCode)
    Util->>Redis: GET shortCode
    alt Cache miss
        Redis-->>Util: null
        Service->>Repo: findUrlValue(originalUrl, shortCode)
        Repo->>DB: SELECT original_url WHERE short_code = ?
        Service->>Redis: SET shortCode → originalUrl (TTL 120s)
    end
    Service->>HitUpdater: recordHitCount(shortCode)
    HitUpdater->>Redis: INCR hitCount:shortCode
    Service-->>Controller: originalUrl
    Controller-->>Client: 200 OK { originalURL }

    Note over Aggregator,DB: Every 30 seconds
    Aggregator->>Redis: KEYS hitCount:*
    Aggregator->>DB: UPDATE hit_count += count
    Aggregator->>Redis: DELETE hitCount keys
```

### Component Diagram — Data Stores

```mermaid
flowchart LR
    subgraph PostgreSQL
        Table["projects.url_mapping"]
        Cols["short_code (PK)<br/>original_url<br/>created_at<br/>hit_count"]
        Table --- Cols
    end

    subgraph RedisKeys["Redis Keys"]
        Cache["{shortCode} → originalUrl<br/>TTL: 300s on create, 120s on read"]
        Hits["hitCount:{shortCode} → counter"]
        Rate["{rateLimitKey}-{IP} → ZSET timestamps"]
    end

    App["Spring Boot App"] --> Table
    App --> Cache
    App --> Hits
    App --> Rate
```

## Database Schema

Hibernate manages the schema via `spring.jpa.hibernate.ddl-auto=update`. The entity maps to the `projects` schema:

| Column | Type | Notes |
|--------|------|-------|
| `short_code` | `VARCHAR` (PK) | 6-character alphanumeric code |
| `original_url` | `VARCHAR` | Full original URL |
| `created_at` | `TIMESTAMP` | Creation time |
| `hit_count` | `INT` | Default `0`; updated by `HitCountAggregator` |

## API Endpoints

Base URL: `http://localhost:8081`

| Method | Endpoint | Description | Notes |
|--------|----------|-------------|-------|
| `POST` | `/api/shorten` | Create a short URL | Rate limited: 2 req / 60s per IP |
| `POST` | `/api/expand` | Resolve short URL to original | Validates `http://devportal.com/` prefix |
| `GET` | `/api/getAllURLMappings` | List all mappings | Query params: `offset` (default `0`), `limit` (default `10`) |

### Request Body (`POST /api/shorten`, `POST /api/expand`)

```json
{
  "url": "https://example.com/some/long/path"
}
```

For `/api/expand`, `url` must be a full short URL (e.g. `http://devportal.com/hYUPPR`).

Validation rules on `url`:
- Must not be blank
- Must match `^(http|https)://[^\s$.?#].[^\s]*$`

### Response Structure

All responses extend `ApiResponse`:

```json
{
  "status": "CREATED",
  "statusCode": 201,
  "message": "Data created successfuly",
  "success": true,
  "timestamp": "2025-05-23T13:02:32.275+00:00",
  "shortURL": "http://devportal.com/hYUPPR"
}
```

`ShortenURLResponse` adds optional fields: `shortURL`, `originalURL`, `urlMappingList`, `count`.

### HTTP Status Codes

| Scenario | Status |
|----------|--------|
| Short URL created | `201 CREATED` |
| Short URL already exists | `200 OK` (returns existing `shortURL`) |
| Expand / list success | `200 OK` |
| Invalid short URL prefix or format | `400 BAD_REQUEST` |
| Short code not found | `204 NO_CONTENT` |
| Rate limit exceeded | `429 TOO_MANY_REQUESTS` |
| Unhandled error | `500 INTERNAL_SERVER_ERROR` |

## Configuration

Key settings in `backend/src/main/resources/application.properties`:

| Property | Value | Purpose |
|----------|-------|---------|
| `server.port` | `8081` | HTTP port |
| `spring.datasource.*` | PostgreSQL connection | Database |
| `spring.redis.host` / `port` | `localhost:6379` | Redis |
| `spring.cache.type` | `redis` | Cache backend |
| `spring.jpa.hibernate.ddl-auto` | `update` | Auto schema sync |
| `logging.config` | `classpath:logging.xml` | Logback config |

Update database credentials before running:

```properties
spring.datasource.url=jdbc:postgresql://localhost:5432/postgres
spring.datasource.username=devportal
spring.datasource.password=devportal
```

Ensure the `projects` schema exists in PostgreSQL, or let Hibernate create the table on first run.

## Prerequisites

- Java 8+
- Maven 3.x (or use the included `mvnw` wrapper)
- PostgreSQL (running on port 5432)
- Redis (running on port 6379)

## Running the Application

```bash
cd backend
mvn spring-boot:run
```

Or with the Maven wrapper:

```bash
cd backend
./mvnw spring-boot:run        # Linux/macOS
mvnw.cmd spring-boot:run      # Windows
```

The API will be available at `http://localhost:8081`.

### WAR Deployment

The project is packaged as a WAR (`<packaging>war</packaging>`) with `ServletInitializer`, so it can also be deployed to an external Tomcat container.

## Logging

Logs are written to `backend/logs/` (created at startup by `LogDirectoryInitializer`):

| Log | Path | Level |
|-----|------|-------|
| Application | `logs/application/application-{dd-MM-yyyy}.log` | DEBUG (root) |
| Audit | `logs/audit/audit-{dd-MM-yyyy}.log` | TRACE (`AUDIT_LOGGER`) |
| Console | stdout | Same as above |

Example application log:

```
2025-05-23 18:41:49.657 ERROR [nio-8081-exec-3] com.devportal.util.Util - Invalid short URL prefix for URL: https://start.spring.io/
```

Audit entries record client IP and action (e.g. shorten, expand, list).

## Short Code Generation

- Length: 6 characters
- Charset: `a-z`, `A-Z`, `0-9` (62 characters)
- Prefix: `http://devportal.com/` (defined in `URLConstants.PREFIX`)
- Uniqueness: checked against PostgreSQL before persisting

## Hit Count Flow

1. On expand, `HitCountUpdater.recordHitCount()` increments `hitCount:{shortCode}` in Redis.
2. `HitCountAggregator` runs every 30 seconds (`@Scheduled(fixedRate = 30000)`).
3. It reads all `hitCount:*` keys, batch-updates PostgreSQL, then deletes the Redis keys.

`HitCountUpdater.updateHitCountAsync()` exists for direct DB updates but is not invoked by the current expand flow.

## Known TODOs

From project notes and code observations:

- Add user authentication
- Dockerize the application
- Enable integration tests (`@SpringBootTest` is commented out in `UrlShortenerApplicationTests`)

## Contributing

Pull requests are welcome. For major changes, please open an issue first to discuss what you would like to change.
