# Step 5.3 – Liquibase: books és book_descriptions tábla

## Mit állít elő

- `db/changelog/changes/004-create-books.yaml` — `books` tábla + indexek
- `db/changelog/changes/005-create-book-descriptions.yaml` — `book_descriptions` tábla
- bekötve a `db.changelog-master.yaml`-ba (008 után, a sor végére)

---

## Tábla struktúrák

Teljes séma: [`Design/DB_SCHEMA.md`](../../../Design/DB_SCHEMA.md). Az alábbiakban csak a döntéseket igénylő pontok.

### `books`

| Oszlop | Típus | Megszorítás |
|--------|-------|-------------|
| `id` | `UUID` | PK |
| `isbn` | `VARCHAR(20)` | UNIQUE, NULL megengedett |
| `title` | `VARCHAR(500)` | NOT NULL |
| `subtitle` | `VARCHAR(500)` | — |
| `authors` | `VARCHAR(1000)` | — (JSON string) |
| `publisher` | `VARCHAR(255)` | — |
| `publish_year` | `INTEGER` | — |
| `page_count` | `INTEGER` | — |
| `language` | `VARCHAR(10)` | — |
| `categories` | `VARCHAR(500)` | — (JSON string) |
| `cover_image_url` | `VARCHAR(500)` | — |
| `status` | `VARCHAR(20)` | NOT NULL |
| `location_id` | `UUID` | FK → locations |
| `source` | `VARCHAR(20)` | — |
| `added_by` | `UUID` | FK → users |
| `version` | `BIGINT` | NOT NULL DEFAULT 0 |
| `created_at` | `TIMESTAMP WITH TIME ZONE` | NOT NULL |
| `updated_at` | `TIMESTAMP WITH TIME ZONE` | NOT NULL |
| `deleted_at` | `TIMESTAMP WITH TIME ZONE` | — |

**Indexek** (004-create-books.yaml-ban, createIndex changeSet-ként):
- `isbn` — az UNIQUE constraint implicit indexet hoz létre
- `status` — szűrő lekérdezésekhez
- `location_id` — JOIN-okhoz

### `book_descriptions`

| Oszlop | Típus | Megszorítás |
|--------|-------|-------------|
| `id` | `UUID` | PK |
| `book_id` | `UUID` | FK → books NOT NULL |
| `language` | `VARCHAR(10)` | NOT NULL |
| `description` | `TEXT` | — |
| `source` | `VARCHAR(20)` | — |
| `created_at` | `TIMESTAMP WITH TIME ZONE` | NOT NULL |
| `updated_at` | `TIMESTAMP WITH TIME ZONE` | NOT NULL |

**Unique constraint:** `(book_id, language)` — implicit indexet hoz létre

---

## Kulcs döntések

- A changeset-ek száma 004 és 005 — a DB_SCHEMA.md-ben tervezett sorrend szerint; a master changelog-ban 008 után kerülnek bekötésre (Liquibase a master sorrendjében futtatja)
- A `DB_SCHEMA.md`-ben tervezett `007-add-indexes.yaml` feleslegessé válik: az indexek a táblalétrehozó changeset-ekbe kerülnek bele, nem külön fájlba
- `isbn` NULL megengedett: PostgreSQL UNIQUE constraint több NULL értéket enged (vonalkód nélküli könyvekhez)
- `authors` és `categories` JSON string (`VARCHAR`) — ADR-006, háztartási méretskálán LIKE-keresés elegendő
- `cover_image_url` Fázis 1-ben mindig NULL — Fázis 2-ban Liquibase migrációval cserélődik `cover_image_key`-re
- ADR-008: `@PrePersist`/`@PreUpdate`, explicit `ZoneOffset.UTC` — `@CreationTimestamp`/`@UpdateTimestamp` tiltott

---

## Elfogadási kritériumok

- App indításkor Liquibase lefuttatja mindkét changesetet hibák nélkül
- `books` és `book_descriptions` táblák létrejönnek HSQLDB-ben és Neon PostgreSQL-en
- `status`, `location_id` indexek és a `(book_id, language)` unique constraint létrejön
