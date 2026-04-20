# Step 3.5 – RoomService

## Mit állít elő

- `RoomService`: üzleti logika — listázás, létrehozás, módosítás, soft delete

---

## Metódusok

### Listázás
- Bemenő paraméterek: `name` szűrő (opcionális) + `Pageable` (rendezés + lapozás)
- A Specification dinamikusan épül fel: csak a nem null szűrők kerülnek be (`LIKE %value%`)
- `active = true` feltétel mindig része a Specification-nek
- Default sort: `name ASC` — ha a hívó nem ad meg rendezést, ez az alapértelmezett
- `locationCount` kiszámítása: egyetlen GROUP BY query-vel (`LEFT JOIN locations ON room_id AND active = true`, `GROUP BY room.id`) — N+1 elkerülése érdekében JPA projection használandó

### Létrehozás
- Bemenő: `name` (kötelező), `description` (opcionális)
- `active = true`, `createdAt`, `updatedAt` automatikusan kerül beállításra

### Módosítás
- Bemenő: `id`, `name`, `description`, `version`
- Optimistic locking: `version` ütközés esetén `ObjectOptimisticLockingFailureException` → 409 Conflict

### Soft delete
- Bemenő: `id`
- Ellenőrzés: ha van aktív (`active = true`) location a roomhoz rendelve → üzleti kivétel → 409 Conflict
- Sikeres esetben: `active = false`

---

## Elfogadási kritériumok

- Listázás üres szűrővel az összes aktív roomot visszaadja `name ASC` sorrendben
- Törlés aktív locationnel rendelkező roomon 409-et dob
- Optimistic locking ütközés 409-et dob
