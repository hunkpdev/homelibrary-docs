# Step 3.3 – Room entitás és repository

## Mit állít elő

- `Room.java` — JPA entitás
- `RoomRepository.java` — Spring Data JPA repository

---

## Kulcs döntések

**Room entitás:**
- Mezők: `id` (UUID), `name`, `description`, `active`, `version`, `createdAt`, `updatedAt`
- `@Version` annotáció a `version` mezőn — optimistic locking
- `@CreationTimestamp` / `@UpdateTimestamp` a timestamp mezőkön

**RoomRepository:**
- Kiterjeszti a `JpaSpecificationExecutor<Room>` interfészt a dinamikus szűréshez
- `findAll(Specification<Room> spec, Pageable pageable)` — az interfészből jön automatikusan

---

## Elfogadási kritériumok

- Az entitás leképezhető a `rooms` táblára (nincs JPA indítási hiba)
- `findAll(spec, pageable)` meghívható (nincs fordítási hiba)
