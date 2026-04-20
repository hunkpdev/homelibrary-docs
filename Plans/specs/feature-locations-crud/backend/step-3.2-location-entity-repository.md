# Step 3.2 – Location entitás és repository

## Mit állít elő

- `backend/src/main/java/.../location/Location.java` — JPA entitás
- `backend/src/main/java/.../location/LocationRepository.java` — Spring Data JPA repository

---

## Kulcs döntések

**Location entitás:**
- Mezők: `id` (UUID), `roomName`, `shelfName`, `description`, `active`, `version`, `createdAt`, `updatedAt`
- `@Version` annotáció a `version` mezőn — optimistic locking (konzisztens a `User` entitással)
- `@CreationTimestamp` / `@UpdateTimestamp` a timestamp mezőkön

**LocationRepository:**
- Kiterjeszti a `JpaSpecificationExecutor<Location>` interfészt a dinamikus szűréshez
- `findAll(Specification<Location> spec, Pageable pageable)` — az interfészből jön automatikusan

---

## Elfogadási kritériumok

- Az entitás leképezhető a `locations` táblára (nincs JPA indítási hiba)
- `findAll(spec, pageable)` meghívható (nincs fordítási hiba)
