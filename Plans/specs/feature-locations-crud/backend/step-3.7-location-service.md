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
- `bookCount`: Feature 3-ban hardcoded `0` — a `books` tábla még nem létezik. Feature 5 (step 5.4) után bővítendő egyetlen GROUP BY query-vel (N+1 elkerülése érdekében JPA projection) — tech-debt backlog

### Létrehozás
- Bemenő: `name` (kötelező), `roomId` (kötelező), `description` (opcionális)
- Room létezésének ellenőrzése `roomId` alapján → nem található esetén 404
- `active = true`, `createdAt`, `updatedAt` automatikusan kerül beállításra

### Módosítás
- Bemenő: `id`, `name`, `description`, `version` (`roomId` nem módosítható — lásd ADR-007)
- Optimistic locking: `version` ütközés esetén `ObjectOptimisticLockingFailureException` → 409 Conflict

### Soft delete
- Bemenő: `id`
- Ellenőrzés: Feature 3-ban kihagyva — a `books` tábla még nem létezik. Feature 5 (step 5.4) után bővítendő (tech-debt backlog)
- Sikeres esetben: `active = false`

---

## Elfogadási kritériumok

- Listázás üres szűrőkkel az összes aktív locationt visszaadja `name ASC` sorrendben
- `roomId` szűrővel csak az adott roomhoz tartozó locationök jelennek meg
- Létrehozás nem létező `roomId`-val 404-et dob
- Törlés bármely locationon sikeres (book check Feature 5-ben kerül be)
- Optimistic locking ütközés 409-et dob
