# Step 3.8 – LocationController

## Mit állít elő

- `LocationController` — `GET /api/locations`, `POST /api/locations`, `PUT /api/locations/{id}`, `DELETE /api/locations/{id}`
- `CreateLocationRequest` — request DTO létrehozáshoz
- `UpdateLocationRequest` — request DTO módosításhoz
- `LocationResponse` — response DTO
- `LocationServiceTest` — unit teszt
- `LocationControllerTest` — unit teszt (MockMvc)

---

## `CreateLocationRequest` DTO

| Mező | Típus | Validáció |
|------|-------|-----------|
| `name` | `String` | `@NotBlank` + `@Size(max = 100)` |
| `roomId` | `UUID` | `@NotNull` |
| `description` | `String` | `@Size(max = 2000)` |

---

## `UpdateLocationRequest` DTO

| Mező | Típus | Validáció |
|------|-------|-----------|
| `name` | `String` | `@NotBlank` + `@Size(max = 100)` |
| `version` | `Long` | `@NotNull` |
| `description` | `String` | `@Size(max = 2000)` |

> `roomId` szándékosan hiányzik — location nem helyezhető át másik roomba (lásd ADR-007)

---

## `LocationResponse` DTO

| Mező | Típus | Leírás |
|------|-------|--------|
| `id` | `UUID` | |
| `name` | `String` | |
| `description` | `String` | |
| `room` | `RoomResponse` | Beágyazott room objektum |
| `bookCount` | `int` | Feature 3-ban hardcoded `0` — Feature 5-ben bővítendő (tech-debt) |
| `version` | `Long` | Optimistic locking |

---

## Endpointok

### `GET /api/locations`
- Jogosultság: `ADMIN`, `VISITOR`
- Query paraméterek: `name`, `roomId` (mind opcionális), `page` (default: 0), `size` (default: 20), `sort` (default: `name,asc`)
- Response 200: `Page<LocationResponse>`

### `GET /api/locations/all`
- Jogosultság: `ADMIN`, `VISITOR`
- Query paraméterek: nincs
- Response 200: `List<LocationResponse>` — összes aktív location, `name ASC` sorrendben, lapozás nélkül
- Kizárólag frontend dropdown feltöltésére

### `POST /api/locations`
- Jogosultság: `ADMIN`
- Request: `CreateLocationRequest`
- Response 201: `LocationResponse`
- Response 400: validációs hiba
- Response 404: room nem található

### `PUT /api/locations/{id}`
- Jogosultság: `ADMIN`
- Request: `UpdateLocationRequest`
- Response 200: `LocationResponse` (növelt `version`)
- Response 404: location nem található
- Response 409: optimistic locking ütközés

### `DELETE /api/locations/{id}`
- Jogosultság: `ADMIN`
- Response 204: No content
- Response 404: location nem található
- Response 409: Feature 5-ben kerül be (book check stub, lásd step 3.7)

---

## Kulcs döntések

- `@PreAuthorize` annotációk a jogosultság kezeléshez
- `@Operation` és `@ApiResponse` annotációk minden endpointon (springdoc-openapi)
- `@Valid` a request DTO-kon

---

## Elfogadási kritériumok

**Unit tesztek** (`LocationServiceTest`):
- Listázás szűrők nélkül az összes aktív locationt visszaadja
- `roomId` szűrővel csak az adott roomhoz tartozó locationök jelennek meg
- Optimistic locking ütközés 409-et dob

**Unit tesztek** (`LocationControllerTest`, MockMvc):
- `GET /api/locations` VISITOR tokennel → 200
- `GET /api/locations/all` VISITOR tokennel → 200, `List` típusú válasz
- `POST /api/locations` VISITOR tokennel → 403
- `POST /api/locations` hiányzó kötelező mezővel → 400
- `PUT /api/locations/{id}` `roomId` megadásával → 400 (mező nem létezik a DTO-ban, deszérializáció hibát dob)

**Manuálisan:**
- Swagger UI-on POST és PUT külön request body-val dokumentált
