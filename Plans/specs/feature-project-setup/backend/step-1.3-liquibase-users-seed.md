# Step 1.3 – Liquibase setup: users tábla + admin seed

## Mit állít elő

- `src/main/resources/db/changelog/db.changelog-master.yaml` — master changelog
- `src/main/resources/db/changelog/changes/001-create-users.yaml` — `users` tábla létrehozása
- `src/main/resources/db/changelog/changes/002-seed-admin-user.yaml` — default admin user seed
- `local-env.example` — szükséges environment variable-ok dokumentációja (projektrootban)

Mindkét changeset minden profilon lefut (local és prod egyaránt).

---

## Fájlstruktúra

```
src/main/resources/
└── db/
    └── changelog/
        ├── db.changelog-master.yaml
        └── changes/
            ├── 001-create-users.yaml
            └── 002-seed-admin-user.yaml

local-env.example          ← commitolva, placeholder értékekkel
```

---

## Admin jelszó kezelési stratégia

A BCrypt hash **nem kerül a repóba** — Liquibase paraméter-szubsztitúción keresztül environment variable-ból érkezik.

| Környezet | Honnan jön az `ADMIN_PASSWORD_HASH` |
|-----------|-------------------------------------|
| Local | IntelliJ run config → Environment Variables |
| Prod (Lambda) | SSM Parameter Store `/homelibrary/admin-password-hash` → CDK injektálja env var-ként (ld. step 1.12) |

**Hash generálása lokálisan** — ugyanazzal az osztállyal, amit az alkalmazás is használ:
```java
new BCryptPasswordEncoder(12).encode("jelszó-ide")
```

---

## local-env.example

```bash
# Szükséges environment variable-ok lokális fejlesztéshez.
# Másold le local-env.sh vagy állítsd be IntelliJ run configban.
# Ez a fájl NEM tartalmaz valós értékeket — ne commitolj valós értékeket!

# BCrypt hash az admin user seed changesethez (cost factor: 12)
# Generálás: htpasswd -bnBC 12 "" <jelszó> | tr -d ':\n'
ADMIN_PASSWORD_HASH=<bcrypt-hash-ide>
```

---

## application.properties kiegészítés

A közös `application.properties`-be kerül (az 1.2-es specben lévők mellé):

```properties
# Liquibase paraméterek
spring.liquibase.parameters.adminPasswordHash=${ADMIN_PASSWORD_HASH}
```

Ha `ADMIN_PASSWORD_HASH` nincs beállítva, Spring Boot induláskor hibával leáll — szándékos fail-fast.

---

## db.changelog-master.yaml

```yaml
databaseChangeLog:
  - include:
      file: db/changelog/changes/001-create-users.yaml
  - include:
      file: db/changelog/changes/002-seed-admin-user.yaml
```

---

## 001-create-users.yaml

```yaml
databaseChangeLog:
  - changeSet:
      id: 001-create-users
      author: homelibrary
      changes:
        - createTable:
            tableName: users
            columns:
              - column:
                  name: id
                  type: UUID
                  constraints:
                    primaryKey: true
                    nullable: false
              - column:
                  name: username
                  type: VARCHAR(50)
                  constraints:
                    unique: true
                    nullable: false
              - column:
                  name: email
                  type: VARCHAR(255)
                  constraints:
                    unique: true
                    nullable: false
              - column:
                  name: password_hash
                  type: VARCHAR(255)
                  constraints:
                    nullable: false
              - column:
                  name: preferred_language
                  type: VARCHAR(10)
                  constraints:
                    nullable: false
              - column:
                  name: role
                  type: VARCHAR(20)
                  constraints:
                    nullable: false
              - column:
                  name: active
                  type: BOOLEAN
                  defaultValueBoolean: true
                  constraints:
                    nullable: false
              - column:
                  name: refresh_token_hash
                  type: VARCHAR(255)
              - column:
                  name: refresh_token_expires_at
                  type: TIMESTAMP WITH TIME ZONE
              - column:
                  name: version
                  type: BIGINT
                  defaultValueNumeric: 0
                  constraints:
                    nullable: false
              - column:
                  name: created_at
                  type: TIMESTAMP WITH TIME ZONE
                  constraints:
                    nullable: false
              - column:
                  name: updated_at
                  type: TIMESTAMP WITH TIME ZONE
                  constraints:
                    nullable: false
```

---

## 002-seed-admin-user.yaml

```yaml
databaseChangeLog:
  - changeSet:
      id: 002-seed-admin-user
      author: homelibrary
      changes:
        - insert:
            tableName: users
            columns:
              - column:
                  name: id
                  value: 00000000-0000-0000-0000-000000000001
              - column:
                  name: username
                  value: admin
              - column:
                  name: email
                  value: admin@homelibrary.local
              - column:
                  name: password_hash
                  value: ${adminPasswordHash}
              - column:
                  name: preferred_language
                  value: hu
              - column:
                  name: role
                  value: ADMIN
              - column:
                  name: active
                  valueBoolean: true
              - column:
                  name: version
                  valueNumeric: 0
              - column:
                  name: created_at
                  valueComputed: NOW()
              - column:
                  name: updated_at
                  valueComputed: NOW()
```

---

## Elfogadási kritériumok

- `ADMIN_PASSWORD_HASH` env var nélkül az alkalmazás nem indul el (fail-fast)
- `local` profilon az `ADMIN_PASSWORD_HASH` env var-ral az alkalmazás elindul, Liquibase lefuttatja mindkét changesetet (log: `Running Changeset: ...`)
- `users` tábla létrejön HSQLDB-ben a séma szerint
- `admin` felhasználó bekerül a táblába `ADMIN` szerepkörrel, a megadott jelszó hash-sel
- Ismételt indításkor a Liquibase nem futtatja újra a changeseteket (`DATABASECHANGELOG` tábla alapján)
- `local-env.example` commitolva van, valós hash értéket nem tartalmaz
