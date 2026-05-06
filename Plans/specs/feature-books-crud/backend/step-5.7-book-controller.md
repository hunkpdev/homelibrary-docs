# Step 5.7 – BookController: CRUD endpointok

## Mit állít elő

- `com.homelibrary.book.BookController` — `GET /api/books`, `GET /api/books/{id}`, `POST`, `PUT /{id}`, `DELETE /{id}`
- `BookControllerTest` — unit tesztek (MockMvc)

> `GET /api/books/isbn/{isbn}` az `IsbnLookupController`-ben van (Feature 4) — nem ütközik, Spring a literal `isbn` szegmenst előnyben részesíti az `{id}` path variable-lal szemben.

---

## Request DTO validáció

### `BookCreateRequest`

| Mező | Típus | Validáció |
|------|-------|-----------|
| `title` | `String` | `@NotBlank`, `@Size(max = 500)` |
| `subtitle` | `String` | `@Size(max = 500)`, nullable |
| `isbn` | `String` | `@Size(max = 20)`, nullable |
| `authors` | `List<String>` | nullable |
| `publisher` | `String` | `@Size(max = 255)`, nullable |
| `publishYear` | `Integer` | `@Min(1000)`, `@Max(2200)`, nullable |
| `pageCount` | `Integer` | `@Min(0)`, nullable |
| `language` | `String` | `@Size(max = 10)`, nullable |
| `categories` | `List<String>` | nullable |
| `locationId` | `UUID` | nullable |
| `status` | `BookStatus` | nullable (service default: `AT_HOME`) |
| `source` | `BookSource` | nullable |
| `description` | `String` | nullable |
| `descriptionLanguage` | `String` | `@Size(max = 10)`, nullable |

> Ha `description` megadott de `descriptionLanguage` null → 400; a service elvárja mindkettőt együtt. Cross-field validáció `@AssertTrue` metódussal a DTO-n.

### `BookUpdateRequest`

Megegyezik `BookCreateRequest`-tel, kiegészítve:

| Mező | Típus | Validáció |
|------|-------|-----------|
| `version` | `Long` | `@NotNull` |

---

## Endpointok

### `GET /api/books`
- Jogosultság: `ADMIN`, `VISITOR`, `DEMO`
- Query paraméterek: `search`, `status`, `locationId`, `category`, `language`, `publishYear`, `page` (default: 0), `size` (default: 20, max: 100), `sort`
- Response 200: `Page<BookResponse>`

### `GET /api/books/{id}`
- Jogosultság: `ADMIN`, `VISITOR`, `DEMO`
- Response 200: `BookResponse`
- Response 404: könyv nem található

### `POST /api/books`
- Jogosultság: `ADMIN`
- Request: `@Valid BookCreateRequest`
- `addedBy`: az authenticated principal UUID-je (`@AuthenticationPrincipal` vagy `SecurityContextHolder`)
- Response 201: `BookResponse`
- Response 400: validációs hiba
- Response 404: location nem található

### `PUT /api/books/{id}`
- Jogosultság: `ADMIN`
- Request: `@Valid BookUpdateRequest`
- Response 200: `BookResponse` (növelt `version`)
- Response 400: validációs hiba
- Response 404: könyv nem található
- Response 409: optimistic locking ütközés

### `DELETE /api/books/{id}`
- Jogosultság: `ADMIN`
- Response 204: No content
- Response 404: könyv nem található

---

## Kulcs döntések

- `@PreAuthorize` annotációk jogosultság kezeléshez
- `@Operation` és `@ApiResponse` annotációk minden endpointon (springdoc-openapi)
- `@Valid` minden request DTO-n
- `size` max 100 — a controller validálja; felette 400

---

## Unit tesztek (`BookControllerTest`, MockMvc)

| Teszt | Elvárt |
|-------|--------|
| `GET /api/books` VISITOR tokennel | 200 |
| `GET /api/books` DEMO tokennel | 200 |
| `GET /api/books/{id}` DEMO tokennel | 200 |
| `POST /api/books` VISITOR tokennel | 403 |
| `POST /api/books` DEMO tokennel | 403 |
| `POST /api/books` hiányzó `title` | 400 |
| `POST /api/books` `description` megadva, `descriptionLanguage` null | 400 |
| `PUT /api/books/{id}` VISITOR tokennel | 403 |
| `DELETE /api/books/{id}` VISITOR tokennel | 403 |
