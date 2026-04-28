# HomeLibrary – Adatbázis Séma

> **Státusz:** v2 – SQL szabvány típusok
> **Utolsó frissítés:** 2026-04-03
> **DB:** PostgreSQL 15 (Neon SaaS)
> **Migráció eszköz:** Liquibase (lásd ADR-005)

> **Tervezési alapelv:** Kizárólag SQL szabványos adattípusokat használunk – nincs gyártófüggő típus (lásd ADR-006).

## Entitások és Kapcsolatok

```
rooms (helyiség)
  │
  └──< locations (konkrét helyszín: polc, szekrény, láda, stb.)
          │
          └──< books
                │
                ├──< loans (kölcsönzések)
                │       │
                │       └── users (kölcsönző)
                │
                └──< book_descriptions (i18n leírások)

users
  ├──< loans (loaned_by)
  └──< books (added_by)
```

## Táblák

### `users`

| Oszlop | Típus | Megszorítás | Leírás |
|--------|-------|-------------|--------|
| `id` | `UUID` | PK | |
| `username` | `VARCHAR(50)` | UNIQUE NOT NULL | Bejelentkezési név |
| `email` | `VARCHAR(255)` | UNIQUE NOT NULL | |
| `password_hash` | `VARCHAR(255)` | NOT NULL | BCrypt |
| `preferred_language` | `VARCHAR(10)` | NULL | `hu` vagy `en`; NULL = admin által létrehozva, első bejelentkezés még nem történt meg — login endpoint tölti ki `Accept-Language` header alapján |
| `role` | `VARCHAR(20)` | NOT NULL | `ADMIN`, `VISITOR` vagy `DEMO` |
| `active` | `BOOLEAN` | DEFAULT true | Soft disable felhasználóhoz |
| `refresh_token_hash` | `VARCHAR(255)` | | BCrypt hash, NULL ha nincs aktív session |
| `refresh_token_expires_at` | `TIMESTAMP WITH TIME ZONE` | | NULL ha nincs aktív session |
| `version` | `BIGINT` | NOT NULL DEFAULT 0 | JPA optimistic locking (`@Version`) |
| `created_at` | `TIMESTAMP WITH TIME ZONE` | NOT NULL | |
| `updated_at` | `TIMESTAMP WITH TIME ZONE` | NOT NULL | |

> Egytáblás jogosultság-kezelés, Spring Security `@PreAuthorize`-zal kezelve.
> Nem szükséges külön `user_roles` tábla a két szerepkörhöz.

---

### `rooms` (helyiségek)

| Oszlop | Típus | Megszorítás | Leírás |
|--------|-------|-------------|--------|
| `id` | `UUID` | PK | |
| `name` | `VARCHAR(100)` | NOT NULL | pl. „Nappali", „Dolgozószoba" |
| `description` | `TEXT` | | Opcionális megjegyzés |
| `active` | `BOOLEAN` | NOT NULL DEFAULT true | Soft delete |
| `version` | `BIGINT` | NOT NULL DEFAULT 0 | JPA optimistic locking (`@Version`) |
| `created_at` | `TIMESTAMP WITH TIME ZONE` | NOT NULL | |
| `updated_at` | `TIMESTAMP WITH TIME ZONE` | NOT NULL | |

---

### `locations` (konkrét helyszínek)

Egy location pontosan egy roomhoz tartozik (`room_id` NOT NULL) — lásd ADR-007.

| Oszlop | Típus | Megszorítás | Leírás |
|--------|-------|-------------|--------|
| `id` | `UUID` | PK | |
| `name` | `VARCHAR(100)` | NOT NULL | pl. „Bal polc", „Felső szekrény", „Sarokláda" |
| `description` | `TEXT` | | Opcionális megjegyzés |
| `room_id` | `UUID` | FK → rooms NOT NULL | Melyik helyiségben van |
| `active` | `BOOLEAN` | NOT NULL DEFAULT true | Soft delete |
| `version` | `BIGINT` | NOT NULL DEFAULT 0 | JPA optimistic locking (`@Version`) |
| `created_at` | `TIMESTAMP WITH TIME ZONE` | NOT NULL | |
| `updated_at` | `TIMESTAMP WITH TIME ZONE` | NOT NULL | |

---

### `books`

| Oszlop | Típus | Megszorítás | Leírás |
|--------|-------|-------------|--------|
| `id` | `UUID` | PK | |
| `isbn` | `VARCHAR(20)` | UNIQUE, NULL megengedett | ISBN-10 vagy ISBN-13. NULL ha a könyv vonalkód nélkül kerül felvételre (antik, ritka, saját kiadású). PostgreSQL több NULL értéket megenged UNIQUE constraint esetén. |
| `title` | `VARCHAR(500)` | NOT NULL | |
| `authors` | `VARCHAR(1000)` | | JSON string, pl. `["Tolkien, J.R.R."]` |
| `publisher` | `VARCHAR(255)` | | |
| `publish_year` | `INTEGER` | | |
| `language` | `VARCHAR(10)` | | Az eredeti könyv nyelve, pl. `hu`, `en` |
| `categories` | `VARCHAR(500)` | | JSON string, pl. `["Fantasy", "Fiction"]` |
| `cover_image_url` | `VARCHAR(500)` | | Fázis 1: külső forrás borítókép URL-je (OpenLibrary/Google Books). Fázis 2: Liquibase migráció — `cover_image_key` (S3 object key) mező kerül be helyette, a service réteg generálja a pre-signed URL-t. |
| `status` | `VARCHAR(20)` | NOT NULL | `AT_HOME`, `LOANED`, `DELETED` |
| `location_id` | `UUID` | FK → locations | Ahol a könyv éppen van |
| `source` | `VARCHAR(20)` | | `OPENLIBRARY`, `GOOGLE_BOOKS`, `MANUAL` |
| `added_by` | `UUID` | FK → users | Ki vette fel a rendszerbe |
| `version` | `BIGINT` | NOT NULL DEFAULT 0 | JPA optimistic locking (`@Version`) |
| `created_at` | `TIMESTAMP WITH TIME ZONE` | NOT NULL | |
| `updated_at` | `TIMESTAMP WITH TIME ZONE` | NOT NULL | |
| `deleted_at` | `TIMESTAMP WITH TIME ZONE` | | Soft delete időbélyege (audit célú). Canonical forrás: `status == 'DELETED'`. Mindig együtt változnak. |

> **Indexek:**
> - `isbn` (unique index)
> - `status` (filter lekérdezésekhez)
> - `location_id` (JOIN-okhoz)

> **Megjegyzés:** Az `authors` és `categories` mezők JSON string-ként vannak tárolva (`VARCHAR`).
> A háztartási méretskálán (max. néhány ezer könyv) LIKE-alapú keresés bőven elegendő, külön index nem szükséges.
> Normalizált tábla Fázis 2-ben mérlegelendő.

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
| `created_at` | `TIMESTAMP WITH TIME ZONE` | NOT NULL | |
| `updated_at` | `TIMESTAMP WITH TIME ZONE` | NOT NULL | |

> **Unique constraint:** `(book_id, language)` – könyvenként egy leírás per nyelv
> Ha az AI fordít, új sor kerül be `AI_TRANSLATED` source-szal (nem felülírja az eredetit).

---

### `loans` (kölcsönzések)

> **Optimistic locking:** A `loans` táblán nincs `version` mező. A `PUT /api/loans/{id}/return` endpointnál konkurens módosítás elvileg lehetséges, de 2-5 felhasználós háztartási alkalmazásnál a valószínűsége elhanyagolható — tudatos döntés a komplexitás csökkentése érdekében.

| Oszlop | Típus | Megszorítás | Leírás |
|--------|-------|-------------|--------|
| `id` | `UUID` | PK | |
| `book_id` | `UUID` | FK → books NOT NULL | |
| `loaned_to_name` | `VARCHAR(255)` | NOT NULL | Kinek (nem feltétlenül regisztrált user) |
| `loaned_to_user_id` | `UUID` | FK → users | Ha regisztrált felhasználónak |
| `loaned_by` | `UUID` | FK → users NOT NULL | Ki adta kölcsön |
| `loaned_at` | `TIMESTAMP WITH TIME ZONE` | NOT NULL | |
| `returned_at` | `TIMESTAMP WITH TIME ZONE` | | NULL = még nincs visszahozva |
| `note` | `TEXT` | | Opcionális megjegyzés |

---

## Liquibase Changelog Struktúra

```
db/changelog/
  db.changelog-master.yaml
  changes/
    001-create-users.yaml
    002-create-rooms.yaml
    003-create-locations.yaml
    004-create-books.yaml
    005-create-book-descriptions.yaml
    006-create-loans.yaml
    007-add-indexes.yaml
```

## Nyitott Kérdések

- [ ] `authors` és `categories` maradjon tömb, vagy kerüljön külön normalizált táblába? (MVP-hez tömb elegendő)
- [ ] `book_history` audit log szükséges-e az MVP-be, vagy Fázis 2?
