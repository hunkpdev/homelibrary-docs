# Step 2.2 – `User` entitás + `UserRepository`

## Mit állít elő

- `com.homelibrary.model.Role` — cross-domain enum
- `com.homelibrary.entity.User` — JPA entitás
- `com.homelibrary.repository.UserRepository` — Spring Data JPA repository

---

## `Role` enum

A `com.homelibrary.model` package-be kerül — az entitás és a többi réteg (Security config, JWT, DTO-k) egyaránt innen importálja.

| Érték |
|---|
| `ADMIN` |
| `VISITOR` |

---

## `User` entitás

A `users` táblára képezi le (az 1.3-as Liquibase changeset hozta létre). Nem implementálja a `UserDetails` interfészt — a Spring Security integrációhoz külön `UserDetailsService` lesz (step 2.5).

**Mezők és mapping:**

| Java mező | DB oszlop | Típus | Megjegyzés |
|---|---|---|---|
| `id` | `id` | `UUID` | `@Id`, `@GeneratedValue(strategy = GenerationType.UUID)` (Hibernate 6+) |
| `username` | `username` | `String` | `unique=true`, `nullable=false` |
| `email` | `email` | `String` | `unique=true`, `nullable=false` |
| `passwordHash` | `password_hash` | `String` | `nullable=false` |
| `preferredLanguage` | `preferred_language` | `String` | `nullable=false` |
| `role` | `role` | `Role` | `@Enumerated(EnumType.STRING)`, `nullable=false` |
| `active` | `active` | `boolean` | `nullable=false` |
| `refreshTokenHash` | `refresh_token_hash` | `String` | nullable |
| `refreshTokenExpiresAt` | `refresh_token_expires_at` | `OffsetDateTime` | nullable |
| `version` | `version` | `Long` | `@Version` — optimistic locking |
| `createdAt` | `created_at` | `OffsetDateTime` | `@Column(updatable=false)`, `@PrePersist` állítja |
| `updatedAt` | `updated_at` | `OffsetDateTime` | `@PrePersist` és `@PreUpdate` állítja |

**Timestamp kezelés:** `@PrePersist` / `@PreUpdate` lifecycle callback-ek — nem szükséges `@EnableJpaAuditing` konfiguráció.

---

## `UserRepository`

`JpaRepository<User, UUID>` kiterjesztése, egyetlen egyedi query metódussal:

| Metódus | Mire kell |
|---|---|
| `Optional<User> findByUsername(String username)` | Login (step 2.6), `UserDetailsService` (step 2.5) |

A `findById` örökölt — a JWT filter (step 2.4) használja.

---

## Elfogadási kritériumok

- `mvn clean package` hiba nélkül lefut
- `local` profilon az alkalmazás elindul, a `User` entitás a meglévő `users` táblára mappolódik hibamentesen
- `UserRepository.findByUsername("admin")` visszaadja a seed admin usert
