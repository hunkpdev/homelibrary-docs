# Step 5.2 – Spring Security: DEMO role globális HTTP method rule-ok

## Mit állít elő

- `SecurityConfig` módosítása (step 2.5-ben jött létre):
  - `@EnableMethodSecurity` hozzáadva
  - Authorization rule-ok kibővítve DEMO-tudatos szabályokkal

---

## Authorization rule struktúra

A jelenlegi `anyRequest().authenticated()` helyett az alábbi sorrendben értékelődnek ki a szabályok:

| Endpoint / metódus | Engedélyezett role-ok |
|---|---|
| Auth + health + Swagger (meglévők) | `permitAll` — változatlan |
| `GET /api/users/**`, `PUT /api/users/**` | `ADMIN`, `VISITOR` — DEMO statikusan kizárva |
| minden más `/api/**` `POST`, `PUT`, `PATCH`, `DELETE` | `ADMIN` only |
| minden más `/api/**` `GET`, `HEAD`, `OPTIONS` | `ADMIN`, `VISITOR`, `DEMO` |

**Role hierarchy kihasználása:** a meglévő `ADMIN > VISITOR` hierarchia miatt `hasRole("VISITOR")` kielégíti az ADMIN-t is, DEMO-t nem — ez adja a `/api/users/**` DEMO-kizárást anélkül, hogy explicit DEMO-blokk rule-t kellene felvenni.

**`/api/users/**` finomítás:** a SecurityConfig csak a belépést engedi (ADMIN vagy VISITOR) — az ownership check (VISITOR csak saját rekordját módosíthatja) method-level `@PreAuthorize`-zal történik a UserControllerben (Feature 7).

---

## Kulcs döntések

- `@EnableMethodSecurity` a `SecurityConfig`-on — a Feature 7 UserController `@PreAuthorize` ownership check-jeihez szükséges
- Whitelist forma: GET/HEAD/OPTIONS-re explicit role-lista, mutáló metódusokra ADMIN-only — nincs külön DEMO-blokk rule
- DEMO role a role hierarchy-n kívül marad (nem kerül be az `ADMIN > VISITOR` láncba)

---

## Elfogadási kritériumok

- DEMO tokennel `GET /api/**` (users kivételével) → `200`
- DEMO tokennel `POST /api/**` → `403`
- DEMO tokennel `GET /api/users/**` → `403`
- VISITOR tokennel `GET /api/users/**` → `200` (controller engedi tovább, ownership check ott)
- ADMIN tokennel minden endpoint → változatlan működés
- Meglévő auth + health + Swagger permit szabályok sértetlenek
