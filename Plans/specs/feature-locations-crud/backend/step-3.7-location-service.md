# Step 3.7 – LocationService

## Mit állít elő

- `LocationService`: üzleti logika — listázás, létrehozás, módosítás, soft delete

---

## Metódusok

### Listázás
- Bemenő paraméterek: `name`, `roomId` szűrők (mind opcionális) + `Pageable` (rendezés + lapozás)
- A Specification dinamikusan épül fel: csak a nem null szűrők kerülnek be (`LIKE %value%` / UUID egyezés)
- `active = true` feltétel mindig része a Specification-nek
- Default sort: `name ASC` — ha a hívó nem ad meg rendezést, ez az alapértelmezett
- `bookCount` kiszámítása: az aktív (nem `DELETED` státuszú) könyvek száma locationnként — count query a `books` táblán, `location_id` alapján

### Létrehozás
- Bemenő: `name` (kötelező), `roomId` (kötelező), `description` (opcionális)
- Room létezésének ellenőrzése `roomId` alapján → nem található esetén 404
- `active = true`, `createdAt`, `updatedAt` automatikusan kerül beállításra

### Módosítás
- Bemenő: `id`, `name`, `description`, `version` (`roomId` nem módosítható — lásd ADR-007)
- Optimistic locking: `version` ütközés esetén `ObjectOptimisticLockingFailureException` → 409 Conflict

### Soft delete
- Bemenő: `id`
- Ellenőrzés: ha van aktív (nem `DELETED` státuszú) könyv a locationhöz rendelve → üzleti kivétel → 409 Conflict
- Sikeres esetben: `active = false`

---

## Elfogadási kritériumok

- Listázás üres szűrőkkel az összes aktív locationt visszaadja `name ASC` sorrendben
- `roomId` szűrővel csak az adott roomhoz tartozó locationök jelennek meg
- Létrehozás nem létező `roomId`-val 404-et dob
- Törlés aktív könyvvel rendelkező locationon 409-et dob
- Optimistic locking ütközés 409-et dob
