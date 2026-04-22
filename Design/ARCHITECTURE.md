# HomeLibrary – Architektúra

> **Utolsó frissítés:** 2026-04-04
> **Státusz:** v3 – végleges, Lambda + Neon PostgreSQL stack

## Tech Stack

### Backend

| Komponens | Technológia | Indok |
|-----------|-------------|-------|
| Language | Java 21 LTS | Meglévő tudás, LTS |
| Framework | Spring Boot 4.x | Meglévő tudás |
| Lambda adapter | `aws-serverless-java-container` | Spring Boot → Lambda bridge |
| Auth | Spring Security + JWT | Elegendő ennél a skálánál, Cognito nem szükséges |
| ORM | Spring Data JPA + Hibernate | Standard Spring stack |
| DB migráció | Liquibase | Verziókövetett sémaváltozások, rollback támogatás (lásd ADR-005) |
| API dokumentáció | springdoc-openapi | OpenAPI spec generálás + Swagger UI (`/swagger-ui.html`), automatikusan naprakész. Publikusan elérhető production-ban is — tudatos döntés: portfolió projekt, álláspályázatnál előny. |
| Build | Maven + maven-shade-plugin | Executable JAR Lambda deployhoz |
| Container | – (JAR alapú) | Lambda-ra JAR-t deploy-olunk, nem Docker image |

### Frontend

| Komponens | Technológia | Indok |
|-----------|-------------|-------|
| Framework | React 18 + TypeScript | Projekt követelmény, TS hosszú távon jobb |
| Build tool | Vite | Gyors, modern (CRA deprecated) |
| UI library | shadcn/ui + Tailwind CSS | Dark mode natívan, könnyű skin váltás |
| i18n | i18next | De facto standard React i18n |
| State | Zustand | Egyszerűbb mint Redux, elég ennél az appnál |
| HTTP | Axios | Standard, interceptorokkal JWT kezelés |
| Vonalkód | react-zxing | Böngészős kamera API wrapper, mobil-optimalizált |
| Routing | React Router v6 | Standard |
| Grid/táblázat | AG Grid React Community | Konfigurációalapú, beépített szűrés/rendezés/lapozás/infinite scroll, dark mode támogatás (lásd ADR-009) |

### Frontend UI irányelvek

| Szempont | Döntés |
|----------|--------|
| Layout | Sidebar + content area (shadcn/ui Sidebar komponens) |
| Könyv megjelenítés | Card alapú |
| Dark mode | Támogatott (shadcn/ui natív CSS variable theming) |
| Színpaletta | Tailwind/shadcn/ui CSS variable alapú – konkrét paletta implementációnál döntendő |
| Grid szűrés | Oszlopfejlécbe integrált szűrősor, hide-olható (nem táblázat feletti szűrősáv) |

### Adatbázis

- **PostgreSQL 15** – [Neon](https://neon.tech) free tier SaaS-on futtatva
- Free tier: 0.5GB tárhely, **perzisztens** (auto-pause inaktivitás után, de adat megmarad)
- Pontosan standard PostgreSQL: azonos JDBC driver, JPA, Liquibase – semmi vendor-specifikus
- Migrációs útvonal: connection string csere → RDS PostgreSQL (ha és amikor szükséges)

### Fejlesztői toolchain

| Tool | Mire | Megjegyzés |
|------|------|------------|
| IntelliJ IDEA Ultimate | IDE | Spring Boot, AWS Toolkit plugin |
| Claude Code CLI | AI-asszisztált fejlesztés | Terminálból, a projekt mappájából indítva |
| SonarLint plugin | Real-time statikus analízis | SonarQube Cloud-hoz kötve |
| Git + GitHub | Verziókezelés | Mono-repo (kód), külön repo (docs) |
| SonarQube Cloud | CI statikus analízis | GitHub Actions-ből, public repo, ingyenes |
| GitHub Actions | CI/CD | OIDC alapú AWS hitelesítés |

---

## AWS Infrastruktúra (free tier kompatibilis)

```
Internet
    │
    ▼
CloudFront (CDN + HTTPS)              ← always free: 1TB transfer/hó
    │                     │
    ▼                     ▼
S3 Bucket            API Gateway      ← always free: 1M req/hó
(React SPA)          (HTTP API)         S3: ~$0.02/GB (fillér)
                          │
                          ▼
                    Lambda Function    ← always free: 1M req/hó, 400K GB-sec
                    (Spring Boot JAR     SnapStart: hidegindítás ~200-600ms
                     Java 21)
                          │
               ┌──────────┴──────────┐
               ▼                     ▼
         Neon PostgreSQL        SSM Parameter Store   ← ingyenes
         (külső SaaS,           (Neon conn string,
          ingyenes)              JWT secret,
                                 Google Books API key)
```

### AWS Szolgáltatások és költségek

| Szolgáltatás | Mire | Költség |
|-------------|------|---------|
| **Lambda** | Spring Boot backend | ✅ Always free (1M req/hó, 400K GB-sec) |
| **API Gateway HTTP API** | Lambda URL + routing | ✅ Always free (1M req/hó) |
| **S3** | React SPA statikus hosting | ~$0.02/GB – fillér |
| **CloudFront** | CDN, HTTPS | ✅ Always free (1TB transfer/hó) |
| **SSM Parameter Store** | Secretek tárolása | ✅ Standard tier ingyenes |
| **CloudWatch** | Logok, monitoring | ✅ Always free (10 alarm, 5GB log) |
| **SNS** | CloudWatch alarm értesítések | ✅ Always free (1000 email/hó) |

> 💡 **Becsült havi költség: ~$0** (S3 tárhelyen kívül, ami fillér nagyságrendű)

### Lambda SnapStart

A Spring Boot JVM hidegindítása Lambda-n SnapStart nélkül 3-8 másodperc lenne.
A SnapStart az inicializált JVM memória-snapshotját menti el, és abból indítja a következő kérést.
Eredmény: ~200-600ms hidegindítás Java 21-en is. Bekapcsolása egyetlen Lambda konfiguráció.

### Kombinált hidegindítási hatás

Ha az alkalmazás hosszabb ideig inaktív (pl. éjszaka), két cold start jelenség egyszerre fordulhat elő:
- **Lambda SnapStart:** ~200-600ms
- **Neon auto-pause felébredés:** ~500ms

Együttes hatás: az első kérés **~700-1100ms** késéssel érkezhet meg. Ez tudatos tradeoff –
az alkalmazás nem sebességkritikus, egy háztartási könyvtár esetén ez teljesen elfogadható.

### Miért nem Secrets Manager?

A Secrets Manager $0.40/secret/hó – 3 secret esetén $1.2/hó.
Az SSM Parameter Store standard tier ingyenes, és a mi use case-ünkre (connection string,
JWT secret, API key tárolás) teljesen megfelelő. Lásd: ADR-004.

---

## CI/CD Pipeline

```
GitHub Push → main branch
        │
        ▼
GitHub Actions
  ├── Backend workflow:
  │     1. mvn test                          (unit tesztek)
  │     2. mvn package                       (shade plugin → fat JAR)
  │     3. SonarQube Cloud scan              (statikus analízis)
  │     4. JAR feltöltés S3-ra
  │     5. aws lambda update-function-code
  │     6. Lambda SnapStart: új verzió publish
  │
  └── Frontend workflow:
        1. npm ci
        2. npm run build                     (Vite → dist/)
        3. SonarQube Cloud scan              (ESLint alapú)
        4. aws s3 sync ./dist → S3
        5. CloudFront invalidation
```

> **Liquibase:** A sémamigrációk app indításkor futnak automatikusan (Spring Boot auto-configuration) — külön pipeline lépés nem szükséges.

> **Tesztstratégia:** Unit tesztek minden service és üzleti logika osztályra. Integrációs tesztek (`@SpringBootTest`) csak a kritikus területekre: auth flow (login/refresh/logout + token rotation) és book státuszátmenetek. Többi endpoint: manuális UI teszt. Staging környezet nincs — a main branch közvetlenül production-re deploy-ol, ami ennél a skálánál elfogadható.

> **AWS hitelesítés GitHub Actions-ből: OIDC** (nem tárolt access key!)
> A GitHub Actions ideiglenes tokent kap az IAM-tól – ez a biztonságos, modern megoldás.
> Nincs AWS_ACCESS_KEY_ID és AWS_SECRET_ACCESS_KEY a GitHub secretekben.

---

## Kód Repo Struktúra (Mono-repo)

```
homelibrary/                        ← GitHub repo: homelibrary
├── backend/                        ← Spring Boot projekt
│   ├── src/
│   │   ├── main/java/
│   │   └── test/java/
│   └── pom.xml
├── frontend/                       ← React projekt
│   ├── src/
│   ├── public/
│   ├── vite.config.ts
│   └── package.json
├── infra/                          ← AWS CDK (TypeScript)
│   ├── bin/
│   │   └── homelibrary.ts          ← CDK app belépési pont
│   ├── lib/
│   │   └── homelibrary-stack.ts    ← Stack definíció
│   ├── package.json
│   └── cdk.json
├── .github/
│   └── workflows/
│       ├── backend-deploy.yml
│       └── frontend-deploy.yml
└── README.md
```

---

## Dokumentáció Repo Struktúra

```
homelibrary-docs/                   ← GitHub repo: homelibrary-docs
├── PROJECT.md
├── Design/
│   ├── ARCHITECTURE.md
│   ├── DB_SCHEMA.md
│   ├── API_DESIGN.md
│   └── ADR/
│       ├── 001-database-choice.md
│       ├── 002-auth-strategy.md
│       ├── 003-isbn-api.md
│       ├── 004-aws-compute-choice.md
│       ├── 005-db-migration-tool.md
│       ├── 006-sql-standard-types.md
│       ├── 007-rooms-locations-normalization.md
│       ├── 008-entity-timestamp-strategy.md
│       └── 009-grid-library-choice.md
└── Plans/
    ├── phase1-feature-order.md
    └── specs/
        └── feature-auth/
            └── refresh-token-rotation.md
```

> A dokumentáció szándékosan külön repóban van a kódtól.
> A fejlesztő menet közben nem szerkeszti – a tervek rögzítik az architektúrális döntéseket.
> Claude Code-ban a docs repo tartalmát kontextusként lehet átadni: `@docs/ARCHITECTURE.md`

---

## Lokális Fejlesztési Környezet

```
# Nincs Docker – a lokális stack IntelliJ-ből vagy terminálból indítható:
# - HSQLDB in-memory  ← Spring Boot automatikusan kezeli (test scope dependency, külön telepítés nélkül)
# - backend           ← Spring Boot embedded Tomcat módban (nem Lambda adapter)
# - frontend          ← Vite dev server (HMR – Hot Module Replacement)
```

**Miért Tomcat lokálisan?**
A Spring Boot Lambda módban fut production-ban, de lokálisan a hagyományos embedded Tomcat
gyorsabb fejlesztési ciklust ad. Lambda-specifikus viselkedés teszteléséhez: AWS SAM CLI
(`sam local start-api`).

**IntelliJ IDEA Ultimate-ből:**
- Spring Boot run config – direktben futtatható, Docker nélkül
- AWS Toolkit plugin – Lambda lokális debug, CloudWatch logok

### Lokális fejlesztői adatkezelés (HSQLDB seed)

A lokál és az AWS környezet élesen elválik egymástól – közös kód nincs köztük.

| Környezet | DB | Adat | Aktiválás |
|-----------|-----|------|-----------|
| Lokál | HSQLDB in-memory | seed fájlból töltve (ha létezik) | `local` Spring profile |
| AWS | Neon PostgreSQL | éles adat | `prod` Spring profile |

**Seed export/import működése (`local` profile):**

- **Első indítás:** üres DB, Liquibase lefuttatja a migrációkat, egyetlen alap admin user kerül be
- **Leállítás előtt:** az exportot Claude Code generálja kérésre – INSERT statement-ek táblánként, `local-data/seed.sql` fájlba
- **Következő indítás:** Spring `ApplicationRunner` betölti a `seed.sql`-t `JdbcTemplate`-tel, ha a fájl létezik
- **Liquibase táblák** (`DATABASECHANGELOG`, `DATABASECHANGELOGLOCK`) nem kerülnek az exportba – ezeket Liquibase mindig maga kezeli

**Gitignore:**
```
local-data/
```
A seed fájl és minden lokális export gitignore-os – nem kerül a repóba.

---

## Biztonsági Megfontolások

- JWT access token: 15 perces élettartam
- Refresh token: HttpOnly cookie (XSS ellen védett), SameSite=Strict, Path=/api/auth
- Refresh token rotation: minden refresh híváskor új token kerül kibocsátásra, a régi érvénytelenítésre (lásd ADR-002)
- CSRF védelem: Az API endpointok Bearer tokennel védettek, ami természeténél fogva CSRF-biztos. A refresh token cookie SameSite=Strict és Path=/api/auth attribútumokkal van beállítva, így a böngésző cross-origin kéréshez nem csatolja. Külön CSRF token nem szükséges.
- CORS: `withCredentials: true` (cookie küldés/fogadás) miatt az `Access-Control-Allow-Origin` nem lehet `*` — explicit CloudFront origin szükséges + `Access-Control-Allow-Credentials: true`
- Brute-force védelem: API Gateway szintű throttling (3 req/sec per IP a login endpointon) az első védelmi vonal. Alkalmazásszintű account lockout nem szükséges — az alkalmazás nem jelent érdemi támadási célpontot ezen a skálán.
- Access token érvényessége logout után: A `POST /api/auth/logout` törli a refresh token cookie-t, de a kiadott access token a 15 perces TTL lejártáig technikailag érvényes marad. Lambda-n in-memory blacklist nem praktikus (nincs perzisztens memória instance-ok között), DynamoDB-alapú blacklist pedig szükségtelen komplexitást és költséget adna ehhez a skálához. A 15 perces TTL elfogadható tradeoff.
- SSM Parameter Store: SOHA nem kerül secret kódba vagy `.env` fájlba plain textként
- Neon PostgreSQL: SSL kapcsolat kötelező (alapértelmezett)
- GitHub Actions → AWS: OIDC, nem tárolt access key
- S3 (SPA): public read, de csak a `dist/` tartalmára
- Swagger UI (`/swagger-ui.html`): publikusan elérhető production-ban — tudatos döntés, portfolió projekt (álláspályázatnál előny). Írási műveletek Bearer token nélkül úgysem hajthatók végre.
- GitHub repo: **public** (SonarQube Cloud free tier feltétele)
  - Secretek soha nem kerülnek a kódba (SSM-ben vannak)
  - `.gitignore` gondosan karbantartva

---

## Monitoring Stratégia

A CloudWatch free tier szintje elegendő — tudatos döntés, elsősorban AWS tanulási célból is.

| Metrika | CloudWatch eszköz |
|---------|------------------|
| Lambda error rate és cold start duration | Log-based metric filter + alarm |
| API Gateway 4xx/5xx arány | Beépített API Gateway metrikák |
| Lambda concurrent executions | Beépített Lambda metrikák (free tier limit figyelése) |
| Neon connection hibák | Lambda log-okból metric filter |

> A CloudWatch alarmok email értesítést küldenek SNS-en keresztül (AWS free tier: 1000 email/hó ingyenes).

---

## ISBN Lookup Stratégia

1. **OpenLibrary API** (elsődleges) – ingyenes, API kulcs nélkül
   `GET https://openlibrary.org/api/books?bibkeys=ISBN:{isbn}&format=json&jscmd=data`
2. **Google Books API** (fallback) – ingyenes, API key kell (SSM-ben tárolva), 1000 req/nap
   `GET https://www.googleapis.com/books/v1/volumes?q=isbn:{isbn}`
3. **Manuális bevitel** – ha egyik API sem talál eredményt

---

## AI Fordítás (Fázis 3 – döntés nyitott)

- **Jelölt A:** Google Gemini API (ingyenes tier, bőkezű kvóta)
- **Jelölt B:** DeepL free tier (500k karakter/hó, fordításra specializált)
- Mindkettőt leteszteljük Fázis 3-ban, akkor döntünk
- A backend hívja (API key SSM-ben, nem a frontend)
- Cache-elés: lefordított szöveg DB-ben (`book_descriptions` tábla, `AI_TRANSLATED` source)
- Fallback: ha az API nem elérhető, az eredeti nyelven jelenik meg a leírás

---

## Nyitott Döntések

- [x] AWS CDK stack nyelve: **TypeScript** – a CDK dokumentáció, példák és közösségi tartalom túlnyomó része TypeScript-ben érhető el, ez a de facto standard CDK nyelvként
- [ ] AI fordítás: Gemini vs DeepL (Fázis 3-ban, tesztelés után döntünk)
