# Copilot Instructions for Spring Boot Shopping Cart

## Project Architecture

This is a **Spring Boot 1.5.3** e-commerce web application using **Spring Security**, **Thymeleaf**, **JPA**, and **H2** database. The app follows a layered architecture with clear separation of concerns.

### Core Components
- **Controllers** (`src/main/java/com/reljicd/controller/`): Handle HTTP requests, return ModelAndView objects
- **Services** (`src/main/java/com/reljicd/service/impl/`): Business logic layer, use `@Transactional` for data operations  
- **Repositories** (`src/main/java/com/reljicd/repository/`): JPA data access layer extending JpaRepository
- **Models** (`src/main/java/com/reljicd/model/`): JPA entities with validation annotations
- **Templates** (`src/main/resources/templates/`): Thymeleaf views with fragment-based composition

### Session-Based Shopping Cart
The shopping cart is implemented as a **session-scoped bean** (`ShoppingCartServiceImpl`) using:
```java
@Scope(value = WebApplicationContext.SCOPE_SESSION, proxyMode = ScopedProxyMode.TARGET_CLASS)
```
Cart state is maintained in a `Map<Product, Integer>` within the user session.

## Development Patterns

### Controller Pattern
- Use `ModelAndView` instead of separate Model/View
- Path variables for entity IDs: `@PathVariable("productId") Long productId`
- Exception handling returns views with error messages: `return shoppingCart().addObject("outOfStockMessage", e.getMessage())`

### Service Layer Pattern  
- Implement interfaces in `service/impl/` packages
- Use constructor injection with `@Autowired`
- Apply `@Transactional` at class level for service implementations

### Template Composition
Templates use Thymeleaf fragments heavily:
```html
<div th:replace="/fragments/header :: navbar"/>
<div th:replace="/fragments/products :: products"/>
```
Key fragments: `header.html`, `footer.html`, `products.html`, `pagination.html`

### Security Configuration
- Database-backed authentication using custom queries in `application.properties`
- Admin user configured via properties: `spring.admin.username/password`
- URL-based authorization in `SpringSecurityConfig.java`

## Key Build & Run Commands

```bash
# Development (using Maven wrapper)
chmod +x scripts/mvnw
scripts/mvnw spring-boot:run

# Build JAR
scripts/mvnw clean package
java -jar target/shopping-cart-0.0.1-SNAPSHOT.jar

# Docker build & run
scripts/run_docker.sh
# OR manually:
mvn clean package
docker build -t shopping-cart:dev -f docker/Dockerfile .
docker run --rm -i -p 8070:8070 --name shopping-cart shopping-cart:dev

# Tests
mvn test
```

## Configuration Files

- **`application.properties`**: Main config (port 8070, H2 database, security queries, admin credentials)
- **`import-h2.sql`**: Database seed data (users with BCrypt passwords, products, roles)
- **`SpringSecurityConfig.java`**: Security configuration using database authentication

## Important URLs & Credentials
- App: `http://localhost:8070/home`
- H2 Console: `http://localhost:8070/h2-console` (JDBC URL: `jdbc:h2:mem:shopping_cart_db`)
- Admin: username=`admin`, password=`admin`
- User: username=`user`, password=`password`

## Deployment
- **Kubernetes**: `deploymentservice.yml` defines deployment with 2 replicas and LoadBalancer service
- **Docker**: Uses OpenJDK 8 Alpine base image, exposes port 8070