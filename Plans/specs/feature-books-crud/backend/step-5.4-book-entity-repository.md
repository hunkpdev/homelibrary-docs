# Step 5.4 – Book entitás és repository

## Mit állít elő

- `com.homelibrary.book.BookStatus` — enum
- `com.homelibrary.book.BookSource` — enum
- `com.homelibrary.book.Book` — JPA entitás
- `com.homelibrary.book.BookRepository` — Spring Data JPA repository

---

## Enumok

**`BookStatus`:** `AT_HOME`, `LOANED`, `DELETED`

**`BookSource`:** `OSZK`, `MANUAL` — külön enum a `book` package-ben (nem az `isbn` package `IsbnSource`-ja); azonos értékek, de eltérő kontextus és várható jövőbeli divergencia

---

## `Book` entitás

Mezők a [`Design/DB_SCHEMA.md`](../../../Design/DB_SCHEMA.md) `books` tábla szerint, beleértve a `subtitle` (`String`, nullable) és `pageCount` (`Integer`, nullable) mezőket — az OSZK MARC21 response-ból érkezhetnek (ld. Feature 4 élő response).

**Kapcsolatok:**
- `location` — `@ManyToOne(fetch = FetchType.LAZY)` → `Location`, nullable (könyv helyszín nélkül is felvehető)
- `addedBy` — `@ManyToOne(fetch = FetchType.LAZY)` → `User`, nullable a DB-ben, de felvételkor mindig kitöltött

**Enum tárolás:**
- `status` → `@Enumerated(EnumType.STRING)`, NOT NULL
- `source` → `@Enumerated(EnumType.STRING)`, nullable

**JSON string mezők:** `authors` és `categories` Java `String` típusként tárolva (nem `List<String>`) — a service réteg végzi a szerializációt/deszerializációt

**Timestamp stratégia:** `@PrePersist` / `@PreUpdate`, explicit `ZoneOffset.UTC` — ADR-008 szerint (`@CreationTimestamp` / `@UpdateTimestamp` tiltott)

**Egyéb:** `@Version` a `version` mezőn (optimistic locking); `deletedAt` nullable `OffsetDateTime` (soft delete audit)

---

## `BookRepository`

```java
public interface BookRepository extends JpaRepository<Book, UUID>, JpaSpecificationExecutor<Book>
```

Nem szükséges egyedi query metódus — a `GET /api/books` szűrési feltételeit (search, status, locationId, category, language, publishYear) a `BookSpecification` kezeli (step 5.6). A `findAll(Specification, Pageable)` az interfészből automatikusan jön.

---

## Elfogadási kritériumok

- Az entitás leképezhető a `books` táblára (nincs JPA indítási hiba)
- `findAll(spec, pageable)` meghívható (nincs fordítási hiba)
- `mvn clean package` hiba nélkül lefut
