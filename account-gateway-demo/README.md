# Account API Gateway + Backend Service (Java 17)

This ZIP contains **two independent Maven/Spring Boot applications**:

| Application | Port | Purpose |
|---|---:|---|
| `account-api-gateway` | 8080 | Receives the public API path and forwards it |
| `account-backend-service` | 8081 initially | Implements `GET /accounts` |

The projects use Java 17, Spring Boot 3.3.5 and Spring Cloud 2023.0.3. The Spring Cloud release train is compatible with Spring Boot 3.3.x. No database, Docker, or external infrastructure is required.

## 1. Requirements

- JDK 17 installed
- Spring Tool Suite (STS) with Maven support
- Internet access on the first Maven build so dependencies can be downloaded
- Postman or a browser/curl for testing

Check Java from a terminal:

```bash
java -version
```

It should report Java 17. In STS, set the installed JRE under **Window > Preferences > Java > Installed JREs** and make Java 17 the default.

## 2. Import both Maven projects into STS

1. Extract this ZIP to a convenient folder.
2. Open STS.
3. Select **File > Import... > Maven > Existing Maven Projects**.
4. For **Root Directory**, browse to the extracted `account-gateway-demo` folder.
5. STS should detect both `account-api-gateway` and `account-backend-service`. Select both and click **Finish**.
6. If dependencies do not download automatically, right-click each project and select **Maven > Update Project...**. Select both projects, enable **Force Update of Snapshots/Releases** if needed, and click **OK**.

You can also import each project separately by selecting its own folder (the folder containing `pom.xml`).

## 3. Download dependencies / build

In STS, right-click each project > **Run As > Maven build...** and enter:

```text
clean package
```

Click **Run**. The first build downloads dependencies from Maven repositories. A successful build ends with `BUILD SUCCESS`.

Alternatively, from a terminal opened in each project directory:

```bash
mvn clean package
```

If the `mvn` command is not found, use STS's embedded Maven or install Apache Maven and add it to PATH.

## 4. Run the backend first

1. In STS, expand `account-backend-service`.
2. Open `src/main/java/com/example/account/AccountBackendServiceApplication.java`.
3. Right-click the file > **Run As > Spring Boot App** (or **Java Application**).
4. Wait until the console shows that Tomcat started on port 8081.

Test the backend directly:

```http
GET http://localhost:8081/accounts
```

Expected HTTP status: `200 OK`

Expected response body:

```text
SUCCESS: Accounts API reached backend service
```

This direct request verifies the backend is listening independently of the Gateway.

## 5. Run the API Gateway

1. Open `AccountApiGatewayApplication.java` in the `account-api-gateway` project.
2. Right-click > **Run As > Spring Boot App**.
3. Wait until the application starts on port 8080.

Test through the Gateway:

```http
GET http://localhost:8080/api/v1/internals/accounts
```

Expected HTTP status: `200 OK`

Expected response body:

```text
SUCCESS: Accounts API reached backend service
```

The Gateway route is configured in `account-api-gateway/src/main/resources/application.yml`:

```yaml
spring:
  cloud:
    gateway:
      routes:
        - id: account-backend-route
          uri: http://localhost:8081
          predicates:
            - Path=/api/v1/internals/**
          filters:
            - StripPrefix=3
```

`StripPrefix=3` removes the first three path segments (`api`, `v1`, and `internals`). Therefore:

```text
Incoming:  /api/v1/internals/accounts
Forwarded: /accounts
```

The Gateway forwards the request to `http://localhost:8081/accounts`.

## 6. Verify isolated ports

The two applications are separate processes and have separate server ports:

- Gateway: `http://localhost:8080`
- Backend: `http://localhost:8081`

Keep both applications running. If either port is already occupied, stop the other process using that port or change the configured port.

## 7. Change the backend from port 8081 to 8082

**Step A — change the backend port**

Open:

`account-backend-service/src/main/resources/application.yml`

Change:

```yaml
server:
  port: 8081
```

to:

```yaml
server:
  port: 8082
```

Save the file and restart the backend application. Confirm in the STS console that Tomcat started on port 8082.

**Step B — update the Gateway destination**

Open:

`account-api-gateway/src/main/resources/application.yml`

Change:

```yaml
uri: http://localhost:8081
```

to:

```yaml
uri: http://localhost:8082
```

Save and restart the Gateway application (or allow DevTools if you have separately added it; this project does not require DevTools).

**Step C — test again**

Direct backend:

```http
GET http://localhost:8082/accounts
```

Gateway:

```http
GET http://localhost:8080/api/v1/internals/accounts
```

Both should return:

```text
SUCCESS: Accounts API reached backend service
```

The Gateway still listens on 8080. Only the backend port and Gateway destination have changed.

## 8. Postman test summary

Create two requests using method **GET**:

| Test | URL | Expected |
|---|---|---|
| Direct backend (initial) | `http://localhost:8081/accounts` | 200 + success message |
| Through Gateway (initial) | `http://localhost:8080/api/v1/internals/accounts` | 200 + success message |
| Direct backend (after port change) | `http://localhost:8082/accounts` | 200 + success message |
| Through Gateway (after port change) | `http://localhost:8080/api/v1/internals/accounts` | 200 + success message |

No request body or special headers are required.

## 9. Troubleshooting

### Gateway returns 503 / connection refused
- Confirm the backend is running.
- Confirm its actual port in the backend STS console.
- Ensure the Gateway `uri` in `application.yml` uses that exact port.
- Restart the Gateway after changing its destination URL.

### Gateway returns 404
- Check the request path is exactly `/api/v1/internals/accounts`.
- Check the route predicate is `Path=/api/v1/internals/**`.
- Check the filter is `StripPrefix=3`.
- Check the backend mapping is `@GetMapping("/accounts")`.

### Port already in use
- Stop the application already using the port, or choose another free port.
- Ensure the Gateway and backend do not use the same port.

### Maven dependencies fail to download
- Check internet access and Maven repository availability.
- In STS, use **Maven > Update Project...** and select **Force Update**.
- Confirm STS is configured to use JDK 17.

### Java version errors
- Configure JDK 17 in STS and ensure the Maven build uses it.
- Run `java -version` and verify it reports version 17.

## 10. Project layout

```text
account-gateway-demo/
├── README.md
├── account-api-gateway/
│   ├── pom.xml
│   └── src/
│       ├── main/java/com/example/gateway/AccountApiGatewayApplication.java
│       ├── main/resources/application.yml
│       ├── main/resources/application.properties
│       └── test/java/com/example/gateway/AccountApiGatewayApplicationTests.java
└── account-backend-service/
    ├── pom.xml
    └── src/
        ├── main/java/com/example/account/AccountBackendServiceApplication.java
        ├── main/java/com/example/account/AccountController.java
        ├── main/resources/application.yml
        ├── main/resources/application.properties
        └── test/java/com/example/account/AccountControllerTests.java
```

## Notes

- These are two separate Maven projects, not a multi-module parent build.
- `application.yml` is the active configuration file. The requested `application.properties` files are included as informational placeholders; avoid duplicating the same settings in both files.
- The Gateway uses the reactive Spring Cloud Gateway starter. Do not add `spring-boot-starter-web` to the Gateway project.
