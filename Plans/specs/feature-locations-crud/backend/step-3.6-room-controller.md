# Step 3.6 – RoomController

## Mit állít elő

- `RoomController` — `GET /api/rooms`, `POST /api/rooms`, `PUT /api/rooms/{id}`, `DELETE /api/rooms/{id}`
- `RoomRequest` — request DTO (létrehozás és módosítás)
- `RoomResponse` — response DTO
- `RoomServiceTest` — unit teszt
- `RoomControllerTest` — unit teszt (MockMvc)

---

## `RoomRequest` DTO

| Mező | Típus | Validáció |
|------|-------|-----------|
| `name` | `String` | `@NotBlank` |
| `description` | `String` | — |
| `version` | `Long` | — (csak PUT-nál kötelező, de egységes DTO) |

---

## `RoomResponse` DTO

| Mező | Típus | Leírás |
|------|-------|--------|
| `id` | `UUID` | |
| `name` | `String` | |
| `description` | `String` | |
| `locationCount` | `int` | Aktív locationök száma |
| `version` | `Long` | Optimistic locking |

---

## Endpointok

### `GET /api/rooms`
- Jogosultság: `ADMIN`, `VISITOR`
- Query paraméterek: `name` (opcionális, részleges egyezés), `page` (default: 0), `size` (default: 20), `sort` (default: `name,asc`)
- Response 200: `Page<RoomResponse>`

### `POST /api/rooms`
- Jogosultság: `ADMIN`
- Request: `RoomRequest` (`version` mező figyelmen kívül hagyva)
- Response 201: `RoomResponse`
- Response 400: validációs hiba

### `PUT /api/rooms/{id}`
- Jogosultság: `ADMIN`
- Request: `RoomRequest` (`version` kötelező az optimistic lockinghoz)
- Response 200: `RoomResponse` (növelt `version`)
- Response 404: room nem található
- Response 409: optimistic locking ütközés

### `DELETE /api/rooms/{id}`
- Jogosultság: `ADMIN`
- Response 204: No content
- Response 404: room nem található
- Response 409: van aktív location a roomhoz rendelve

---

## Kulcs döntések

- `@PreAuthorize` annotációk a jogosultság kezeléshez
- `@Operation` és `@ApiResponse` annotációk minden endpointon (springdoc-openapi)
- `@Valid` a request DTO-kon

---

## Elfogadási kritériumok

**Unit tesztek** (`RoomServiceTest`):
- Listázás szűrő nélkül az összes aktív roomot visszaadja
- Törlés aktív locationnel rendelkező roomon 409-et dob
- Optimistic locking ütközés 409-et dob

**Unit tesztek** (`RoomControllerTest`, MockMvc):
- `GET /api/rooms` VISITOR tokennel → 200
- `POST /api/rooms` VISITOR tokennel → 403
- `POST /api/rooms` hiányzó kötelező mezővel → 400
- `DELETE /api/rooms/{id}` aktív locationnel → 409

**Manuálisan:**
- Swagger UI-on mind a négy endpoint dokumentált és elérhető
