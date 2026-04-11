# 1. Fázis – Implementációs terv

> **Státusz:** Tervezés alatt
> **Utolsó frissítés:** 2026-04-04
> **Hatókör:** MVP – az első működő, deployolható alkalmazáshoz szükséges core feature-ök

## Fejlesztési workflow

Minden feature vertikálisan (teljes stack egyszerre) kerül implementálásra, stepekre bontva.

**Stepenként:**
1. Rövid egyeztetés (mit produce-ol a step, mely osztályok/interfészek érintettek)
2. Claude Code megírja a kódot + unit teszteket (Clean Code + OWASP elvek)
3. Lead dev review + SonarLint ellenőrzés
4. Findingok javítása ha van → vissza a 3. pontra
5. Unit tesztek futtatása lokálban
6. Commit + push

**Feature lezárásakor (minden step elkészülte után):**
- CI ellenőrzés (GitHub Actions)
- Integrációs tesztek ahol releváns (`@SpringBootTest`: auth flow, könyv státuszátmenetek)
- Manuális GUI teszt

**Branch stratégia:** `feature/<name>` branch → merge commit a main-re PR-en keresztül

---

## Feature sorrend és függőségek

```
1. Project Setup
       │
       ▼
2. Auth ──────────────────────────────────────────┐
       │                                          │
       ▼                                          │
3. Locations CRUD                                 │
       │                                          │
       ▼                                          │
4. ISBN Lookup                                    │
       │                                          │
       ▼                                          │
5. Books CRUD ← függ: Locations + Auth + ISBN     │
       │                                          │
       ▼                                          │
6. Loans ← függ: Books + Auth                    │
       │                                          │
       ▼                                          │
7. Felhasználókezelés ← függ: Auth ──────────────┘
```

---

## Feature 1 – Project Setup

**Cél:** Deployolható Spring Boot skeleton és React alap — fut lokálisan (HSQLDB) és Lambda-n (Neon) is. Üzleti logika még nincs, de minden infrastruktúra össze van drótozva.

### Backend

| Step | Mit állít elő |
|------|---------------|
| 1.1 | Maven projekt: `pom.xml` minden függőséggel (Spring Boot, Lambda adapter, JPA, Liquibase, HSQLDB, springdoc-openapi, teszt lib-ek) |
| 1.2 | Application properties: `local` profil (HSQLDB, embedded Tomcat) és `prod` profil (Neon, Lambda adapter) |
| 1.3 | Liquibase setup: master changelog + `users` tábla changeset + default admin user seed changeset (BCrypt hash-elt jelszó, minden profilon lefut) |
| 1.4 | Lambda handler bekötés (`StreamLambdaHandler`) + health check endpoint (`GET /api/health`) |
| 1.5 | HSQLDB seed mechanizmus: `ApplicationRunner` betölti a `local-data/seed.sql`-t `local` profilon, ha a fájl létezik |

### Frontend

| Step | Mit állít elő |
|------|---------------|
| 1.6 | Vite + React + TypeScript projekt setup: shadcn/ui, Tailwind CSS, React Router, Zustand, Axios, i18next alap konfiguráció |
| 1.7 | Alap layout: sidebar + content area (shadcn/ui Sidebar komponens), dark mode toggle |
| 1.8 | Axios instance: `baseURL`, `withCredentials: true` |

### Infra (CDK)

| Step | Mit állít elő |
|------|---------------|
| 1.9 | CDK projekt setup (`infra/`): TypeScript, `cdk.json`, alap stack struktúra |
| 1.10 | Lambda + API Gateway HTTP API: Spring Boot JAR deploy, CORS konfiguráció (explicit CloudFront origin) |
| 1.11 | S3 + CloudFront: React SPA hosting, HTTPS |
| 1.12 | SSM Parameter Store: `/homelibrary/neon-connection-string`, `/homelibrary/jwt-secret`, `/homelibrary/admin-password-hash` paraméterek |
| 1.13 | IAM + OIDC: GitHub Actions szerepkör, minimális jogosultságok (Lambda update, S3 sync, CloudFront invalidation) |
| 1.14 | GitHub Actions workflow-ok: backend deploy (JAR → Lambda) + frontend deploy (dist → S3 + invalidation) |

---

## Feature 2 – Auth

**Cél:** Teljes autentikációs flow — login, token refresh rotációval, logout. Spring Security filter chain és frontend auth state felállítva a következő feature-ökhöz.

**Részletes spec:** [`Plans/specs/feature-auth/refresh-token-rotation.md`](specs/feature-auth/refresh-token-rotation.md) — tartalmazza a teljes backend flow-t, cookie konfigurációt, Axios interceptor implementációt és a tesztelési szempontokat.

**Integrációs teszt:** `@SpringBootTest` — login → refresh (rotation ellenőrzés) → logout flow.

### Infra

| Step | Mit állít elő |
|------|---------------|
| 2.1 | SonarQube Cloud bekötés: backend és frontend GitHub Actions workflow kiegészítése scan lépéssel, szükséges GitHub Secrets (`SONAR_TOKEN`, `SONAR_ORGANIZATION`, `SONAR_PROJECT_KEY`) |

### Backend

| Step | Mit állít elő |
|------|---------------|
| 2.2 | `User` entitás + `UserRepository` (a Liquibase changeset az 1.3-ban már elkészül) |
| 2.3 | `JwtProperties` (`@ConfigurationProperties` az `app.jwt.*` property-khez) + `JwtUtil`: access token generálás, validálás, claim kinyerés |
| 2.4 | `JwtAuthenticationFilter`: Bearer token validálás minden kérésnél |
| 2.5 | Spring Security konfiguráció: filter chain, role hierarchia, publikus endpointok (`/api/auth/**`, `/api/health`, `/swagger-ui/**`, `/v3/api-docs/**`) |
| 2.6 | `CookieProperties` (`@ConfigurationProperties` az `app.cookie.*` property-khez) + `POST /api/auth/login`: hitelesítés, access token + refresh token cookie kibocsátás (HttpOnly, Secure, SameSite=Strict, Path=/api/auth, `app.cookie.secure=false` local profilon) |
| 2.7 | `POST /api/auth/refresh`: refresh token validálás, rotation (új token kibocsátás, régi érvénytelenítés DB-ben) |
| 2.8 | `POST /api/auth/logout`: refresh token cookie törlése + DB-ben érvénytelenítés |

### Frontend

| Step | Mit állít elő |
|------|---------------|
| 2.9 | Zustand auth store: `accessToken`, `user` tárolása, `setAccessToken`, `clearAuth` műveletek |
| 2.10 | Axios interceptor: 401 kezelés, refresh token rotation race condition védelem (spec szerint) |
| 2.11 | Login oldal: felhasználónév + jelszó form, hibaüzenet kezelés |
| 2.12 | Protected route wrapper: bejelentkezett felhasználó ellenőrzése, redirect /login-ra |
| 2.13 | Logout gomb a sidebar-ban |

---

## Feature 3 – Locations CRUD

**Cél:** Teljes helyiség/polc kezelés — előfeltétele a könyvek elhelyezésének.

**Jogosultság:** GET — ADMIN, VISITOR | POST, PUT, DELETE — ADMIN

### Backend

| Step | Mit állít elő |
|------|---------------|
| 3.1 | Liquibase: `locations` tábla changeset |
| 3.2 | `Location` entitás + `LocationRepository` |
| 3.3 | `LocationService`: összes aktív lekérés, létrehozás, módosítás, soft delete (könyv hozzárendelés ellenőrzéssel) |
| 3.4 | `LocationController`: `GET /api/locations`, `POST`, `PUT /{id}`, `DELETE /{id}` |

### Frontend

| Step | Mit állít elő |
|------|---------------|
| 3.5 | Locations oldal: helyiségek/polcok listája, könyv darabszámmal |
| 3.6 | Létrehozás / szerkesztés form (dialog vagy inline), törlés megerősítéssel |

---

## Feature 4 – ISBN Lookup

**Cél:** Külső API integráció — ISBN beolvasás/bevitel után könyv adatok előtöltése OpenLibrary-ból vagy Google Books-ból.

**Jogosultság:** GET — ADMIN

### Backend

| Step | Mit állít elő |
|------|---------------|
| 4.1 | `OpenLibraryClient`: HTTP hívás, válasz leképezés belső DTO-ra |
| 4.2 | `GoogleBooksClient`: HTTP hívás, válasz leképezés, API kulcs SSM-ből (lokálisan: application properties) |
| 4.3 | `ISBNLookupService`: elsődleges (OpenLibrary) → fallback (Google Books) → nem található |
| 4.4 | `ISBNLookupController`: `GET /api/books/isbn/{isbn}` |

### Frontend

| Step | Mit állít elő |
|------|---------------|
| 4.5 | ISBN beviteli mező a könyv felvétel formon: lookup gomb, találat esetén mezők előtöltése, nem találat esetén üzenet |

### Infra (CDK)

| Step | Mit állít elő |
|------|---------------|
| 4.6 | SSM Parameter Store: `/homelibrary/google-books-api-key` paraméter hozzáadása a stack-hez |

---

## Feature 5 – Books CRUD

**Cél:** Core könyvkezelés — felvétel (kézi + ISBN lookup előtöltéssel), szűrős listázás, módosítás, soft delete, státusz és helyszín változtatás.

**Jogosultság:** GET — ADMIN, VISITOR | POST, PUT, DELETE — ADMIN

**Integrációs teszt:** `@SpringBootTest` — könyv státuszátmenetek (AT_HOME → LOANED → AT_HOME → DELETED).

### Backend

| Step | Mit állít elő |
|------|---------------|
| 5.1 | Liquibase: `books`, `book_descriptions` tábla changeset + indexek |
| 5.2 | `Book` entitás + `BookRepository` (szűrő query metódusokkal) |
| 5.3 | `BookDescription` entitás + `BookDescriptionRepository` |
| 5.4 | `BookService`: létrehozás, lekérés id alapján, szűrős listázás (keresés, státusz, helyszín, kategória, nyelv, év), módosítás, soft delete |
| 5.5 | `BookController`: `GET /api/books`, `GET /{id}`, `POST`, `PUT /{id}`, `DELETE /{id}` |
| 5.6 | `BookController`: `PUT /{id}/status`, `PUT /{id}/location` |

### Frontend

| Step | Mit állít elő |
|------|---------------|
| 5.7 | Könyvlista oldal: card grid, TanStack Table alapú szűrősor (státusz, helyszín, kategória, nyelv, keresés) |
| 5.8 | Könyv részletek oldal/panel |
| 5.9 | Könyv felvétel form: ISBN bevitel → lookup előtöltés (Feature 4) + kézi kitöltés |
| 5.10 | Könyv szerkesztés form, soft delete megerősítéssel |
| 5.11 | Státusz és helyszín módosítás (inline akciók a card-on vagy részletek oldalon) |

---

## Feature 6 – Loans

**Cél:** Kölcsönzés nyilvántartás — könyv kölcsönadása, visszavétele, aktív/lezárt kölcsönzések listázása.

**Jogosultság:** GET, POST, PUT — ADMIN

### Backend

| Step | Mit állít elő |
|------|---------------|
| 6.1 | Liquibase: `loans` tábla changeset |
| 6.2 | `Loan` entitás + `LoanRepository` |
| 6.3 | `LoanService`: kölcsönzés létrehozása (könyv státusz → LOANED), visszavétel (könyv státusz → AT_HOME), szűrős listázás |
| 6.4 | `LoanController`: `GET /api/loans`, `POST /api/loans`, `PUT /api/loans/{id}/return` |

### Frontend

| Step | Mit állít elő |
|------|---------------|
| 6.5 | Kölcsönzések oldal: aktív és lezárt kölcsönzések listája |
| 6.6 | Könyv kölcsönadása form (kinek, megjegyzés), visszavétel gomb megerősítéssel |

---

## Feature 7 – Felhasználókezelés

**Cél:** Admin kezeli a felhasználókat (létrehozás, listázás, szerepkör/státusz módosítás). Felhasználók saját jelszavukat és nyelvi beállításukat módosíthatják.

**Jogosultság:** GET, POST, DELETE — ADMIN | PUT — ADMIN (bármely user) + VISITOR (csak saját: jelszó, nyelv)

### Backend

| Step | Mit állít elő |
|------|---------------|
| 7.1 | `UserService`: listázás, lekérés id alapján, létrehozás, módosítás (mező szintű jogosultság ellenőrzéssel), soft delete (active → false) |
| 7.2 | `UserController`: `GET /api/users`, `GET /{id}`, `POST`, `PUT /{id}`, `DELETE /{id}` |

### Frontend

| Step | Mit állít elő |
|------|---------------|
| 7.3 | Felhasználók oldal (admin): lista, új felhasználó létrehozás, szerepkör és státusz módosítás |
| 7.4 | Saját profil oldal: jelszó módosítás, nyelvi beállítás |
