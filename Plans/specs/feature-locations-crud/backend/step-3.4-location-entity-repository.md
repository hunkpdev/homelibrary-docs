# Step 3.4 – Location entitás és repository

## Mit állít elő

- `Location.java` — JPA entitás
- `LocationRepository.java` — Spring Data JPA repository

---

## Kulcs döntések

**Location entitás:**
- Mezők: `id` (UUID), `name`, `description`, `room` (OneToOne FK → Room), `active`, `version`, `createdAt`, `updatedAt`
- `@OneToOne` kapcsolat a `Room` entitásra (`room_id` FK, NOT NULL — lásd ADR-007)
- `@Version` annotáció a `version` mezőn — optimistic locking
- `@CreationTimestamp` / `@UpdateTimestamp` a timestamp mezőkön

**LocationRepository:**
- Kiterjeszti a `JpaSpecificationExecutor<Location>` interfészt a dinamikus szűréshez
- `findAll(Specification<Location> spec, Pageable pageable)` — az interfészből jön automatikusan

---

## Elfogadási kritériumok

- Az entitás leképezhető a `locations` táblára (nincs JPA indítási hiba)
- `findAll(spec, pageable)` meghívható (nincs fordítási hiba)
