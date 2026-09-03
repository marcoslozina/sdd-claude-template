---
name: lang-java
description: Java 21+ standards - hexagonal package layout, records for value objects and DTOs, domain entities and use cases, Spring Boot DI, and JUnit 5 + Mockito + AssertJ + Testcontainers testing. Use when writing or reviewing Java code, .java files, Gradle/Maven builds, or Spring Boot services.
---

# Skill: Java

## Project setup

**Gradle (preferred):**
```bash
gradle init --type java-application --dsl kotlin
```

**Maven:**
```bash
mvn archetype:generate -DgroupId=com.example -DartifactId=name
```

Java 21+ with records, sealed classes and pattern matching available.

## Mandatory conventions

- Java 21+
- Records for value objects and immutable DTOs
- `Optional<T>` only in returns, never as a parameter
- Unchecked domain exceptions (`extends RuntimeException`)
- Packages by layer, not by type (`domain.user`, not `entities.User`)

## Layer structure

```
src/main/java/com/example/
  domain/
    model/           # entities, value objects, aggregates
    port/            # interfaces (inbound and outbound)
    exception/       # domain exceptions
  application/
    usecase/         # use cases (implement inbound ports)
    service/         # application services
  infrastructure/
    adapter/
      in/            # REST controllers, consumers
      out/           # repos, HTTP clients, etc.
    config/          # Spring beans, configuration
src/test/java/
  unit/
  integration/
```

## Key patterns

### Outbound port
```java
// domain/port/UserRepository.java
public interface UserRepository {
    Optional<User> findById(UserId id);
    void save(User user);
}
```

### Domain entity
```java
// domain/model/User.java
public class User {
    private final UserId id;
    private String name;
    private Email email;

    private User(UserId id, String name, Email email) {
        this.id = id;
        this.name = name;
        this.email = email;
    }

    public static User create(String name, String email) {
        return new User(UserId.generate(), name, new Email(email));
    }
}
```

### Value Object with a record
```java
// domain/model/UserId.java
public record UserId(UUID value) {
    public static UserId generate() {
        return new UserId(UUID.randomUUID());
    }
}
```

### Use Case
```java
// application/usecase/CreateUserUseCase.java
public class CreateUserUseCase {
    private final UserRepository repository;

    public CreateUserUseCase(UserRepository repository) {
        this.repository = repository;
    }

    public UserId execute(String name, String email) {
        var user = User.create(name, email);
        repository.save(user);
        return user.getId();
    }
}
```

### With Spring Boot (automatic DI)
```java
@Service
public class CreateUserUseCase { ... }

@Repository
public class JpaUserRepository implements UserRepository { ... }
```

## Testing

```bash
./gradlew test              # everything
./gradlew test --tests "*CreateUser*"
```

Stack: JUnit 5 + Mockito + AssertJ

Naming: `should_<result>_when_<condition>`

```java
@Test
void should_throw_duplicate_error_when_email_already_exists() {
    // given
    when(repository.findByEmail(any())).thenReturn(Optional.of(existingUser));
    // when / then
    assertThatThrownBy(() -> useCase.execute("Juan", "a@b.com"))
        .isInstanceOf(DuplicateEmailException.class);
}
```

- Unit: Mockito for ports, test use cases in isolation
- Integration: `@SpringBootTest` + Testcontainers for repos

## Common architecture decisions in Java

For these choices, apply the decision protocol (this project's CLAUDE.md if it defines one, otherwise dev-harness's docs/DECISION_PROTOCOL.md):
- **Framework:** Spring Boot vs Quarkus vs Micronaut vs plain Java
- **Build:** Gradle (Kotlin DSL) vs Maven
- **Persistence:** JPA/Hibernate vs JOOQ vs plain JDBC
- **REST:** Spring MVC vs Spring WebFlux (reactive)
- **Messaging:** Kafka vs RabbitMQ vs SQS
- **Integration tests:** Testcontainers vs in-memory H2

## When to use what

| Case | Use |
|------|------|
| Immutable value object | `record` |
| Entity with mutable state | `class` with a private constructor |
| Result that can fail | `Optional<T>` or a domain exception |
| Typed config | `@ConfigurationProperties` |
| API DTO | `record` with Jackson annotations |
