# Step 1.1 – Maven projekt setup

## Mit állít elő

- `pom.xml` minden szükséges függőséggel
- `HomelibraryApplication.java` main osztály
- Alap package struktúra

---

## Package struktúra

```
com.homelibrary
├── HomelibraryApplication.java
├── config/          ← Spring Security, CORS, stb. (következő stepekben)
├── controller/      ← REST controllerek (következő stepekben)
├── service/         ← üzleti logika (következő stepekben)
├── repository/      ← JPA repository-k (következő stepekben)
├── entity/          ← JPA entitások (következő stepekben)
└── dto/             ← request/response DTO-k (következő stepekben)
```

---

## Függőségek

### Spring Boot (BOM: 4.0.5)

| Dependency | Mire |
|-----------|------|
| `spring-boot-starter-web` | REST API, embedded Tomcat (local profilon) |
| `spring-boot-starter-data-jpa` | JPA + Hibernate |
| `spring-boot-starter-security` | Spring Security alap |
| `spring-boot-starter-validation` | Bean Validation (@Valid) |
| `spring-boot-starter-test` | JUnit 5, Mockito, MockMvc |

### Adatbázis

| Dependency | Scope | Mire |
|-----------|-------|------|
| `org.postgresql:postgresql` | runtime | Neon PostgreSQL (prod profil) |
| `org.hsqldb:hsqldb` | runtime | In-memory DB (local profil) |

### Liquibase

| Dependency | Mire |
|-----------|------|
| `org.liquibase:liquibase-core` | Séma migráció (verzió: Spring Boot BOM-ból) |

### Lambda adapter

| Dependency | Verzió | Mire |
|-----------|--------|------|
| `com.amazonaws.serverless:aws-serverless-java-container-springboot4` | `3.0.1` | Spring Boot → Lambda bridge |

### JWT

| Dependency | Verzió | Mire |
|-----------|--------|------|
| `io.jsonwebtoken:jjwt-api` | `0.12.6` | JWT API |
| `io.jsonwebtoken:jjwt-impl` | `0.12.6` | JWT implementáció (runtime) |
| `io.jsonwebtoken:jjwt-jackson` | `0.12.6` | JWT JSON szerializáció (runtime) |

### API dokumentáció

| Dependency | Verzió | Mire |
|-----------|--------|------|
| `org.springdoc:springdoc-openapi-starter-webmvc-ui` | `3.0.2` | Swagger UI + OpenAPI spec |

---

## Build plugin

`maven-shade-plugin` — executable fat JAR generálás Lambda deployhoz:

```xml
<plugin>
    <groupId>org.apache.maven.plugins</groupId>
    <artifactId>maven-shade-plugin</artifactId>
    <configuration>
        <createDependencyReducedPom>false</createDependencyReducedPom>
    </configuration>
    <executions>
        <execution>
            <phase>package</phase>
            <goals><goal>shade</goal></goals>
        </execution>
    </executions>
</plugin>
```

---

## Java verzió

```xml
<properties>
    <java.version>21</java.version>
</properties>
```

---

## Elfogadási kritériumok

- `mvn clean package` hiba nélkül lefut
- `HomelibraryApplication` elindul `local` Spring profilon (következő stepben konfigurálva)
- A generált JAR tartalmazza az összes függőséget (fat JAR)
