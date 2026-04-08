# Step 1.2 – Application properties

## Mit állít elő

- `src/main/resources/application.properties` — közös alap konfiguráció
- `src/main/resources/application-local.properties` — local profil (HSQLDB, embedded Tomcat)
- `src/main/resources/application-prod.properties` — prod profil (Neon PostgreSQL, Lambda adapter)

---

## Profil aktiválása

| Környezet | Aktiválás |
|-----------|-----------|
| Local fejlesztés | `SPRING_PROFILES_ACTIVE=local` (IntelliJ run config vagy `mvn spring-boot:run -Dspring-boot.run.profiles=local`) |
| Lambda (prod) | Lambda environment variable: `SPRING_PROFILES_ACTIVE=prod` |

---

## application.properties (közös)

```properties
spring.application.name=homelibrary

# Liquibase – mindkét profilon fut
spring.liquibase.change-log=classpath:db/changelog/db.changelog-master.xml

# JWT – értékét prod-on SSM-ből kapja (ld. lent), local-on fix placeholder
app.jwt.secret=${JWT_SECRET:local-dev-secret-not-for-production}
app.jwt.access-token-expiration-ms=900000
# 900000 ms = 15 perc
app.jwt.refresh-token-expiration-ms=604800000
# 604800000 ms = 7 nap

# Cookie secure flag – false local-on, true prod-on
app.cookie.secure=true

# OpenAPI – portfólió/tanuló projekt, minden profilon publikusan elérhető
springdoc.swagger-ui.enabled=true
springdoc.api-docs.enabled=true
```

---

## application-local.properties

```properties
# HSQLDB in-memory
spring.datasource.url=jdbc:hsqldb:mem:homelibrary;DB_CLOSE_DELAY=-1
spring.datasource.driver-class-name=org.hsqldb.jdbc.JDBCDriver
spring.datasource.username=sa
spring.datasource.password=

spring.jpa.database-platform=org.hibernate.dialect.HSQLDialect
spring.jpa.hibernate.ddl-auto=none

# Embedded Tomcat
server.port=8080

# Cookie – http://localhost-on Secure cookie nem megy
app.cookie.secure=false

```

---

## application-prod.properties

```properties
# Neon PostgreSQL – connection string SSM Parameter Store-ból
# Lambda environment variable-ként injektálva: SPRING_DATASOURCE_URL
spring.datasource.url=${SPRING_DATASOURCE_URL}
spring.datasource.driver-class-name=org.postgresql.Driver
spring.datasource.username=${SPRING_DATASOURCE_USERNAME:}
spring.datasource.password=${SPRING_DATASOURCE_PASSWORD:}

spring.jpa.database-platform=org.hibernate.dialect.PostgreSQLDialect
spring.jpa.hibernate.ddl-auto=none

# Lambda adapter – ne indítson embedded Tomcat-ot
spring.main.web-application-type=none
```

> **Megjegyzés – Neon connection string formátuma:**
> A Neon `jdbc:postgresql://<host>/<db>?sslmode=require` formátumú connection string-et használ.
> A Lambda CDK stack az SSM `/homelibrary/neon-connection-string` paraméterből olvassa ki és injektálja
> `SPRING_DATASOURCE_URL` environment variable-ként (ld. step 1.12).

---

## Elfogadási kritériumok

- `mvn spring-boot:run -Dspring-boot.run.profiles=local` — az alkalmazás elindul, HSQLDB-t használ
- `local` profilon `http://localhost:8080/swagger-ui/index.html` elérhető
- `app.cookie.secure` értéke `false` local profilon (HTTPS nélkül a cookie működik)
