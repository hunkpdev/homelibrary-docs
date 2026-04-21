# Step 3.3 – Room entitás és repository

## Mit állít elő

- `Room.java` — JPA entitás
- `RoomRepository.java` — Spring Data JPA repository

---

## Kulcs döntések

**Room entitás:**
- Mezők: `id` (UUID), `name`, `description`, `active`, `version`, `createdAt`, `updatedAt`
- `@Version` annotáció a `version` mezőn — optimistic locking
- `@PrePersist` / `@PreUpdate` lifecycle callback-ek a timestamp mezőkön, explicit UTC timezone-nal (lásd ADR-008)
- **Nincs `@OneToMany` collection** a Location felé — unidirectional kapcsolat (Location → Room), a `locationCount` aggregation query-vel számított (lásd step 3.5)

**RoomRepository:**
- Kiterjeszti a `JpaSpecificationExecutor<Room>` interfészt a dinamikus szűréshez
- `findAll(Specification<Room> spec, Pageable pageable)` — az interfészből jön automatikusan

---

## Elfogadási kritériumok

- Az entitás leképezhető a `rooms` táblára (nincs JPA indítási hiba)
- `findAll(spec, pageable)` meghívható (nincs fordítási hiba)
