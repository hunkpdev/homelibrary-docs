# HomeLibrary – Adatbázis Séma

> **Státusz:** v1 – Draft
> **Utolsó frissítés:** 2026-03-28
> **DB:** PostgreSQL 15 (Neon SaaS)
> **Migráció eszköz:** Flyway

## Entitások és Kapcsolatok

```
users
  │
  ├──< loans (kölcsönzések)
  │       │
  │       └── books
  │
  └── books
        │
        ├── locations (helyiség/polc)
        └──< book_descriptions (i18n leírások)
```

## Táblák

### `users`

| Oszlop | Típus | Megszorítás | Leírás |
|--------|-------|-------------|--------|
| `id` | `UUID` | PK | |
| `username` | `VARCHAR(50)` | UNIQUE NOT NULL | Bejelentkezési név |
| `email` | `VARCHAR(255)` | UNIQUE NOT NULL | |
| `password_hash` | `VARCHAR(255)` | NOT NULL | BCrypt |
| `preferred_language` | `VARCHAR(10)` | NOT NULL | pl. `hu`, `en` |
| `role` | `VARCHAR(20)` | NOT NULL | `ADMIN` vagy `VISITOR` |
| `active` | `BOOLEAN` | DEFAULT true | Soft disable felhasználóhoz |
| `created_at` | `TIMESTAMPTZ` | NOT NULL | |
| `updated_at` | `TIMESTAMPTZ` | NOT NULL | |

> Egytáblás jogosultság-kezelés, Spring Security `@PreAuthorize`-zal kezelve.
> Nem szükséges külön `user_roles` tábla a két szerepkörhöz.

---

### `locations` (helyiségek és polcok)

| Oszlop | Típus | Megszorítás | Leírás |
|--------|-------|-------------|--------|
| `id` | `UUID` | PK | |
| `room_name` | `VARCHAR(100)` | NOT NULL | pl. „Nappali", „Dolgozószoba" |
| `shelf_name` | `VARCHAR(100)` | | pl. „Jobb polc", „Alsó sor" |
| `description` | `TEXT` | | Opcionális megjegyzés |
| `active` | `BOOLEAN` | DEFAULT true | Soft delete |
| `created_at` | `TIMESTAMPTZ` | NOT NULL | |

---

### `books`

| Oszlop | Típus | Megszorítás | Leírás |
|--------|-------|-------------|--------|
| `id` | `UUID` | PK | |
| `isbn` | `VARCHAR(20)` | UNIQUE | ISBN-10 vagy ISBN-13 |
| `title` | `VARCHAR(500)` | NOT NULL | |
| `authors` | `TEXT[]` | | PostgreSQL tömb, pl. `{"Tolkien, J.R.R."}` |
| `publisher` | `VARCHAR(255)` | | |
| `publish_year` | `INTEGER` | | |
| `language` | `VARCHAR(10)` | | Az eredeti könyv nyelve, pl. `hu`, `en` |
| `categories` | `TEXT[]` | | pl. `{"Fantasy", "Fiction"}` |
| `cover_image_key` | `VARCHAR(500)` | | S3 object key a borítóképhez |
| `status` | `VARCHAR(20)` | NOT NULL | `AT_HOME`, `LOANED`, `DELETED` |
| `location_id` | `UUID` | FK → locations | Ahol a könyv éppen van |
| `source` | `VARCHAR(20)` | | `OPENLIBRARY`, `GOOGLE_BOOKS`, `MANUAL` |
| `added_by` | `UUID` | FK → users | Ki vette fel a rendszerbe |
| `created_at` | `TIMESTAMPTZ` | NOT NULL | |
| `updated_at` | `TIMESTAMPTZ` | NOT NULL | |
| `deleted_at` | `TIMESTAMPTZ` | | Soft delete időbélyege |

> **Indexek:**
> - `isbn` (unique index)
> - `status` (filter lekérdezésekhez)
> - `location_id` (JOIN-okhoz)
> - `authors` GIN index (szabad szöveges keresés tömbben)
> - `categories` GIN index (szabad szöveges keresés tömbben)

> **Megjegyzés:** Az `authors` és `categories` mezők PostgreSQL tömbként vannak tárolva.
> Ez elegendő az MVP-hez – GIN indexszel kereshetők. Normalizált tábla Fázis 2-ben mérlegelendő.

---

### `book_descriptions`

A könyv leírását i18n-nel tároljuk: könyvenként és nyelvenként egy sor.
Az AI fordítás is ide kerül, `AI_TRANSLATED` source-szal.

| Oszlop | Típus | Megszorítás | Leírás |
|--------|-------|-------------|--------|
| `id` | `UUID` | PK | |
| `book_id` | `UUID` | FK → books NOT NULL | |
| `language` | `VARCHAR(10)` | NOT NULL | pl. `hu`, `en` |
| `description` | `TEXT` | | |
| `source` | `VARCHAR(20)` | | `ORIGINAL`, `AI_TRANSLATED`, `MANUAL` |
| `created_at` | `TIMESTAMPTZ` | NOT NULL | |
| `updated_at` | `TIMESTAMPTZ` | NOT NULL | |

> **Unique constraint:** `(book_id, language)` – könyvenként egy leírás per nyelv
> Ha az AI fordít, új sor kerül be `AI_TRANSLATED` source-szal (nem felülírja az eredetit).

---

### `loans` (kölcsönzések)

| Oszlop | Típus | Megszorítás | Leírás |
|--------|-------|-------------|--------|
| `id` | `UUID` | PK | |
| `book_id` | `UUID` | FK → books NOT NULL | |
| `loaned_to_name` | `VARCHAR(255)` | NOT NULL | Kinek (nem feltétlenül regisztrált user) |
| `loaned_to_user_id` | `UUID` | FK → users | Ha regisztrált felhasználónak |
| `loaned_by` | `UUID` | FK → users NOT NULL | Ki adta kölcsön |
| `loaned_at` | `TIMESTAMPTZ` | NOT NULL | |
| `returned_at` | `TIMESTAMPTZ` | | NULL = még nincs visszahozva |
| `note` | `TEXT` | | Opcionális megjegyzés |

---

## Flyway Migráció Névkonvenció

```
db/migration/
  V1__create_users_and_locations.sql
  V2__create_books.sql
  V3__create_book_descriptions.sql
  V4__create_loans.sql
  V5__add_indexes.sql
```

## Nyitott Kérdések

- [ ] `authors` és `categories` maradjon tömb, vagy kerüljön külön normalizált táblába? (MVP-hez tömb elegendő)
- [ ] `book_history` audit log szükséges-e az MVP-be, vagy Fázis 2?
