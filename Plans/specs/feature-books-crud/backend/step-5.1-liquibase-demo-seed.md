# Step 5.1 – Liquibase: demo user seed

## Mit állít elő

- `db/changelog/changes/009-seed-demo-user.yaml` — beégetett demo user seed changeset
- bekötve a `db.changelog-master.yaml`-ba (a feature 4 changeset, 008 után)

---

## Kulcs döntések

- Jelszó: `"demo"` — nyilvános, szándékos (portfólió célra, DEMO role read-only)
- BCrypt hash cost factor: 12 (konzisztens az admin seed-del, step 1.3)
- Hash **beégetve** a changesetbe — nincs env var, nincs paraméteres szubsztitúció (szemben az admin seed-del)
- Fix UUID: `00000000-0000-0000-0000-000000000002`
- `preferred_language`: `en` (portfólió bemutató célú fiók)
- Minden profilon lefut (`local` és `prod` egyaránt)

---

## Elfogadási kritériumok

- App indításkor Liquibase lefuttatja a changesetet hibák nélkül (`local` és `prod` profilon)
- `demo` felhasználó bekerül a `users` táblába `DEMO` role-lal
- `demo` / `"demo"` jelszópárral be lehet jelentkezni az alkalmazásba
