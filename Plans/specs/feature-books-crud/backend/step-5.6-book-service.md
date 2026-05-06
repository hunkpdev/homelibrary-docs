# Step 5.6 – BookService

## Mit állít elő

- `com.homelibrary.book.BookService` — üzleti logika
- `com.homelibrary.book.BookSpecification` — dinamikus Specification factory
- DTO-k (ugyanabban a package-ben):
  - `BookCreateRequest` — POST body; tartalmazza `subtitle` (nullable String) és `pageCount` (nullable Integer) mezőket az OSZK response-szal való paritás érdekében
  - `BookUpdateRequest` — PUT body (BookCreateRequest + `version`)
  - `BookSearchParams` — GET /api/books query paraméterek
  - `BookResponse` — egységes válasz lista és detail nézethez; tartalmazza `subtitle` és `pageCount` mezőket
  - `BookLocationResponse` — nested location (id, name, room)
  - `BookRoomResponse` — nested room (id, name)
- `com.homelibrary.book.BookServiceTest` — unit tesztek

**Mellékhatás — LocationService tech-debt lezárása:** a `books` tábla most már létezik, a Feature 3-ban elhalasztott két feladat itt oldandó meg:
- `LocationService.softDelete`: aktív könyv ellenőrzés — ha van `active = true` könyv a locationhöz → 409
- `LocationService` listázás: `bookCount` számított értékként (GROUP BY query, nem hardcoded 0)

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
- Ha `description` és `descriptionLanguage` megadva → `BookDescription` létrehozása `ORIGINAL` source-szal

### `getById(UUID id) → BookResponse`
- Nem található → 404
- Leírás lekérése: authenticated user `preferred_language`-je a `SecurityContextHolder`-ből, az adott nyelvű `BookDescription` keresése → ha nincs, bármely elérhető (első) → ha nincs leírás, `description: null`

### `search(BookSearchParams params, Pageable pageable) → Page<BookResponse>`
- `BookSpecification` alapján dinamikus szűrés
- Leírás minden könyvhöz: az authenticated user `preferred_language`-je alapján egyetlen query-vel (`BookDescriptionRepository.findAllByBookIdIn(ids, language)`) — N+1 elkerülése
- `authors` és `categories`: JSON string → `List<String>` (ObjectMapper)

### `update(UUID id, BookUpdateRequest request) → BookResponse`
- Nem található → 404
- `locationId` változás esetén: új location létezés + aktív ellenőrzés → 404 ha nem elérhető
- `ObjectOptimisticLockingFailureException` → 409 Conflict
- Ha `description` és `descriptionLanguage` megadva → `BookDescription` upsert: `findByBookIdAndLanguage` → létezik: frissítés, nem létezik: létrehozás

### `softDelete(UUID id) → void`
- Nem található → 404
- `status` → `DELETED`, `deletedAt` → `OffsetDateTime.now(ZoneOffset.UTC)`

---

## Kulcs döntések

- **`authors` / `categories` konverzió:** Jackson `ObjectMapper` a service rétegben — entity `String` ↔ DTO `List<String>`. Nem `@Converter` az entitáson, mert a `BookSpecification` LIKE predikátumnak a nyers VARCHAR oszlopra kell hivatkoznia
- **Leírás listázáskor:** egyetlen `findAllByBookIdIn` hívás az egész laphoz, nem N+1 — a `BookDescriptionRepository`-ban custom query metódus szükséges
- **Törölt könyvek szűrése:** `status != DELETED` mindig be van égetve a `BookSpecification`-be — törölt könyv semmilyen felhasználónak nem jelenik meg a listában
- **`version` benne van a `BookResponse`-ban** — az update form-nak szüksége van rá az optimistic locking miatt

---

## Unit tesztek (`BookServiceTest`)

| Teszt | Mit ellenőriz |
|-------|---------------|
| `create_validRequest_savesBookAndDescription` | book + description létrejön |
| `create_invalidLocationId_throws404` | nem létező location → 404 |
| `update_optimisticLockConflict_throws409` | verziószám ütközés → 409 |
| `softDelete_setsStatusAndDeletedAt` | status DELETED, deletedAt kitöltött |
| `search_withStatusFilter_returnsFilteredResults` | Specification szűr |
| `authors_serializationRoundtrip` | List<String> → JSON String → List<String> |
