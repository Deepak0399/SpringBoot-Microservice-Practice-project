# SpringBoot-Micro (Practice Microservices Project)

A multi-module Spring Boot microservices practice project showcasing service discovery, inter-service communication (OpenFeign), data persistence with MySQL, layered architecture, and consistent API contracts.

This repository contains four microservices:
- serviceRegistry — Eureka Server (Service Discovery)
- employeeService — Employee CRUD service
- accountService — Account service that validates employee existence via OpenFeign call to employeeService
- plotService — Plot CRUD service (standalone example service)


## Architecture Overview

```
+------------------+         registers         +--------------------+
| employeeService  |-------------------------->|  serviceRegistry   |
|  (8081)          |<--------------------------|  (Eureka Server)   |
+------------------+                           +--------------------+
         ^                                                   ^
         |                                                   |
         |  OpenFeign (by service name: EMPLOYEE-SERVICE)    |
         |                                                   |
+------------------+         registers         +--------------------+
| accountService   |-------------------------->|  serviceRegistry   |
|  (8082)          |                           +--------------------+
|  validates       |
|  employee by     |
|  calling employee|
+------------------+

+------------------+         registers         +--------------------+
| plotService      |-------------------------->|  serviceRegistry   |
|  (8083)          |                           +--------------------+
+------------------+
```

Key flows
- Service discovery: All business services register with Eureka at serviceRegistry:8080.
- Inter-service call: accountService uses OpenFeign client `EmployeeClient` to call employeeService by service-id `EMPLOYEE-SERVICE` when creating an account.
- Databases: Each service uses its own MySQL schema: `employee`, `account`, `plot` (one DB schema per service).


## Modules and Responsibilities

1) serviceRegistry (Eureka Server)
- Port: 8080
- Purpose: Central registry for service discovery.
- No DB required.

2) employeeService
- Port: 8081
- Responsibilities: Manage employees (create, list, get, delete).
- DB: MySQL schema `employee`.
- Primary endpoints (base path: `/api/employees`):
  - POST `/` — create employee
  - GET `/` — list employees
  - GET `/{id}` — get employee by id
  - DELETE `/{id}` — delete employee by id

3) accountService
- Port: 8082
- Responsibilities: Manage accounts and validate `employeeId` via employeeService.
- DB: MySQL schema `account`.
- Inter-service: Uses OpenFeign `EmployeeClient` with `name = "EMPLOYEE-SERVICE"` and calls `GET /api/employees/{id}`.
- Primary endpoints (base path: `/api/accounts`):
  - POST `/` — create account (validates employee existence)
  - GET `/` — list accounts
  - GET `/{id}` — get account by id

4) plotService
- Port: 8083
- Responsibilities: Manage plots (create, list, get). Standalone example.
- DB: MySQL schema `plot`.
- Primary endpoints (base path: `/api/plots`):
  - POST `/` — create plot
  - GET `/` — list plots
  - GET `/{id}` — get plot by id


## Tech Stack and Dependencies

Common
- Java: 17
- Spring Boot: 3.5.12 (via `spring-boot-starter-parent`)
- Spring Cloud Release Train: 2025.0.2 (via `spring-cloud-dependencies`)
- Build: Maven
- Core Starters used:
  - spring-boot-starter-web (REST controllers)
  - spring-boot-starter-data-jpa (JPA + Hibernate)
  - spring-boot-starter-validation (Jakarta Bean Validation)
  - spring-boot-starter-test (tests)
- Cloud/Discovery:
  - spring-cloud-starter-netflix-eureka-server (Eureka server) — in serviceRegistry module
  - spring-cloud-starter-netflix-eureka-client — in business services
  - spring-cloud-starter-openfeign — used by accountService (and included in plotService POM, ready for future calls)
- Utilities:
  - modelmapper (object mapping DTO <-> Entity)
  - lombok (boilerplate reduction; requires annotation processing)
  - mysql-connector-j (runtime MySQL driver)
  - spring-boot-devtools (optional, runtime only)

Exact per-module dependency highlights (see each `pom.xml` for details):
- accountService: Web, Validation, Data JPA, OpenFeign, Eureka Client, ModelMapper, Lombok, MySQL, DevTools, Tests
- employeeService: Web, Validation, Data JPA, Eureka Client, ModelMapper, Lombok, MySQL, DevTools, Tests
- plotService: Web, Validation, Data JPA, Eureka Client, OpenFeign, ModelMapper, Lombok, MySQL, DevTools, Tests
- serviceRegistry: Eureka Server (Spring Cloud Netflix) + Spring Boot basics


## Configuration

Eureka (Service Discovery)
- Server (serviceRegistry/src/main/resources/application.properties):
  - `server.port=8080`
  - `eureka.client.fetch-registry=false`
  - `eureka.client.register-with-eureka=false`
- Clients (employeeService, accountService, plotService `application.properties`):
  - `eureka.client.service-url.defaultZone=http://localhost:8080/eureka`
  - `eureka.client.register-with-eureka=true`
  - `eureka.client.fetch-registry=true`
  - Optional: `eureka.instance.prefer-ip-address=true`

Service ports and names
- employeeService: `spring.application.name=employee-service`, `server.port=8081`
- accountService: `spring.application.name=account-service`, `server.port=8082`
- plotService: `spring.application.name=plot-service`, `server.port=8083`
- Eureka service-id used by OpenFeign in accountService: `EMPLOYEE-SERVICE` (case-insensitive mapping to `employee-service` in discovery)

Databases (MySQL)
- employeeService DB: `jdbc:mysql://localhost:3306/employee`
- accountService DB: `jdbc:mysql://localhost:3306/account`
- plotService DB: `jdbc:mysql://localhost:3306/plot`
- Shared settings in services:
  - `spring.datasource.driver-class-name=com.mysql.cj.jdbc.Driver`
  - `spring.jpa.hibernate.ddl-auto=update`
  - `spring.jpa.show-sql=true`

Important: Replace credentials in each service's `application.properties` with your own
- `spring.datasource.username=<YOUR_DB_USERNAME>`
- `spring.datasource.password=<YOUR_DB_PASSWORD>`


## How to Run (Local)

Prerequisites
- JDK 17 installed and `JAVA_HOME` set
- Maven 3.9+
- MySQL running locally
- Ports 8080, 8081, 8082, 8083 available

1) Create databases in MySQL
- Create empty schemas (no tables required; Hibernate will create them):
  - `CREATE DATABASE employee;`
  - `CREATE DATABASE account;`
  - `CREATE DATABASE plot;`

2) Configure credentials
- In each module's `src/main/resources/application.properties`, set your MySQL username/password.

3) Start Eureka Server first
- From `serviceRegistry` module:
  - `mvn spring-boot:run`
  - Visit `http://localhost:8080` to see Eureka dashboard.

4) Start business services (order does not matter after Eureka is up)
- In separate terminals (or your IDE), from each module root run:
  - employeeService: `mvn spring-boot:run`
  - accountService: `mvn spring-boot:run`
  - plotService: `mvn spring-boot:run`
- After startup, confirm all register in Eureka at `http://localhost:8080`.


## API Usage (Sample)

employeeService (8081)
- Create employee
```
curl -X POST http://localhost:8081/api/employees \
  -H "Content-Type: application/json" \
  -d '{
        "name": "Alice",
        "email": "alice@example.com",
        "phone": "9999999999"
      }'
```
- Get by id
```
curl http://localhost:8081/api/employees/{employeeId}
```

accountService (8082)
- Create account (validates employeeId by calling employeeService via Feign)
```
curl -X POST http://localhost:8082/api/accounts \
  -H "Content-Type: application/json" \
  -d '{
        "accNo": "AC-1001",
        "balance": 1200.0,
        "employeeId": "<existing-employee-id>"
      }'
```
- Get by id
```
curl http://localhost:8082/api/accounts/{accountId}
```

plotService (8083)
- Create plot
```
curl -X POST http://localhost:8083/api/plots \
  -H "Content-Type: application/json" \
  -d '{
        "name": "Plot-A",
        "location": "City Center"
      }'
```
- Get by id
```
curl http://localhost:8083/api/plots/{plotId}
```


## Inter-service Call Details (accountService -> employeeService)

- Feign client: `com.practice.accountservice.external.client.EmployeeClient`
- Declaration:
```
@FeignClient(name = "EMPLOYEE-SERVICE")
public interface EmployeeClient {
    @GetMapping("/api/employees/{id}")
    ResponseEntity<ApiResponse<EmployeeResponse>> getEmployeeById(@PathVariable String id);
}
```
- Usage: In `AccountServiceImpl.saveAccount(...)`, the service logs and invokes `employeeClient.getEmployeeById(employeeId)` before persisting the account.
- Discovery: Feign resolves `EMPLOYEE-SERVICE` via Eureka; no hard-coded host/port needed.


## Project Conventions

- DTOs for request payloads (e.g., `EmployeeDTO`, `AccountDTO`, `PlotDTO`).
- Entities per service with independent schemas.
- `ApiResponse<T>` wrapper provides consistent `status`, `message`, and `data` across services.
- Centralized exception handling via `GlobalExceptionHandler` classes in each service.
- ModelMapper for DTO <-> Entity conversion.


## Building & Testing

- Build a module:
  - `mvn -q -DskipTests package`
- Run tests for a module:
  - `mvn test`
- You can also import each module into IntelliJ IDEA/VS Code as individual Spring Boot projects.


## Environment & Versions

- Java 17
- Spring Boot 3.5.12
- Spring Cloud 2025.0.2
- MySQL 8.x (Connector/J provided at runtime)


## Troubleshooting

- Service not appearing in Eureka dashboard:
  - Ensure Eureka server (8080) started first.
  - Verify client properties: `eureka.client.service-url.defaultZone=http://localhost:8080/eureka`.
  - Check that ports 8081/8082/8083 are free and services have started without errors.

- Database connection errors:
  - Confirm the DB name exists and credentials are correct in `application.properties`.
  - Ensure MySQL is running and accessible at `localhost:3306`.

- Feign call failures when creating an account:
  - Verify employeeService is up and registered as `employee-service` in Eureka.
  - Use Eureka dashboard to confirm instance health.
  - Check logs in accountService for the call to `getEmployeeById`.

- Lombok annotations not recognized in IDE:
  - Enable annotation processing in your IDE settings.

  