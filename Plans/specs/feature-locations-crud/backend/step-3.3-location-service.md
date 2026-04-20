# Step 3.3 – LocationService

## Mit állít elő

- `LocationService`: üzleti logika — listázás, létrehozás, módosítás, soft delete

---

## Metódusok

### Listázás
- Bemenő paraméterek: `roomName`, `shelfName`, `description` szűrők (mind opcionális) + `Pageable` (rendezés + lapozás)
- A Specification dinamikusan épül fel: csak a nem null szűrők kerülnek be (`LIKE %value%` jellegű feltételek)
- `active = true` feltétel mindig része a Specification-nek
- Default sort: `roomName ASC` — ha a hívó nem ad meg rendezést, ez az alapértelmezett
- `bookCount` kiszámítása: az aktív (nem `DELETED` státuszú) könyvek száma locationnként — count query a `books` táblán, `location_id` alapján

### Létrehozás
- Bemenő: `roomName` (kötelező), `shelfName` (kötelező), `description` (opcionális)
- `active = true`, `createdAt`, `updatedAt` automatikusan kerül beállításra

### Módosítás
- Bemenő: `id`, `roomName`, `shelfName`, `description`, `version`
- Optimistic locking: a `version` mező ütközés esetén `ObjectOptimisticLockingFailureException` → 409 Conflict

### Soft delete
- Bemenő: `id`
- Ellenőrzés: ha van aktív (nem `DELETED` státuszú) könyv a locationhöz rendelve → üzleti kivétel → 409 Conflict
- Sikeres esetben: `active = false`

---

## Elfogadási kritériumok

- Listázás üres szűrőkkel az összes aktív locationt visszaadja `roomName ASC` sorrendben
- Több szűrő egyszerre kombinálható
- Törlés aktív könyvvel rendelkező locationon 409-et dob
- Optimistic locking ütközés 409-et dob
