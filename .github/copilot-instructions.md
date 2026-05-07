# Copilot Instructions

## Build & Run

```bash
# Build
./mvnw clean package

# Run (app starts on port 8088)
./mvnw spring-boot:run

# Run tests
./mvnw test

# Run a single test class
./mvnw test -Dtest=LbmApplicationTests
```

H2 console is available at `http://localhost:8088/h2` when the app is running.

## Architecture

Spring Boot 2.1.3 / Java 8 MVC app with JSP views. Package root: `com.verinon.lbm`.

```
controller/   — Spring MVC @Controller classes (BookController, MemberController, LoginController, ErrorController)
pojos/        — JPA @Entity model classes (BookPojo, MemberPojo, SmartBookSystem)
jpa/          — Spring Data JPA repositories (BookRepository, MemberRepository, SmartBookSystemRepository)
services/     — Business logic services (BookServices, MemberServices, SmartServices, LoginServices)
security/     — Spring Security config (LoginSecurityConfig)
```

Views are JSP files under `src/main/webapp/WEB-INF/jsps/`. Shared layout fragments (header, footer, navigation) live in `WEB-INF/jsps/commons/` as `.jspf` files.

Static assets (CSS) are in `src/main/resources/static/`.

## Key Conventions

**In-memory data, not JPA persistence at runtime.** Despite the presence of `@Entity` annotations and JPA repositories, all runtime data is stored in `static List<>` fields inside the `@Service` classes. The JPA/repository calls are commented out throughout the codebase. Do not wire in the repositories unless intentionally migrating to database-backed persistence.

**"Borrow" is spelled "barrow" throughout the codebase** — field names (`book_date_of_barrow`), variable names, URL paths (`/show-barrow-list`, `/del-smartbs`), and JSP file names all use this spelling. Maintain this spelling for consistency.

**SmartBookSystem supports up to 3 books per borrow transaction.** The `SmartBookSystem` entity stores three separate book name fields (`bookName`, `bookName2`, `bookName3`) rather than a collection.

**Authentication is hardcoded.** A single in-memory user `admin / admin` is configured in `LoginSecurityConfig`. CSRF is disabled. The login page is at `/userlogin` and redirects to `/home` on success.

**Controller methods use raw `ModelMap`** instead of `@ModelAttribute` — data is pushed with `model.put(key, value)` and retrieved in JSPs via `${key}`.

**`MemberServices` is instantiated manually in one controller method** (`BookController.showMainPageForBookOperations`) rather than being `@Autowired`, which means it operates on a separate instance from the Spring-managed bean. Keep this in mind when tracing member data flow from that endpoint.
