# Step 5.5 – BookService

## Mit állít elő

- `com.homelibrary.book.BookService` — üzleti logika
- `com.homelibrary.book.BookSpecification` — dinamikus Specification factory
- DTO-k (ugyanabban a package-ben):
  - `BookCreateRequest` — POST body; tartalmazza `subtitle` (nullable String), `pageCount` (nullable Integer) és `description` (nullable String) mezőket
  - `BookUpdateRequest` — PUT body (BookCreateRequest + `version`)
  - `BookSearchParams` — GET /api/books query paraméterek
  - `BookResponse` — egységes válasz lista és detail nézethez; tartalmazza `subtitle`, `pageCount` és `description` mezőket
  - `BookLocationResponse` — nested location (id, name, room)
  - `BookRoomResponse` — nested room (id, name)
- `com.homelibrary.book.BookServiceTest` — unit tesztek

**Mellékhatás — LocationService tech-debt lezárása:** a `books` tábla most már létezik, a Feature 3-ban elhalasztott két feladat itt oldandó meg:
- `LocationService.softDelete`: aktív könyv ellenőrzés — ha van `status != 'DELETED'` könyv a locationhöz → 409
- `LocationService` listázás: `bookCount` számított értékként (GROUP BY query, `WHERE b.status != 'DELETED'` szűrővel — törölt könyvek nem számítanak)

---

## `BookSpecification`

Statikus factory metódusok, `Specification<Book>` típust adnak vissza. Minden predikátum csak akkor kerül be, ha a szűrőérték nem null:

| Szűrő | Predikátum |
|-------|------------|
| `search` | `LIKE %value%` a `title`-n ÉS `authors`-on (OR) |
| `status` | exact egyezés |
| `locationId` | exact egyezés |
| `category` | `LIKE %"value"%` a `categories` JSON string-en — dupla idézőjelek biztosítják a JSON elemhatáron való illesztést; a value-ban szereplő `"` karakter escape-elendő (SQL injection védelem) |
| `language` | exact egyezés |
| `publishYear` | exact egyezés |

A predikátumok `and`-del láncolódnak. **`status != DELETED` mindig benne van** — törölt könyv soha nem jelenik meg a listában, függetlenül a `status` szűrőparamétertől. Ha a hívó `status=DELETED`-et küld, az eredmény üres lap (a két feltétel együtt kizárja egymást — a controller validálhatja, de a service önmagában is biztonságos).

---

## `BookService` metódusai

### `create(BookCreateRequest request, UUID addedByUserId) → BookResponse`
- `locationId` megadva → `locationRepository.findById` + aktív ellenőrzés → nem található vagy inaktív esetén 404
- `authors` és `categories`: `List<String>` → JSON string (Jackson `ObjectMapper`)
- `status` mindig `AT_HOME` — a request nem tartalmazza, a service állítja be
- `addedBy` beállítása a paraméterből
- `description` közvetlenül a `books` táblára kerül (nullable)

### `getById(UUID id) → BookResponse`
- Nem található → 404
- `description` közvetlenül a `Book` entitásból kerül a response-ba

### `search(BookSearchParams params, Pageable pageable) → Page<BookResponse>`
- `BookSpecification` alapján dinamikus szűrés
- `authors` és `categories`: JSON string → `List<String>` (ObjectMapper)

### `update(UUID id, BookUpdateRequest request) → BookResponse`
- Nem található → 404
- `locationId` változás esetén: új location létezés + aktív ellenőrzés → 404 ha nem elérhető
- `ObjectOptimisticLockingFailureException` → 409 Conflict
- `description` közvetlenül frissül a `books` táblán

### `softDelete(UUID id) → void`
- Nem található → 404
- `status` → `DELETED`, `deletedAt` → `OffsetDateTime.now(ZoneOffset.UTC)`

---

## Kulcs döntések

- **`authors` / `categories` konverzió:** Jackson `ObjectMapper` a service rétegben — entity `String` ↔ DTO `List<String>`. Nem `@Converter` az entitáson, mert a `BookSpecification` LIKE predikátumnak a nyers VARCHAR oszlopra kell hivatkoznia
- **Törölt könyvek szűrése:** `status != DELETED` mindig be van égetve a `BookSpecification`-be — törölt könyv semmilyen felhasználónak nem jelenik meg a listában
- **`version` benne van a `BookResponse`-ban** — az update form-nak szüksége van rá az optimistic locking miatt

---

## Elfogadási kritériumok

- Könyv létrehozásakor `status` mindig `AT_HOME`, `description` elmentve
- Nem létező vagy inaktív `locationId` → 404
- Konkurens módosítás (`ObjectOptimisticLockingFailureException`) → 409 Conflict
- Soft delete után `status = DELETED`, `deletedAt` kitöltött
- `BookSpecification` kombinált szűrők esetén AND kapcsolatban szűr; `status != DELETED` mindig érvényes
- `authors` és `categories`: `List<String>` ↔ JSON String konverzió veszteségmentes
