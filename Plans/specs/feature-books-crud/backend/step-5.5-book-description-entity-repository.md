# Step 5.5 – BookDescription entitás és repository

## Mit állít elő

- `com.homelibrary.book.DescriptionSource` — enum
- `com.homelibrary.book.BookDescription` — JPA entitás
- `com.homelibrary.book.BookDescriptionRepository` — Spring Data JPA repository

---

## `DescriptionSource` enum

`ORIGINAL`, `AI_TRANSLATED`, `MANUAL` — külön enum a `book` package-ben (nem a `BookSource`); Fázis 3 AI fordítás is ide kerül `AI_TRANSLATED` értékkel

---

## `BookDescription` entitás

Mezők a [`Design/DB_SCHEMA.md`](../../../Design/DB_SCHEMA.md) `book_descriptions` tábla szerint.

**Kapcsolat:**
- `book` — `@ManyToOne(fetch = FetchType.LAZY)` → `Book`, NOT NULL; unidirectional (a `Book` entitáson nincs `@OneToMany` collection)

**Enum tárolás:** `source` → `@Enumerated(EnumType.STRING)`, nullable

**Nincs `@Version` mező** — a `book_descriptions` táblán nincs optimistic locking (lásd DB_SCHEMA.md)

**Timestamp stratégia:** `@PrePersist` / `@PreUpdate`, explicit `ZoneOffset.UTC` — ADR-008 szerint

**Unique constraint:** `(book_id, language)` — DB szinten Liquibase kezeli (step 5.3); JPA szinten nincs `@UniqueConstraint` annotáció (redundáns lenne)

---

## `BookDescriptionRepository`

```java
public interface BookDescriptionRepository extends JpaRepository<BookDescription, UUID>
```

Szükséges query metódusok:
- `findByBookIdAndLanguage(UUID bookId, String language)` — leírás lekérése könyv + nyelv alapján (létezés-ellenőrzés és frissítés előtt)
- `findAllByBookId(UUID bookId)` — egy könyv összes leírása (könyv törléskor)
- `findAllByBookIdInAndLanguage(Collection<UUID> bookIds, String language)` — lap szintű leírás-lekérés a listázáshoz (N+1 elkerülése); a step 5.6 `search` metódusa használja

---

## Elfogadási kritériumok

- Az entitás leképezhető a `book_descriptions` táblára (nincs JPA indítási hiba)
- `findByBookIdAndLanguage` és `findAllByBookId` meghívható (nincs fordítási hiba)
- `mvn clean package` hiba nélkül lefut
