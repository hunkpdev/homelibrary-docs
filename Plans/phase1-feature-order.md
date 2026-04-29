# 1. Fázis – Implementációs terv

> **Státusz:** Tervezés alatt
> **Utolsó frissítés:** 2026-04-29
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

**OpenAPI dokumentáció:** Minden controller endpoint `@Operation` és `@ApiResponse` annotációkkal látandó el — a Swagger UI automatikusan naprakész dokumentációt ad.

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
       │
       ▼
8. i18n kiegészítések ← függ: Project Setup (1.6, 1.7)
       │
       ▼
9. Demo role ← függ: Auth + minden controller (3–7)
```

---

## Feature 1 – Project Setup

**Cél:** Deployolható Spring Boot skeleton és React alap — fut lokálisan (HSQLDB) és Lambda-n (Neon) is. Üzleti logika még nincs, de minden infrastruktúra össze van drótozva.

### Backend

| Step | Mit állít elő |
|------|---------------|
| [1.1](specs/feature-project-setup/backend/step-1.1-maven-setup.md) | Maven projekt: `pom.xml` minden függőséggel (Spring Boot, Lambda adapter, JPA, Liquibase, HSQLDB, springdoc-openapi, teszt lib-ek) |
| [1.2](specs/feature-project-setup/backend/step-1.2-application-properties.md) | Application properties: `local` profil (HSQLDB, embedded Tomcat) és `prod` profil (Neon, Lambda adapter) |
| [1.3](specs/feature-project-setup/backend/step-1.3-liquibase-users-seed.md) | Liquibase setup: master changelog + `users` tábla changeset + default admin user seed changeset (BCrypt hash-elt jelszó, minden profilon lefut) |
| [1.4](specs/feature-project-setup/backend/step-1.4-lambda-handler-health.md) | Lambda handler bekötés (`StreamLambdaHandler`) + health check endpoint (`GET /api/health`) |
| [1.5](specs/feature-project-setup/backend/step-1.5-hsqldb-seed-mechanism.md) | HSQLDB seed mechanizmus: `ApplicationRunner` betölti a `local-data/seed.sql`-t `local` profilon, ha a fájl létezik |

### Frontend

| Step | Mit állít elő |
|------|---------------|
| [1.6](specs/feature-project-setup/frontend/step-1.6-vite-react-setup.md) | Vite + React + TypeScript projekt setup: shadcn/ui, Tailwind CSS, React Router, Zustand, Axios, i18next alap konfiguráció |
| [1.7](specs/feature-project-setup/frontend/step-1.7-base-layout.md) | Alap layout: sidebar + content area (shadcn/ui Sidebar komponens), dark mode toggle |
| [1.8](specs/feature-project-setup/frontend/step-1.8-axios-instance.md) | Axios instance: `baseURL`, `withCredentials: true` |

### Infra (CDK)

| Step | Mit állít elő |
|------|---------------|
| [1.9](specs/feature-project-setup/infra/step-1.9-cdk-project-setup.md) | CDK projekt setup (`infra/`): TypeScript, `cdk.json`, alap stack struktúra |
| [1.10](specs/feature-project-setup/infra/step-1.10-lambda-api-gateway.md) | Lambda + API Gateway HTTP API: Spring Boot JAR deploy, CORS konfiguráció (explicit CloudFront origin) |
| [1.11](specs/feature-project-setup/infra/step-1.11-s3-cloudfront.md) | S3 + CloudFront: React SPA hosting, HTTPS |
| [1.12](specs/feature-project-setup/infra/step-1.12-ssm-parameters.md) | SSM Parameter Store: `/homelibrary/neon-connection-string`, `/homelibrary/jwt-secret`, `/homelibrary/admin-password-hash` paraméterek |
| [1.13](specs/feature-project-setup/infra/step-1.13-iam-oidc.md) | IAM + OIDC: GitHub Actions szerepkör, minimális jogosultságok (Lambda update, S3 sync, CloudFront invalidation) |
| [1.14](specs/feature-project-setup/infra/step-1.14-github-actions.md) | GitHub Actions workflow-ok: backend deploy (JAR → Lambda) + frontend deploy (dist → S3 + invalidation) |

---

## Feature 2 – Auth

**Cél:** Teljes autentikációs flow — login, token refresh rotációval, logout. Spring Security filter chain és frontend auth state felállítva a következő feature-ökhöz.

**Részletes spec:** [`Plans/specs/feature-auth/refresh-token-rotation.md`](specs/feature-auth/refresh-token-rotation.md) — tartalmazza a teljes backend flow-t, cookie konfigurációt, Axios interceptor implementációt és a tesztelési szempontokat.

**Integrációs teszt:** `@SpringBootTest` — login → refresh (rotation ellenőrzés) → logout flow.

### Infra

| Step | Mit állít elő |
|------|---------------|
| [2.1](specs/feature-auth/infra/step-2.1-ci-code-quality.md) | CI code quality: SonarQube Cloud (backend + frontend, külön projektek, QG-blokkolt deploy), human-readable test report lokálban és GitHub Check fülön |

### Backend

| Step | Mit állít elő |
|------|---------------|
| [2.2](specs/feature-auth/backend/step-2.2-user-entity-repository.md) | `User` entitás + `UserRepository` (a Liquibase changeset az 1.3-ban már elkészül) |
| [2.3](specs/feature-auth/backend/step-2.3-jwt-properties-util.md) | `JwtProperties` (`@ConfigurationProperties` az `app.jwt.*` property-khez) + `JwtUtil`: access token generálás, validálás, claim kinyerés |
| [2.4](specs/feature-auth/backend/step-2.4-jwt-authentication-filter.md) | `JwtAuthenticationFilter`: Bearer token validálás minden kérésnél |
| [2.5](specs/feature-auth/backend/step-2.5-security-config.md) | Spring Security konfiguráció: filter chain, role hierarchia, publikus endpointok (`/api/auth/**`, `/api/health`, `/swagger-ui/**`, `/v3/api-docs/**`) |
| [2.6](specs/feature-auth/backend/step-2.6-login-endpoint.md) | `CookieProperties` (`@ConfigurationProperties` az `app.cookie.*` property-khez) + `POST /api/auth/login`: hitelesítés, access token + refresh token cookie kibocsátás (HttpOnly, Secure, SameSite=Strict, Path=/api/auth, `app.cookie.secure=false` local profilon). Ha `user.preferred_language === NULL` (első bejelentkezés), a login endpoint az `Accept-Language` HTTP header alapján tölti ki (`hu` magyar locale-ra, `en` egyébként) — innentől a mező mindig kitöltött. |
| [2.7](specs/feature-auth/backend/step-2.7-refresh-endpoint.md) | `POST /api/auth/refresh`: refresh token validálás, rotation (új token kibocsátás, régi érvénytelenítés DB-ben) |
| [2.8](specs/feature-auth/backend/step-2.8-logout-integration-test.md) | `POST /api/auth/logout`: refresh token cookie törlése + DB-ben érvénytelenítés |

### Frontend

| Step | Mit állít elő |
|------|---------------|
| [2.9](specs/feature-auth/frontend/step-2.9-auth-store.md) | Zustand auth store: `accessToken`, `user` tárolása, `setAccessToken`, `clearAuth` műveletek |
| [2.10](specs/feature-auth/frontend/step-2.10-axios-interceptor.md) | Axios interceptor: 401 kezelés, refresh token rotation race condition védelem (spec szerint) |
| [2.11](specs/feature-auth/frontend/step-2.11-login-page.md) | Login oldal: felhasználónév + jelszó form, hibaüzenet kezelés |
| [2.12](specs/feature-auth/frontend/step-2.12-protected-route.md) | Protected route wrapper: bejelentkezett felhasználó ellenőrzése, redirect /login-ra |
| [2.13](specs/feature-auth/frontend/step-2.13-logout-button.md) | Role-alapú menü szűrés a sidebar-ban (az 1.7-ben minden menüpont megjelenik, itt kerül bekötésre a láthatóság role szerint) + logout gomb |

---

## Feature 3 – Rooms és Locations CRUD

**Cél:** Teljes helyiség és helyszín kezelés — előfeltétele a könyvek elhelyezésének. Lásd ADR-007.

**Jogosultság:** GET — ADMIN, VISITOR | POST, PUT, DELETE — ADMIN

### Backend

| Step | Mit állít elő |
|------|---------------|
| [3.1](specs/feature-locations-crud/backend/step-3.1-liquibase-rooms.md) | Liquibase: `rooms` tábla changeset (`002-create-rooms.yaml`) |
| [3.2](specs/feature-locations-crud/backend/step-3.2-liquibase-locations.md) | Liquibase: `locations` tábla changeset (`003-create-locations.yaml`, `room_id` FK-val) |
| [3.3](specs/feature-locations-crud/backend/step-3.3-room-entity-repository.md) | `Room` entitás + `RoomRepository` |
| [3.4](specs/feature-locations-crud/backend/step-3.4-location-entity-repository.md) | `Location` entitás + `LocationRepository` |
| [3.5](specs/feature-locations-crud/backend/step-3.5-room-service.md) | `RoomService`: listázás (Specification + Pageable), létrehozás, módosítás, soft delete (aktív location ellenőrzéssel) |
| [3.6](specs/feature-locations-crud/backend/step-3.6-room-controller.md) | `RoomController`: `GET /api/rooms`, `POST`, `PUT /{id}`, `DELETE /{id}` |
| [3.7](specs/feature-locations-crud/backend/step-3.7-location-service.md) | `LocationService`: listázás (Specification + Pageable, `roomId` szűrővel), létrehozás, módosítás, soft delete (aktív könyv ellenőrzéssel) |
| [3.8](specs/feature-locations-crud/backend/step-3.8-location-controller.md) | `LocationController`: `GET /api/locations`, `POST`, `PUT /{id}`, `DELETE /{id}` |

### Frontend

| Step | Mit állít elő |
|------|---------------|
| [3.9](specs/feature-locations-crud/frontend/step-3.9-rooms-locations-page.md) | Rooms + Locations oldal: flat location lista AG Grid Community-vel (sort, szöveges + kaszkádos room/location dropdown szűrő, lapozás), `bookCount` |
| [3.10](specs/feature-locations-crud/frontend/step-3.10-room-modals.md) | Room form modalok: létrehozás / szerkesztés / törlés megerősítés |
| [3.11](specs/feature-locations-crud/frontend/step-3.11-location-modals.md) | Location form modalok: létrehozás / szerkesztés / törlés megerősítés — `roomId` autocomplete dropdownnal (szerkeszthető placeholder), csoportfejlécből előre kitöltve |

---

## Feature 4 – ISBN Lookup

**Cél:** OSZK NEKTÁR Z39.50 integráció — ISBN beolvasás/bevitel után könyv adatok előtöltése. Manuális bevitel mint fallback. Lásd ADR-003, ADR-010.

**Jogosultság:** GET — `ADMIN` vagy `DEMO` (`hasAnyRole('ADMIN', 'DEMO')`) — VISITOR 403

**DEMO rate limit:** 5 keresés/session (Spring Cache, JWT-hez kötve), 50 keresés/nap (DB tábla, lazy reset) — 429 Too Many Requests limit eléréskor

### Backend

| Step | Mit állít elő |
|------|---------------|
| [4.1](specs/feature-isbn-lookup/backend/step-4.1-liquibase-demo-rate-limit.md) | Liquibase: `demo_isbn_daily_stats` tábla changeset |
| [4.2](specs/feature-isbn-lookup/backend/step-4.2-nektar-client.md) | `OszkNektarClient`: YAZ4J Z39.50 kapcsolat, `@attr 1=7` ISBN keresés, marc4j MARC21 parsing, `IsbnLookupResult` record + `IsbnSource` enum (`OSZK`, `MANUAL`); időablak-ellenőrzés nincs (connection timeout fedezi a kiesést) |
| [4.3](specs/feature-isbn-lookup/backend/step-4.3-isbn-lookup-service.md) | `IsbnLookupService`: OSZK lookup → ha `Optional.empty()` → `found: false`; DEMO rate limit (session: Spring Cache JWT-kötve, napi: DB lazy reset) |
| [4.4](specs/feature-isbn-lookup/backend/step-4.4-isbn-lookup-controller.md) | `IsbnLookupController`: `GET /api/books/isbn/{isbn}` — 200 `found:true`, 200 `found:false`, 429 DEMO rate limit esetén |

### Frontend

| Step | Mit állít elő |
|------|---------------|
| [4.5](specs/feature-isbn-lookup/frontend/step-4.5-isbn-scanner-input.md) | `IsbnScannerInput` React komponens: `react-zxing` integráció, kamera elérhetőség detektálás (MediaDevices API), kamera nézet elsődleges / kézi szövegmező fallback — ha a kamera elérhető, alapból az aktív |
| [4.6](specs/feature-isbn-lookup/frontend/step-4.6-isbn-lookup-ui.md) | `IsbnLookupPanel`: scanner + API hívás + eredmény megjelenítés; `onResult` callbacken adja át az adatokat a szülő (könyv form, Feature 5) számára |

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
| 5.7 | Könyvlista oldal: card grid, AG Grid Community alapú szűrősor (státusz, helyszín, kategória, nyelv, keresés) |
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

**Jogosultság:** GET lista, POST, DELETE — ADMIN (VISITOR és DEMO kizárva) | GET saját, PUT saját (csak jelszó + preferredLanguage) — VISITOR is (`@PreAuthorize` ownership check); DEMO minden `/api/users/**` végpontból kizárva

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

---

## Feature 8 – i18n kiegészítések

**Cél:** Böngésző locale alapján automatikus nyelvválasztás és manuális váltási lehetőség — a meglévő i18next alap (step 1.6) és sidebar layout (step 1.7) kiegészítése.

**Megjegyzés:** Technikailag már Feature 1 után implementálható (csak project setup-tól függ), de kényelmi feature — tudatos döntés, hogy a fontosabb feature-ök (3–7) után kerül sorra.

### Frontend

| Step | Mit állít elő |
|------|---------------|
| 8.1 | `i18next-browser-languagedetector` plugin bekötése: böngésző locale detektálás, hu → `hu`, egyéb → `en` fallback. Bejelentkezett user esetén `user.preferred_language` az egyetlen forrás (`PUT /api/users/{id}` perzisztálva) — `localStorage` réteg nincs. Anonymous oldalakon (login form) csak `navigator.language` autodetect, perzisztencia nincs. |
| 8.2 | `LanguageSwitcher` komponens: flag ikon gomb a sidebar alján (dark mode toggle mellé), mindig a másik nyelvet jelöli, kattintásra nyelvváltás i18next-en keresztül |

---

## Feature 9 – Demo role

**Cél:** Portfólió bemutató célú DEMO role — minden oldal és form látható, kizárólag biztonságos HTTP metódusok (GET/HEAD/OPTIONS) engedélyezve; minden state-modifying művelet GUI-n disabled, API-n 403-mal elutasítva.

**Jogosultság:** GET, HEAD, OPTIONS — DEMO role számára is elérhetők; minden state-modifying metódus (POST, PUT, PATCH, DELETE) → 403

### Backend

| Step | Mit állít elő |
|------|---------------|
| 9.1 | `Role` enum bővítése `DEMO` értékkel |
| 9.2 | Liquibase changeset: beégetett demo user seed (BCrypt hash-elt jelszó, `DEMO` role) — analóg az admin seed changeset-tel (step 1.3) |
| 9.3 | Spring Security: DEMO role számára kizárólag a biztonságos (GET, HEAD, OPTIONS) HTTP metódusok engedélyezve `/api/**` alatt; minden state-modifying metódus (POST, PUT, PATCH, DELETE és bármi egyéb) ADMIN-only — globális `httpSecurity` rule-lal SecurityConfig-ban (whitelist forma: GET/HEAD-re explicit role-list, `anyRequest()` ADMIN-only; nincs külön DEMO-blokk rule). Kivétel: `/api/users/**` — DEMO és VISITOR statikusan kizárva a SecurityConfig-ban; VISITOR self-access (`GET /{id}`, `PUT /{id}` saját rekordra) method-level `@PreAuthorize`-zal finomítva a controllerben (`@EnableMethodSecurity` szükséges). |

### Frontend

| Step | Mit állít elő |
|------|---------------|
| 9.4 | DEMO role detektálás a Zustand auth store `user.role === 'DEMO'` alapján — nem külön `isDemo` boolean flag a store-ban (a `user` objektumból származtatott érték, ld. Feature 2.9). Minden mutáció gombon (mentés, törlés, küldés) `disabled` állapot DEMO role esetén — cross-cutting, az összes feature (3–7) érintett. Pattern: `<MutationButton>` wrapper komponens (`src/components/common/MutationButton.tsx`) — a logika egyszer van megírva, DEMO esetén auto-disabled + tooltip ("Demo módban a mentés letiltva"); a meglévő save/delete/submit gombok cseréje erre a komponensre. A kliensoldali blokk UX cél; a tényleges biztonságot a step 9.3 szerver oldali szabálya adja, ami DevTools-tampering ellen is védett. |
