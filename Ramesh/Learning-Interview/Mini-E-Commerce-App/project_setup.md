# Mini E-Commerce App — Project Structure & Environment Setup Guide

A step-by-step guide to scaffolding a microservices-based mini e-commerce
application with a **React** frontend, a **Spring Cloud API Gateway**, and
three backend microservices — **Product** (PostgreSQL), **Order** (MySQL),
and **Inventory** (MongoDB) — with **Eureka** service discovery.

---

## 1. Architecture Overview

```
                              ┌─────────────────────┐
                              │   React Frontend     │
                              │   (localhost:3000)   │
                              └──────────┬───────────┘
                                         │ HTTP (REST/JSON)
                                         ▼
                              ┌─────────────────────┐
                              │  Spring Cloud API    │
                              │  Gateway (:8080)     │
                              └──────────┬───────────┘
                                         │
                       ┌─────────────────┼─────────────────┐
                       ▼                 ▼                 ▼
              ┌────────────────┐ ┌────────────────┐ ┌────────────────┐
              │ Product Service│ │  Order Service │ │Inventory Service│
              │    (:8081)     │ │    (:8082)     │ │    (:8083)      │
              └───────┬────────┘ └───────┬────────┘ └───────┬────────┘
                      ▼                  ▼                  ▼
              ┌────────────────┐ ┌────────────────┐ ┌────────────────┐
              │  PostgreSQL    │ │     MySQL      │ │    MongoDB      │
              │    (:5432)     │ │    (:3306)     │ │    (:27017)     │
              └────────────────┘ └────────────────┘ └────────────────┘

                        ┌───────────────────────┐
                        │ Eureka Discovery Server│
                        │        (:8761)         │
                        └───────────────────────┘
```

**Why Eureka?** The gateway and services need to find each other without
hardcoded hostnames/ports. Eureka gives each service a name (e.g.
`product-service`) that the gateway routes to dynamically — this also makes
the setup portable to Docker/Kubernetes where IPs change on every restart.

---

## 2. Prerequisites

Install the following before starting:

| Tool | Version | Purpose |
|---|---|---|
| JDK | 17+ | Spring Boot services |
| Maven | 3.9+ | Build tool for Java services |
| Node.js | 18+ | React frontend |
| npm or yarn | latest | Frontend package management |
| Docker & Docker Compose | latest | Running PostgreSQL, MySQL, MongoDB |
| Git | latest | Version control |
| IDE | IntelliJ IDEA / VS Code | Development |

Verify installations:

```bash
java -version
mvn -version
node -v
npm -v
docker -v
docker compose version
```

---

## 3. Project Folder Structure

Create the root project folder and the layout below:

```
Mini-E-Commerce-App/
├── docker-compose.yml
├── project_setup.md
├── .gitignore
├── discovery-server/                 # Eureka server
│   ├── pom.xml
│   └── src/main/java/.../DiscoveryServerApplication.java
│   └── src/main/resources/application.yml
│
├── api-gateway/                      # Spring Cloud Gateway
│   ├── pom.xml
│   └── src/main/java/.../ApiGatewayApplication.java
│   └── src/main/resources/application.yml
│
├── product-service/                  # PostgreSQL
│   ├── pom.xml
│   └── src/main/java/com/ecommerce/product/
│   │   ├── ProductServiceApplication.java
│   │   ├── controller/ProductController.java
│   │   ├── entity/Product.java
│   │   ├── repository/ProductRepository.java
│   │   └── service/ProductService.java
│   └── src/main/resources/application.yml
│
├── order-service/                    # MySQL
│   ├── pom.xml
│   └── src/main/java/com/ecommerce/order/
│   │   ├── OrderServiceApplication.java
│   │   ├── controller/OrderController.java
│   │   ├── entity/Order.java
│   │   ├── repository/OrderRepository.java
│   │   └── service/OrderService.java
│   └── src/main/resources/application.yml
│
├── inventory-service/                # MongoDB
│   ├── pom.xml
│   └── src/main/java/com/ecommerce/inventory/
│   │   ├── InventoryServiceApplication.java
│   │   ├── controller/InventoryController.java
│   │   ├── document/InventoryItem.java
│   │   ├── repository/InventoryRepository.java
│   │   └── service/InventoryService.java
│   └── src/main/resources/application.yml
│
└── frontend/                         # React app
    ├── package.json
    ├── .env
    ├── public/
    └── src/
        ├── api/
        │   ├── axiosClient.js
        │   ├── productApi.js
        │   ├── orderApi.js
        │   └── inventoryApi.js
        ├── components/
        ├── pages/
        └── App.js
```

Create it quickly with:

```bash
mkdir Mini-E-Commerce-App && cd Mini-E-Commerce-App
mkdir discovery-server api-gateway product-service order-service inventory-service frontend
```

---

## 4. Discovery Server (Eureka)

Generate via [start.spring.io](https://start.spring.io) (Dependencies:
`Eureka Server`) or manually add to `discovery-server/pom.xml`:

```xml
<parent>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-parent</artifactId>
    <version>3.2.5</version>
</parent>

<properties>
    <java.version>17</java.version>
    <spring-cloud.version>2023.0.1</spring-cloud.version>
</properties>

<dependencies>
    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-starter-netflix-eureka-server</artifactId>
    </dependency>
</dependencies>

<dependencyManagement>
    <dependencies>
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-dependencies</artifactId>
            <version>${spring-cloud.version}</version>
            <type>pom</type>
            <scope>import</scope>
        </dependency>
    </dependencies>
</dependencyManagement>
```

`DiscoveryServerApplication.java`:

```java
package com.ecommerce.discovery;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.cloud.netflix.eureka.server.EnableEurekaServer;

@EnableEurekaServer
@SpringBootApplication
public class DiscoveryServerApplication {
    public static void main(String[] args) {
        SpringApplication.run(DiscoveryServerApplication.class, args);
    }
}
```

`application.yml`:

```yaml
server:
  port: 8761

eureka:
  client:
    register-with-eureka: false
    fetch-registry: false
  server:
    enable-self-preservation: false
```

Run: `mvn spring-boot:run` → dashboard at `http://localhost:8761`.

---

## 5. Database Configuration (Docker Compose)

Instead of installing PostgreSQL, MySQL, and MongoDB locally, run them all
via Docker Compose at the project root (`docker-compose.yml`):

```yaml
version: "3.8"

services:
  postgres-db:
    image: postgres:16
    container_name: product-postgres
    restart: unless-stopped
    environment:
      POSTGRES_DB: productdb
      POSTGRES_USER: product_user
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD}
    ports:
      - "5432:5432"
    volumes:
      - postgres_data:/var/lib/postgresql/data

  mysql-db:
    image: mysql:8.4
    container_name: order-mysql
    restart: unless-stopped
    environment:
      MYSQL_DATABASE: orderdb
      MYSQL_USER: order_user
      MYSQL_PASSWORD: ${MYSQL_PASSWORD}
      MYSQL_ROOT_PASSWORD: ${MYSQL_ROOT_PASSWORD}
    ports:
      - "3306:3306"
    volumes:
      - mysql_data:/var/lib/mysql

  mongo-db:
    image: mongo:7
    container_name: inventory-mongo
    restart: unless-stopped
    environment:
      MONGO_INITDB_ROOT_USERNAME: ${MONGO_ROOT_USER}
      MONGO_INITDB_ROOT_PASSWORD: ${MONGO_ROOT_PASSWORD}
      MONGO_INITDB_DATABASE: inventorydb
    ports:
      - "27017:27017"
    volumes:
      - mongo_data:/data/db

volumes:
  postgres_data:
  mysql_data:
  mongo_data:
```

Create a `.env` file **next to `docker-compose.yml`** (and add it to
`.gitignore` — never commit real credentials):

```
POSTGRES_PASSWORD=change_me
MYSQL_PASSWORD=change_me
MYSQL_ROOT_PASSWORD=change_me
MONGO_ROOT_USER=admin
MONGO_ROOT_PASSWORD=change_me
```

Start all three databases:

```bash
docker compose up -d
docker compose ps
```

---

## 6. Product Service (Spring Boot + PostgreSQL)

`product-service/pom.xml` key dependencies:

```xml
<dependencies>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-web</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-data-jpa</artifactId>
    </dependency>
    <dependency>
        <groupId>org.postgresql</groupId>
        <artifactId>postgresql</artifactId>
        <scope>runtime</scope>
    </dependency>
    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-starter-netflix-eureka-client</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-validation</artifactId>
    </dependency>
</dependencies>
```

`application.yml`:

```yaml
server:
  port: 8081

spring:
  application:
    name: product-service
  datasource:
    url: jdbc:postgresql://localhost:5432/productdb
    username: product_user
    password: ${DB_PASSWORD}
  jpa:
    hibernate:
      ddl-auto: update
    show-sql: true
    properties:
      hibernate:
        dialect: org.hibernate.dialect.PostgreSQLDialect

eureka:
  client:
    service-url:
      defaultZone: http://localhost:8761/eureka/
```

`entity/Product.java` (schema design):

```java
package com.ecommerce.product.entity;

import jakarta.persistence.*;
import java.math.BigDecimal;

@Entity
@Table(name = "products")
public class Product {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(nullable = false)
    private String name;

    private String description;

    @Column(nullable = false)
    private BigDecimal price;

    private String category;

    // getters and setters
}
```

`controller/ProductController.java`:

```java
package com.ecommerce.product.controller;

import com.ecommerce.product.entity.Product;
import com.ecommerce.product.service.ProductService;
import org.springframework.web.bind.annotation.*;

import java.util.List;

@RestController
@RequestMapping("/api/products")
public class ProductController {

    private final ProductService productService;

    public ProductController(ProductService productService) {
        this.productService = productService;
    }

    @GetMapping
    public List<Product> getAllProducts() {
        return productService.getAllProducts();
    }

    @GetMapping("/{id}")
    public Product getProduct(@PathVariable Long id) {
        return productService.getProductById(id);
    }

    @PostMapping
    public Product createProduct(@RequestBody Product product) {
        return productService.createProduct(product);
    }
}
```

Run: `mvn spring-boot:run` (from `product-service/`).

---

## 7. Order Service (Spring Boot + MySQL)

`order-service/pom.xml` key dependencies (same as Product, swap driver):

```xml
<dependency>
    <groupId>com.mysql</groupId>
    <artifactId>mysql-connector-j</artifactId>
    <scope>runtime</scope>
</dependency>
```

`application.yml`:

```yaml
server:
  port: 8082

spring:
  application:
    name: order-service
  datasource:
    url: jdbc:mysql://localhost:3306/orderdb?useSSL=false&serverTimezone=UTC
    username: order_user
    password: ${DB_PASSWORD}
    driver-class-name: com.mysql.cj.jdbc.Driver
  jpa:
    hibernate:
      ddl-auto: update
    properties:
      hibernate:
        dialect: org.hibernate.dialect.MySQLDialect

eureka:
  client:
    service-url:
      defaultZone: http://localhost:8761/eureka/
```

`entity/Order.java`:

```java
package com.ecommerce.order.entity;

import jakarta.persistence.*;
import java.math.BigDecimal;
import java.time.LocalDateTime;

@Entity
@Table(name = "orders")
public class Order {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(nullable = false)
    private Long productId;

    @Column(nullable = false)
    private Integer quantity;

    private BigDecimal totalPrice;

    @Enumerated(EnumType.STRING)
    private OrderStatus status;

    private LocalDateTime createdAt = LocalDateTime.now();

    // getters and setters
}
```

`OrderService` calls the Product and Inventory services via the gateway
(or a `WebClient`/`RestTemplate` with `@LoadBalanced` and the Eureka
service name, e.g. `http://product-service/api/products/{id}`), so orders
can validate price and stock before persisting.

---

## 8. Inventory Service (Spring Boot + MongoDB)

`inventory-service/pom.xml` key dependency:

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-mongodb</artifactId>
</dependency>
```

`application.yml`:

```yaml
server:
  port: 8083

spring:
  application:
    name: inventory-service
  data:
    mongodb:
      uri: mongodb://admin:${MONGO_PASSWORD}@localhost:27017/inventorydb?authSource=admin

eureka:
  client:
    service-url:
      defaultZone: http://localhost:8761/eureka/
```

`document/InventoryItem.java` (schema design — MongoDB is schema-less, but
model the document shape explicitly):

```java
package com.ecommerce.inventory.document;

import org.springframework.data.annotation.Id;
import org.springframework.data.mongodb.core.mapping.Document;

@Document(collection = "inventory")
public class InventoryItem {

    @Id
    private String id;

    private Long productId;
    private Integer availableQuantity;
    private String warehouseLocation;

    // getters and setters
}
```

`repository/InventoryRepository.java`:

```java
package com.ecommerce.inventory.repository;

import com.ecommerce.inventory.document.InventoryItem;
import org.springframework.data.mongodb.repository.MongoRepository;

import java.util.Optional;

public interface InventoryRepository extends MongoRepository<InventoryItem, String> {
    Optional<InventoryItem> findByProductId(Long productId);
}
```

---

## 9. API Gateway Configuration & Service Discovery

`api-gateway/pom.xml` key dependencies:

```xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-gateway</artifactId>
</dependency>
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-netflix-eureka-client</artifactId>
</dependency>
```

`application.yml` — routes resolved dynamically via Eureka
(`lb://` = load-balanced by service name, no hardcoded ports):

```yaml
server:
  port: 8080

spring:
  application:
    name: api-gateway
  cloud:
    gateway:
      discovery:
        locator:
          enabled: true
      routes:
        - id: product-service
          uri: lb://product-service
          predicates:
            - Path=/api/products/**
        - id: order-service
          uri: lb://order-service
          predicates:
            - Path=/api/orders/**
        - id: inventory-service
          uri: lb://inventory-service
          predicates:
            - Path=/api/inventory/**

eureka:
  client:
    service-url:
      defaultZone: http://localhost:8761/eureka/
```

With this in place, the frontend only ever talks to
`http://localhost:8080/api/...` — the gateway hides the internal
topology entirely.

**Startup order:** `discovery-server` → `api-gateway` + the three
services (any order) → `frontend`.

---

## 10. React Frontend Setup & API Integration

```bash
cd frontend
npx create-react-app . --template minimal
# or: npm create vite@latest . -- --template react
npm install axios react-router-dom
```

`.env` (Create React App requires the `REACT_APP_` prefix):

```
REACT_APP_API_BASE_URL=http://localhost:8080/api
```

`src/api/axiosClient.js`:

```javascript
import axios from "axios";

const axiosClient = axios.create({
  baseURL: process.env.REACT_APP_API_BASE_URL,
  headers: { "Content-Type": "application/json" },
  timeout: 8000,
});

axiosClient.interceptors.request.use((config) => {
  const token = localStorage.getItem("token");
  if (token) config.headers.Authorization = `Bearer ${token}`;
  return config;
});

export default axiosClient;
```

`src/api/productApi.js`:

```javascript
import axiosClient from "./axiosClient";

export const getProducts = () => axiosClient.get("/products");
export const getProduct = (id) => axiosClient.get(`/products/${id}`);
export const createProduct = (data) => axiosClient.post("/products", data);
```

`src/api/orderApi.js` and `src/api/inventoryApi.js` follow the same
pattern against `/orders` and `/inventory`.

Run the frontend: `npm start` → `http://localhost:3000`.

---

## 11. Security Considerations & Best Practices

1. **Centralize auth at the gateway.** Validate JWTs once in a Spring
   Cloud Gateway `GlobalFilter`, so individual services don't each
   reimplement authentication. Downstream services can trust a header
   like `X-User-Id` set by the gateway after validation.
2. **Never commit secrets.** Keep DB passwords, JWT signing keys, and
   Mongo credentials in `.env` / environment variables, not in
   `application.yml` directly. Add `.env` to `.gitignore`.
3. **Use HTTPS in front of the gateway** in any non-local environment
   (terminate TLS at a reverse proxy/load balancer if not in Spring
   directly).
4. **Restrict CORS at the gateway**, not in every service:
   ```yaml
   spring:
     cloud:
       gateway:
         globalcors:
           cors-configurations:
             '[/**]':
               allowedOrigins: "http://localhost:3000"
               allowedMethods: [GET, POST, PUT, DELETE]
   ```
5. **Least-privilege DB users.** Each service's DB user should only
   have access to its own database/schema (`product_user` cannot touch
   `orderdb`, etc.).
6. **Validate input at service boundaries** (`@Valid` + Bean Validation
   annotations on request DTOs) — don't trust the gateway alone.
7. **Don't expose internal service ports externally.** In Docker/K8s,
   only the gateway and frontend ports should be published; service and
   DB ports stay on the internal network.
8. **Rate limiting.** Spring Cloud Gateway supports a
   `RequestRateLimiter` filter (Redis-backed) to throttle abusive
   clients at a single choke point.
9. **Rotate and scope secrets** separately per environment
   (dev/staging/prod) — never reuse production credentials locally.
10. **Log correlation IDs**, not sensitive payloads (card numbers,
    passwords) — add a gateway filter that injects a trace/request ID
    header propagated to all downstream services for debugging.

---

## 12. Running the Full Stack (Quick Reference)

```bash
# 1. Start databases
docker compose up -d

# 2. Start discovery server
cd discovery-server && mvn spring-boot:run

# 3. Start services (separate terminals)
cd product-service && mvn spring-boot:run
cd order-service && mvn spring-boot:run
cd inventory-service && mvn spring-boot:run

# 4. Start the gateway
cd api-gateway && mvn spring-boot:run

# 5. Start the frontend
cd frontend && npm start
```

Verify:
- Eureka dashboard: `http://localhost:8761`
- Gateway health: `http://localhost:8080/actuator/health` (add
  `spring-boot-starter-actuator` to each service to enable this)
- Frontend: `http://localhost:3000`
