# Step 5.6 – BookService

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
| `category` | `LIKE %value%` a `categories` JSON string-en |
| `language` | exact egyezés |
| `publishYear` | exact egyezés |

A predikátumok `and`-del láncolódnak. **`status != DELETED` mindig benne van** — törölt könyv soha nem jelenik meg a listában, függetlenül a `status` szűrőparamétertől. Ha a hívó `status=DELETED`-et küld, az eredmény üres lap (a két feltétel együtt kizárja egymást — a controller validálhatja, de a service önmagában is biztonságos).

---

## `BookService` metódusai

### `create(BookCreateRequest request, UUID addedByUserId) → BookResponse`
- `locationId` megadva → `locationRepository.findById` + aktív ellenőrzés → nem található vagy inaktív esetén 404
- `authors` és `categories`: `List<String>` → JSON string (Jackson `ObjectMapper`)
- `status` alapértelmezetten `AT_HOME` ha a request nem adja meg
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

## Unit tesztek (`BookServiceTest`)

| Teszt | Mit ellenőriz |
|-------|---------------|
| `create_validRequest_savesBook` | book létrejön, description elmentve |
| `create_invalidLocationId_throws404` | nem létező location → 404 |
| `update_optimisticLockConflict_throws409` | verziószám ütközés → 409 |
| `softDelete_setsStatusAndDeletedAt` | status DELETED, deletedAt kitöltött |
| `search_withStatusFilter_returnsFilteredResults` | Specification szűr |
| `authors_serializationRoundtrip` | List<String> → JSON String → List<String> |
