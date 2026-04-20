# Step 3.4 – LocationController

## Mit állít elő

- `LocationController` — `GET /api/locations`, `POST /api/locations`, `PUT /api/locations/{id}`, `DELETE /api/locations/{id}`
- `LocationRequest` — request DTO (létrehozás és módosítás)
- `LocationResponse` — response DTO
- `LocationServiceTest` — unit teszt
- `LocationControllerTest` — unit teszt (MockMvc)

---

## `LocationRequest` DTO

| Mező | Típus | Validáció |
|------|-------|-----------|
| `roomName` | `String` | `@NotBlank` |
| `shelfName` | `String` | `@NotBlank` |
| `description` | `String` | — |
| `version` | `Long` | — (csak PUT-nál kötelező, de egységes DTO) |

---

## `LocationResponse` DTO

| Mező | Típus | Leírás |
|------|-------|--------|
| `id` | `UUID` | |
| `roomName` | `String` | |
| `shelfName` | `String` | |
| `description` | `String` | |
| `bookCount` | `int` | Aktív (nem `DELETED`) könyvek száma |
| `version` | `Long` | Optimistic locking |

---

## Endpointok

### `GET /api/locations`
- Jogosultság: `ADMIN`, `VISITOR`
- Query paraméterek: `roomName`, `shelfName`, `description` (mind opcionális, részleges egyezés), `page` (default: 0), `size` (default: 20), `sort` (default: `roomName,asc`)
- Response 200: `Page<LocationResponse>`

### `POST /api/locations`
- Jogosultság: `ADMIN`
- Request: `LocationRequest` (`version` mező figyelmen kívül hagyva)
- Response 201: `LocationResponse`
- Response 400: validációs hiba

### `PUT /api/locations/{id}`
- Jogosultság: `ADMIN`
- Request: `LocationRequest` (`version` kötelező az optimistic lockinghoz)
- Response 200: `LocationResponse` (növelt `version`)
- Response 404: location nem található
- Response 409: optimistic locking ütközés

### `DELETE /api/locations/{id}`
- Jogosultság: `ADMIN`
- Response 204: No content
- Response 404: location nem található
- Response 409: van aktív könyv a locationhöz rendelve

---

## Kulcs döntések

- `@PreAuthorize` annotációk a jogosultság kezeléshez (konzisztens a Feature 2 security config-jával)
- `@Operation` és `@ApiResponse` annotációk minden endpointon (springdoc-openapi)
- `@Valid` a request DTO-kon

---

## Elfogadási kritériumok

**Unit tesztek** (`LocationServiceTest`):
- Listázás szűrők nélkül az összes aktív locationt visszaadja
- Több szűrő egyszerre kombinálható
- Törlés aktív könyvvel rendelkező locationon 409-et dob
- Optimistic locking ütközés 409-et dob

**Unit tesztek** (`LocationControllerTest`, MockMvc):
- `GET /api/locations` VISITOR tokennel → 200
- `POST /api/locations` VISITOR tokennel → 403
- `POST /api/locations` hiányzó kötelező mezővel → 400
- `DELETE /api/locations/{id}` aktív könyvvel → 409

**Manuálisan:**
- Swagger UI-on mind a négy endpoint dokumentált és elérhető
